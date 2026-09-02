# Contrato: `options` con cantidad en carrito, pedidos y ventas

Cubre FR-002 a FR-004, FR-006, FR-007, FR-011 sobre los cuatro puntos de entrada reales que hoy
aceptan una lista de opciones elegidas (research.md, Decisión 1). **Cambia el shape** del campo —
a diferencia de la mayoría de contratos de specs anteriores, aquí el nombre y la forma del campo
cambian, no solo se agrega uno nuevo al lado.

## Forma nueva compartida: `OptionSelectionIn`

```json
{ "option_id": "3fa8...", "quantity": 2 }
```

`quantity` es opcional en el request, default `1` — el mismo significado que "incluir este id en
`option_ids`" tenía antes de esta spec. Reemplaza `option_ids: list[UUID]` por
`options: list[OptionSelectionIn]` en los cuatro schemas siguientes.

## `POST /cart/items` y `PATCH /cart/items/{id}`

- **Request** — `CartItemIn`/`CartItemUpdate` (`app/api/v1/cart/schemas.py:40-49`):
  ```json
  {
    "product_variant_id": "…",
    "quantity": 1,
    "options": [
      { "option_id": "bobombun-id", "quantity": 2 },
      { "option_id": "gomitas-id", "quantity": 1 }
    ]
  }
  ```
  `option_id` repetido dentro del mismo `options[]` → `422` ("opción repetida en la selección") —
  nunca se fusionan silenciosamente dos entradas del mismo id (research.md Decisión 1/2).
- **Response 200/201** — `CartItemResponse.options` (antes `CartItemOption[]` con solo
  `id`/`option_id`) gana `quantity`:
  ```json
  { "id": "…", "option_id": "bobombun-id", "quantity": 2 }
  ```
- **Efecto**: `unit_price` sigue siendo un snapshot (`compute_line_price`), ahora calculado sobre
  `ChosenOption` — `extra_price × quantity` por opción (research.md Decisión 2). Para una opción de
  un grupo "conteo" con `quantity > 1` en el request → `422` ("esta opción no admite más de una
  unidad") antes de calcular ningún precio (research.md Decisión 3).

## `POST /orders` (comanda de mesero) y el endpoint equivalente de venta de mostrador

- **Request** — la línea de ítem de ambos endpoints gana el mismo cambio: `option_ids: list[UUID]`
  → `options: list[OptionSelectionIn]`. Mismo criterio de rechazo por id repetido y por `quantity >
  1` en grupo "conteo" que el carrito.
- **Response** — el detalle de la orden/venta creada refleja `quantity` por opción igual que el
  carrito.
- **Sin cambio en el momento de aplicar la venta**: `POST /cart/submit` sigue copiando el snapshot
  ya calculado (`unit_price`, y ahora también `quantity` por opción) de `CartItem`/`CartItemOption`
  a `OrderItem`/`OrderItemOption` sin recalcular nada (`app/api/v1/cart/service.py:591-610`) — ese
  copiado se extiende para incluir la columna `quantity` nueva, sin tocar su lógica de "no
  recomputar".

## Confirmación de pedido / checkout (consumo de inventario)

- **Sin cambio de forma**: `_confirm_order_impl`/`checkout_and_send`
  (`app/api/v1/orders/checkout.py`) ya cargan las opciones elegidas desde las filas persistidas
  (`_item_options`), no desde el request — ahora esas filas traen `quantity`, así que
  `plan_line_consumption` recibe `ChosenOption` con la cantidad correcta sin que este endpoint
  cambie su propia lógica, solo el tipo de dato que fluye a través de él.

## Registro de venta y pago

- **`SaleItem.options`** (JSONB): cada dict gana `"quantity"` (ver data-model.md). `build_sale`
  (`app/api/v1/sales/builder.py`) no cambia — sigue sin inspeccionar el contenido de `options`, solo
  lo persiste tal cual junto con `unit_price`/`line_total` ya calculados (FR-011: el detalle de la
  venta conserva la cantidad, pero el cálculo de `Sale`/`Payment` en sí no cambia — research.md,
  hallazgo de la investigación previa a esta spec).

## Sin cambios

- El chequeo preventivo de disponibilidad (`check_availability`, `RN-CAT-24`) — recibe el
  diccionario `required` ya agregado por `plan_line_consumption`, sin cambio de forma propio.
- `Sale`, `Payment`, `Invoice` (modelos) — ningún campo ni cálculo nuevo; absorben el precio de
  línea ya resuelto igual que hoy (FR-011, research.md).
