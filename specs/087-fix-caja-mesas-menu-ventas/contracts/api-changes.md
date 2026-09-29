# Contratos: cambios de API — spec 087

Solo se documentan los endpoints **nuevos o cuyo contrato cambia**. El resto de la API de
`pos-backend` no se toca. Base: `app/api/v1/`.

## 1. Caja — Arqueo Parcial (eliminado)

### `POST /cash/shifts/{shift_id}/partial-count` — **eliminado**

Se retira por completo (router, servicio, schemas de request/response). Cualquier llamada
posterior a este path debe devolver `404 Not Found` (ruta inexistente), no un `410 Gone`
especial — no hace falta mantener compatibilidad, el único consumidor es `pos-heladeria` del
mismo repo/spec.

No existía un endpoint `GET` de historial de arqueos parciales — no hay nada que retirar ahí.

## 2. Caja — Cierre de turno (sin cambio de contrato)

`GET /cash/shifts/{shift_id}/report` no cambia su schema de respuesta. El cambio de esta
historia es puramente de presentación en el frontend (impresión aislada del shell) — no
requiere tocar `ShiftReportResponse`.

## 3. Órdenes — crear pedido (`customer_name` pasa a requerido)

### `POST /orders`

**Antes**: `customer_name` opcional en el request body.

**Después**: `customer_name` requerido (string no vacío) cuando el pedido se crea para
`order_type` en `{DINE_IN, TAKEAWAY, DELIVERY}`. Si viene vacío/ausente, la API responde
`422 Unprocessable Entity` con un mensaje explícito ("El nombre del cliente es obligatorio").

Aplica al mismo cambio a cualquier otro endpoint de creación de pedido de mesa/para
llevar/domicilio que se identifique durante la implementación (ver D7 en `research.md`).

**Respuesta** (`OrderResponse` o equivalente): expone `table_order_number` (nullable,
presente solo para pedidos `DINE_IN`) además de los campos ya existentes — nuevo campo, no
rompe consumidores que lo ignoren.

## 4. Órdenes — agregar producto a un pedido específico (nuevo)

### `POST /orders/{order_id}/items` — **nuevo**

Reemplaza, para el flujo de Terminal de Mesas, al uso de
`POST /orders/tables/{table_id}/items` (`consolidation.add_item_to_table`), que asumía una
única orden abierta por mesa — supuesto inválido ahora que FR-007 permite pedidos paralelos.

**Request**: mismo payload de ítem(s) que el endpoint anterior (producto, presentación,
adicionales, cantidad, notas).

**Comportamiento**:
- Si el pedido (`order_id`) tiene estado no terminal (`recibida`/`abierta`/`bloqueada`): anexa
  los ítems al mismo pedido, sin crear un registro nuevo (FR-008). Responde `200`/`201` con el
  pedido actualizado (mismo `id`, total recalculado).
- Si el pedido tiene estado terminal (`pagada`/`cancelada`): responde `409 Conflict` con un
  mensaje explícito que el frontend traduce en la guía "este pedido ya está pagado, crea uno
  nuevo" (FR-009).

**`POST /orders/tables/{table_id}/items`** (el endpoint anterior, basado en mesa): a decidir
en implementación si se retira o se conserva como atajo válido solo cuando la mesa tiene como
máximo un pedido abierto (rechazando con error explícito si hay más de uno). El frontend de
Terminal de Mesas deja de usarlo de todas formas, porque "Agregar producto" en pantalla ya
tiene siempre el `order_id` identificado.

## 5. Ventas — detalle de venta (sin cambio de contrato si ya serializa `paid_amount`/`change_given`)

### `GET /sales/{sale_id}`

Verificar en implementación si `SaleResponse` ya serializa `paid_amount` y `change_given`
(ya existen como columnas desde la migración `f5a6b7c8d9e0`). Si faltan en el schema de
respuesta, añadirlos — campos nuevos opcionales, no rompe consumidores existentes.

No se añade ningún campo booleano "tuvo componente en efectivo": el frontend lo deriva de
`Sale.payments` (ya expuesto), que incluye el método de cada pago.

## 6. Menú QR / checkout / Pagos por confirmar / detalle de venta — sin cambio de contrato

El fix de FR-011 (Precio Promocional + Σ Adicionales) es un cambio de **valor calculado**, no
de forma del contrato: los mismos campos de total que ya devuelven `compute_checkout_preview`,
`compute_draft_preview`, `pay_order`, `checkout_and_send`, `confirm_cash_payment_attempt` y
`approve_payment_attempt` simplemente calculan el valor correcto. Ningún schema cambia de
forma.

---

## Adenda 2026-09-29 — US8 a US11

## 7. Órdenes — nombre y grupo de las opciones de cada ítem (US9, FR-015) — cambio aditivo

Aplica a toda respuesta que serialice `OrderItemResponse.options` (`GET /orders/{id}`,
listados de pedidos de mesa/para llevar/domicilio, eventos de tiempo real que reutilizan el
mismo schema).

### `OrderItemOptionResponse`

| Campo | Tipo | Estado |
|---|---|---|
| `id` | UUID | existente |
| `option_id` | UUID | existente |
| `quantity` | int | existente |
| `name` | string \| null | **nuevo** — `options.name`, resuelto en lectura (JOIN) |
| `group_name` | string \| null | **nuevo** — `option_groups.name` del grupo dueño de la opción |

- Ambos son solo lectura y opcionales: los consumidores que no los conocen no se rompen; el
  frontend los usa como respaldo cuando su `lookup()` del menú vigente no encuentra la opción.
- Sin columna nueva ni migración (ver `data-model.md` §6). Las consultas ya cargan
  `OrderItem.options` con `selectinload`, pero no `Option` ni su grupo: `name`/`group_name` se
  derivan con un `column_property` (subconsulta escalar correlacionada, tasks T066), que viaja
  en el mismo SELECT y evita un N+1 (verificar en implementación con el conteo de queries de un pedido con varias
  opciones).

## 8. `POST /orders/draft-preview` — sin cambio de contrato (US10, FR-016)

Se conserva el body actual (`DraftPreviewIn`: `items: OrderItemIn[]`, `delivery_fee`). Lo que
cambia es **qué manda el frontend**: el conjunto vigente completo (ítems guardados de tipo
producto + borradores) en vez de solo los borradores, para que el backend reevalúe las
promociones sobre todo el pedido (`research.md` D13). Ningún schema cambia de forma.

## 9. US8 y US11 — sin cambio de contrato

Presentación y estilos, solo frontend.

**Nota 2026-09-29 (US8-b)**: `GET /menu/qr-token/{token}` ya entrega el nombre de cada variante como
`presentation_name` (+ `presentation_id`), igual que `GET /menu`; no hay campo `name`. No cambia el
contrato: el mapper del comensal (`diner.service.ts`) pasa a leer el campo correcto.
