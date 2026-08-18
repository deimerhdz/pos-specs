# Contrato: `POST /cart/submit` con pago (US1, US2, US3, US4)

Endpoint ya existente (`app/api/v1/cart/router.py`, `service.submit_cart`). **Cambia de forma**:
gana un cuerpo obligatorio. Auth: `Depends(get_session_context)`, sin cambio (invariante A-24 —
el tenant/mesa/comensal siempre vienen del `x-session-token`, nunca del body).

## Antes (spec 024)

```text
POST /cart/submit
Body: {} (vacío)
→ crea CustomerOrder en 'recibida', sin ningún dato de pago
```

## Ahora (esta spec)

```text
POST /cart/submit
Body:
  SubmitCartIn:
    payment_method_id: UUID (requerido)
    receipt_file_url: str | null = null   # requerido si el método NO es efectivo

Response: 201
  OrderResponse   # mismo shape de siempre — current_payment_attempt ya viene poblado
```

**Precondiciones y validaciones** (en el orden en que se evalúan):

1. El carrito abierto del comensal no debe estar vacío → `409` (sin cambio respecto a hoy).
2. El comensal no debe tener ya una orden activa (`recibida`/`abierta`/`bloqueada`) → `409`
   `"Ya tienes una orden activa; espera a que finalice antes de enviar otra."` — mismo mensaje que
   hoy, ahora respaldado además por `idx_active_order_per_participant` (data-model.md) como
   garantía de última instancia ante una confirmación duplicada (FR-013).
3. `payment_method_id` debe existir y estar `active = true` → `404` si no existe, `409` si existe
   pero está inactivo (un método pudo desactivarse mientras el comensal estaba en la pantalla de
   pago).
4. Si el método es efectivo (`is_cash = true`): `receipt_file_url` debe ser `null`/ausente → `422`
   si se envía uno (dato inesperado).
5. Si el método NO es efectivo: `receipt_file_url` es requerido → `422` si falta (FR-006 — no se
   crea el pedido de una transferencia sin comprobante).
6. Disponibilidad de stock (chequeo preventivo, sin cambio respecto a hoy).

**Camino feliz**: crea `CustomerOrder` (`status='recibida'`) + sus `OrderItem` (copiados del
carrito, igual que hoy) + un `OrderPaymentAttempt` (`status='pendiente'`, `payment_method_id`, y
`receipt_file_url` si aplica) — **una sola transacción**, un solo `commit()`. Si el `commit()`
viola `idx_active_order_per_participant` (carrera entre dos envíos casi simultáneos del mismo
comensal), la transacción hace rollback completo y el endpoint responde el mismo `409` del punto 2
— ningún pedido a medias queda creado.

**Responses**:

| Código | Caso |
|---|---|
| `201` | Pedido creado, con su primer intento de pago adjunto. |
| `404` | `payment_method_id` no existe. |
| `409` | Carrito vacío, orden activa ya existente (incluida la carrera resuelta por el índice), o método encontrado pero inactivo. |
| `422` | `receipt_file_url` presente con efectivo, o ausente con transferencia; o falta de stock (`content-type` del error sin cambio respecto a hoy). |

## Lo que NO cambia

- `GET /cart/orders` (`my_orders`) y `POST /cart/orders/{id}/cancel` — mismo `OrderResponse`,
  mismo comportamiento; ahora todo pedido que devuelven ya trae `current_payment_attempt` desde su
  creación (antes podía ser `null` momentáneamente).
- `POST /cart/orders/{order_id}/payment-attempts`, `POST
  /cart/payment-attempts/{id}/receipt/presign`, `POST /cart/payment-attempts/{id}/receipt` (spec
  024, reintento tras rechazo) — sin ningún cambio, siguen operando sobre una orden que ya existe.
- Todo lo posterior a que el pedido existe (`GET /orders/{id}/payment-attempts`, `approve`,
  `reject`, `confirm-cash`, `confirm_order`) — spec 024, sin tocar.
