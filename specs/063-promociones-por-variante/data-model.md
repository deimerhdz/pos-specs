# Data Model: Refactorización del módulo de promociones — modelo por conjunto explícito de variantes

Todas las entidades viven en el schema **`tenant`** (por-tenant, vía `@for_each_tenant_schema`).
Las decisiones de diseño detrás de cada elección están en [research.md](./research.md); este
documento se limita a columnas, restricciones, relaciones, compatibilidad, migración (incluido el
**paso de datos**) y rollback (Principio VIII).

**Resumen del cambio de esquema** — en **dos revisiones Alembic** (Principio VI, ver §"Migración y
rollback"):

*Revisión `063a` (aditiva, Incremento A) — deja el motor viejo y todas las tablas viejas intactos:*

| Acción | Objeto |
|---|---|
| **Tabla nueva** | `promotion_variants` (hija de `promotions`, sin atributos de precio) |
| **Columnas nuevas** | `promotions.closed_by_refactor_at`; `sales.applied_promotions`; `invoices.applied_promotions`; `customer_orders.discount`; `customer_orders.applied_promotions` |
| **CHECKs** | `ck_promotion_min_qty` (`min_qty >= 1`) nuevo; `ck_promotion_type` **ampliado** a `('percent','fixed','combo','qty_price','qty_price_presentation','package_price')`; `ck_customer_order_discount_non_negative` nuevo |
| **Paso de datos** | `percent` → filas en `promotion_variants` (foto fija); `combo`/`fixed`/`qty_price`/`qty_price_presentation` no terminales → `status='finished'` + `closed_by_refactor_at` |

*Revisión `063b` (destructiva, Incremento F) — cuando ningún módulo referencia ya la estructura vieja:*

| Acción | Objeto |
|---|---|
| **Columnas borradas** | `promotions.priority`; `product_variants.presentation_id` (+ FK + índice) |
| **Tablas borradas** | `promotion_targets`; `promotion_combo_items`; `promotion_presentation_rules`; `presentations` |
| **CHECKs** | `ck_promotion_type` **estrechado con escape** a `type IN ('percent','package_price') OR status = 'finished'`; `ck_promotion_qty_price_pack` borrado |

Cero cambios de importe en `sales` / `invoices` / `payments` ya emitidas (Principio VII).

---

## Entidad nueva: `PromotionVariant` (`promotion_variants`)

Tabla hija de `promotions`, análoga a `promotion_combo_items` (`app/models/promotion.py:192-217`)
**sin** la columna `quantity`. Una fila = "esta variante pertenece al conjunto elegible de esta
promoción" (FR-001). **No lleva `type`, `value` ni `min_qty`** (clarification 2026-08-31).

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `id` | UUID (PK) | No | `uuid4` | `UUIDPrimaryKeyMixin`. |
| `promotion_id` | UUID (FK → `promotions.id`) | No | — | `ondelete="CASCADE"`, indexado. Borrar la promoción borra sus filas de conjunto. |
| `product_variant_id` | UUID (FK → `product_variants.id`) | No | — | `ondelete="CASCADE"` — FR-011: una variante **eliminada** sale del conjunto de toda promoción sin dejar fila huérfana. (Una variante **desactivada** no se borra; el motor la filtra por `product_variants.active`, ver §"Reglas de validación".) |

**Restricciones**:

| Nombre | Tipo | Definición |
|---|---|---|
| `pk__promotion_variants` | PK | `(id)` |
| `fk__promotion_variants__promotion_id__promotions` | FK | `promotion_id → promotions.id`, `ON DELETE CASCADE` |
| `fk__promotion_variants__product_variant_id__product_variants` | FK | `product_variant_id → product_variants.id`, `ON DELETE CASCADE` |
| `uq__promotion_variants__promotion_id__product_variant_id` | UNIQUE | `(promotion_id, product_variant_id)` — última red de "una variante aparece una sola vez en el conjunto". |
| `ix__promotion_variants__promotion_id` | INDEX | `(promotion_id)` |
| `ix__promotion_variants__product_variant_id` | INDEX | `(product_variant_id)` — el bloqueo de solape (FR-014) y el motor cruzan por esta columna. |

**Relaciones**:
- `Promotion.variants: list[PromotionVariant]`, `cascade="all, delete-orphan"` (igual que
  `Promotion.targets` / `Promotion.combo_items` hoy, `promotion.py:95-100`).
- `PromotionVariant.product_variant: ProductVariant` (viewonly no requerido; el motor y el
  serializador cargan la variante para el precio normal y la descripción, FR-005).

---

## Entidad modificada: `Promotion` (`promotions`)

`app/models/promotion.py`.

**Se borra — en el Incremento F (revisión `063b` + T061c), no en el Incremento A**, porque
`promotions/service.py` (motor viejo), `catalog/router.py` y `menu/router.py` referencian esto hasta
las Phases 4/8:
- Columna `priority` (`promotion.py:83`) y su `server_default="0"` (research.md D3, A-58).
- Clases `PromotionTarget` (`promotion.py:136-189`), `PromotionComboItem` (`promotion.py:192-217`),
  `PromotionPresentationRule` (`promotion.py:220-256`) y las relaciones `targets`, `combo_items`,
  `presentation_rules` (`promotion.py:95-103`).
- `CheckConstraint` `ck_promotion_qty_price_pack` (`promotion.py:124-127`, `type <> 'qty_price'
  OR min_qty >= 2`) — el tipo `qty_price` deja de existir.
- El import de `Presentation` y el bloque `TYPE_CHECKING`.

**Se modifica**:
- `PROMOTION_TYPES` (`promotion.py:38`): en el Incremento A **se amplía** a
  `("percent", "fixed", "combo", "qty_price", "qty_price_presentation", "package_price")` y **se
  queda así** — sirve para leer las promociones `finished` de tipo viejo que dejó la migración. La
  restricción "solo dos tipos" vive en el enum de **entrada** de Pydantic (`PromotionType` en
  `PromotionCreate` / `PromotionShapeUpdate`, T018), no en `PROMOTION_TYPES`.
- `CheckConstraint` `ck_promotion_type` (`promotion.py:106-109`): `063a` lo amplía con
  `'package_price'`; `063b` lo estrecha **con escape** a
  `type IN ('percent','package_price') OR status = 'finished'` (ninguna promoción viva con tipo
  viejo; las `finished` históricas conservan su `type`).
- `type` (`promotion.py:71`): se queda en `String(50)` (no se estrecha — `package_price` = 13
  chars cabe de sobra; estrechar sería otra migración de columna sin beneficio, research.md D2). En
  `PromotionResponse` es `str` libre (serializa promociones `finished` de tipo viejo, FR-025).
- `value` (`promotion.py:73-75`): sin cambio de columna. Semántica: `percent` → 0 < value ≤ 100
  (`ck_promotion_percent_range` se **conserva**); `package_price` → precio total de `min_qty`
  unidades (value > 0 recomendado; el `ck_promotion_value_positive` `value >= 0` se conserva, el
  servicio exige `> 0` para `package_price`).
- `min_qty` (`promotion.py:93`): sin cambio de columna (`server_default="1"`). **CHECK nuevo** (en
  `063a`) `ck_promotion_min_qty` `min_qty >= 1` (antes solo lo garantizaba Pydantic `ge=1` y, para
  `qty_price`, el CHECK que se borra en `063b`).

**Se agrega**:
- `closed_by_refactor_at: Mapped[Optional[datetime]] = mapped_column(DateTime, nullable=True)` —
  `NULL` para toda promoción salvo las que la migración de la spec 063 pasó a `finished`
  (`combo`/`fixed`/`qty_price`/`qty_price_presentation` no terminales). Es la fuente del aviso
  "recrea a mano" (FR-025, research.md D12). Se expone en `PromotionResponse` y se filtra con
  `GET /promotions?closed_by_refactor=true`.

**Sin cambio**: `name` (único), `description`, `status` + `ck_promotion_status` +
`PROMOTION_TRANSITIONS` (`promotion.py:20-28`, la máquina de estados de FR-015 es la misma),
`starts_at`, `ends_at`, `days_of_week`, `start_time`, `end_time`, `Index
ix_promotions_status_ends_at`.

---

## Entidad modificada: `ProductVariant` (`product_variants`)

`app/models/product_variant.py`. **Se borra** la columna `presentation_id` (`product_variant.py:48-53`),
su `ForeignKey` a `presentations.id`, su `index=True`, la relación `presentation`
(`product_variant.py:54`) y el import de `Presentation` (FR-027, research.md D13, A-63) — en el
**Incremento F** (revisión `063b` + T061c), junto con el resto del retiro de `Presentation`.

Además, la integración que la **spec 040 añadió al catálogo** se revierte en el mismo incremento
(T061d): `api/v1/catalog/schemas.py` (`VariantCreate` / `VariantUpdate` / `VariantResponse` pierden
`presentation_id`) y `api/v1/catalog/router.py` (`_resolve_presentation_id`, import de `Presentation`).
En `pos-heladeria`: `modules/products/services/product.service.ts` y
`modules/tables/services/diner.service.ts` (T053 / T059).

Columnas existentes (`id`, `product_id`, `name`, `sku`, `price`, `active`, `display_order`) —
**sin cambio**. El motor de promociones ahora referencia `product_variants` **solo** por
`promotion_variants.product_variant_id` y por `product_variants.active` / `.price`.

**Compatibilidad hacia atrás**: ninguna — la columna nace de la spec 040. El `downgrade` de `063b`
la recrea `NULL`.

---

## Entidades modificadas: `Sale` (`sales`), `Invoice` (`invoices`), `CustomerOrder` (`customer_orders`)

### `sales` — persistencia del resultado (FR-021, A-64)

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `applied_promotions` | `JSONB` | No | `'[]'::jsonb` (`server_default`) | **NUEVA.** Snapshot al emitir: `[{"promotion_id": "<uuid>", "name": "<snapshot>", "amount": "<decimal string>"}]`, una entrada por promoción que descontó alguna línea de ese cobro, con su **monto agregado** (no por línea). Mismo patrón inmutable que `sale_items.options` (`sale.py:122-124`). |

**Sin cambio**: `discount` (`sale.py:55`) sigue siendo el agregado (ya incluye el descuento de
promociones). `promotion_id` (`sale.py:67-69`, FK `SET NULL`) **se conserva**: se sigue poblando
cuando **una sola** promoción explica todo el descuento; con dos o más queda `NULL` (como hoy),
pero `applied_promotions` lo cubre → **cierra A-29**. `sale_items.combo_id` (`sale.py:134-136`)
**se conserva** (histórico, FR-024, Principio VII); deja de escribirse.

### `invoices` — snapshot de factura (FR-021)

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `applied_promotions` | `JSONB` | No | `'[]'::jsonb` | **NUEVA.** `issue_for_sale` la copia de `sale.applied_promotions` dentro de la transacción del cobro, igual que hoy copia `discount=sale.discount` (`invoices/service.py:67`). Inmutable. |

**Sin cambio**: `discount` (`invoice.py:35`).

### `customer_orders` — descuento del pedido (FR-021)

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `discount` | `Numeric(12, 2)` | No | `0` (`server_default="0"`) | **NUEVA.** Hoy `CustomerOrder` no tiene ningún campo de descuento. Se fija en el cobro (`pay_order`, `_close_unified`, `_close_split`) con el mismo agregado que la `Sale`. |
| `applied_promotions` | `JSONB` | No | `'[]'::jsonb` | **NUEVA.** Mismo contenido que `sales.applied_promotions`, fijado en el cobro. |
| `ck_customer_order_discount_non_negative` | CHECK | — | — | `discount >= 0`. |

**Compatibilidad hacia atrás**: `discount` nace `0`, `applied_promotions` nace `'[]'` para todo
pedido existente. Sin backfill; no retroactivo (FR-021).

---

## Entidades BORRADAS

| Entidad | Tabla | Qué la reemplaza |
|---|---|---|
| `Presentation` | `presentations` | Nada — el alcance se resuelve por `promotion_variants` (FR-027). |
| `PromotionTarget` | `promotion_targets` | `promotion_variants` (FR-003 — no hay alcance por producto/categoría). |
| `PromotionComboItem` | `promotion_combo_items` | Nada — el tipo `combo` se retira (FR-024). |
| `PromotionPresentationRule` | `promotion_presentation_rules` | `promotion_variants` (FR-027). |

El `DROP TABLE` va en la migración **después** del paso de datos (§Migración). Las FKs
`product_variants.presentation_id → presentations.id` y `promotion_*_id → promotions.id` se
resuelven con el orden de borrado (primero las hijas, luego `presentations`).

---

## Entidades NO modificadas (pero relevantes al cálculo)

| Entidad | Por qué es relevante | Cambio |
|---|---|---|
| `SaleItem` / `OrderItem` / `CartItem` | Tienen `combo_id` (FK `SET NULL`) y, en order/cart, `promotion_id` para el snapshot por línea (spec 038, commit `a4afbe0`). El desglose por línea de venta queda **fuera de alcance** (FR-021). | Ninguno de esquema. `combo_id` deja de escribirse; `promotion_id` de order/cart se sigue usando para el snapshot de preview del carrito (spec 038), poblado desde `by_line` del motor nuevo. |
| `AuditLog` (`audit_logs`) | El "historial de modificaciones" (A-42) se **deprecia** — no se construye ni se expone (clarification 2026-08-31). `record_audit` de `create`/`update`/`shape`/`status`/`duplicate`/`delete` sigue registrando lo que ya registra. | Ninguno. |
| `Product` / `Category` | El alcance ya no depende de ellos (FR-003). El formulario los usa **solo** como filtro para poblar el selector de variantes (FR-004). | Ninguno. |

---

## Reglas de validación (resumen por historia de usuario)

| Regla | Capa(s) | FR / Historia |
|---|---|---|
| Una promoción tiene exactamente un `(type, value, min_qty)` y ≥ 1 fila en `promotion_variants` | Pydantic (`variant_ids` no vacía) + servicio (`_apply_variant_set`) + `NOT NULL`/FK de tabla | FR-001, US1-CA3 |
| `type ∈ {percent, package_price}` para toda promoción **viva** (nada nuevo fuera de eso) | Pydantic (`PromotionType` de entrada) + servicio + `ck_promotion_type` (`… OR status='finished'`) | FR-002 |
| `percent`: `0 < value <= 100` | Pydantic (`_percent_range`) + `ck_promotion_percent_range` | FR-002, US1-CA4 |
| `package_price`: `value > 0`, `min_qty >= 1` | Pydantic + servicio + `ck_promotion_min_qty` | FR-002, FR-006 |
| El alcance se resuelve SIEMPRE por `promotion_variants`; los filtros del formulario no se guardan; ninguna variante creada después entra sola | Servicio (guarda la lista concreta) + motor (`SELECT` por `promotion_variants` en cada cobro) | FR-003, FR-004, FR-010, US1-CA2 |
| Antes de confirmar, la UI muestra tipo + condición en lenguaje llano + lista de variantes con su **precio normal vigente** | Backend expone `variants: [{product_variant_id, description, unit_price}]` en `PromotionResponse`; frontend lo pinta | FR-005, US1-CA1 |
| `package_price`: si `value >= min_qty × (menor price del conjunto)` → **409** al guardar/activar | Servicio (`_guard_package_is_discount`) en `create` / `update_shape` / `change_status→active` | FR-016, SC-002, US-Edge |
| Crear o activar con conjunto que comparte ≥1 variante con otra promoción no terminal **y** ventanas de fecha ∧ días ∧ horas que se intersectan → **409** nombrando promoción + variantes | Servicio (`_guard_variant_overlap`) en `create` / `update_shape` / `change_status→active` | FR-014, FR-014a, SC-003, US3 |
| En `active`/`paused`: editable solo `name`, `description`, `ends_at`, `days_of_week`, `start_time`, `end_time`. Bloqueado `type`, `value`, `min_qty`, `variant_ids` | Servicio (`update` valida contra `promo.status`; `update_shape` exige `draft`) | FR-018, US5-CA1/CA2 |
| Duplicar → copia en `draft` con mismo tipo/valor/`min_qty`/conjunto/vigencia, nombre distinto | Servicio (`duplicate`) | FR-017, US5-CA4 |
| Solo el **administrador del tenant** crea/edita/duplica/cambia estado/elimina; el cajero solo visualiza | `require_tenant_admin` en el router (ya vigente) | FR-019, US5-CA5 |
| Una variante `active = false` o eliminada no cuenta como unidad elegible; si el conjunto queda sin variantes elegibles la promoción deja de descontar pero **no** cambia de estado | Motor (`evaluate_variant_sets` filtra `_variant_active`; FK `CASCADE` quita filas de variantes borradas) | FR-011, US2-CA9, US-Edge |
| Estados y transiciones `Borrador → {Activa, Finalizada}`, `Activa → {Pausada, Finalizada}`, `Pausada → {Activa, Finalizada}`, `Finalizada → {}` | `PROMOTION_TRANSITIONS` (sin cambio) + `change_status` | FR-015, US5-CA3 |

---

## Cálculo del descuento — invariantes de datos (no de esquema)

Ninguna de estas magnitudes se persiste **antes** de emitir (FR-020); se listan como el contrato
que `evaluate_variant_sets` debe cumplir (research.md D5/D6). Lo que **sí** se persiste al emitir
es el agregado + la lista (`applied_promotions`, `discount`), nunca el desglose por línea.

| Invariante | FR |
|---|---|
| `grupos = total_unidades_elegibles // min_qty`; solo grupos completos descuentan; el remanente va a precio normal | FR-007 |
| Las `grupos × min_qty` unidades de los grupos se eligen por **precio unitario descendente** (desempate `product_variant_id` asc, luego `line_id` asc); nunca por la posición de la línea | FR-008 |
| `package_price`: descuento de un grupo = `max(0, Σ precios normales de sus unidades − value)` | FR-006, FR-009 |
| `percent`: descuento de un grupo = `round(value% × Σ precios normales de sus unidades)`, a peso | FR-006 |
| El descuento de un grupo se reparte repartiendo el **importe cobrado**: cada línea contribuyente cobra `floor(aporte_línea − descuento_grupo × aporte_línea / aporte_total)`; los pesos que falten se suman al cobrado de la línea de la variante de **id más alto** (desempate: `line_id` más alto) | FR-008a, SC-005 |
| El descuento aplicado a una línea nunca supera el precio normal de sus unidades; ninguna línea queda por debajo de $0; un grupo no se aplica si deja la línea con total mayor que sin promoción | FR-009 |
| El total y el reparto por línea **no dependen del orden** de las líneas del pedido | FR-008, SC-005 |
| Una línea nunca acumula dos promociones — garantizado por el bloqueo de solape real (FR-014), sin reconciliación en el motor | FR-014 |
| `applied_promotions` lista **toda** promoción que descontó ≥1 línea, con su monto agregado; `discount` es la suma | FR-021 |

**Definición operativa de "id más alto"** (FR-008a; determinismo SC-005): comparación nativa de
`uuid.UUID` (ordena por `.int`, coincide con el orden byte a byte de PostgreSQL). Clave primaria:
`product_variants.id`. Desempate (dos líneas de la **misma** variante): el `id` de la fila de
línea del camino de cobro (`order_items.id` / `cart_items.id` / `sale_items.id`). Ninguna clave
depende del orden de captura ni del motor de base de datos. Mismo criterio que la spec 040
(`_unit_sort_key`, `service.py:522-529`).

---

## Migración y rollback (Principio VIII)

**Dos revisiones Alembic** (Principio VI: el Incremento A queda 100% aditivo y verificable con la
suite existente en verde; el borrado se difiere al Incremento F, cuando ningún módulo referencia ya
la estructura vieja — hasta la Phase 8 `menu/router.py` importa `Presentation`):

- **`063a`** (aditiva) — `down_revision = "e1c455751dbc"` (head verificado con el grafo de
  revisiones: `e1c455751dbc_merge_domicilio_presentations.py`, que ya fusiona `f03274730367`
  —presentaciones, spec 040— con `d427cd419e79` —domicilio—).
- **`063b`** (destructiva) — `down_revision = "<rev 063a>"`.

Patrón en ambas: `@for_each_tenant_schema` + guarda `_has_table(schema, "promotions")` (como
`d3e4f5a6b7c8_promotions.py` y `f03274730367`). Nombres de constraint con `op.f(...)`.

### Revisión `063a` — `upgrade(schema)` (Incremento A)

```text
if not _has_table(schema, "promotions"): return

# --- 1. estructura nueva (compatible hacia atrás) ---
1.  create_table "promotion_variants"      (id, promotion_id FK CASCADE,
                                            product_variant_id FK CASCADE;
                                            UniqueConstraint(promotion_id, product_variant_id);
                                            index(promotion_id), index(product_variant_id))
2.  add_column   "promotions.closed_by_refactor_at"  TIMESTAMP NULL
3.  add_column   "sales.applied_promotions"          JSONB NOT NULL server_default "'[]'"
4.  add_column   "invoices.applied_promotions"       JSONB NOT NULL server_default "'[]'"
5.  add_column   "customer_orders.discount"          NUMERIC(12,2) NOT NULL server_default "0"
    add_column   "customer_orders.applied_promotions" JSONB NOT NULL server_default "'[]'"
    create_check_constraint "ck_customer_order_discount_non_negative"  "discount >= 0"
6.  create_check_constraint "ck_promotion_min_qty"  "min_qty >= 1"
7.  drop_constraint "ck_promotion_type"
    create_check_constraint "ck_promotion_type"
        "type IN ('percent','fixed','combo','qty_price','qty_price_presentation','package_price')"
    # AMPLIADO: se agrega 'package_price'; NO se quita ningún valor viejo todavía.

# --- 2. PASO DE DATOS (con targets/combo/presentation TODAVÍA presentes) ---
8.  # percent -> conjunto de variantes (foto fija, FR-026)
    for promo in SELECT id FROM promotions WHERE type = 'percent':
        variant_ids = (
            SELECT pv.id FROM product_variants pv
            JOIN products p ON p.id = pv.product_id
            WHERE pv.active = true AND (
                pv.product_id IN (SELECT product_id  FROM promotion_targets
                                  WHERE promotion_id = promo AND product_id  IS NOT NULL)
             OR p.category_id  IN (SELECT category_id FROM promotion_targets
                                  WHERE promotion_id = promo AND category_id IS NOT NULL)
             OR NOT EXISTS (SELECT 1 FROM promotion_targets WHERE promotion_id = promo)  # percent global
            )
        )
        INSERT INTO promotion_variants (id, promotion_id, product_variant_id)
        SELECT gen_random_uuid(), promo, vid FROM unnest(variant_ids) vid
        ON CONFLICT (promotion_id, product_variant_id) DO NOTHING
9.  # combo / fixed / qty_price / qty_price_presentation no terminales -> Finalizada (FR-025)
    UPDATE promotions
       SET status = 'finished', closed_by_refactor_at = now()
     WHERE type IN ('combo','fixed','qty_price','qty_price_presentation')
       AND status <> 'finished'
```

### Revisión `063a` — `downgrade(schema)`

```text
7'. drop_constraint "ck_promotion_type"
    create_check_constraint "ck_promotion_type"
        "type IN ('percent','fixed','combo','qty_price','qty_price_presentation')"  # el vigente de spec 040
6'. drop_constraint "ck_promotion_min_qty"
5'. drop_constraint "ck_customer_order_discount_non_negative"
    drop_column "customer_orders.applied_promotions" ; drop_column "customer_orders.discount"
4'. drop_column "invoices.applied_promotions"
3'. drop_column "sales.applied_promotions"
2'. drop_column "promotions.closed_by_refactor_at"
1'. drop_table  "promotion_variants"
#   sin reversa del paso de datos: las promociones 'finished' NO se reactivan; las filas de
#   promotion_variants se pierden. `promotion_targets` etc. siguen intactas (nunca se borraron aquí).
```

### Revisión `063b` — `upgrade(schema)` (Incremento F)

```text
if not _has_table(schema, "promotions"): return

# --- borrado de lo que el modelo nuevo deja sin sentido ---
1.  drop_table "promotion_presentation_rules"     (spec 040)
2.  drop_table "promotion_combo_items"
3.  drop_table "promotion_targets"
4.  drop_index / drop_constraint FK / drop_column  "product_variants.presentation_id"
5.  drop_table "presentations"                     (después de 4: ya sin FK entrante)
6.  drop_column "promotions.priority"
# --- CHECKs ---
7.  drop_constraint "ck_promotion_qty_price_pack"   (constraint muerto: `qty_price` ya no se crea)
8.  drop_constraint  "ck_promotion_type"
    create_check_constraint "ck_promotion_type"
        "type IN ('percent','package_price') OR status = 'finished'"
    # ESTRECHADO con escape: ninguna promoción VIVA (draft/active/paused) puede tener un tipo
    # viejo, pero las que 063a pasó a 'finished' conservan su `type` original (combo/qty_price/
    # fixed/qty_price_presentation) como registro histórico y aparecen en el aviso de FR-025.
```

`PROMOTION_TYPES` del modelo se queda en los 6 valores (sirve para leer las filas `finished`
históricas); solo el `PromotionType` de **entrada** (`PromotionCreate` / `PromotionShapeUpdate`,
T018) es `{PERCENT, PACKAGE_PRICE}`, y el servicio no deja nacer ninguna promoción fuera de eso.
`PromotionResponse.type` es `str` libre.

### Revisión `063b` — `downgrade(schema)` — simétrico, sin pérdida de dato histórico

```text
8'. drop_constraint "ck_promotion_type"
    create_check_constraint "ck_promotion_type"
        "type IN ('percent','fixed','combo','qty_price','qty_price_presentation')"   # el vigente de spec 040
7'. create_check_constraint "ck_promotion_qty_price_pack"  "type <> 'qty_price' OR min_qty >= 2"
6'. add_column  "promotions.priority"  INTEGER NOT NULL server_default "0"
5'. create_table "presentations"                  (estructura de spec 040, vacía)
4'. add_column  "product_variants.presentation_id" UUID NULL
    create_foreign_key ... ondelete="SET NULL";  create_index
3'. create_table "promotion_targets"              (estructura de spec 013, vacía)
2'. create_table "promotion_combo_items"          (estructura de spec 013, vacía)
1'. create_table "promotion_presentation_rules"   (estructura de spec 040, vacía)
```

Notas:
- **`063a` nunca lee ni toca `promotion_targets` para borrarlas** — solo las lee en el paso de
  datos. Su `DROP` es exclusivo de `063b`.
- `gen_random_uuid()` está disponible en PostgreSQL 13+ (el proyecto usa 16); si el patrón del
  repo prefiere `uuid_generate_v4()` o generar en Python, se sigue ese.
- Los `server_default` de las columnas JSONB / `discount` se pueden **retirar tras `063a`**
  (`op.alter_column(..., server_default=None)`) si el patrón del repo lo hace; es opcional y no
  afecta compatibilidad.
- `ck_promotion_percent_range` y `ck_promotion_value_positive` (`promotion.py:114-122`) **no se
  tocan** en ninguna de las dos revisiones.

**Por qué el rollback es seguro (Principio VII/VIII)**:
- Ningún importe de `sales` / `invoices` / `payments` ya emitido se toca en ningún sentido. El
  `discount` y el `total` que se cobraron siguen persistidos; solo se pierde la **lista** de
  promociones del snapshot (`applied_promotions`), que es información añadida por esta spec, no un
  hecho contable — A-29 volvería a su estado previo (sin registrar), que es de dónde parte.
- El paso de datos **no es simétrico y se documenta como tal**: tras revertir **ambas** revisiones
  (`063b` recrea `promotion_targets` **vacía**, `063a` borra `promotion_variants`), las promociones
  `percent` quedan sin alcance hasta que un administrador las reconfigure; las promociones que la
  migración pasó a `finished` **no** se reactivan (y `063b` downgrade tampoco restaura el `type` de
  las que ya eran de tipo viejo — nunca cambió). Esto es aceptable porque (a) un
  `downgrade` de una migración de refactor es una operación de emergencia, no de rutina; (b) la
  configuración de promociones es re-capturable por el administrador (no transaccional, no
  contable); (c) ninguna venta ya emitida cambia. Mismo criterio que la spec 040 §Rollback
  ("asignaciones de catálogo re-capturables, nunca un importe cobrado").
- `Sale.promotion_id` / `*.combo_id` son FK `ON DELETE SET NULL` a `promotions`: recrear
  `promotion_targets` etc. vacías no las rompe.

**Datos de scratch** (`tenant_default`): la guarda `_has_table` evita fallos en schemas sin las
tablas base, igual que las migraciones existentes.

### Prueba de la migración (no cubierta por los characterization tests)

SQLite (los char tests) **no** ejecuta `@for_each_tenant_schema` ni valida el paso de datos. Ambas
revisiones se prueban contra **PostgreSQL real** (`quickstart.md` Paso 1, subpasos 1a y 1b): sobre
un tenant con promociones de los cinco tipos, `alembic upgrade head` / `downgrade -1` / `upgrade
head` para `063a` (Incremento A) y luego para `063b` (Incremento F), verificando el estado y la
forma de cada promoción y que ninguna `Sale`/`Invoice` cambió de importe en ningún punto.
