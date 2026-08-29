# Contrato: `POST /orders` — campos de domicilio

Cubre FR-001 a FR-010. Endpoint **existente** (`app/api/v1/orders/router.py`,
`orders.service.create_order`) — agrega tres campos nuevos, opcionales según `order_type`.

## `POST /orders`

- **Request** (`OrderCreate`, campos relevantes; `channel`, `participant_id`, `items`,
  `hold_for_payment` sin cambio de forma):
  ```json
  {
    "channel": "POS",
    "order_type": "DELIVERY",
    "dining_table_id": null,
    "customer_name": "Ana Torres",
    "delivery_address": "Cra 45 #12-30, apto 301",
    "delivery_phone": "3011234567",
    "delivery_fee": 6000,
    "items": [ { "product_variant_id": "...", "quantity": 1 } ],
    "hold_for_payment": true
  }
  ```
  - `delivery_address` (**nuevo**): texto libre, `str | None`. Obligatorio (no vacío tras
    `.strip()`) cuando `order_type == "DELIVERY"`; ignorado para cualquier otro `order_type`.
  - `delivery_phone` (**nuevo**): texto libre, `str | None`. Siempre opcional, incluso con
    `order_type == "DELIVERY"`.
  - `delivery_fee` (**nuevo**): número no negativo, `Decimal | None`. Obligatorio (no `None`)
    cuando `order_type == "DELIVERY"`; ignorado para cualquier otro `order_type`.

- **Response 201** (`OrderResponse`, campos relevantes):
  ```json
  {
    "id": "...",
    "channel": "POS",
    "order_type": "DELIVERY",
    "dining_table_id": null,
    "customer_name": "Ana Torres",
    "delivery_address": "Cra 45 #12-30, apto 301",
    "delivery_phone": "3011234567",
    "delivery_fee": 6000.00,
    "...": "..."
  }
  ```
  Los tres campos nuevos siempre presentes en la respuesta (`null` para cualquier orden que no sea
  `DELIVERY`, nueva o histórica).

- **Validaciones y errores nuevos**:
  - `422` si `order_type == "DELIVERY"` y falta `customer_name`, `delivery_address`, o
    `delivery_fee` (data-model.md, "Reglas de validación"). Mensaje: indica que un pedido a
    domicilio requiere nombre del cliente, dirección y valor del domicilio.
  - `422` si `delivery_fee` es negativo (reforzado además por `CheckConstraint` en base de datos —
    defensa en profundidad).
  - Validaciones ya existentes sin cambio de forma (spec 055): combinación canal×tipo de orden;
    `422` si `order_type` es `TAKEAWAY`/`DELIVERY` y `dining_table_id` no es `null` — ya cubre
    "Domicilio", sin cambios.

- **Ejemplo — pedido "Domicilio" válido** (el que arma `manual-order-page.component.ts` al
  confirmar con la pestaña "Domicilio"):
  ```json
  {
    "channel": "POS",
    "order_type": "DELIVERY",
    "dining_table_id": null,
    "customer_name": "Ana Torres",
    "delivery_address": "Cra 45 #12-30, apto 301",
    "delivery_phone": null,
    "delivery_fee": 6000,
    "items": [ { "product_variant_id": "9c1e...", "quantity": 2 } ],
    "hold_for_payment": true
  }
  ```
  → `201`, orden creada sin mesa, `status: "recibida"`, teléfono `null` (opcional, no
  diligenciado).

- **Ejemplo — rechazo por campos obligatorios faltantes**:
  ```json
  {
    "channel": "POS", "order_type": "DELIVERY",
    "customer_name": "Ana Torres",
    "items": [ { "product_variant_id": "9c1e...", "quantity": 1 } ]
  }
  ```
  (sin `delivery_address` ni `delivery_fee`) → `422 Unprocessable Entity`, sin crear ningún
  registro.

## Sin cambios de forma en otros endpoints

- `GET /orders`, `GET /orders/{id}`: `OrderResponse` gana los tres campos nuevos (cambio aditivo,
  mismo criterio que `order_type` en spec 055) — cualquier cliente que ignore campos desconocidos
  no nota ruptura.
- `POST /cart/submit` (flujo QR), `POST /tables/{id}/consolidate`, `POST /tables/{id}/items`
  (consolidación/mesero): sin cambio de forma — ninguno de estos caminos construye órdenes
  `DELIVERY`, siguen sin conocer estos tres campos.
