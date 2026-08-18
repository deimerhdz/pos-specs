# Contrato: revisión de pagos por el cajero (US2, US3, US5)

Router: `app/api/v1/orders/router.py` (junto a los demás endpoints de staff sobre `CustomerOrder`).
Auth: `Depends(get_current_user)` — cualquier staff autenticado (research.md Decisión 7, no hay rol
`CAJERO` dedicado hoy).

## `GET /orders/{order_id}/payment-attempts` — NUEVO

Historial completo de intentos de una orden, para el cajero/back-office (FR-016).

```text
Response: 200
  [
    {
      id, payment_method_id, payment_method_name, status,
      amount_received, change_amount, receipt_file_url,
      rejection_reason,          # SÍ visible aquí (a diferencia del router de cart)
      resolved_by_user_id, resolved_at, created_at
    },
    ...
  ]   # orden cronológico, incluye rechazados y el confirmado/pendiente vigente
```

- `404` si `order_id` no existe.
- Sin filtro de tenant explícito en el contrato porque `get_db`/el schema activo ya resuelven el
  tenant del usuario autenticado, igual que el resto de endpoints de `orders`.

## `POST /orders/payment-attempts/{attempt_id}/approve` — NUEVO

Aprueba un comprobante de transferencia (Acceptance Scenario 4, US2).

```text
Request: (sin body)

Response: 200
  { id, status: "confirmado", resolved_by_user_id, resolved_at }
```

- `SELECT ... WHERE id = :id AND status = 'pendiente' WITH FOR UPDATE` (research.md Decisión 9).
- `404` si no existe o ya no está `pendiente` — **no** se distingue con un mensaje "ya fue resuelto"
  vs. "no existe": ambos devuelven `409 Conflict` si la fila existe pero no está `pendiente` (para
  que dos cajeros casi simultáneos vean un conflicto claro, no un 404 confuso), y `404` solo si el
  `id` no existe en absoluto.
- `409` si el método del intento es efectivo (`is_cash = true`) — se aprueba con
  `confirm-cash`, no con `approve`.
- `409` si `receipt_file_url IS NULL` (no hay nada que aprobar todavía).

## `POST /orders/payment-attempts/{attempt_id}/reject` — NUEVO

Rechaza un comprobante con motivo obligatorio (FR-014, Acceptance Scenario 5 y 6, US2).

```text
Request:
  { reason: str }   # requerido, no vacío — 422 si falta o está vacío

Response: 200
  { id, status: "rechazado", resolved_by_user_id, resolved_at }
  # rejection_reason NO se re-expone en esta respuesta del router de staff tampoco es necesario:
  # el cajero ya lo acaba de escribir; queda disponible vía GET .../payment-attempts
```

- Mismas condiciones de bloqueo/estado que `approve` (`WITH FOR UPDATE`, solo sobre `pendiente`,
  solo transferencia).
- Tras el rechazo, el comensal puede iniciar un intento nuevo (US5) — este endpoint no crea esa fila
  nueva, la crea el comensal vía `POST /cart/orders/{order_id}/payment-attempts` cuando decida
  reintentar.

## `POST /orders/payment-attempts/{attempt_id}/confirm-cash` — NUEVO

Confirma un pago en efectivo, calculando el cambio (FR-009/FR-010/FR-010a, US3).

```text
Request:
  { amount_received: Decimal }   # > 0

Response: 200
  {
    id, status: "confirmado",
    amount_received, change_amount,   # change_amount = amount_received - total_orden
    resolved_by_user_id, resolved_at
  }
```

- `SELECT ... WHERE id = :id AND status = 'pendiente' WITH FOR UPDATE`, igual que `approve`/`reject`.
- `409` si el método del intento no es efectivo (`is_cash = false`) — ese caso usa `approve`/`reject`.
- **`422` si `amount_received < total_orden`** (FR-010a, Acceptance Scenario 4 de US3) — el sistema
  impide confirmar, no solo advierte; el total de la orden se calcula igual que en el resto del
  código de cobro (suma de líneas activas, `estado_cocina != "anulado"` si aplica el mismo filtro que
  `confirm_order` usa para stock).
- `change_amount = amount_received - total_orden` (`0` si son iguales, Acceptance Scenario 2 de
  US3).

## Responses comunes a los tres endpoints de resolución

| Código | Caso |
|---|---|
| `200` | Resuelto. |
| `404` | `attempt_id` no existe. |
| `409` | Ya no está `pendiente` (resuelto por otra petición — FR-018/SC-007), o método incompatible con el endpoint usado (`approve`/`reject` vs. `confirm-cash`). |
| `422` | `reason` vacío (`reject`), o `amount_received` ausente/`<= 0`/menor al total (`confirm-cash`). |
| `401` | Sin autenticar. |
