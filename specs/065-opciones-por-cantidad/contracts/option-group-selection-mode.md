# Contrato: `selection_mode` y topes de cantidad en los endpoints de grupo de opciones

Cubre FR-001, FR-008, FR-009 sobre endpoints que ya existen
(`app/api/v1/catalog/router.py`, `app/api/v1/catalog/schemas.py`).

## `POST /option-groups`

- **Request** — `OptionGroupCreate`, campos nuevos:
  ```json
  {
    "name": "Toppings",
    "min_select": 0,
    "max_select": 1,
    "pricing_type": "con_recargo",
    "selection_mode": "cantidad",
    "max_quantity_per_option": 3,
    "max_total_quantity": 5
  }
  ```
  `selection_mode` es **opcional, con default `"conteo"`** (a diferencia de `pricing_type`, que no
  tiene default) — omitirlo crea un grupo en modo "conteo", idéntico al comportamiento anterior a
  esta spec (FR-001). `max_quantity_per_option`/`max_total_quantity` son opcionales y solo tienen
  efecto cuando `selection_mode="cantidad"`; enviarlos con `selection_mode="conteo"` no es un error,
  pero no producen ningún efecto (se guardan, no se aplican).
- **Response 201** — `OptionGroupResponse`, incluye los tres campos nuevos.

## `PATCH /option-groups/{group_id}`

- **Request** — `OptionGroupUpdate`, mismos tres campos, todos opcionales (patrón ya usado por
  `pricing_type`/`active`): ausente no toca el valor actual.
  ```json
  { "selection_mode": "cantidad", "max_quantity_per_option": 3, "max_total_quantity": 5 }
  ```
- **Response 200** — `OptionGroupResponse` con los valores ya actualizados.
- **Sin efecto lateral al cambiar de modo**: a diferencia de `pricing_type` (que fuerza `extra_price
  = 0` al pasar a "incluido", spec 063), cambiar `selection_mode` no modifica ninguna `Option` del
  grupo — el precio y el insumo de cada opción son independientes del modo (FR-005). El único efecto
  es que `min_select`/`max_select` dejan de exigirse (modo "cantidad") o los topes dejan de exigirse
  (modo "conteo"), evaluado en el momento de vender, no al guardar el grupo.

## Sin cambios

- `GET /option-groups`, `GET /variants/{id}/option-groups`: agregan los tres campos nuevos al body
  de `OptionGroupResponse`, sin cambios de comportamiento adicionales.
- `POST /option-groups/{group_id}/options`, `PATCH /options/{option_id}`: sin cambio de forma —
  `extra_price`/`inventory_item_id`/`item_quantity` de una opción no cambian de significado; solo
  cambia cómo se multiplican al momento de vender (ver
  [cart-order-option-quantity.md](./cart-order-option-quantity.md)).
