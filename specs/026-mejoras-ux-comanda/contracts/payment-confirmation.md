# Contrato: confirmación de pago fusiona el envío a cocina

Cubre FR-001/FR-002/FR-013 (spec 024, reutilizado) sobre los dos endpoints que resuelven un
Intento de Pago a favor del comensal. **El request y el response no cambian de forma** — solo su
efecto secundario.

## `POST /orders/{order_id}/payment-attempts/{attempt_id}/confirm-cash`

- **Request**: sin cambios — `{ "amount_received": <decimal> }`.
- **Response 200**: sin cambios de forma — `PaymentAttemptResponse` (incluye `amount_received`,
  `change_amount`, ya presentes hoy).
- **Efecto (nuevo)**: además de marcar el intento `confirmado` y calcular `change_amount`, la misma
  transacción ejecuta ahora el equivalente de `confirm_order` sobre la orden asociada (descuenta
  inventario, `status: recibida → abierta`). Ambos cambios comparten un único `commit`.
- **Errores (nuevos, propagados desde la lógica de `confirm_order` ya existente)**:
  - `409 Conflict` — la orden no tiene ítems, o algún otro precondición de `confirm_order` falla.
  - `409 Conflict` (u el código que ya use `deduct_order_items`) — stock insuficiente para algún
    insumo del pedido. **En este caso, el intento de pago NO queda confirmado** (rollback completo,
    FR-002) — la respuesta de error dispara el mismo camino de `toast.error(...)` que ya maneja el
    frontend para cualquier fallo de este endpoint.
- **Sin cambios**: la validación ya existente de `amount_received >= total` (FR-010a, spec 024)
  sigue ocurriendo antes que cualquier lógica nueva — un monto insuficiente nunca llega a intentar
  el envío a cocina.

## `POST /orders/{order_id}/payment-attempts/{attempt_id}/approve`

- **Request**: sin cambios — sin body.
- **Response 200**: sin cambios de forma — `PaymentAttemptResponse`.
- **Efecto (nuevo)**: igual que arriba — aprobar el comprobante ahora también descuenta inventario
  y envía la orden a cocina en la misma transacción.
- **Errores (nuevos)**: mismos que arriba.

## `POST /orders/{order_id}/payment-attempts/{attempt_id}/reject`

- **Sin cambios de ningún tipo** — rechazar un comprobante nunca ha enviado ni enviará una orden a
  cocina (spec 024, FR-014); esta spec no lo toca.

## Idempotencia (sin cambios)

La garantía ya existente de spec 024 (FR-018: una misma confirmación nunca produce más de un
efecto, ante doble clic o reintento de red) se conserva sin modificación — el `with_for_update`
sobre el Intento de Pago ya evita que dos resoluciones concurrentes del mismo intento tengan
efecto ambas.
