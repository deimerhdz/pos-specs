# Data Model: Corrección de Bugs en Promociones y Productos

Único cambio de esquema de esta spec (research.md D1). El resto de las correcciones (bugs 1, 3, 4,
5, y la guarda de exclusividad de producto del bug 3) son comportamiento de servicio/UI sobre
tablas ya existentes, sin cambio de columnas.

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
