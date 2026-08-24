# Quickstart: Validación de las Correcciones de Cobro, Anulación y Descuento

**Spec**: [spec.md](./spec.md) | **Contratos**: [contracts/api-contracts.md](./contracts/api-contracts.md)

Guía de validación manual/automatizada end-to-end, una escena por historia de usuario. No
sustituye los tests unitarios/característicos de cada repo — es la prueba de que el conjunto se
comporta como el spec exige.

## Prerrequisitos

- `pos-backend` corriendo localmente (API + PostgreSQL 16 con al menos un tenant de prueba) con un
  turno de caja (`cash_shift`) abierto para el cajero de prueba.
- `pos-heladeria` corriendo localmente contra ese backend (`ng serve` o equivalente).
- Un usuario staff autenticado en la Terminal de Mesas — repetir cada escenario relevante también
  con un usuario de rol Administrador, dado que las Historias 1 y 2 no admiten excepción por rol.
- Al menos una mesa libre.

## Escenario 1 — Un pedido ya pagado no se puede anular (User Story 1)

1. Crear una orden manual ("+ Crear Orden Manual" o F3), agregar un producto, y cobrarla
   ("Cobrar, Facturar y Enviar a Cocina").
2. Sobre esa misma orden ya pagada, intentar anular el ítem desde el panel de pedido.
   - **Esperado**: no aparece la acción "Anular" para ese ítem.
3. Invocar directamente `POST /orders/items/{item_id}/void` con el `item_id` de ese pedido (p. ej.
   con `curl` o la consola de red del navegador).
   - **Esperado**: `409` con el detalle "El pedido ya fue pagado y no puede anularse"; el ítem
     sigue con su `estado_cocina` sin cambios, sin `OrderItemVoidLog` nuevo.
4. Repetir el paso 3 sobre un pedido **sin pagar** (orden `abierta` recién creada, sin cobrar).
   - **Esperado**: la anulación se sigue permitiendo exactamente como antes de esta spec.
5. Con un ítem ya en `estado_cocina = "listo"` mientras la orden **todavía no está pagada**, marcar
   la preparación como avanzada de nuevo (`PATCH /orders/items/{item_id}/kitchen`).
   - **Esperado**: sigue funcionando sin cambios — `transition_kitchen` no se ve afectado por esta
     spec.

## Escenario 2 — Ningún rol puede aplicar descuento manual (User Story 2)

1. Abrir la Terminal de Mesas con una cuenta manual activa (sesión iniciada como Administrador).
   - **Esperado**: no existe ningún atajo "Aplicar descuento (F4)" ni control equivalente en la
     pantalla, ni el atajo de teclado F4 produce ningún efecto.
2. Agregar un producto que **no** califique para ninguna promoción activa.
   - **Esperado**: el descuento mostrado es `$0`.
3. Agregar un producto que sí califique para una promoción activa.
   - **Esperado**: el descuento aparece calculado automáticamente por el motor de promociones, sin
     ningún campo editable.
4. Invocar directamente `POST /orders/{order_id}/checkout-and-send` con `"discount": 5000` en el
   body.
   - **Esperado**: `422` (validación del esquema — `discount` solo admite `0`); el pedido no se
     cobra.
5. Repetir el paso 4 con `"discount": 0` (u omitiendo el campo).
   - **Esperado**: el cobro se completa con normalidad, sin relación con este cambio.

## Escenario 3 — La insignia "Listo" solo aparece cuando el pedido realmente ya se cobró (User Story 3)

1. Crear una orden manual directamente en `abierta` (mesero registra ítems sin pasar por
   "+ Crear Orden Manual"/`hold_for_payment`, o usar el fixture equivalente), sin cobrarla.
2. Marcar su único ítem como `listo` en cocina (`PATCH /orders/items/{item_id}/kitchen` dos veces,
   o el botón "✓ Listo" de la terminal).
3. Observar el listado de mesas y el detalle del pedido.
   - **Esperado**: la mesa **no** muestra la insignia "Listo" — muestra "Pago pendiente"; el
     encabezado del pedido dice "en preparación" o el equivalente para pago pendiente, no "listo
     para cobrar".
4. Cobrar ese mismo pedido (vía el flujo de cobro manual correspondiente).
5. Observar de nuevo el listado de mesas y el detalle.
   - **Esperado**: ahora sí aparece "Listo" en ambos lugares, de inmediato tras confirmarse el pago.
6. Repetir los pasos 1-3 pero dejando el ítem en `en_preparacion` (no `listo`) sobre un pedido ya
   pagado (usar el pedido del escenario 1).
   - **Esperado**: tampoco muestra "Listo" — hacen falta ambas condiciones a la vez.

## Escenario 4 — Una sola acción para imprimir la factura ya emitida (User Story 4)

1. Cobrar un pedido de un solo comensal (Escenario 1, paso 1) y observar el diálogo de éxito que
   aparece justo después.
   - **Esperado**: el diálogo no ofrece ningún botón de impresión para este caso (un solo
     comprobante) — solo el botón "Cerrar".
2. Cerrar el diálogo y mirar la barra lateral de esa mesa.
   - **Esperado**: aparece exactamente una acción "Imprimir Factura" (ya no "Reimprimir Factura
     POS" ni ningún duplicado).
3. Usar esa acción.
   - **Esperado**: se reimprime el mismo documento ya emitido; `GET /invoices?order_id=` sigue
     devolviendo un único documento, sin duplicados.
4. Cobrar una cuenta con varios comensales usando cierre dividido (`billing_mode='split'`,
   mecanismo ya existente de spec 010/026) y observar el diálogo de éxito.
   - **Esperado**: el diálogo sigue ofreciendo "Imprimir todos" y un botón "Imprimir" por
     comensal — este caso no cambia.
5. Sobre un pedido que nunca llegó a pagarse (cancelado, o pago rechazado sin reintento), revisar
   las acciones disponibles.
   - **Esperado**: "Imprimir Factura" no aparece — solo "Imprimir Pre-cuenta" mientras el pedido
     siga activo.

## Verificación automatizada (por repo)

- **Backend** (`pos-backend`): `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`
  — el characterization test `test_transition_kitchen_y_void_item_no_validan_status_de_la_orden_a16`
  (`test_orders_kitchen.py`) requiere actualización explícita: la aserción sobre `void_item` pasa a
  esperar `409` sobre una orden pagada (citando esta spec y A-16 en el docstring); la aserción sobre
  `transition_kitchen` se mantiene sin cambios. Agregar tests nuevos para el guard de `void_item`
  contra una orden `abierta` con `Sale` asociada (el caso que motivó la spec, distinto del caso
  `status == "pagada"` que ya cubre el test existente) y para el rechazo `422` de
  `CheckoutAndSendIn.discount != 0`.
- **Frontend** (`pos-heladeria`): `ng test` — actualizar `pos-terminal.store.spec.ts`
  (`describe('deriveTableStatus', ...)`) para cubrir el nuevo campo `paid` en ambas ramas ("listo"
  vs. "pago_pendiente" según pago), y `pos-checkout-panel.component.spec.ts` (test T033) para
  reflejar el nuevo texto/acción "Imprimir Factura". Agregar specs nuevos para: la ocultación del
  botón "Anular" cuando `selectedOrder().paid === true`, la ausencia del atajo F4 y su popover, y
  el texto de cabecera de tres estados en `pos-order-panel.component.ts`.
