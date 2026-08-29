# Contrato: `POST /orders` — canal estandarizado + tipo de orden

Cubre FR-001, FR-003, FR-006, FR-007, FR-009, FR-011. Endpoint **existente**
(`app/api/v1/orders/router.py`, `orders.service.create_order`) — cambia el catálogo de valores
aceptados en `channel`, y se agrega `order_type`.

## `POST /orders`

- **Request** (`OrderCreate`, campos relevantes; el resto — `participant_id`, `customer_name`,
  `notes`, `items`, `hold_for_payment` — sin cambio de forma):
  ```json
  {
    "channel": "POS",
    "order_type": "TAKEAWAY",
    "dining_table_id": null,
    "customer_name": "Consumidor final",
    "items": [ { "product_variant_id": "...", "quantity": 1 } ],
    "hold_for_payment": true
  }
  ```
  - `channel`: uno de `"POS" | "QR_MENU" | "WHATSAPP" | "API"` (antes `"qr" | "counter" |
    "waiter"`). Default si se omite: `"POS"` (antes `"counter"`).
  - `order_type` (**nuevo**): uno de `"DINE_IN" | "TAKEAWAY" | "DELIVERY"`. Default si se omite:
    `"DINE_IN"`.

- **Response 201** (`OrderResponse`, campos relevantes):
  ```json
  { "id": "...", "channel": "POS", "order_type": "TAKEAWAY", "dining_table_id": null, "...": "..." }
  ```
  `order_type` nuevo, siempre presente en órdenes creadas desde esta versión (puede ser `null` solo
  en órdenes de antes de esta mejora sin mesa — ver data-model.md, backfill).

- **Validaciones y errores nuevos**:
  - `400` si la combinación `channel`/`order_type` no está permitida (data-model.md, tabla de
    combinaciones) — p. ej. `{"channel": "WHATSAPP", "order_type": "DINE_IN"}`. Mensaje: indica
    ambos valores recibidos y que esa combinación no es válida.
  - `422` si `order_type` es `TAKEAWAY` o `DELIVERY` y `dining_table_id` no es `null` — un pedido
    para llevar/domicilio no puede llevar mesa asociada.
  - Validaciones ya existentes sin cambio de forma: `400` si `hold_for_payment` + `channel ==
    QR_MENU` (antes `QR`); `404` mesa/participante inexistente; `409` conflicto QR↔staff en la
    misma sesión de mesa (solo aplica cuando `dining_table_id` no es `null`, así que nunca afecta a
    `TAKEAWAY`/`DELIVERY`).

- **Ejemplo — pedido "Para Llevar" válido** (el que arma `manual-order-page.component.ts` al
  confirmar con la pestaña "Para Llevar"):
  ```json
  {
    "channel": "POS",
    "order_type": "TAKEAWAY",
    "dining_table_id": null,
    "customer_name": "Consumidor final",
    "items": [ { "product_variant_id": "9c1e...", "quantity": 2 } ],
    "hold_for_payment": true
  }
  ```
  → `201`, orden creada sin mesa, `status: "recibida"` (igual que hoy con `hold_for_payment: true`).

- **Ejemplo — combinación rechazada**:
  ```json
  { "channel": "WHATSAPP", "order_type": "DINE_IN", "items": [ ... ] }
  ```
  → `400 Bad Request`, sin crear ningún registro.

## Sin cambios de forma en otros endpoints

- `GET /orders`, `GET /orders/{id}`: `OrderResponse` gana el campo `order_type` (mismo tipo de
  cambio aditivo que tuvieron `paid`/`current_payment_attempt` en specs 024/029) — cualquier
  cliente que ignore campos desconocidos no nota ruptura.
- `POST /cart/submit` (flujo QR, `cart/service.py::submit_cart`): sin cambio de forma — sigue sin
  aceptar `channel` ni `order_type` del comensal; internamente sigue creando siempre
  `channel="QR_MENU"` (antes `"qr"`) + `order_type="DINE_IN"`, fijos.
- `POST /tables/{id}/consolidate`, `POST /tables/{id}/items` (consolidación/mesero): sin cambio de
  forma — internamente crean siempre `channel="POS"` + `order_type="DINE_IN"` +
  `is_consolidation_order=true` (campo interno, no expuesto), fijos.
