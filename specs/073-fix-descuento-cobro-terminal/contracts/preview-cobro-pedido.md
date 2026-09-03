# Contrato: Preview de cobro de un pedido ya creado

**Cubre**: FR-001 a FR-007a — User Stories 1, 2 y 3 (spec.md). Reusado sin cambios por la
**quinta superficie** (revisión de pago del cajero para pedidos QR, FR-021–FR-024, US7 — ver
[revision-pago-cajero-qr.md](./revision-pago-cajero-qr.md)).
**Research**: [research.md](../research.md) D3, D4, D6 (y D13/D14 para el consumo desde US7).

## `GET /orders/{order_id}/checkout-preview`

Desglose autoritativo de un pedido **ya persistido** y aún no cobrado (mesa individual atendida
por la Terminal en estado `recibida`, para llevar o domicilio). Solo lectura — no bloquea el
pedido, no toca inventario, no requiere turno de caja abierto.

### Request

Sin cuerpo. `order_id` en la ruta.

### Response `200 OK` — `CheckoutPreviewResponse`

```json
{
  "subtotal": "16000.00",
  "discount": "8000.00",
  "delivery_fee": "0.00",
  "total": "8000.00",
  "promotion_evaluated_at": "2026-09-02T19:59:03.120000Z"
}
```

Forma exacta = `CheckoutPreview` de [data-model.md](../data-model.md). `discount`/`delivery_fee`
en `"0.00"` cuando no aplican (FR-004: el frontend decide con ese valor si pinta la fila o no —
"solo cuando su valor es mayor que cero"). `promotion_evaluated_at` es aware UTC (sufijo de zona),
serializado con el mismo `UtcDatetime` que `SaleResponse.sold_at`.

### Errores

| Código | Cuándo | Detalle |
|---|---|---|
| `404` | `order_id` no existe | igual que el resto de endpoints de `orders` (`get_or_404`). |
| `409` | `order.status` es terminal (`pagada`/`cancelada`) — cualquier otro estado es cobrable, incluido `recibida` (pedido QR pendiente de confirmación, US7) | mensaje explícito del estado actual, mismo patrón que `checkout_and_send`/`pay_order`. |

### Implementación (referencia, no prescriptiva de línea exacta — detalle en tasks.md)

`checkout.compute_checkout_preview(db, order_id)`:
1. Carga el `CustomerOrder` (`get_or_404`).
2. `lines = order_sale_lines(db, order.id)` — reuso literal (`checkout.py:202-240`), ya excluye
   ítems `anulado` (Edge Case "Ítems anulados en cocina").
3. `raw_subtotal = sum(line.line_total for line in lines)`.
4. `instant = promotion_evaluation_instant([order], now=datetime.now(timezone.utc))`
   (research.md D3 — con una sola orden, equivale a `order.promotion_evaluated_at or now`;
   `now` aware UTC, nunca naive).
5. `discount, _, _ = auto_discount(db, lines, instant)`.
6. `delivery_fee = order.delivery_fee or Decimal("0")`.
7. `total = compute_total(raw_subtotal, discount, tax=0, tip=0, delivery_fee)` (research.md D6).
8. Devuelve `CheckoutPreviewResponse(subtotal=raw_subtotal, discount=discount,
   delivery_fee=delivery_fee, total=total, promotion_evaluated_at=instant)`.

**No persiste nada** — a diferencia de `pay_order`/`checkout_and_send`, no hay `db.commit()`, no
hay `build_sale`, no hay `_deduct_and_open`. Es idempotente y se puede llamar tantas veces como
haga falta mientras el cajero mira el panel de cobro.

## Consumo desde `pos-heladeria` (referencia, ver research.md D10/D11)

- `PosTerminalStore` gana `checkoutPreview`/`checkoutPreviewLoading`/`checkoutPreviewStale`
  (mismo molde que `sessionBill`/`billLoading`/`billStale`, `pos-terminal.store.ts:1620-1679`), y
  `loadCheckoutPreview(orderId)` que llama a este endpoint.
- `PosCheckoutPanelComponent` (línea 174) pasa `store.checkoutPreview()?.total` en vez de
  `store.totals().total` a `[total]` de `app-payment-input`, y deshabilita "Cobrar"
  (FR-007a) mientras `checkoutPreviewLoading()` es verdadero o `checkoutPreview()` es `null`.
- Se dispara: al seleccionar el pedido (mismo punto donde hoy resetea `paymentDraft`,
  `pos-checkout-panel.component.ts:324-332`) y cada vez que algo pueda haber cambiado la cuenta
  (mismo evento que hoy marca `billStale`).
- Antes de `checkout()` (línea 334-337), research.md D11: vuelve a pedir el preview; si el
  `total` recibido difiere del que se le mostró al cajero, no somete el pago — pide confirmación
  con el total nuevo primero (FR-007).

### Consumo adicional desde la quinta superficie (US7 — pedidos QR)

`PaymentAttemptReviewPanelComponent` llama a `DiningSessionService.checkoutPreview(order.id)` —
mismo endpoint, sin pasar por `PosTerminalStore` (el flujo QR usa `DiningSessionService` directo).
Detalle completo en [revision-pago-cajero-qr.md](./revision-pago-cajero-qr.md) (D14/D15). El
chequeo previo de `confirm_cash_payment_attempt` también pasa a usar
`compute_checkout_preview(...).total` en el backend (D13), reemplazando `_order_total`.
