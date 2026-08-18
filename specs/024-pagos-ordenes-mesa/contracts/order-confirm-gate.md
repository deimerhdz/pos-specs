# Contrato: gate de pago sobre `confirm_order` (US4)

Endpoint ya existente: `POST /orders/{order_id}/confirm` (`app/api/v1/orders/router.py:165-178` →
`checkout.confirm_order`). **No cambia de forma** (mismo path, método, `response_model`) — cambia
únicamente su precondición interna.

## Comportamiento actual (sin esta spec)

```text
1. SELECT CustomerOrder WHERE id=:id WITH FOR UPDATE
2. 404 si no existe
3. 409 si status != "recibida"
4. 409 si no quedan ítems consumibles (estado_cocina != "anulado")
5. deduct_order_items(...) — descuenta inventario
6. status = "abierta"; commit
```

## Comportamiento nuevo (esta spec, FR-017)

Se agrega un paso **entre el 3 y el 4** (antes de tocar inventario — si el pago no está confirmado,
no debe haber ningún efecto secundario, ni siquiera de intentar descontar stock):

```text
1. SELECT CustomerOrder WHERE id=:id WITH FOR UPDATE
2. 404 si no existe
3. 409 si status != "recibida"
3a. 409 si NO existe OrderPaymentAttempt WHERE order_id=:id AND status='confirmado'   # NUEVO
4. 409 si no quedan ítems consumibles
5. deduct_order_items(...)
6. status = "abierta"; commit
```

**Mensaje del 409 nuevo** (paso 3a): distinto del 409 del paso 3, para que el cajero/frontend pueda
distinguir "la orden no está en el estado correcto" de "falta confirmar el pago" — ej. `"La orden no
tiene un pago confirmado"` vs. `"Solo se confirman pedidos en 'recibida'"`. El código de estado HTTP
es el mismo (`409`) en ambos casos; la distinción es de mensaje/detalle, no de código, consistente
con cómo `confirm_order` ya distingue sus dos 409 actuales (pasos 3 y 4) solo por mensaje.

## Casos cubiertos (Acceptance Scenarios de US4)

| Escenario | Resultado |
|---|---|
| Único intento `pendiente` (sin resolver) | `409` en el paso 3a — igual que "rechazado", no hay `confirmado` |
| Único intento `rechazado` | `409` en el paso 3a — un `rechazado` no cuenta como `confirmado`, aunque el comensal ya haya visto el rechazo |
| Un intento `confirmado` (efectivo confirmado o comprobante aprobado) | Pasa el paso 3a, sigue el flujo normal (descuenta stock, `status = "abierta"`) |
| Confirmar dos veces la misma orden (doble clic / dos cajeros) | La segunda llamada falla en el paso 3 (`status` ya es `"abierta"`, no `"recibida"`) — mismo mecanismo de bloqueo que ya existe hoy vía `WITH FOR UPDATE`, sin cambio |

## Fuera de este contrato

- No cambia el `response_model` de `POST /orders/{order_id}/confirm` (`OrderResponse`, sin campos
  nuevos).
- No cambia el evento `events.order_confirmed(...)` emitido tras el commit (`router.py:180`).
- No se toca `checkout.pay_order`/`close_session` (cobro de mesa consolidado) — ese flujo ocurre
  después de `abierta`/`bloqueada`, fuera de alcance de esta spec.
