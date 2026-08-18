# Data Model: Revisión y Pago Antes de Enviar el Pedido (Skeilopos)

Esta spec no agrega ninguna tabla ni columna. El único cambio de esquema es un índice único
parcial nuevo sobre una tabla ya existente (`customer_orders`, spec 007/017). Las decisiones detrás
de esta elección están en [research.md](./research.md) (Decisión 4); este documento se limita a la
definición exacta y su alcance.

## CustomerOrder (`customer_orders`) — MODIFICADA (solo índice)

Ningún campo cambia. Se agrega:

| Restricción | Definición | Reemplaza a |
|---|---|---|
| `idx_active_order_per_participant` | Índice único parcial: `UNIQUE (participant_id) WHERE status NOT IN ('pagada', 'cancelada')` | La validación de solo-aplicación que `submit_cart` ya hacía (spec 024, un `SELECT` antes del `INSERT`, sin garantía bajo concurrencia) |

**Semántica del índice**:
- Cubre únicamente filas con `participant_id IS NOT NULL` — Postgres no considera dos `NULL`
  iguales para efectos de un índice único, así que las órdenes de mostrador/mesero (`channel`
  `counter`/`waiter`, siempre `participant_id = NULL`) no se ven afectadas en absoluto.
- El predicado `WHERE status NOT IN ('pagada', 'cancelada')` es idéntico al que ya usaba
  `_NON_TERMINAL_ORDER_STATUSES = ('recibida', 'abierta', 'bloqueada')` en
  `app/api/v1/cart/service.py` — mismo criterio de "activa", ahora también a nivel de base de
  datos, no solo de aplicación.
- Como con cualquier índice único parcial de este proyecto (`idx_active_session_per_table`,
  `idx_open_cart_per_participant`, `idx_pending_payment_attempt_per_order`), un segundo `INSERT`
  que lo violaría no falla silenciosamente: `submit_cart` captura la `IntegrityError`
  correspondiente y la traduce al mismo `409` que ya devolvía el chequeo de aplicación (ver
  contracts/submit-cart-with-payment.md).

## OrderPaymentAttempt (`order_payment_attempts`) — SIN CAMBIOS DE ESQUEMA

Ninguna columna ni restricción cambia respecto a spec 024. Cambia únicamente **cuándo** nace la
primera fila de esta tabla para una orden dada: antes (spec 024), en una llamada HTTP separada,
después de que la orden ya existía; ahora, en la misma transacción que crea la orden
(`POST /cart/submit`, ver contracts/submit-cart-with-payment.md).

Las filas creadas por el reintento tras un rechazo (spec 024, Historia 5 — `POST
/cart/orders/{order_id}/payment-attempts`, sin cambios en esta spec) siguen naciendo exactamente
igual que hoy: en una llamada separada, sobre una orden que ya existe.

## Transición: qué reemplaza a qué

```text
spec 024 (antes de esta spec):
  POST /cart/submit                                    → crea CustomerOrder (sin pago)
  POST /cart/orders/{id}/payment-attempts               → crea el 1er OrderPaymentAttempt
  [solo transferencia] POST .../receipt/presign          → presign (ligado al attempt)
  [solo transferencia] POST .../receipt                  → adjunta el archivo

spec 025 (esta spec):
  [solo transferencia] POST /cart/payment-receipt/presign → presign (NO ligado a nada — nuevo)
  [solo transferencia] (el comensal sube el archivo directo a R2 con la URL firmada)
  POST /cart/submit                                      → crea CustomerOrder + 1er
                                                             OrderPaymentAttempt juntos,
                                                             recibiendo payment_method_id y
                                                             (si aplica) receipt_file_url

  Reintento tras un rechazo (spec 024, Historia 5) — SIN CAMBIOS, porque ahí la orden ya existe:
  POST /cart/orders/{id}/payment-attempts
  POST /cart/payment-attempts/{id}/receipt/presign
  POST /cart/payment-attempts/{id}/receipt
```

## Reglas de validación (resumen por historia de usuario)

| Regla | Dónde se aplica | Historia |
|---|---|---|
| No se registra ningún pedido mientras el comensal no complete el pago | Ausencia de fila — no hay `INSERT` hasta `POST /cart/submit` | US1 |
| Efectivo no lleva `receipt_file_url` | Validación de `submit_cart` (`422` si se envía uno) | US2 |
| Transferencia exige `receipt_file_url` | Validación de `submit_cart` (`422` si falta) | US3 |
| Cambiar de método de transferencia antes de enviar no tiene restricción | No aplica ninguna regla — no existe ningún recurso creado todavía que gestionar | US4 |
| Una orden activa por comensal (spec 024) + ninguna confirmación duplicada crea dos pedidos (nuevo, FR-013) | `idx_active_order_per_participant` (único mecanismo para ambas) | US1, US4; edge cases |
| Reintento tras fallo de creación no exige volver a subir el archivo | Estado en memoria del cliente (`receipt_file_url` ya obtenido) — sin cambio de esquema | Edge case (FR-012) |
