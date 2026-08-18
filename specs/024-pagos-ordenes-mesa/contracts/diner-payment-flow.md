# Contrato: flujo de pago del comensal (US2, US3, US5, US6)

Router: `app/api/v1/cart/router.py`. Auth: **todos** estos endpoints usan
`Depends(get_session_context)` (resuelve tenant/mesa/comensal desde `x-session-token`, invariante
A-24) — ninguno acepta `tenant_id`/`participant_id`/`order_id` de propietario distinto al del token.

## `GET /cart/payment-methods` — NUEVO

Lista los métodos de pago activos del tenant del comensal (FR-004).

```text
Response: 200
  [ { id, name, type, is_cash, payment_info } ]   # sin "active" — solo se listan los activos
```

- Filtra `WHERE active = true` — un método desactivado nunca aparece, aunque el comensal ya tuviera
  la pantalla abierta (Acceptance Scenario 1, US2).
- `payment_info` viaja completo (es justamente lo que el comensal necesita ver para transferir,
  Acceptance Scenario 2, US2).

## `POST /cart/orders/{order_id}/payment-attempts` — NUEVO

Crea un intento de pago nuevo para una orden del comensal autenticado (FR-012 para transferencia,
FR-008 para efectivo — ambos casos crean la fila; el comprobante se adjunta después por separado).

```text
Request:
  { payment_method_id: UUID }

Response: 201
  {
    id, order_id, payment_method_id, status: "pendiente",
    amount_received: null, receipt_file_url: null, created_at
  }
```

**Reglas**:
- `404` si `order_id` no existe o no pertenece al comensal del token (mismo patrón que
  `cancel_my_order`, `service.py:429`).
- `409` si la orden ya tiene un intento `pendiente` (FR-015a — el índice único parcial de
  `order_payment_attempts` es la garantía última; el servicio hace primero un `SELECT` para devolver
  un 409 con mensaje claro en vez de dejar que la violación de índice llegue como error de
  integridad crudo).
- `409` si `order.status != "recibida"` (no tiene sentido pagar una orden ya `abierta`/`bloqueada`/
  `pagada`/`cancelada`).
- No serializa `rejection_reason` en ningún caso — este router es el del comensal (Clarification 3).

## `POST /cart/payment-attempts/{attempt_id}/receipt/presign` — NUEVO

Pide una URL firmada para subir el archivo del comprobante a R2 (research.md Decisión 6). Reutiliza
`app/core/storage.py::generate_presigned_put_url`/`build_object_key`, no el endpoint
`/uploads/presign` (gateado por `require_tenant_admin`, inaccesible para el comensal).

```text
Request:
  { content_type: str }   # mismo whitelist de CONTENT_TYPE_EXTENSIONS ya usado por /uploads/presign

Response: 200
  { upload_url: str, key: str, public_url: str, expires_in: int }
```

- `404` si `attempt_id` no pertenece a una orden del comensal del token.
- `409` si el intento no está `pendiente`, o si su método no es de transferencia (`is_cash = true`
  no tiene comprobante).
- `422` si `content_type` no está en la whitelist.
- El comensal sube el archivo directo a R2 con `PUT {upload_url}` (fuera de la API, igual que el
  flujo de imágenes de producto) y luego llama al endpoint siguiente con el `public_url` recibido.

## `POST /cart/payment-attempts/{attempt_id}/receipt` — NUEVO

Asocia el archivo ya subido a R2 con el intento de pago (FR-012).

```text
Request:
  { file_url: str }   # el public_url devuelto por el presign anterior

Response: 200
  { id, status: "pendiente", receipt_file_url: file_url, ... }
```

- `404`/`409` mismas condiciones que el presign (intento del comensal, `pendiente`, método de
  transferencia).
- `409` si el intento ya tiene `receipt_file_url` (un comprobante por intento — un reintento crea un
  intento nuevo, no reemplaza el archivo de uno existente).
- Tras esta llamada, el intento sigue `pendiente` — queda a la espera de que el cajero lo revise
  (Acceptance Scenario 3, US2). No hay autoconfirmación posible desde este router (FR-013).

## `GET /cart/orders/{order_id}` — MODIFICADO (campo nuevo en la respuesta)

Ya existe (parte de `my_orders`/detalle de orden del comensal). Gana un resumen del estado de pago
para que el comensal vea "pendiente de revisión" / "rechazado, puedes reintentar" / "pago
confirmado" sin exponer `rejection_reason`:

```text
Response gana:
  current_payment_attempt: {
    id, status: "pendiente" | "confirmado" | "rechazado",
    payment_method_name: str
  } | null
```

- Acceptance Scenario 7 (US2) y Acceptance Scenario 3 (US3): mientras no haya intento `confirmado`,
  la orden sigue viéndose como pendiente de pago para el comensal — este campo es la única fuente
  que el frontend necesita para mostrar ese estado, sin inferirlo de `order.status` (que no cambia
  hasta `confirm_order`).

## Interacción con `POST /cart/submit` — MODIFICADO (US6)

Ya existe (`submit_cart`, `service.py:471`). Gana una validación antes de crear la `CustomerOrder`:

- `409 Conflict` si el comensal ya tiene una `CustomerOrder` con `status NOT IN ('pagada',
  'cancelada')` (research.md Decisión 8, FR-005). Mensaje explícito para que el frontend distinga
  este caso de otros 409 posibles (ej. carrito vacío, si existiera esa validación hoy).
- Sin cambio en el resto del comportamiento de `submit_cart` (crea la orden en `recibida`, cierra el
  carrito).
