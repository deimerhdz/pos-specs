# Contrato: administración de promociones (CRUD multi-regla, validaciones, estados)

> **Reemplaza** la versión de este contrato escrita para el modelo plano — la que describe
> exactamente lo que ya está implementado en `app/api/v1/promotions/{schemas,service,router}.py`
> de la rama de feature `refactor/063-promociones-por-variante` (verificado 2026-09-01:
> `PromotionCreate` en `schemas.py:104`, `_guard_variant_overlap` en `service.py:338`,
> `_guard_package_is_discount` en `service.py:383`). Este documento describe el payload y las
> validaciones **con el nivel de regla añadido** (FR-001, FR-001a). Solo el **administrador del
> tenant** gestiona promociones (`require_tenant_admin`, sin cambio); el cajero solo visualiza
> (FR-019, sin cambio).

---

## 1. Modelo de payload

### `PromotionRuleIn` (nuevo — antes estos campos vivían directo en `PromotionCreate`)

| Campo | Tipo | Reglas |
|---|---|---|
| `type` | `"percent"` \| `"package_price"` | obligatorio; ningún otro valor (FR-002) |
| `value` | Decimal | `percent`: `0 < value <= 100`; `package_price`: `value > 0` |
| `min_qty` | int | `>= 1` |
| `variant_ids` | `list[UUID]` | **≥ 1** (FR-001a); sin repetidos dentro de la misma regla; cada uuid debe existir y ser de una variante del tenant |

### `PromotionCreate` (`POST /promotions`)

| Campo | Tipo | Reglas |
|---|---|---|
| `name` | str | 1–255, único (409 legible) — **sin cambio**, es de la promoción |
| `description` | str \| null | ≤ 2000 — sin cambio |
| `status` | `"draft"` \| `"active"` | default `"draft"` — sin cambio |
| `starts_at` | datetime | **obligatoria** (FR-012) — sin cambio |
| `ends_at` | datetime \| null | ≥ `starts_at` si viene — sin cambio |
| `days_of_week` | str CSV `0..6` \| null | sin cambio |
| `start_time` / `end_time` | time \| null | sin cambio |
| `rules` | `list[PromotionRuleIn]` | **NUEVO, reemplaza `type`/`value`/`min_qty`/`variant_ids` a nivel de promoción — ≥ 1 regla (FR-001)** |

**Validación cruzada nueva en el payload** (antes de tocar la base de datos): ninguna
`variant_ids` de una regla puede intersectar la `variant_ids` de otra regla del mismo `rules` —
ver §2, `_guard_variant_overlap` caso FR-001a.

**Se eliminan del payload** (sin cambio respecto del modelo plano, ya retirado ahí):
`priority`, `targets`, `combo_items`, `presentation_rules`, `confirm_precio_no_uniforme`,
`confirm_sin_descuento`.

### `PromotionShapeUpdate` (`PATCH /promotions/{id}/shape`, **solo en `draft`**)

`rules?: list[PromotionRuleIn]` — **reemplaza la lista completa de reglas** de la promoción (no
hay endpoint para editar una sola regla suelta: en `Borrador` todo el bloque de reglas es
editable de una vez, FR-018). Antes: `type?`, `variant_ids?` sueltos.

### `PromotionUpdate` (`PATCH /promotions/{id}`, escalares de la promoción)

**Sin cambio de forma**: `name?`, `description?`, `ends_at?`, `days_of_week?`, `start_time?`,
`end_time?`. Nunca tuvo `type`/`value`/`min_qty` en el modelo plano tampoco (FR-018 ya los excluía
de este endpoint) — la diferencia ahora es que esos campos ni siquiera existen en `Promotion`,
viven en `PromotionRule`, así que no hay nada que rechazar aquí: simplemente no están en el
schema.

---

## 2. Validaciones de negocio (servicio)

Todas en `promotions/service.py`:

| Validación | Dónde | Respuesta | FR |
|---|---|---|---|
| `rules` no vacía | `create`, `update_shape`, `change_status→active` | 422 "Una promoción necesita al menos una regla" | FR-001 |
| `variant_ids` de cada regla no vacía | ídem | 422 "Cada regla necesita al menos una variante" | FR-001a |
| `percent` → `0 < value <= 100`; `package_price` → `value > 0`, `min_qty >= 1` | Pydantic + `ck_promotion_rule_*` | 422 | FR-002 |
| **FR-001a**: ninguna variante se repite entre dos reglas de la **misma** promoción | `_guard_variant_overlap`, primer chequeo (antes de comparar contra otras promociones) | **409** `{error: "regla_variante_duplicada", rule_index_a, rule_index_b, variant_ids: [...]}` | FR-001a |
| **FR-016**: por regla de tipo `package_price`, `value >= min_qty × (menor precio entre las variantes de esa regla)` | `_guard_package_is_discount`, una vez por regla | **409** `{error, rule_index, value, min_qty, cheapest_unit_price, variant_id}` | FR-016, SC-002 |
| **FR-014**: una regla comparte ≥1 variante con una regla de **otra** promoción no terminal **y** las ventanas de sus respectivas promociones se intersectan | `_guard_variant_overlap`, segundo chequeo (sin cambio de criterio respecto del modelo plano, ahora comparando conjuntos de reglas en vez de conjuntos de promociones) | **409** `{error: "solape", conflicts: [{promotion_id, promotion_name, rule_id, variant_ids: [...]}]}` | FR-014, FR-014a, SC-003 |
| **FR-018**: en `active`/`paused`, ningún campo de ninguna regla editable, ni agregar/quitar reglas | `update` rechaza el payload si trae `rules`; `update_shape` exige `draft` | 409 "Solo una promoción en borrador puede cambiar sus reglas" | FR-018 |
| Máquina de estados | `change_status` + `PROMOTION_TRANSITIONS` (sin cambio) | 409 "Transición no permitida: {from} -> {to}" | FR-015 |
| Solo admin del tenant | `require_tenant_admin` en el router (sin cambio) | 403 | FR-019 |

### `_guard_variant_overlap` — dos chequeos, en este orden

**Chequeo 1 — intra-promoción (FR-001a, nuevo)**: para las reglas del payload que se está
guardando (`rules` de `create`/`update_shape`), cualquier par `(rule_a, rule_b)` con
`variant_ids_a ∩ variant_ids_b ≠ ∅` bloquea, **sin mirar fecha/día/hora** — las reglas de una misma
promoción comparten vigencia por definición (viven en la misma fila de `promotions`), así que la
intersección temporal es siempre 100% y no hace falta calcularla.

**Chequeo 2 — inter-promoción (FR-014/FR-014a, mismo criterio que el modelo plano)**: para cada
regla del payload contra cada regla `c` de otra promoción `p_c` en estado `draft`/`active`/
`paused` (`p_c.id != promotion.id`):

```
comparten_variante = variant_ids ∩ {pv.product_variant_id for pv in c.variants} ≠ ∅
fecha:   NOT (promotion.starts_at.date() > (p_c.ends_at.date() if p_c.ends_at else +∞)
             OR p_c.starts_at.date() > (promotion.ends_at.date() if promotion.ends_at else +∞))
días:    (not promotion.days_of_week) OR (not p_c.days_of_week)
         OR ({d for d in promotion.days_of_week.split(',')} ∩ {d for d in p_c.days_of_week.split(',')} ≠ ∅)
horas:   (promotion.start_time is None) OR (p_c.start_time is None)
         OR _in_time_window(promotion.start_time, p_c.start_time, p_c.end_time)
         OR _in_time_window(p_c.start_time, promotion.start_time, promotion.end_time)
bloquea  = comparten_variante AND fecha AND días AND horas
```

Idéntico al criterio que ya tenía el modelo plano, comparando `promotion`/`p_c` en vez de
`promo`/`c` directamente — la única diferencia es de dónde sale `variant_ids` (de una regla, no de
la promoción completa) y que el mensaje de conflicto ahora nombra también `rule_id`.

---

## 3. Respuestas

### `PromotionResponse`

```
id, name, description, status,
starts_at, ends_at, days_of_week, start_time, end_time,
closed_by_refactor_at,                    # sin cambio — marca de "finalizada por la migración" (FR-025)
rules: [                                  # NUEVO — reemplaza type/value/min_qty/variants a nivel de promoción
  {
    id, type, value, min_qty,
    condition_text,                       # variant_set_condition_text(rule) (FR-005)
    variants: [
      { product_variant_id, description, unit_price }
    ]
  }
]
```

**Compatibilidad con promociones ya `finished` del modelo plano** (migradas por `063a`, o cerradas
por `063c` si llegaron a existir promociones `finished` sin ninguna fila resultante en
`promotion_rules` — no debería ocurrir porque el paso de datos de `063c` crea una regla por cada
fila de `promotions` sin condicionar por `status`, así que toda promoción, incluida una `finished`,
termina con exactamente una regla que conserva su `type`/`value`/`min_qty` histórico): se serializan
igual, con `rules` de longitud 1.

**Se eliminan** (respecto del modelo plano): `type`, `value`, `min_qty`, `condition_text` y
`variants` a nivel de promoción — se mueven dentro de cada elemento de `rules`.

### Endpoints (`/promotions`)

| Verbo | Ruta | Cambio respecto del modelo plano |
|---|---|---|
| GET | `/promotions` | Sin cambio de query params (`closed_by_refactor`, `X-Server-Time`). `PromotionResponse` ahora anida `rules`. |
| POST | `/promotions` | Payload con `rules: list[PromotionRuleIn]`; 409 de FR-001a (nuevo) / FR-014 / FR-016 (por regla); 409 de nombre duplicado sin cambio. |
| PATCH | `/promotions/{id}` | Sin cambio de forma — solo campos de vigencia/nombre. |
| PATCH | `/promotions/{id}/shape` | `rules` completo reemplaza a `type`/`variant_ids`; solo `draft`; revalida FR-001a / FR-014 / FR-016 sobre el `rules` nuevo completo. |
| PATCH | `/promotions/{id}/status` | Máquina de estados sin cambio; `→ active` revalida FR-001a / FR-014 / FR-016 sobre **todas** las reglas actuales de la promoción. |
| POST | `/promotions/{id}/duplicate` | Copia en `draft` con **todas** las reglas de la promoción (tipo/valor/`min_qty`/conjunto de cada una) + misma vigencia (FR-017). |
| DELETE | `/promotions/{id}` | Sin cambio (204; cascada borra `promotion_rules` y sus `promotion_variants`). |

`record_audit` sigue registrando `create`/`update`/`update_shape`/`status`/`duplicate`/`delete` —
sin cambio de granularidad (a nivel de promoción, no de regla individual).

---

## 4. Selección del conjunto (FR-004) — solo backend-relevante

**Sin cambio de principio, ahora por regla**: el backend recibe y guarda `variant_ids` **dentro de
cada regla**. Los filtros "por producto", "por categoría" y "por texto" siguen siendo
exclusivamente del frontend para poblar el selector de **una** regla a la vez
(`contracts/superficies-consumo.md` §3). El backend no guarda ningún filtro y no re-resuelve el
conjunto de ninguna regla: una variante creada después en una categoría usada como filtro no entra
sola en ninguna regla existente (FR-003, US1-CA3).
