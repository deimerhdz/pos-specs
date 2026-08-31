# Contrato: administración de promociones (CRUD, validaciones, estados)

Reemplaza el contrato de administración de la spec 013 y la parte de forma de la spec 040. Solo el
**administrador del tenant** gestiona promociones (`require_tenant_admin`, ya vigente); el cajero
solo visualiza (FR-019). Ver [research.md](../research.md) D2/D3/D4/D11/D16 y
[data-model.md](../data-model.md) §"Reglas de validación".

---

## 1. Modelo de payload

### `PromotionCreate` (`POST /promotions`)

| Campo | Tipo | Reglas |
|---|---|---|
| `name` | str | 1–255, único (409 legible) |
| `description` | str \| null | ≤ 2000 |
| `type` | `"percent"` \| `"package_price"` | obligatorio; ningún otro valor |
| `value` | Decimal | `percent`: `0 < value <= 100`; `package_price`: `value > 0` |
| `min_qty` | int | `>= 1` (`percent` suele ser 1; `package_price` = tamaño del grupo) |
| `status` | `"draft"` \| `"active"` | default `"draft"`; `"finished"` prohibido al crear |
| `starts_at` | datetime | **obligatoria** (FR-012) |
| `ends_at` | datetime \| null | ≥ `starts_at` si viene |
| `days_of_week` | str CSV `0..6` \| null | conjunto; null/vacío = todos los días |
| `start_time` / `end_time` | time \| null | ambas o ninguna; admite cruce de medianoche |
| `variant_ids` | `list[UUID]` | **≥ 1** (FR-001); sin repetidos; cada uuid debe existir y ser de una variante del tenant |

**Se eliminan del payload**: `priority`, `targets`, `combo_items`, `presentation_rules`,
`confirm_precio_no_uniforme`, `confirm_sin_descuento` (research.md D3/D15, A-58/A-65).

### `PromotionShapeUpdate` (`PATCH /promotions/{id}/shape`, **solo en `draft`**)

`type?`, `variant_ids?`. (Sin `targets`/`combo_items`/`presentation_rules`.)

### `PromotionUpdate` (`PATCH /promotions/{id}`, escalares)

`name?`, `description?`, `ends_at?`, `days_of_week?`, `start_time?`, `end_time?`.
`value?` y `min_qty?` se aceptan en el schema pero el **servicio los rechaza (422)** si
`status != "draft"` (FR-018). `starts_at` no editable tras crear.

---

## 2. Validaciones de negocio (servicio)

Todas en `promotions/service.py`, invocadas donde se indica:

| Validación | Dónde | Respuesta | FR |
|---|---|---|---|
| `variant_ids` no vacía | `create`, `update_shape`, `change_status→active` | 422 "Una promoción necesita al menos una variante" | FR-001, US1-CA3 |
| `percent` → `0 < value <= 100` | Pydantic + `ck_promotion_percent_range` | 422 | FR-002, US1-CA4 |
| `package_price` → `value > 0`, `min_qty >= 1` | servicio + `ck_promotion_min_qty` | 422 | FR-002 |
| **FR-016**: `package_price` y `value >= min_qty × (menor price entre las variantes del conjunto)` | `_guard_package_is_discount` en `create` / `update_shape` / `change_status→active` | **409** `{error, value, min_qty, cheapest_unit_price, variant_id}` | FR-016, SC-002 |
| **FR-014**: conjunto comparte ≥1 variante con otra promo no terminal **y** fecha ∧ días ∧ horas se intersectan | `_guard_variant_overlap` en `create` / `update_shape` / `change_status→active` | **409** `{error, conflicts: [{promotion_id, promotion_name, variant_ids: [...]}]}` | FR-014, FR-014a, SC-003 |
| **FR-018**: en `active`/`paused`, `value`/`min_qty` no editables | `update` (contra `promo.status` real) | 422 "Duplica la promoción para cambiar el valor o la cantidad" | FR-018, US5-CA2 |
| **FR-018**: `type`/`variant_ids` solo en `draft` | `update_shape` (ya exige `draft`) | 409 "Solo una promoción en borrador puede cambiar de tipo o de conjunto" | FR-018 |
| Máquina de estados | `change_status` + `PROMOTION_TRANSITIONS` (sin cambio) | 409 "Transición no permitida: {from} -> {to}" | FR-015, US5-CA3 |
| Solo admin del tenant | `require_tenant_admin` en el router | 403 | FR-019, US5-CA5 |

### `_guard_variant_overlap` — detalle de la intersección (FR-014a)

Para `promo` (con `variant_ids`) contra cada candidata `c` en
`status IN ('draft','active','paused')`, `c.id != promo.id`:

```
comparten_variante = variant_ids ∩ {pv.product_variant_id for pv in c.variants}  ≠ ∅
fecha:   NOT (promo.starts_at.date() > (c.ends_at.date() if c.ends_at else +∞)
             OR c.starts_at.date() > (promo.ends_at.date() if promo.ends_at else +∞))
días:    (not promo.days_of_week) OR (not c.days_of_week)
         OR ({d for d in promo.days_of_week.split(',')} ∩ {d for d in c.days_of_week.split(',')} ≠ ∅)
horas:   (promo.start_time is None) OR (c.start_time is None)
         OR _in_time_window(promo.start_time, c.start_time, c.end_time)
         OR _in_time_window(c.start_time, promo.start_time, promo.end_time)
bloquea  = comparten_variante AND fecha AND días AND horas
```

Se reutilizan `_in_time_window` (`service.py:78`), y la lógica de `_ranges_overlap` /
`_csv_overlap` / `_times_overlap` (`service.py:708-727`) **acotada a intersección simultánea de
las tres dimensiones** (hoy `find_overlaps` exige las tres pero devuelve advertencia; el cambio
es que ahora bloquea y compara por variante compartida en vez de target — A-59). `find_overlaps`,
`_scope_overlap`, `OverlapResponse`, `PromotionWithOverlaps` y el campo `overlaps` de las
respuestas **se eliminan**.

---

## 3. Respuestas

### `PromotionResponse`

```
id, name, description, type, value, status, min_qty,
starts_at, ends_at, days_of_week, start_time, end_time,
closed_by_refactor_at,                    # nuevo — marca de "finalizada por la migración" (FR-025)
condition_text,                           # nuevo — variant_set_condition_text(promo) (FR-005)
variants: [                               # nuevo — reemplaza targets/combo_items/presentation_rules
  { product_variant_id, description,      #   "Producto - Variante"
    unit_price }                          #   precio normal vigente (FR-005)
]
```

**`type` en la respuesta es `str` libre, no el enum de entrada**: las promociones que la migración
`063a` pasó a `finished` conservan su `type` original (`combo` / `qty_price` /
`qty_price_presentation` / `fixed`, FR-025) y el listado —incluido el aviso
`?closed_by_refactor=true`— debe poder serializarlas. Solo `PromotionCreate` /
`PromotionShapeUpdate` restringen la entrada a `{percent, package_price}`. Para una promoción
`finished` de tipo viejo, `condition_text` puede ser `null` y `variants` vacío.

**Se eliminan**: `priority`, `targets`, `combo_items`, `presentation_rules`, `overlaps`.

### Endpoints (`/promotions`)

| Verbo | Ruta | Cambio |
|---|---|---|
| GET | `/promotions` | `list_query` ordena por `name` (ya no `priority`). Nuevo query param `closed_by_refactor: bool` → filtra `closed_by_refactor_at IS NOT NULL` (aviso de FR-025). `X-Server-Time` intacto (A-09). |
| POST | `/promotions` | payload nuevo; 409 de FR-014 / FR-016 / nombre duplicado. Respuesta `PromotionResponse` (sin `overlaps`). |
| PATCH | `/promotions/{id}` | escalares; 422 de FR-018. |
| PATCH | `/promotions/{id}/shape` | `type` / `variant_ids`; solo `draft`; revalida FR-014 / FR-016. |
| PATCH | `/promotions/{id}/status` | máquina de estados; `→ active` revalida FR-014 / FR-016 / conjunto no vacío. |
| POST | `/promotions/{id}/duplicate` | copia en `draft` con mismo tipo/valor/`min_qty`/conjunto/vigencia (FR-017). |
| DELETE | `/promotions/{id}` | sin cambio (204). |

`record_audit` sigue registrando `create`/`update`/`update_shape`/`status`/`duplicate`/`delete`.
El "historial de modificaciones" (A-42) **no se construye ni se expone** (clarification
2026-08-31).

---

## 4. Selección del conjunto (FR-004) — solo backend-relevante

El backend recibe y guarda **`variant_ids`** (lista concreta). Los filtros "por producto", "por
categoría" y "por texto de presentación/nombre" son **exclusivamente del frontend** para poblar el
selector (`contracts/superficies-consumo.md` §3). El backend **no** guarda ningún filtro y **no**
re-resuelve el conjunto: una variante creada después en una categoría que se usó como filtro
**no** entra en la promoción (FR-003, US1-CA2). Para ayudar al selector, el frontend puede usar
los endpoints de catálogo ya existentes (`GET /catalog/...`, `GET /products`) — no se agrega
ninguno.
