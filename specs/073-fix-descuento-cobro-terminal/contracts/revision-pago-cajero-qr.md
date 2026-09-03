# Contrato: Revisión de pago del cajero para pedidos de comensal por QR ("Pagos por confirmar")

**Cubre**: FR-021 a FR-024 — User Story 7 (spec.md). **Anomalía**: ninguna nueva — el cambio de
vigencia ya está en `A-70` (que enumera `confirm_cash_payment_attempt` y `approve_payment_attempt`
entre sus 8 call sites).
**Research**: [research.md](../research.md) D13, D14, D15 (adenda 2026-09-03).

El panel **"Pagos por confirmar"** es la revisión del cajero de un intento de pago declarado por
un comensal por QR: pulsa **"Confirmar efectivo"** (teclea el efectivo recibido) o **"Aprobar
comprobante"** (transferencia con comprobante subido), lo que confirma el intento, envía el pedido
a cocina y emite la venta/factura en la misma llamada (spec 028). Esta spec lo trata como una
**quinta superficie de cobro**: el importe que muestra, el que valida en el chequeo previo y el
que registra en la venta deben ser el mismo y salir de la misma autoridad (FR-021).

## Sin endpoint nuevo — reusa `GET /orders/{order_id}/checkout-preview`

El endpoint de [preview-cobro-pedido.md](./preview-cobro-pedido.md) ya devuelve el desglose
autoritativo `{subtotal, discount, delivery_fee, total, promotion_evaluated_at}` para cualquier
`CustomerOrder` que no esté en estado terminal (`pagada`/`cancelada`). Un pedido QR pendiente de
confirmación está en `recibida` → el endpoint responde `200` sin cambio de ruta ni de esquema.

El frontend ya tiene el método de servicio: `DiningSessionService.checkoutPreview(orderId)`.

## Backend — el chequeo previo del efectivo pasa a la cuenta autoritativa (FR-023)

### `confirm_cash_payment_attempt` (`app/api/v1/orders/checkout.py`)

| Hoy | Con esta spec |
|---|---|
| `total = _order_total(db, attempt.order_id)` — suma `unit_price × cantidad + delivery_fee`, **sin descuento** (`checkout.py:939-955`, `:1142`) | `total = compute_checkout_preview(db, attempt.order_id).total` — subtotal − descuento por promoción (instante congelado) + domicilio, misma función de solo lectura que muestra el panel |
| `if amount_received < total:` → 422 *"El monto recibido (X) es menor al total de la orden (Y)"* con `Y` inflado | mismo chequeo, ahora contra el `total` real: rechaza **solo** cuando el efectivo no cubre el total con descuento |
| `attempt.change_amount = amount_received - total` (`:1152`) con `total` inflado | mismo cálculo con el `total` real → coincide al peso con `Sale.change_given` que `build_sale` calcula después (FR-022) |
| `_order_total` (única llamada en `:1142`) | **se elimina** — no queda ninguna segunda fuente de verdad del total |

La emisión de la venta que sigue (`promotion_evaluation_instant` + `auto_discount` + `build_sale`
con `promotion_evaluated_at=instant`) **ya está correcta** desde la tarea T028 — no cambia.

### `approve_payment_attempt` (transferencia) — sin cambio de backend

Ya construye la venta con `sum(line_total) − promo_discount + delivery_fee` e instante congelado
(`checkout.py:1033-1044`, T028) y no tiene chequeo previo de "monto" (el importe es el total
exacto, no lo teclea el cajero). El único hueco es que el cajero no ve ese total antes de aprobar
— se cubre en el frontend (FR-024).

### Errores del preview reusado

| Código | Cuándo |
|---|---|
| `404` | el `order_id` del intento no existe (no debería ocurrir — el intento referencia un pedido real) |
| `409` | el pedido ya está `pagada`/`cancelada` — p. ej. otro cajero resolvió el intento primero; el panel ya maneja el `409` del propio `confirm`/`approve` con el mismo criterio |

## Frontend — `PaymentAttemptReviewPanelComponent` (`payment-attempt-review-panel.component.ts`)

### Carga del preview (FR-021/FR-022)

- Signals locales `checkoutPreview` / `checkoutPreviewLoading` / `checkoutPreviewError` + método
  `loadCheckoutPreview()` que llama a `DiningSessionService.checkoutPreview(this.order.id)` en
  `ngOnChanges` (junto a `this.load()` de intentos, `:240-243`). Molde señal-loading-error como el
  resto de spec 073, **local al componente** (este flujo no pasa por `PosTerminalStore`).
- `orderTotal()` (`:266-272`) → `checkoutPreview()?.total` (hoy: suma local `unit_price × quantity`).
- `cashChangePreview()` (`:278-282`) calcula el vuelto sobre `checkoutPreview()?.total`.
- Desglose nuevo en la plantilla del panel: `Subtotal` / `Descuento` / `Domicilio` / `Total`
  (FR-022, mismo formato agregado de FR-004); `Descuento` y `Domicilio` solo cuando su valor es
  `> 0`.
- Estado **"calculando"** (español de Colombia, Principio XIII) mientras
  `checkoutPreviewLoading()` es `true` o `checkoutPreview()` es `null`: "Confirmar efectivo" y
  "Aprobar comprobante" deshabilitados; nunca un total provisional (FR-024 → regla de FR-007a).

### `PaymentValidationBlockComponent` (`payment-validation-block.component.ts`)

- Se elimina la fila de pie `$ {{ total(order) | number }}` (`:102-106`) y el método `total(order)`
  (`:136-147`) — el total autoritativo y su desglose los muestra ahora el
  `app-payment-attempt-review-panel` embebido. Con esto desaparece el último uso de
  `discounted_line_total` (el total congelado del carrito, que "disimulaba el fallo").

### Reconfirmación cuando el total cambió (FR-024)

- **Al abrir el panel**: si `checkoutPreview()?.total` difiere del total que la tarjeta venía
  mostrando (el `Σ discounted_line_total` del carrito), el panel muestra un aviso "el total cambió
  respecto al declarado por el comensal: antes $X, ahora $Y" y exige que el cajero lo reconozca
  antes de habilitar las acciones. Cubre el Edge Case "La tarjeta de 'Pagos por confirmar' muestra
  un total desactualizado" y el de promoción pausada entre el pedido y el cobro (FR-009a: el
  estado vivo ya hace que el preview traiga el total recalculado).
- **Justo antes de `approve()` / `confirm()`**: se vuelve a pedir `checkoutPreview(order.id)`; si
  el `total` cambió respecto al último mostrado, se detiene la acción, se presenta el total nuevo
  y se exige una segunda confirmación explícita — **nunca** se deja que el 422 del backend sea el
  aviso (mismo patrón que [research.md D11](../research.md)). Para "Confirmar efectivo" se
  re-valida además que el `amountReceived` tecleado siga cubriendo el total nuevo.

## Invariantes que cualquier implementación debe preservar

1. **Una sola autoridad del total en las tres puntas**: lo que el panel muestra, lo que el chequeo
   previo de `confirm_cash_payment_attempt` valida y lo que `build_sale` registra salen todos de
   `compute_checkout_preview` / `auto_discount` con el mismo instante congelado (FR-021). No queda
   ningún cálculo de total sin descuento en el camino del pedido QR (se elimina `_order_total`).
2. **El vuelto se calcula sobre el `Total` real** (con descuento y domicilio), nunca sobre el
   subtotal bruto (FR-022) — tanto en la vista previa del panel como en `attempt.change_amount` y
   `Sale.change_given`.
3. **El estado de la promoción se lee vivo** (FR-009a): si se pausó entre el pedido y el cobro, el
   preview trae el total sin descuento y el panel lo marca (FR-024) — no hay 422 sorpresa al
   pulsar el botón.
4. **El instante congelado manda para la vigencia temporal** (FR-009): un pedido QR creado a las
   19:59 dentro de una promoción vigente hasta las 20:00, confirmado a las 20:05, conserva el
   descuento. Ya cubierto por `A-70` + T027/T027a (el flujo QR congela su instante al confirmar el
   carrito).
5. **Ningún campo nuevo en `OrderPaymentAttempt` / `PaymentAttemptResponse`** — el instante viaja
   por `Sale.promotion_evaluated_at` (FR-011a), el total por `CheckoutPreviewResponse`.
6. **`approve_payment_attempt` no se toca en backend** — solo se deja constancia en el commit de
   que se revisó y ya emitía la venta con el total correcto desde T028.

## Acceptance Scenarios cubiertos (spec.md, Historia 7)

| Escenario | Cubierto por |
|---|---|
| 1 — desglose `16000 / −8000 / 8000` en el panel | D14 (desglose FR-022) |
| 2 — $10.000 efectivo → venta $8.000, vuelto $2.000, sin el mensaje de "monto menor al total" | D13 (chequeo previo) + D14 |
| 3 — $8.000 efectivo → vuelto $0, primer intento | D13 |
| 4 — $5.000 efectivo → "faltan $3.000" (sobre $8.000) | D13 (`amount_received < total` real) |
| 5 — transferencia: cajero ve $8.000 antes de "Aprobar", venta $8.000 | D14 (preview visible) — backend ya correcto |
| 6 — pedido QR 19:59 / cobro 20:05 → descuento aplicado, total $8.000 | `A-70` + instante congelado (T027/T027a), preview lo refleja |
| 7 — promoción pausada entre pedido y cobro, total sube $8.000→$16.000 → panel marca el cambio y exige confirmación | D15 (FR-024 + FR-009a) |
