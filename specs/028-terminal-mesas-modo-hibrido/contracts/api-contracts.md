# Contratos de API: Rediseño Híbrido de la Terminal de Mesas

**Spec**: [spec.md](./../spec.md) | **Datos**: [data-model.md](./../data-model.md)

Todos los endpoints están bajo el prefijo existente `pos-backend` `app/api/v1/`. Autenticación:
staff autenticado (`User`) para todo lo que no sea el flujo público del comensal QR — sin cambios
respecto del esquema de auth ya existente.

## Endpoints reutilizados sin cambios (Out of Scope de esta spec)

Estos ya implementan las reglas de negocio que el spec reutiliza explícitamente (spec 024,
spec 026); esta spec solo decide dónde y cómo se exponen en la UI.

| Endpoint | Uso en esta spec |
|---|---|
| `GET /orders/{order_id}/payment-attempts` | Alimenta el bloque "Validación de Pago Requerida" (FR-002), filtrado en frontend a `channel="qr"`. |
| `POST /orders/payment-attempts/{attempt_id}/approve` | Botón "Confirmar y Enviar a Cocina" sobre un comprobante de transferencia (FR-002). |
| `POST /orders/payment-attempts/{attempt_id}/reject` | Botón "Rechazar" (FR-002) — único significado de "rechazar" en la nueva UI (D5). Requiere `reason` no vacío. |
| `POST /orders/payment-attempts/{attempt_id}/confirm-cash` | Botón "Confirmar y Enviar a Cocina" sobre un pago en efectivo QR (FR-002); ya calcula y devuelve el cambio (reutilizado de spec 026). |
| `POST /table-sessions/{id}/close` (`close_session`) | Sigue siendo el mecanismo de "cobrar y cerrar" cuando **sí** queda algo por cobrar (p. ej. cierre unificado clásico) — sin cambios; no se usa para FR-016 (ver endpoint nuevo abajo). |
| `GET /invoices?order_id=` / `GET /invoices/{id}` | Base de "Reimprimir Factura POS" (FR-012) y de la impresión automática (FR-003) — el frontend regenera el ticket a partir de la respuesta, sin endpoint de "reimprimir" dedicado. |
| `GET /table-sessions/{id}` / cuenta de la sesión (`compute_bill`) | Alimenta el modo "Resumen de Cuenta" de solo lectura (FR-005) y "Imprimir Pre-cuenta" (FR-007). |

## Endpoints nuevos

### `POST /orders` — extensión aditiva (soporta D1 / T1)

Campo nuevo, opcional, en el payload existente `OrderCreate`:

```jsonc
{
  // ...campos existentes sin cambios (dining_table_id, participant_id, items, channel, ...)
  "hold_for_payment": false   // NUEVO. Default false = comportamiento actual exacto.
}
```

**Comportamiento cuando `hold_for_payment: true`** (usado exclusivamente por la nueva UI
"+ Crear Orden Manual", FR-004/FR-005):
- La orden se crea con `status = "recibida"` en vez de `"abierta"`.
- No se descuenta inventario ni se marca ningún ítem visible en cocina en este paso.
- La orden **no** debe aparecer en la cola de "Validación de Pago Requerida" (esa cola se filtra
  por `channel == "qr"` además de `status == "recibida"`).
- Requiere `channel` en `{counter, waiter}` — `400` si se envía `hold_for_payment: true` con
  `channel: "qr"` (las órdenes QR ya usan su propio flujo de `recibida`).

**Comportamiento cuando se omite o es `false`**: idéntico al actual (sin cambios, ninguna llamada
existente se ve afectada).

**Regla de no-mezcla de orígenes (FR-013)**: `409 Conflict` si la `table_session_id` resultante ya
tiene una orden activa (no terminal) cuyo `channel` pertenece a la familia opuesta (`qr` vs.
`counter`/`waiter`), en cualquiera de las dos direcciones.

### `POST /orders/{order_id}/checkout-and-send` — nuevo (soporta D3 / FR-011)

Reemplaza, **solo para órdenes creadas con `hold_for_payment: true`**, la secuencia
`block_order` + `pay_order` (que no aplica a este flujo — ver D3 en research.md).

**Request**:
```jsonc
{
  "version": 3,                 // lock optimista, igual que BlockIn.version hoy
  "cash_shift_id": "uuid",
  "payments": [ { "method": "efectivo", "amount": 20000 } ],  // uno o varios, igual que PayIn
  "discount": 0, "tax": 0, "tip": 0,
  "billing_customer_name": "Consumidor Final"   // FR-010, default si se omite
}
```

**Comportamiento** (una sola transacción):
1. Verifica `order.status == "recibida"` y `order.version == version` (`409` en caso contrario,
   mismo formato de error que `block_order` hoy).
2. Verifica `ensure_open_shift(cash_shift_id)` (igual que `pay_order`).
3. Calcula líneas, promociones y descuentos (reutiliza `order_sale_lines` +
   `promotions.evaluate`/`combo_discount_for_lines`, igual que `pay_order`).
4. `build_sale(...)` → `Sale` + `Invoice` emitidos, con `customer_name = billing_customer_name`.
5. Ejecuta la misma transición que `_confirm_order_impl` (`recibida → abierta`, descuento de
   inventario, ítems visibles en cocina).
6. Si el paso 5 falla (p. ej. stock insuficiente), revierte toda la transacción — no queda un
   estado intermedio "pagado sin cocina" (misma garantía que spec 026 FR-002).
7. Emite los mismos eventos de tiempo real que ya emite el flujo QR equivalente
   (`payment_completed`, `table_status_changed`), para que el listado de mesas (FR-014) se
   actualice sin polling.

**Response**: `Sale` (mismo shape que hoy devuelve `pay_order`).

**Idempotencia**: doble clic o reintento con el mismo `version` en una orden que ya avanzó a
`abierta` devuelve `409` (mismo mecanismo de lock optimista que ya exige spec 024 FR-018) — nunca
genera una segunda venta.

### `POST /table-sessions/{table_session_id}/release` — nuevo (soporta D2 / FR-016 / T2)

**Request**: sin cuerpo (o `{}` — no recibe `payments` ni `billing_mode`, a diferencia de
`close_session`).

**Comportamiento**:
1. Carga la sesión con el mismo lock de fila que `close_session` (`RN-MESA-01`).
2. `409 "La sesión ya está {status}"` si no está `active` (igual que `close_session`).
3. `409` si `has_billable_orders(...)` es verdadero, con mensaje explícito
   ("Todavía hay algo por cobrar en esta mesa; cóbralo antes de liberarla.") — condición **inversa**
   a la de `close_session`.
4. `409` (mismo `detail` shape) si `_assert_closable(...)` falla — pedidos `recibida` sin
   confirmar, o ítems de cocina sin terminar; reutiliza la función tal cual, sin reabrir sus reglas.
5. `checkout.close_table_sessions(...)` + `table.status = "libre"`, commit.
6. Emite el mismo evento `session_closed` / `table_status_changed` que ya emite `close_session`.

**Response**: `204 No Content` o un resumen mínimo `{ "dining_table_id": "...", "status": "libre" }`.

**Idempotencia**: dos clics casi simultáneos (mismo cajero, o dos cajeros) — el segundo espera el
lock de fila, ve `status != "active"` y recibe `409` sin doble efecto (mismo mecanismo que ya
garantiza `close_session`).

## Contrato de frontend (sin backend nuevo)

| Elemento | Contrato |
|---|---|
| Modal "Ver Comprobante" (D4) | Componente que recibe `receipt_file_url: string` y lo muestra en overlay dentro de la misma pantalla; reemplaza el `<a target="_blank">` actual en `payment-attempt-review-panel.component.ts`. |
| Plantilla "Imprimir Pre-cuenta" (D6) | Nueva función `sessionBillToReceipt(bill: SessionBillResponse | BillResponse, ctx)` en `receipt.util.ts`, análoga a `saleToReceipt` mismo pero antes de que exista `Sale`; reutiliza `printReceiptHtml` sin cambios. |
| Insignias de mesa (FR-014) | Derivadas en el store (`pos-terminal.store.ts`) a partir de `DiningTable.status` + presencia de intentos de pago `pendiente` por sesión — no requiere nuevo endpoint, solo nueva lógica de mapeo local. |
