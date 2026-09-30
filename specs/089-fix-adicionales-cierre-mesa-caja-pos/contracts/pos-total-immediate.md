# Contrato — Total inmediato y sin modales en el POS

**Historia**: 2 · **FR**: 025–031 · **Decisiones**: research D14–D18 · **Decisión de negocio**: A-96

## 1. Estados de la cifra en pantalla

| Estado | Origen | ¿Cobrable? | Indicador |
|---|---|---|---|
| **Confirmado** | Respuesta de `GET /orders/{id}/checkout-preview` (o `POST /orders/draft-preview` en pedido manual) | Sí | ninguno |
| **Estimado** | Cálculo local: último confirmado + Σ `subtotal` de las líneas recién guardadas | **No** (Cobrar deshabilitado) | "actualizando…" discreto junto al total |

`PosTerminalStore` gana `checkoutPreviewEstimate = signal<{subtotal:number; total:number} | null>(null)`. `null` = no hay estimación (la cifra visible es la confirmada).

## 2. Flujo de `saveOrder()` (agregar productos a una orden abierta)

1. Antes de la primera llamada: `estimate = confirmado.total + Σ draftLines.subtotal` (síncrono, < 0,2 s). El panel muestra ese total al instante.
2. Se envían las líneas (`addOrderItems`, una por línea, secuenciales; `submitting = true` serializa altas concurrentes).
3. **Éxito**: `reload()` y `loadCheckoutPreview(orderId)`; al recibir el preview se limpia el estimado. La cifra pasa a ser la del servidor, **sin diálogo**, aunque difiera.
4. **Fallo en una línea**: se limpia el estimado, `loadCheckoutPreview(orderId)` (refleja las líneas que sí se guardaron antes del fallo), toast de error. Nunca queda un importe fantasma (FR-029).
5. **Servidor sin respuesta**: igual que (4); si el preview también falla, `checkoutPreview = null` ⇒ el panel muestra "Calculando el total…" con Cobrar deshabilitado (comportamiento ya vigente, FR-007a de la spec 073) y el error habitual (FR-031).

Aplica a mesa, domicilio y para llevar (mismo `saveOrder`/`selectedOrderId`).

## 2b. Quitar o reemplazar una línea (`voidPersistedItem`, reemplazo de línea)

FR-025 cubre también quitar y modificar. Misma causa raíz (D15): hoy solo se hace `reload()` y el preview queda viejo.

1. Tras la confirmación habitual "¿Anular este ítem?" (se conserva; no es el modal de total) y antes de llamar al servidor: `estimate = confirmado.total − lineTotal(línea)` (reemplazo: menos la línea vieja, más la nueva).
2. Tras el éxito: `reload()` y `loadCheckoutPreview(orderId)`; se limpia el estimado y la cifra pasa a ser la del servidor sin diálogo.
3. Fallo o servidor sin respuesta: igual que §2 (4) y (5) — se limpia el estimado y se revierte al confirmado.

## 3. Cobrar sin modal (`pos-checkout-panel::checkout`)

```
1. if (estimate != null || loading || !preview) return           // no cobrable aún
2. beforeTotal = total visible
3. await loadCheckoutPreview(orderId); fresh = checkoutPreview()
4. if (!fresh) return                                             // no se cobra un total no verificado
5. if (fresh.total != beforeTotal):
       mostrar aviso NO bloqueante en el panel:
         "El total cambió: ahora es $X (antes $Y). Revisa el importe y vuelve a pulsar Cobrar."
       return                                                     // NO cobra; el siguiente Cobrar cobra el total visible
6. checkoutAndSend(...)
```
- **Sin `confirm.ask`** en ningún paso (FR-027, SC-010).
- El aviso vive en el propio panel (mismo bloque visual que la barra `checkoutPreviewStale`); se descarta al volver a pulsar Cobrar, al cambiar de orden o al llegar un preview sin diferencia.
- La barra existente `checkoutPreviewStale` ("El total cambió · Actualizar", evento `session.bill_changed`) **se conserva**: es el aviso no bloqueante para cambios externos y ya cumple FR-028.

## 4. "Pagos por confirmar" (`payment-attempt-review-panel`)

- `reconfirmIfTotalChanged()`: sin `confirm.ask`. Si el total difiere del visible, actualiza `checkoutPreview`, muestra el aviso no bloqueante **dentro de la tarjeta** y **no resuelve** el pago; el siguiente clic (Aprobar/Confirmar) usa el total actualizado (FR-030, FR-028).
- `actionsBlocked()` deja de exigir `totalChangeAck`; sigue bloqueando mientras carga o no hay total autoritativo.
- El texto informativo `totalChanged()` (difiere del que declaró el comensal) se conserva **sin exigir acuse**.

## 5. Pedido manual (`manual-order-page::confirm`)

Sin `confirm.ask`: al volver a pedir `draft-preview`, si `fresh.total != before` se actualiza la cifra, se muestra el aviso no bloqueante y **no se crea** el pedido hasta un segundo clic (FR-015a de la spec 073 → FR-028 de esta).

## 6. Producto agotado con nombre (backend)

`orders/service.py::create_order`, `orders/consolidation.py::add_item_to_order` y el reemplazo de `orders/kitchen.py` capturan el error de descuento de inventario y responden:

```
HTTP 400 Bad Request   (el status actual de `InsufficientStockError` se conserva; el texto es lo que cambia)
{ "detail": { "error": "«Hamburguesa · Doble» está agotado: falta Queso",
              "producto": "Hamburguesa · Doble", "insumo": "Queso" } }
```
Etiqueta = `catalog_engine.consumption.variant_label`. `extractError` del frontend ya lee `detail.error`; el toast y `store.error` muestran ese texto. Sin cambio en el Menú QR (su `check_availability` ya devuelve `{insumo, disponible, requerido}`).

## 7. Pruebas

- Store: `voidPersistedItem` baja el total al instante y revierte si falla (§2b); `saveOrder` publica estimado antes del primer POST; lo limpia y confía en el preview al terminar; en fallo revierte al confirmado y nombra el producto; dos altas seguidas suman una sola vez.
- Panel: Cobrar deshabilitado con estimado; sin `confirm.ask`; primer Cobrar con cambio ⇒ aviso y sin cobro, segundo ⇒ cobra; idem "Pagos por confirmar" y pedido manual.
- Se actualizan (citando **A-96**) los casos de `pos-checkout-panel.component.spec.ts`, `pos-terminal.store.spec.ts`, `payment-attempt-review-panel.component.spec.ts` y `manual-order-page.component.spec.ts` que esperan el modal.
