# Data Model: Catálogo de Presentaciones y Rediseño de Promociones

Todas las entidades nuevas viven en el schema **`tenant`** (por-tenant, vía
`@for_each_tenant_schema`). Las decisiones detrás de cada elección están en
[research.md](./research.md) (D1–D10); este documento se limita a columnas, restricciones,
relaciones, el paso de datos y el rollback (Principio VIII).

**Resumen del cambio de esquema** (una sola migración aditiva, sin contraparte destructiva —
D8):

| Acción | Objeto |
|---|---|
| **Tabla nueva** | `presentations` |
| **Tabla nueva** | `category_presentations` (puente `categories` ↔ `presentations`) |
| **Paso de datos** | Por cada schema de tenant: una `Presentation` activa por cada `product_variants.name` distinto ya existente (FR-019); ninguna fila en `category_presentations` |
| **Sin cambio de columnas** | `product_variants`, `categories`, `promotions`, `promotion_rules`, `promotion_variants` — ninguna gana ni pierde columnas |
| **Cambio de valor por defecto (código, no esquema)** | `ensure_default_variant` pasa de crear `name="Single"` a `name="Presentación única"` (D7, requiere anomalía **A-74** en `registro-de-anomalias.md` antes de implementar) |

Cero cambios de importe en `sales`/`invoices`/`customer_orders` ya emitidas (Principio VII): esta
spec no toca ninguna tabla de hechos contables, solo catálogo.

---

## Entidad nueva: `Presentation` (`presentations`)

`app/models/presentation.py`. Una fila = una etiqueta reutilizable del catálogo de
presentaciones de un tenant (FR-001). Espejo de `OptionGroup` (`app/models/option_group.py`).

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `id` | UUID (PK) | No | `uuid4` | `UUIDPrimaryKeyMixin`, igual que el resto del dominio. |
| `name` | String(255) | No | — | Único dentro del tenant (`UniqueConstraint`/`unique=True`, igual que `OptionGroup.name`). La unicidad se valida tanto al crear como al editar (FR-003) con `ensure_unique(..., exclude_id=...)` (`app/core/crud.py`, ya soporta el caso de edición). |
| `active` | Boolean | No | `true` | Alternable en ambos sentidos desde el listado (US1 Escenario 5). Una presentación inactiva deja de ofrecerse para nuevas asociaciones a categoría (FR-002) pero no se excluye de las ya existentes. |
| `created_at`/`updated_at` | DateTime | No/Sí | — | `TimestampMixin`, igual que el resto del dominio. |

**Restricciones**:

| Nombre | Tipo | Definición |
|---|---|---|
| `pk__presentations` | PK | `(id)` |
| `uq__presentations__name` | UNIQUE | `(name)` — por tenant, vía schema-per-tenant. |

**Relaciones**: ninguna relación ORM directa hacia `ProductVariant`/`PromotionVariant` — el
catálogo de Presentaciones es una etiqueta de ayuda (research.md D9), no una FK en esas tablas
(a diferencia del `Presentation`/`presentation_id` de spec 040, retirado por spec 063 — A-63 — y
que **no** se reintroduce, spec.md §Contexto).

**Sin endpoint de borrado físico** (D4): el ciclo de vida completo de una `Presentation` es
`POST` (crear activa), `PATCH` (renombrar y/o cambiar `active` en cualquier sentido). FR-002 se
cumple porque el borrado físico simplemente no es una operación expuesta, para ninguna
presentación, esté o no referenciada.

---

## Entidad nueva: `CategoryPresentation` (`category_presentations`)

`app/models/category_presentation.py`. Una fila = "esta presentación está habilitada para esta
categoría" (FR-004). Espejo de `VariantOptionGroup` (`app/models/variant_option_group.py`), sin
columnas de negocio propias porque FR-004 no pide cardinalidad ni configuración por fila, solo
pertenencia.

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `id` | UUID (PK) | No | `uuid4` | `UUIDPrimaryKeyMixin`. |
| `category_id` | UUID (FK → `categories.id`) | No | — | `ondelete="CASCADE"` — borrar la categoría borra sus asociaciones (no borra las `Presentation` referenciadas). Indexado. |
| `presentation_id` | UUID (FK → `presentations.id`) | No | — | **Sin** `ondelete="CASCADE"`: `Presentation` no tiene borrado físico (D4), así que nunca hay un DELETE de la fila padre que deba propagarse. Indexado. |

**Restricciones**:

| Nombre | Tipo | Definición |
|---|---|---|
| `pk__category_presentations` | PK | `(id)` |
| `fk__category_presentations__category_id__categories` | FK | `category_id → categories.id`, `ON DELETE CASCADE` |
| `fk__category_presentations__presentation_id__presentations` | FK | `presentation_id → presentations.id` |
| `uq__category_presentations__category_id__presentation_id` | UNIQUE | `(category_id, presentation_id)` — una misma presentación no se asocia dos veces a la misma categoría. |

**Relaciones**:
- `Category.presentation_links: list[CategoryPresentation]` (o relación `secondary=` directa
  `Category.presentations: list[Presentation]` vía tabla asociativa — decisión de implementación,
  ambas formas son válidas en SQLAlchemy 2.0; se recomienda `secondary=` simple ya que no hay
  columnas propias en la puente, análogo a como podría modelarse pero **no** como hoy modela
  `VariantOptionGroup` — esa sí necesita ser entidad propia porque lleva columnas de negocio).
- Sin relación inversa obligatoria en `Presentation` (no se necesita navegar de presentación a
  categorías para ningún flujo de esta spec).

**Reemplazo total en cada guardado de categoría** (research.md D5): `PATCH /categories/{id}` con
`presentation_ids` hace `DELETE FROM category_presentations WHERE category_id = :id` seguido de un
`INSERT` por cada id nuevo — mismo patrón que `_replace_option_groups`
(`app/api/v1/catalog/service.py:178-`). Cambiar las presentaciones de una categoría **no** toca
ninguna `ProductVariant` ya creada (FR-008) — la asociación solo se lee en el momento de crear un
producto nuevo (D6).

---

## Sin cambio de modelo: `Category`, `Product`, `ProductVariant`

- **`Category`** (`app/models/category.py`): sin columnas nuevas. Gana la relación ORM hacia
  `Presentation` vía `CategoryPresentation` (arriba). `CategoryResponse` (schema Pydantic) gana un
  campo de solo lectura `presentations: list[PresentationSummary]`; `CategoryCreate`/
  `CategoryUpdate` ganan `presentation_ids: list[UUID] | None` (ver
  [contracts/categoria-herencia-producto.md](./contracts/categoria-herencia-producto.md)).
- **`ProductVariant`** (`app/models/product_variant.py`): sin cambio de columnas ni de
  constraints. Cambia únicamente el valor por defecto que el código le asigna cuando no hay
  variantes explícitas ni presentaciones de categoría (D6/D7): antes `"Single"`, ahora
  `"Presentación única"`. El `server_default="Single"` de la columna se actualiza al mismo texto
  por consistencia documental (nunca es la ruta de escritura real: el ORM siempre pasa `name`
  explícito en `ensure_default_variant`).
- **`Product`**: sin cambios. `ProductService.create_product` gana lógica nueva (D6) pero ninguna
  columna nueva.
- **`Promotion`/`PromotionRule`/`PromotionVariant`**: sin ningún cambio (modelo, servicio,
  schemas, router) — confirmado en research.md D9. Esta spec no reintroduce `presentation_id` en
  ninguna tabla de promociones (spec.md §Contexto, A-65 sin cambios).

---

## Migración Alembic

Una sola revisión, `down_revision = "ef0abdf40889"` (head actual verificado en `develop`),
`@for_each_tenant_schema`:

1. `CREATE TABLE tenant.presentations (...)` con las columnas/constraints de arriba.
2. `CREATE TABLE tenant.category_presentations (...)` con las columnas/constraints de arriba.
3. **Paso de datos** (FR-019), por cada schema ya migrado (`tenant_default` + cada
   `shared.tenants.schema`):
   ```sql
   INSERT INTO {schema}.presentations (id, name, active, created_at, updated_at)
   SELECT gen_random_uuid(), v.name, true, now(), now()
   FROM (SELECT DISTINCT name FROM {schema}.product_variants) AS v;
   ```
   Comparación exacta de texto, sin `TRIM`/`LOWER` (FR-019: "comparación exacta de texto").
   Guardas `_has_table(schema, "product_variants")` antes de insertar, mismo patrón defensivo que
   `387ef3e638cd`/`94144eaa60b5`, por si algún schema de tenant aún no tiene la tabla (no debería
   ocurrir en `develop`, pero mantiene el patrón del repo).
4. No se inserta ninguna fila en `category_presentations` (FR-019 explícito).

**Downgrade**: `DROP TABLE category_presentations`, `DROP TABLE presentations` — no intenta
revertir selectivamente el paso de datos (mismo criterio que `063a`/`94144eaa60b5`: el downgrade
revierte estructura, no reconstruye el estado de datos previo a un `INSERT` masivo ya aplicado).

**Testing de la migración**: función de paso de datos factorizada en `_seed_sql(schema) ->
str`/función pura equivalente, ejercitable contra SQLite en memoria vía
`schema_translate_map={"tenant": None}` en un characterization test nuevo
(`test_presentations_migration.py`), mismo patrón que `test_promotions_migration.py`.
