# Contrato: `POST /cart/submit` (US1, US2)

Endpoint ya existente (`app/api/v1/cart/router.py:137-167`, `service.submit_cart`). **No cambia de
forma** (mismo body `SubmitCartIn`, mismo `response_model=OrderResponse`, mismo `201`) — cambia su
**efecto secundario sobre el carrito** y el **contenido** de `OrderResponse.items[]`, más un nuevo
caso de `409`.

## Antes (spec 025)

```text
POST /cart/submit
Body: SubmitCartIn { payment_method_id: UUID, receipt_file_url: str | null }
Auth: x-session-token (SessionContext), sin cambio

→ 201 OrderResponse
  - crea CustomerOrder 'recibida' + OrderItem[] (unit_price sin descuento) + OrderPaymentAttempt
  - el Cart del participante pasa a status='confirmado' (fila sobrevive, nunca se vuelve a leer)
  - GET /cart posterior: _get_or_create_open_cart no encuentra fila 'abierto' → abre una nueva
```

## Ahora (esta spec)

```text
POST /cart/submit
Body: SubmitCartIn { payment_method_id: UUID, receipt_file_url: str | null }   # SIN CAMBIO
Auth: x-session-token (SessionContext)                                          # SIN CAMBIO

→ 201 OrderResponse
  - crea CustomerOrder 'recibida' + OrderItem[] + OrderPaymentAttempt          # sin cambio
  - cada OrderItem gana su snapshot de descuento (discounted_unit_price/discounted_line_total),
    calculado con el mismo motor que ya usa GET /cart (research.md Decisión 4)
  - el Cart del participante y sus líneas se ELIMINAN físicamente (db.delete(cart)), en la misma
    transacción — no queda ninguna fila, ni 'confirmado' ni de ningún otro status
  - GET /cart posterior: _get_or_create_open_cart no encuentra ninguna fila para el participante
    (antes tampoco encontraba una 'abierto', pero ahora tampoco existe una 'confirmado' huérfana)
    → abre una nueva, igual que antes — comportamiento observable idéntico en GET /cart
```

**Precondiciones y validaciones** (orden en que se evalúan — reordenado respecto a spec 025,
research.md Decisión 3):

1. Se carga el `Cart` `'abierto'` del participante, si existe, **sin lanzar todavía** ningún error.
2. Se calcula, una sola vez, si el participante tiene una orden no terminal previa
   (`_NON_TERMINAL_ORDER_STATUSES`, sin cambio).
3. **Si el carrito no existe o está vacío**:
   - y el participante **sí** tiene una orden no terminal previa → `409`
     `"Tu pedido ya fue enviado; revisa el estado en Mis pedidos."` — **NUEVO** (FR-007).
   - si no, y el carrito no existe (nunca hubo uno) → `404` `"No hay carrito abierto para el
     comensal"` — sin cambio (FR-009).
   - si no, y el carrito existe pero está vacío → `409` `"El carrito está vacío"` — sin cambio
     (FR-009).
4. **Si el carrito tiene ítems** y el participante ya tiene una orden no terminal → `409`
   `"Ya tienes una orden activa; espera a que finalice antes de enviar otra."` — sin cambio (FR-008).
5. Comanda manual activa de la misma mesa (`channel in ('counter','waiter')`) → `409` — sin cambio.
6. `payment_method_id` existe y está activo → `404`/`409` — sin cambio.
7. Reglas de comprobante según método (efectivo vs. transferencia) → `422` — sin cambio.
8. Disponibilidad de stock (chequeo preventivo) → `422` — sin cambio.

**Camino feliz**: crea `CustomerOrder` + `OrderItem[]` (con snapshot de descuento) +
`OrderPaymentAttempt`, **elimina** el `Cart`/`CartItem[]`/`CartItemOption[]` del participante — una
sola transacción, un solo `commit()`. Si el `commit()` viola `idx_active_order_per_participant`
(carrera entre dos envíos casi simultáneos), rollback completo (incluye deshacer el borrado del
carrito) y el mismo `409` de FR-008 — sin cambio respecto a spec 025.

**Responses**:

| Código | Caso |
|---|---|
| `201` | Pedido creado; `items[]` incluye `discounted_unit_price`/`discounted_line_total` cuando aplicó descuento; el carrito del participante queda en cero filas. |
| `404` | `payment_method_id` no existe, o no hay ni carrito ni orden previa (comensal que nunca agregó nada). |
| `409` | **Nuevo caso**: carrito vacío/inexistente + orden no terminal previa → mensaje de "ya fue enviado". Casos sin cambio: carrito vacío sin orden previa, orden activa ya existente con carrito no vacío (incluida la carrera resuelta por el índice), método de pago inactivo, comanda manual activa. |
| `422` | Sin cambio — comprobante inconsistente con el método, o falta de stock. |

## Lo que NO cambia

- El shape del body (`SubmitCartIn`) y de la respuesta (`OrderResponse` como tipo) — solo gana
  campos opcionales en `items[]` (ver [cart-orders.md](./cart-orders.md)).
- `events.order_created` (router, línea 157) — se sigue publicando después del commit, con el mismo
  cálculo de `total` sin descuento (research.md Decisión 6, deliberadamente fuera de alcance).
- Todo lo posterior a que el pedido existe (confirmación de staff, cobro, cancelación por el
  comensal) — specs 024/025/029, sin tocar.
