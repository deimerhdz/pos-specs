# Data Model: Promociones de Precio por Cantidad Configuradas por Presentación

Todas las entidades viven en el schema **`tenant`** (por-tenant, vía `@for_each_tenant_schema`). Las
decisiones de diseño detrás de cada elección están en [research.md](./research.md); este documento
se limita a columnas, restricciones, relaciones, compatibilidad, migración y rollback (Principio
VIII).

**Resumen del cambio de esquema**: 2 tablas nuevas (`presentations`,
`promotion_presentation_rules`), 1 columna nueva (`product_variants.presentation_id`), 1 `CHECK`
ampliado (`ck_promotions_type`). Cero migraciones de datos. Cero cambios en `sales` /
`sale_invoices` / `payments` (Principio VII).

---

## Entidad nueva: `Presentation` (`presentations`)

Concepto de catálogo compartido del tenant (p. ej. "8oz", "16oz") al que variantes de distintos
productos pueden apuntar. **No es** la variante de producto ni su `name` libre (research.md D1).

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `id` | UUID (PK) | No | `uuid4` | `UUIDPrimaryKeyMixin`. |
| `name` | `String(100)` | No | — | No vacío (`min_length=1` en el schema, recorte de espacios como `VariantCreate`, `catalog/schemas.py:11`). Único por tenant. |
| `active` | `Boolean` | No | `true` (`server_default`) | Baja lógica. Una presentación inactiva no se ofrece en el selector de variante ni se puede elegir en una regla nueva. La baja está **bloqueada** mientras una regla de una promoción `active` la referencie (FR-020 — ver "Reglas de validación"). |
| `created_at` | `DateTime` | No | `now()` | `TimestampMixin`. |
| `updated_at` | `DateTime` | Sí | — | `TimestampMixin`. |

**Restricciones**:

| Nombre | Tipo | Definición |
|---|---|---|
| `pk__presentations` | PK | `(id)` |
| `uq__presentations__name` | UNIQUE | `(name)` — por schema, es unicidad por tenant. |

**Relaciones**:
- `ProductVariant.presentation` (N variantes → 1 presentación), vía `product_variants.presentation_id`.
- `PromotionPresentationRule.presentation` (N reglas → 1 presentación).

---

## Entidad nueva: `PromotionPresentationRule` (`promotion_presentation_rules`)

Tabla hija de `promotions`, análoga a `promotion_combo_items` (`app/models/promotion.py:178-203`).
Una fila = una regla de la promoción: `(presentación, cantidad mínima, precio total del paquete)`
(FR-001). Solo se usa cuando `promotion.type == "qty_price_presentation"`.

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `id` | UUID (PK) | No | `uuid4` | `UUIDPrimaryKeyMixin`. |
| `promotion_id` | UUID (FK → `promotions.id`) | No | — | `ondelete="CASCADE"`, indexado. Borrar la promoción borra sus reglas. |
| `presentation_id` | UUID (FK → `presentations.id`) | No | — | `ondelete="CASCADE"` (research.md D10: una regla que pierde su presentación es basura; una promoción `draft` se recompleta antes de activar, que revalida). |
| `min_qty` | `Integer` | No | — | Cantidad mínima del paquete. `CHECK >= 1` — **a diferencia de `qty_price`** (`min_qty >= 2`), aquí `1` es válido: "precio especial por unidad de esa presentación" (CL-7). |
| `pack_price` | `Numeric(12, 2)` | No | — | Precio total del paquete. `CHECK >= 0`. Es el `value` de la regla; no se hereda de `Promotion.value` (research.md D3). |

**Restricciones**:

| Nombre | Tipo | Definición |
|---|---|---|
| `pk__promotion_presentation_rules` | PK | `(id)` |
| `fk__promotion_presentation_rules__promotion_id__promotions` | FK | `promotion_id → promotions.id`, `ON DELETE CASCADE` |
| `fk__promotion_presentation_rules__presentation_id__presentations` | FK | `presentation_id → presentations.id`, `ON DELETE CASCADE` |
| `uq__promotion_presentation_rules__promotion_id__presentation_id` | UNIQUE | `(promotion_id, presentation_id)` — última red de "no dos reglas para la misma presentación dentro de una promoción" (FR-006, 1ª parte). |
| `ck__promotion_presentation_rules__min_qty` | CHECK | `min_qty >= 1` |
| `ck__promotion_presentation_rules__pack_price` | CHECK | `pack_price >= 0` |
| `ix__promotion_presentation_rules__promotion_id` | INDEX | `(promotion_id)` |

**Relaciones**:
- `Promotion.presentation_rules: list[PromotionPresentationRule]`, `cascade="all, delete-orphan"`
  (igual que `Promotion.targets` y `Promotion.combo_items`, `promotion.py:84-89`).

---

## Entidad modificada: `ProductVariant` (`product_variants`)

`app/models/product_variant.py`. **Una columna nueva**, nada más.

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `presentation_id` | UUID (FK → `presentations.id`) | **Sí** | `NULL` | **NUEVA.** `ondelete="SET NULL"` (research.md D1): si la presentación se borra, la variante queda sin presentación, no se rompe. Indexada (el motor filtra por ella en cada cobro, research.md D5/D11). |

Columnas existentes (`id`, `product_id`, `name`, `sku`, `price`, `active`) — **sin cambio**. En
particular `name` sigue siendo texto libre por producto (`UniqueConstraint("product_id", "name")`
intacto): puede decir "8oz" y **además** apuntar a la presentación "8oz", o no apuntar a ninguna.

**Restricciones nuevas**:

| Nombre | Tipo | Definición |
|---|---|---|
| `fk__product_variants__presentation_id__presentations` | FK | `presentation_id → presentations.id`, `ON DELETE SET NULL` |
| `ix__product_variants__presentation_id` | INDEX | `(presentation_id)` |

**Compatibilidad hacia atrás**: toda variante existente arranca con `presentation_id = NULL` → fuera
de toda regla por presentación (FR-008). La "Single" de un producto sin tamaños
(`catalog/service.py:63-80`) tampoco entra salvo asignación explícita. Sin backfill (research.md
D15).

---

## Entidad modificada: `Promotion` (`promotions`)

`app/models/promotion.py`. **Sin columnas nuevas.** Se amplía la lista de tipos y **el ancho de la
columna `type`**.

- `PROMOTION_TYPES` (constante Python, `promotion.py:30`): `("percent", "fixed", "combo",
  "qty_price", "qty_price_presentation")`.
- `Promotion.type`: la columna se creó como `varchar(20)` (`d3e4f5a6b7c8_promotions.py`) — cabía
  hasta `qty_price` (9). `qty_price_presentation` tiene **22 caracteres**, así que la migración
  `f03274730367` **ensancha `promotions.type` a `varchar(50)`** (`op.alter_column`, `downgrade`
  simétrico `50 → 20`) y el modelo pasa a `String(50)`. Sin esta ampliación el `INSERT` falla en
  PostgreSQL con `StringDataRightTruncation` → 500 (SQLite no valida el ancho, por eso los
  characterization tests no lo detectan — ver `quickstart.md` Paso 1).
- `CheckConstraint` `ck_promotion_type` (`promotion.py:92-95`): `type IN ('percent', 'fixed',
  'combo', 'qty_price', 'qty_price_presentation')`.
- `Promotion.value`: **no se usa** en el tipo nuevo (el precio vive por regla en
  `promotion_presentation_rules.pack_price`). Igual que `value`/`min_qty` de `Promotion` "no se
  usan" en `qty_price` (`promotion.py:39-42`). Su `CHECK value >= 0` sigue vigente sin conflicto
  (default `0`).
- `Promotion.min_qty`: tampoco se usa en el tipo nuevo (la cantidad vive por regla). El `CHECK
  "type <> 'qty_price' OR min_qty >= 2"` no aplica al tipo nuevo (su `type` no es `'qty_price'`), así
  que `min_qty` queda en su default `1` sin romper nada.
- Vigencia (`starts_at`, `ends_at`, `days_of_week`, `start_time`, `end_time`, `status`, `priority`):
  **modelo y máquina de estados sin cambio** — la promoción nueva usa el mismo modelo de vigencia y
  la misma máquina de estados (`PROMOTION_TRANSITIONS`, `promotion.py:21-26`) que el resto (spec 013
  FR-002). La comparación de **hora** con cruce de medianoche (22:00–02:00) ya está soportada
  (`_in_time_window`, `service.py:76-91`, FR-003). La atribución de **día** cuando la ventana cruza
  la medianoche (FR-004/CL-8: las horas tras medianoche pertenecen al día de inicio) **NO** está
  soportada hoy — `_valid_now` compara `now.weekday()` de forma independiente. Esta spec corrige
  `_valid_now` para todos los tipos de promoción; el cambio de comportamiento queda registrado como
  A-55 en `registro-de-anomalias.md` (research.md D18, tarea T009a).

**`AUTO_TYPES`** (`service.py:47`): **NO** cambia. El tipo nuevo queda fuera del motor
línea-por-línea, igual que `combo` (research.md D4).

---

## Entidades NO modificadas (pero relevantes al cálculo)

| Entidad | Por qué es relevante | Cambio |
|---|---|---|
| `Sale` (`sales`) | `promotion_id` (FK `promotions.id`, `ondelete="SET NULL"`, `sale.py:64`) se rellena si una única promoción explica todas las líneas descontadas (spec 012 FR-026, `single_promotion_id`). La promoción por presentación entra a esa misma regla. | Ninguno. |
| `OrderItem` (`order_items`) / `CartItem` (`cart_items`) | Tienen `promotion_id` FK `SET NULL` (`order_item.py:57`, `cart_item.py:36`) para el snapshot de descuento por línea (commit `a4afbe0`). El desglose por línea de la modalidad nueva puede poblarlos igual. | Ninguno de esquema. |
| `PromotionTarget` (`promotion_targets`) | El tipo nuevo **no lo usa** (research.md D3). `qty_price`/`percent`/`fixed` siguen usándolo idéntico (FR-016). | Ninguno. |
| `PromotionComboItem` (`promotion_combo_items`) | Precedente estructural de `promotion_presentation_rules`. Los combos siguen intactos. | Ninguno. |
| `Category` (`categories`) / `Product` (`products`) | El alcance por presentación es ortogonal a categoría/producto. | Ninguno. |

---

## Reglas de validación (resumen por historia de usuario)

| Regla | Capa(s) donde se aplica | FR / Historia |
|---|---|---|
| Una promoción `qty_price_presentation` tiene ≥ 1 regla; cada regla = `(presentation_id, min_qty ≥ 1, pack_price ≥ 0)` | Pydantic (`PresentationRuleIn`, lista no vacía) + servicio + `CHECK`s de tabla | FR-001, CL-7 |
| No dos reglas para la misma `presentation_id` dentro de una promoción | Pydantic (lista sin `presentation_id` repetido) + servicio + `UniqueConstraint(promotion_id, presentation_id)` | FR-006 (1ª parte), US1-CA2 |
| Al guardar/activar, ninguna regla apunta a una `presentation_id` ya cubierta por una regla de **otra** promoción `qty_price_presentation` con `status == "active"` → **409** nombrando el conflicto | Servicio, en `create` / `update_shape` **y** en `change_status → active` (research.md D8) | FR-006 (2ª parte), US1-CA3, CL-4 |
| Solo `type == "qty_price_presentation"` acepta `presentation_rules`; otros tipos que las envíen → 422 | Pydantic + servicio (patrón de `_combo_items_required`, `schemas.py:168-180`) | research.md D4 |
| Al guardar una regla, si las variantes activas que la referencian no tienen precio uniforme → **422** con el detalle, salvo `confirm_precio_no_uniforme = true` | Servicio, solo en `create` / `update_shape` (nunca retroactivo) | FR-017, FR-018, US3-CA1/CA2/CA3, CL-1/CL-1b |
| Al guardar una regla, si `pack_price / min_qty ≥ precio_ref` (no es descuento real) → **422** con el detalle, salvo `confirm_sin_descuento = true` | Servicio, solo en `create` / `update_shape` | FR-022 |
| La interfaz muestra, antes de confirmar, el resumen de todas las reglas y cuántas variantes alcanza cada una | Backend expone `applicable_count` por regla; frontend lo pinta ("Resumen de la Regla" + "Productos Aplicables") | FR-005, US1-CA1 |
| No se puede eliminar ni desactivar una presentación referenciada por una regla de una promoción `active` → **409** con la lista de promociones | Servicio de `presentations` (`DELETE` y `PATCH active=false`) | FR-020, US4-CA2, CL-2 |
| El alcance de una regla se resuelve SIEMPRE por `product_variants.presentation_id`, nunca por `ProductVariant.name`; incluye variantes creadas después | `presentation_package_discount_for_lines` hace el `SELECT` en cada cobro (research.md D11) | FR-007, FR-019, US4-CA1 |
| Una variante `active = false` no cuenta como unidad elegible para completar un paquete | `presentation_package_discount_for_lines` filtra `ProductVariant.active` | FR-015, CL-1c |
| Una variante sin `presentation_id` no entra en ninguna regla | `NULL` nunca casa el `SELECT` por `presentation_id` | FR-008 |

---

## Cálculo del descuento — invariantes de datos (no de esquema)

Ninguna de estas magnitudes se persiste (FR-014); se listan como el contrato del cálculo que
`presentation_package_discount_for_lines` debe cumplir (research.md D5/D6):

| Invariante | FR |
|---|---|
| `paquetes = total_unidades_elegibles // min_qty`; solo paquetes completos descuentan; el resto va a `precio_ref` | FR-010 |
| `precio_ref` = menor `unit_price` vigente entre las variantes elegibles que aportan unidades a esa presentación en el pedido; único para todas sus unidades; nunca variante por variante | FR-011, FR-017 |
| Cada unidad de un paquete completo se cobra a `(pack_price / min_qty).quantize(Decimal("1"), ROUND_HALF_UP)` a peso; el residuo se asigna a la unidad de la línea de "identificador más alto" (ver la definición operativa bajo esta tabla) | FR-011, CL-9 |
| Las unidades sobrantes se cobran a `precio_ref` y se toman de la(s) línea(s) de "identificador más alto" (misma definición operativa) | FR-011 |
| El descuento por línea es derivado: `unidades_línea_en_presentación × precio_ref − cobrado_a_la_línea`; la suma por línea cuadra con el descuento total al peso, sin depender del orden de las líneas | FR-011, SC-005 |
| Una línea nunca acumula el descuento de una promoción heredada de producto y el de una de presentación; gana la de **menor total para esa línea** | FR-013 |
| Nunca se aplica el descuento por presentación a una línea si la deja con total mayor que sin promoción | FR-023 |
| Presentaciones sin regla en la promoción no reciben descuento de esta promoción | FR-012 |
| Nunca se mezclan unidades de presentaciones distintas dentro de un mismo paquete | FR-009 |

**Definición operativa de "identificador más alto" (FR-011, CL-9; determinismo SC-005 / CA-5)**:
para asignar el residuo del redondeo y para elegir de qué línea(s) salen las unidades sobrantes, el
orden es por el **valor del UUID** de la fila (comparación nativa de `uuid.UUID`, que ordena por
`.int` y coincide con el orden byte a byte de PostgreSQL). "Más alto" = `max(...)`.
- Clave primaria: `product_variants.id` de la variante de la línea.
- Desempate (dos líneas de la **misma** variante): el `id` de la fila de línea del camino de cobro
  correspondiente (`order_items.id` / `cart_items.id` / `sale_items.id`).
Ninguna de las dos claves depende del orden de captura de las líneas ni del motor de base de datos;
nunca se usa la posición de la línea en la lista.

---

## Migración y rollback (Principio VIII)

**Revisión Alembic nueva** — `down_revision = "187e491e597a"` (head verificado con `alembic heads`).
Patrón: `@for_each_tenant_schema` + guarda `_has_table` (como `d3e4f5a6b7c8_promotions.py` y
`e3f4a5b6c7d8_products_tracks_inventory.py`). Nombres de constraint con `op.f(...)`.

### `upgrade(schema)`

```text
if not _has_table(schema, "promotions"): return
1. create_table "presentations"                       (id, name, active server_default 'true',
                                                        created_at, updated_at;
                                                        UniqueConstraint(name))
2. create_table "promotion_presentation_rules"         (id, promotion_id FK CASCADE,
                                                        presentation_id FK CASCADE,
                                                        min_qty  CHECK >= 1,
                                                        pack_price Numeric(12,2) CHECK >= 0;
                                                        UniqueConstraint(promotion_id,
                                                        presentation_id);
                                                        index(promotion_id))
3. add_column "product_variants.presentation_id" UUID NULL
   create_foreign_key ... ondelete="SET NULL"
   create_index ix__product_variants__presentation_id
4. alter_column "promotions.type"  varchar(20) -> varchar(50)   (qty_price_presentation = 22 chars)
5. drop_constraint  ck_promotions_type
   create_check_constraint ck_promotions_type
       "type IN ('percent','fixed','combo','qty_price','qty_price_presentation')"
```

### `downgrade(schema)` — simétrico, sin pérdida de dato histórico

```text
if not _has_table(schema, "promotions"): return
5'. drop_constraint ck_promotions_type
    create_check_constraint "type IN ('percent','fixed','combo','qty_price')"   (el CHECK vigente
                                                                                 del modelo hoy)
4'. alter_column "promotions.type"  varchar(50) -> varchar(20)   (el CHECK ya prohíbe el valor de
                                                                  22 chars, así que no hay filas
                                                                  que lo impidan)
3'. drop_index / drop_constraint FK / drop_column product_variants.presentation_id
2'. drop_table promotion_presentation_rules
1'. drop_table presentations
```

**Por qué el rollback es seguro**: nada histórico se escribe en estas estructuras. El descuento
nunca se persiste (FR-014). `Sale.promotion_id` / `OrderItem.promotion_id` / `CartItem.promotion_id`
son FK `ON DELETE SET NULL` hacia `promotions` — si una promoción `qty_price_presentation` llegara a
existir y luego se hiciera `downgrade`, primero habría que finalizar/borrar esas promociones
(operación de administración, no de migración); una vez sin filas de ese tipo, el `downgrade` no
encuentra nada que romper. El `drop_column presentation_id` descarta asignaciones de catálogo (no
transaccionales, re-capturables por el administrador), nunca un importe cobrado.

**Datos de scratch** (`tenant_default`): la guarda `_has_table` evita fallos en schemas que no
tienen las tablas base, igual que las migraciones existentes.
