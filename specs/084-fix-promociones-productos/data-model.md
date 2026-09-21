# Data Model: Corrección de Bugs en Promociones y Productos

Único cambio de esquema de esta spec (research.md D1). El resto de las correcciones (bugs 1, 3, 4,
5, y la guarda de exclusividad de producto del bug 3) son comportamiento de servicio/UI sobre
tablas ya existentes, sin cambio de columnas.

> **Enmienda 2026-09-20 (A-79)** — la sección `ProductVariant (modificada)` de abajo describe la
> versión intermedia (2026-09-17: `presentation_id` **opcional**, `name` conservado y sincronizado
> por cascada). Esa versión ya está implementada (migración `a9f7d0310f6b`) y queda **sustituida**
> por [§ Enmienda 2026-09-20](#enmienda-2026-09-20--productvariant-sin-name-a-79) al final de este
> archivo. Se mantiene el texto original como registro de lo que se construyó primero.

## `ProductVariant` (modificada)

Tabla existente `product_variants` (`app/models/product_variant.py`). Columnas actuales sin
cambio: `id`, `product_id` (FK `products.id`, `ondelete=CASCADE`), `name` (String255, `NOT NULL`,
`server_default='Presentación única'`), `sku` (nullable, unique), `price` (Numeric 12,2), `active`
(bool), `display_order` (int).

**Columna nueva**:

| Columna | Tipo | Nullable | Default | FK |
|---|---|---|---|---|
| `presentation_id` | `UUID` (mismo tipo que `presentations.id`) | Sí | `NULL` | `presentations.id`, `ondelete='SET NULL'` |

**Constraint nuevo**: `UniqueConstraint("product_id", "presentation_id", name="uq_product_variant_presentation")`.

En PostgreSQL, un `UniqueConstraint` no considera iguales dos filas con `presentation_id IS NULL`
(NULL nunca es igual a NULL) — así que múltiples variantes del mismo producto sin presentación
asociada siguen permitidas sin condición especial; el constraint solo actúa cuando dos variantes
del mismo producto comparten el mismo `presentation_id` no nulo (spec.md FR-006).

**Relationship nueva**: `presentation: Mapped["Presentation | None"] = relationship()` (sin
`back_populates` de lista — desde `Presentation` no se navega a sus variantes, no hay necesidad
funcional; si se requiere en la implementación, se agrega como `viewonly` para no complicar el
ciclo de vida de `Presentation`, que no gana responsabilidades nuevas).

**Validaciones de servicio** (no expresables como `CHECK` porque cruzan filas):

- Al guardar una variante con `presentation_id` no nulo (`_save_variant_entry`,
  `app/api/v1/catalog/service.py`): la presentación debe existir, pertenecer al mismo tenant y
  estar `active=True` (spec.md FR-001); si otra variante del mismo producto ya usa esa
  presentación, se rechaza con el mismo estilo de error que `variante_duplicada` (spec.md FR-006).
- Al guardar con `presentation_id` no nulo, el `name` recibido en el payload se **ignora** y se
  reemplaza por `Presentation.name` (spec.md FR-002/FR-003) — el campo sigue siendo `NOT NULL` en
  la tabla, solo cambia quién decide su valor.
- Al guardar con `presentation_id = null` (o al pasar de una presentación asociada a "Sin
  presentación"), el `name` del payload se usa tal cual, conservando el texto que tenía la
  variante en ese momento como punto de partida en el formulario (spec.md FR-005) — sin
  validación adicional distinta a la ya vigente (`UniqueConstraint(product_id, name)`).

### Efecto en cascada

(Fuera de la fila de `ProductVariant`, mismo commit de servicio.) Al
renombrar una `Presentation` (`PATCH /presentations/{id}`, `app/api/v1/presentations/router.py`),
en la misma transacción se ejecuta `UPDATE product_variants SET name = :nuevo_nombre WHERE
presentation_id = :id AND product_id IN (SELECT id FROM products WHERE tenant actual)` — en la
práctica, dado que cada schema ya está aislado por tenant (`@for_each_tenant_schema`), el `WHERE`
solo necesita `presentation_id = :id` (spec.md FR-004).

**Guarda previa a la cascada** (spec.md FR-004, edge case agregado tras `/speckit-analyze`): antes
de ejecutar el `UPDATE`, el servicio MUST verificar que ningún `product_id` afectado tenga ya otra
variante (sin `presentation_id`, o con uno distinto) cuyo `name` sea igual a `:nuevo_nombre` — el
`UniqueConstraint(product_id, name)` existente lo rechazaría a nivel de base de datos con un
`IntegrityError` no controlado si no se valida antes. La consulta de verificación:

```sql
SELECT pv.product_id, pv.name FROM product_variants pv
WHERE pv.name = :nuevo_nombre
  AND pv.presentation_id IS DISTINCT FROM :presentation_id
  AND pv.product_id IN (
    SELECT product_id FROM product_variants WHERE presentation_id = :presentation_id
  );
```

Si devuelve alguna fila, el `PATCH` completo se rechaza con `409` antes de tocar `Presentation` o
cualquier `ProductVariant` (ver contracts/variante-presentacion.md).

## Migración (Alembic)

**Archivo nuevo**: `app/alembic/versions/<rev>_084_variante_presentacion.py`

- `down_revision = 'da7581f7bb18'` (head actual verificado, migración aditiva de spec 083).
- `@for_each_tenant_schema` (mismo patrón que la migración de spec 083): por cada schema de
  tenant —
  - `ALTER TABLE product_variants ADD COLUMN presentation_id UUID NULL REFERENCES
    presentations(id) ON DELETE SET NULL;`
  - `ALTER TABLE product_variants ADD CONSTRAINT uq_product_variant_presentation UNIQUE
    (product_id, presentation_id);`
- Sin paso de datos: toda fila existente de `product_variants` nace con `presentation_id = NULL`,
  comportamiento idéntico al actual (Principio VII, no retroactivo — spec.md FR-007).
- **Rollback (`downgrade`)**: por cada schema de tenant, `ALTER TABLE product_variants DROP
  CONSTRAINT uq_product_variant_presentation;` seguido de `ALTER TABLE product_variants DROP
  COLUMN presentation_id;` — sin pérdida de datos que revertir porque la columna nunca tuvo un
  paso de datos que reescribir (mismo criterio que el `downgrade` de spec 083, `da7581f7bb18`).

## Entidades sin cambio de esquema (referencia)

- **`Presentation`** (`app/models/presentation.py`): sin cambio de columnas. Gana efecto
  secundario en su `PATCH` (cascada de renombre, ver arriba) — comportamiento de servicio, no de
  modelo.
- **`CategoryPresentation`**: sin cambio, no participa en esta spec.
- **`Promotion` / `PromotionRule` / `PromotionVariant`** (`app/models/promotion.py`): sin cambio de
  columnas. La exclusividad de producto (spec.md FR-021–025) y el reemplazo de la regla combinada
  por N reglas (FR-016–020) son comportamiento de servicio/frontend sobre las mismas tablas — cada
  `PromotionRule` sigue guardando su propia lista de `PromotionVariant`, exactamente como hoy;
  simplemente se crean más filas de `PromotionRule` por acción del usuario en vez de una sola con
  más variantes.

---

## Enmienda 2026-09-20 — `ProductVariant` sin `name` (A-79)

Segundo cambio de esquema de esta spec, sobre la tabla ya modificada arriba. Reemplaza el
criterio de `research.md` D2 (columna copiada + cascada) por el nombre derivado por JOIN que D2
había descartado; ver `research.md` D10 para el porqué del cambio de criterio.

### Cambios sobre `product_variants`

| Elemento | Antes (tras `a9f7d0310f6b`) | Después |
|---|---|---|
| `name` | `String(255) NOT NULL`, `server_default='Presentación única'` | **eliminada** |
| `uq__product_variants__product_id__name` | `UNIQUE (product_id, name)` | **eliminada** |
| `presentation_id` | `UUID NULL` | `UUID NOT NULL` |
| FK `fk__product_variants__presentation_id__presentations` | `ON DELETE SET NULL` | `ON DELETE RESTRICT` |
| `uq__product_variants__product_id__presentation_id` | `UNIQUE (product_id, presentation_id)` | sin cambio — pasa a ser la única unicidad por producto y, con la columna `NOT NULL`, ya no tiene la excepción de "varios `NULL`" |
| `relationship presentation` | `Mapped[Presentation \| None]` | `Mapped[Presentation]`, `lazy="joined"` (toda lectura de nombre lo necesita; evita N+1) |

`ProductVariant` gana una propiedad de conveniencia de solo lectura
`presentation_name -> str` (`return self.presentation.name`), sin columna, usada por los schemas
Pydantic (`from_attributes`) para armar `presentation_name` sin repetir el acceso en cada servicio.

**`RESTRICT` y no `SET NULL`/`CASCADE`**: `SET NULL` es incompatible con `NOT NULL`; `CASCADE`
borraría variantes (y con ellas recetas, opciones y líneas históricas) al borrar una presentación.
No existe endpoint de borrado de `Presentation` (solo `PATCH active`), así que `RESTRICT` es una
red de seguridad de base de datos, no una ruta funcional (spec.md US7, escenario 7).

**Consulta de nombre en lecturas** (las lecturas de `ProductVariant.name` inventariadas en A-79
y `research.md` D10 — 11 archivos de `pos-backend`): `select(..., Presentation.name).join(Presentation, Presentation.id ==
ProductVariant.presentation_id)`; en ORM, `variant.presentation.name`.

### Validaciones de servicio (reemplazan a las de la sección original)

- **Guardar variante** (`_save_variant_entry`): `presentation_id` recibido `null` ⇒ se resuelve la
  presentación "Presentación única" (get-or-create, ver abajo); no nulo ⇒ debe existir, ser del
  tenant y estar `active=True` **salvo** que sea la que la variante ya tiene (una variante puede
  seguir usando una presentación que se desactivó después). Si otra variante del mismo producto, activa
  **o desactivada**, ya usa esa presentación ⇒ `409` (`variante_duplicada` pasa a buscar por
  `presentation_id`; el mensaje conserva el flujo "reactívala en vez de crear otra" si la que
  choca está desactivada).
- **Resolver "Presentación única"**: `SELECT ... FROM presentations WHERE name = 'Presentación
  única' LIMIT 1` (sin filtrar por `active`); si no hay, `INSERT` con `active=true`. Sin columna
  `is_system` ni marca especial: si el administrador la renombra, la próxima resolución crea otra
  (research.md D11). Concurrencia: `presentations.name` es `UNIQUE`; ante `IntegrityError` se
  vuelve a consultar y se reutiliza la ganadora.
- **Herencia de categoría** (`_apply_inherited_or_default_variant`): `VariantSaveIn(presentation_id=p.id)`
  en vez de `VariantSaveIn(name=p.name)`.
- **`ensure_default_variant`**: crea la variante con `presentation_id` = "Presentación única"
  resuelta; deja de escribir el literal.
- **`PATCH /presentations/{id}`**: se **elimina** la guarda `_rename_conflict` y la cascada
  `UPDATE product_variants SET name` — ya no hay columna que sincronizar. Queda solo la unicidad de
  `Presentation.name`.

### Migración (Alembic) — `<rev>_084_variante_sin_nombre.py`

- `down_revision` = cabeza de Alembic al momento de implementar (hoy `a9f7d0310f6b`; confirmar con
  `alembic heads`, tarea T043).
- `@for_each_tenant_schema`, idempotente (`_has_column`) como `a9f7d0310f6b`. Orden por schema:

```sql
-- 1) Presentaciones faltantes: un nombre libre = una fila del catálogo (coincidencia EXACTA).
INSERT INTO {schema}.presentations (id, name, active, created_at, updated_at)
SELECT gen_random_uuid(), n.name, true, now(), now()
FROM (SELECT DISTINCT name FROM {schema}.product_variants WHERE presentation_id IS NULL) n
WHERE NOT EXISTS (SELECT 1 FROM {schema}.presentations p WHERE p.name = n.name);

-- 2) Enlazar cada variante sin presentación con la fila de su nombre.
UPDATE {schema}.product_variants pv
SET presentation_id = p.id
FROM {schema}.presentations p
WHERE pv.presentation_id IS NULL AND p.name = pv.name;

-- 3) Endurecer.
ALTER TABLE {schema}.product_variants ALTER COLUMN presentation_id SET NOT NULL;
-- FK: DROP CONSTRAINT fk__product_variants__presentation_id__presentations,
--     ADD CONSTRAINT ... FOREIGN KEY (presentation_id) REFERENCES {schema}.presentations(id) ON DELETE RESTRICT;
ALTER TABLE {schema}.product_variants DROP CONSTRAINT uq__product_variants__product_id__name;
ALTER TABLE {schema}.product_variants DROP COLUMN name;
```

  - **Coincidencia exacta**, sin `lower`/`trim`: `presentations.name` es `UNIQUE` sensible a
    mayúsculas y el `UNIQUE(product_id, name)` que se retira también lo era, así que dentro de un
    producto dos nombres distintos nunca colapsan en la misma presentación (no puede violarse
    `uq__..__presentation_id` en el paso 2). `gen_random_uuid()` es el mismo mecanismo que ya usa
    `da7581f7bb18` (spec 083) para sembrar `presentations`.
  - Las variantes que ya tenían `presentation_id` **no se tocan** (su `name` ya coincidía por la
    cascada de US1; si por alguna razón difiriera, gana la presentación).
  - La verificación posterior al paso 2 (`SELECT count(*) ... WHERE presentation_id IS NULL`) MUST
    dar 0 antes del paso 3; si no, `RAISE` y la migración aborta sin efecto.
- **Rollback (`downgrade`)**, por schema y sin pérdida: `ADD COLUMN name VARCHAR(255) NULL`;
  `UPDATE product_variants pv SET name = p.name FROM presentations p WHERE p.id = pv.presentation_id`;
  `ALTER COLUMN name SET NOT NULL`, `SET DEFAULT 'Presentación única'`; re-crear
  `uq__product_variants__product_id__name`; `presentation_id` vuelve a `NULL`-able con FK
  `ON DELETE SET NULL`. Es reversible porque dos variantes del mismo producto nunca comparten
  presentación, así que los nombres re-derivados no pueden chocar. Las filas del catálogo creadas
  en el paso 1 **no** se borran al revertir (no se distinguen de las creadas a mano; inocuas).
- **Despliegue en tres pasos** (research.md D10): la migración de arriba es el paso **C**
  (destructivo, el último). Los pasos A (backend que añade `presentation_id`/`presentation_name` a
  las respuestas manteniendo `name`) y B (frontend) no tocan el esquema. La migración repite el
  enlazado de nombres libres precisamente para absorber lo que el frontend anterior haya creado
  entre A y C.

### Efecto sobre el resto de entidades

- **`Presentation`**: sin cambio de columnas. Deja de tener el efecto secundario de cascada en su
  `PATCH` (ver arriba).
- **`OrderItem` / `CartItem` / `Sale`**: sin cambio de columnas; ninguno guarda `variant.name` como
  columna (la descripción de venta se compone al checkout con el nombre entonces vigente y queda
  como texto: `checkout.py:294`, `checkout.py:447`, `sales/service.py:228`). Solo cambia la
  expresión que compone ese texto (`presentation.name`).
- **`Promotion` / `PromotionRule` / `PromotionVariant`**: sin cambio de columnas. La aplicación
  masiva (US4) sigue emparejando por etiqueta de presentación; como cada variante ahora tiene
  siempre una presentación (nombre único en el catálogo), el emparejamiento es equivalente y más
  robusto que antes.
