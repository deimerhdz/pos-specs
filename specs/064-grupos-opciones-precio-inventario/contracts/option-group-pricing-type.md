# Contrato: `pricing_type` en los endpoints de grupo de opciones y opción

Cubre FR-001 a FR-005, FR-015 sobre endpoints que ya existen
(`app/api/v1/catalog/router.py`, `app/api/v1/catalog/schemas.py`).

## `POST /option-groups`

- **Request** — `OptionGroupCreate`, campo nuevo **obligatorio**:
  ```json
  {
    "name": "Sabores",
    "min_select": 1,
    "max_select": 1,
    "pricing_type": "incluido"
  }
  ```
  A diferencia de `Product.tracks_inventory` (spec 027), `pricing_type` **no tiene default en el
  schema** — omitirlo responde `422` (FR-001: el administrador debe elegirlo explícitamente).
- **Response 201** — `OptionGroupResponse`, incluye `"pricing_type": "incluido"` en el body.
- **Efecto**: sin efecto adicional sobre `Option` — un grupo nuevo nace sin opciones.

## `PATCH /option-groups/{group_id}`

- **Request** — `OptionGroupUpdate`, campo nuevo opcional:
  ```json
  { "pricing_type": "incluido" }
  ```
  Mismo patrón que `active`/`min_select`: ausente no toca el valor actual.
- **Response 200** — `OptionGroupResponse` con el valor ya actualizado.
- **Efecto side-effect (FR-004)**: si el valor pasa de `"con_recargo"` a `"incluido"`, el sistema
  ejecuta `UPDATE options SET extra_price = 0 WHERE option_group_id = :group_id` como parte de la
  misma transacción — no rechaza el `PATCH`, lo aplica. Si pasa de `"incluido"` a `"con_recargo"`,
  no hay efecto adicional (los precios ya están en $0 por construcción; el administrador los edita
  después, opción por opción).
- **Sin confirmación en el backend**: pedir confirmación antes de este `PATCH` cuando el grupo
  tiene opciones con precio es responsabilidad del frontend (research.md Decisión 2) — el backend
  no la exige ni la valida, mismo criterio que `product-tracks-inventory-field.md` (spec 027)
  documentó para su propio switch.

## `POST /option-groups/{group_id}/options`

- **Request** — `OptionCreate`, sin cambio de forma. Gana una regla de validación cruzada:
  ```json
  { "name": "Fresa", "extra_price": 500 }
  ```
  Si el `OptionGroup` de `group_id` tiene `pricing_type == "incluido"` y `extra_price != 0` →
  `422 "Los grupos «Incluido» no permiten precio distinto de $0."` (FR-002). Con
  `pricing_type == "con_recargo"`, sin cambio respecto al comportamiento actual (`RN-CAT-04`).
- **Response 201** — `OptionResponse`, sin cambio de forma.

## `PATCH /options/{option_id}`

- **Request** — `OptionUpdate`, sin cambio de forma. Gana la misma regla de validación cruzada que
  `POST .../options` cuando el body trae `extra_price` explícito: se resuelve el `pricing_type` del
  grupo de la opción (`option.option_group_id`), no el que traiga (si trajera) el body — no existe
  un endpoint para mover una opción de grupo, así que el grupo de referencia es siempre el actual.
- **Response 200** — `OptionResponse`, sin cambio de forma.
- **Sin cambio a `RN-CAT-38`**: la regla de "desvincular `inventory_item_id` resetea
  `item_quantity` a `0`" sigue exactamente igual — es independiente de `pricing_type`.

## Sin cambios

- `GET /option-groups`, `GET /variants/{id}/option-groups`: agregan `pricing_type` al body de
  `OptionGroupResponse`, sin cambios de comportamiento adicionales.
- `DELETE /option-groups/{group_id}`, `DELETE /options/{option_id}`: soft-delete sin cambio, sin
  relación con `pricing_type`.
- El guardado unificado de producto (`POST`/`PATCH /products`, spec 043) no cambia de forma — no
  transporta `pricing_type` (es un atributo del `OptionGroup`, no de la asignación
  variante↔grupo). Un producto que asigna un grupo "Incluido" o "Con recargo" a una de sus
  variantes no necesita declarar nada adicional; el precio de cada opción ya viene resuelto desde
  el catálogo compartido.
