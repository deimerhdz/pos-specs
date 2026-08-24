# Research: Correcciones de Cobro, Anulación y Descuento en la Terminal de Mesas

**Spec**: [spec.md](./spec.md) | **Fecha**: 2026-08-21

Este documento resuelve las decisiones técnicas necesarias para implementar el spec, descubiertas
al inspeccionar el estado real de `../pos-backend` y `../pos-heladeria` — ambos ya en producción
con spec 028 mergeada (`11bc912` en `pos-heladeria`, `049a91a` en `pos-backend`). Ninguna decisión
aquí reabre alcance de negocio ya cerrado en el spec (Principio XI de la
[Constitución](../../.specify/memory/constitution.md)): son decisiones **técnicas** de cómo
implementar lo que el spec ya definió, no decisiones de negocio nuevas.

## D1: Consolidar impresión — retirar `printReceipt`/`lastReceipts` como acción de usuario, dejar `printOrderInvoice` como la única

**Decisión**: el diálogo de éxito que aparece justo tras cobrar (`table-sessions.component.ts`,
señal `successOpen`) deja de ofrecer un botón de impresión para el caso de **una sola venta**
(`store.lastReceipts().length === 1`, el caso reportado — un pedido con un solo comensal). Ese
caso pasa a reutilizar exclusivamente `store.printOrderInvoice(orderId)` — el mismo método que ya
usa el botón "Imprimir Factura" (renombrado desde "Reimprimir Factura POS") de la barra lateral,
verificado contra el backend. El diálogo de éxito **conserva** sus botones "🧾 Imprimir todos" /
"🧾 Imprimir" por comensal para el caso de cuenta dividida (`lastReceipts().length > 1`) — ese caso
no es el duplicado reportado: no existe hoy ninguna acción equivalente en la barra lateral para
imprimir el ticket de cada comensal por separado de una cuenta ya dividida.

**Razón**: hoy coexisten dos implementaciones para reimprimir el mismo documento de un pedido de
un solo comensal:
- `store.printReceipt(index?)` (`pos-terminal.store.ts:1373-1379`), alimentado por la señal en
  memoria `lastReceipts` (`:240`), poblada solo justo después de `checkoutAndSend`/aprobación de
  pago (`:1264-1333`) — sin llamada al backend en el momento de imprimir, se pierde si se recarga
  la página. Botones "🧾 Imprimir factura" / "🧾 Imprimir todos" / "🧾 Imprimir" en el diálogo de
  éxito (`table-sessions.component.ts:144-176`).
- `store.printOrderInvoice(orderId)` (`:1435-1445`), vía `resolveSaleForOrder` (`:1418-1433`) que
  primero mira la misma caché y, si no está, consulta el backend
  (`DiningSessionService.findSaleForOrder` → `GET /invoices?order_id=` + `GET /sales/{sale_id}`,
  `dining-session.service.ts:75-84`). Botón "🧾 Reimprimir Factura POS" en la barra lateral
  (`pos-checkout-panel.component.ts:159-164`), visible en todo momento que haya un pedido
  seleccionado con cuenta (`store.sessionBill()` + `store.selectedOrder()`), sin relación con si el
  diálogo de éxito está abierto o no.

Como el botón de la barra lateral queda visible tan pronto el pedido tiene cuenta —exactamente el
momento en que también se abre el diálogo de éxito tras cobrar—, el cajero ve ambas acciones casi
al mismo tiempo al confirmar el pago: el reporte del usuario. Ambas terminan llamando a
`buildReceiptHtml` + `printReceiptHtml` (`services/receipt.util.ts`) sobre el mismo documento —
la diferencia es solo el origen del dato (caché volátil vs. backend verificado), una duplicación de
implementación además de UI. Se retiene la versión respaldada por el backend porque es la única
que sigue funcionando igual sin importar cuánto tiempo pase o si la página se recargó — la caché no
sobrevive un refresh.

**Alternativas consideradas**:
- *Unificar ambos botones para que llamen al mismo método, sin retirar ninguno de los dos*:
  descartada — el spec pide explícitamente "una sola acción", no dos botones idénticos por debajo;
  dejar dos accesos visibles seguiría pareciendo duplicado aunque compartieran implementación.
- *Retirar el botón de la barra lateral y quedarse con el del diálogo*: descartada — el del diálogo
  solo existe mientras el diálogo está abierto (una ventana de tiempo corta); el spec exige poder
  reimprimir la factura en cualquier momento posterior (FR-002), no solo justo tras cobrar.
- *Eliminar también "Imprimir todos" del caso de cuenta dividida*: descartada — no es el duplicado
  reportado (no hay una acción equivalente en la barra lateral para ese caso) y retirarla sin razón
  sería una regresión no autorizada por el spec (Principio VI).

## D2: La insignia "Listo" pasa a exigir un pago ya registrado — nuevo campo `paid` en `OrderResponse`, no `CustomerOrder.status`

**Decisión**: agregar un campo booleano computado `paid` a `OrderResponse` (backend) — verdadero
si existe una `Sale` con `customer_order_id` igual al id de la orden — y usarlo, junto con el
estado de cocina ya existente, en dos sitios del frontend:
1. `deriveTableStatus(orders, tableStatus)` (`pos-terminal.store.ts:147-166`): solo devuelve
   `'listo'` cuando **todas** las órdenes activas de la mesa con ítems están `paid` y sus ítems
   están `estado_cocina === 'listo'`. Si la cocina ya terminó pero al menos una orden relevante no
   está `paid`, devuelve `'pago_pendiente'` — la insignia ya existente ("Pago pendiente", chip
   índigo, ya incluida en `NEEDS_STAFF`), en vez de inventar un estado nuevo.
2. El texto de cabecera del pedido seleccionado (`pos-order-panel.component.ts:28`, hoy
   `store.kitchenReady() ? 'listo para cobrar' : 'en preparación'`): pasa a tres ramas — "listo
   para cobrar" solo si además `paid`; "pago pendiente" si la cocina ya terminó pero no hay pago
   registrado; "en preparación" en cualquier otro caso. `store.kitchenReady()` **no cambia** — sigue
   controlando, sin tocar pago, cuándo se oculta el botón "Marcar pedido listo" (uso puramente de
   cocina, correcto tal como está).

**Razón**: `deriveTableStatus` hoy decide `'listo'` mirando exclusivamente `estado_cocina` de los
ítems — nunca el estado de pago. La exclusión de `status === 'pagada'` en `tableOrders()`
(`:348-353`, filtro previo a `deriveTableStatus`) no protege de este bug: se confirmó leyendo
`pos-backend/app/api/v1/orders/checkout.py` que **ninguno** de los dos caminos de pago vigentes en
la Terminal de Mesas deja la orden en `status = 'pagada'` — `checkout_and_send` (`:474`, manual) y
`_confirm_order_impl`/`_deduct_and_open` (`:335`, QR) terminan en `status = 'abierta'`, con la
`Sale` ya emitida (`build_sale(..., customer_order_id=order.id, ...)`) pero el status del pedido
sin cambiar. `status = 'pagada'` solo ocurre en el camino legado `block_order` → `pay_order`
(`checkout.py:293`) — el mesero abre la cuenta directamente en `'abierta'`, cocina la va
terminando, y el cobro ocurre al final. Eso significa que el filtro de `tableOrders()` es
efectivamente letra muerta para el camino QR/mostrador actual (nunca excluye nada ahí, porque esas
órdenes nunca llegan a `'pagada'`), mientras que `deriveTableStatus` no tiene ninguna otra señal de
pago para decidir `'listo'` — exactamente la mezcla que produce el defecto de la captura adjunta al
spec: un pedido de mesero (`'abierta'`, sin `Sale` todavía) con cocina ya en `'listo'` cae en la
misma rama que un pedido genuinamente ya pagado.

`Sale.customer_order_id` (indexado, `sale.py:40`) es la señal correcta y única que cubre **ambos**
caminos por igual: se confirma poblada tanto en `pay_order` (`checkout.py`, cerca de línea 280)
como en `checkout_and_send` (`:474`). `OrderResponse` hoy no expone ningún campo de pago
(`orders/schemas.py:172-189`) — de ahí la necesidad del campo nuevo, para que el frontend no tenga
que hacer una consulta extra por pedido solo para pintar el listado de mesas.

**El patrón de consulta ya existe y ya está probado en producción**: `has_billable_orders`
(`app/api/v1/table_sessions/service.py:65-83`) — agregado en un hotfix posterior a spec 028, para
un problema hermano (una mesa QR con `Sale` ya emitida pero `status="abierta"` nunca se liberaba
porque `has_billable_orders` solo miraba `status`) — ya calcula exactamente esta condición:
```python
ya_facturados = select(Sale.customer_order_id).where(Sale.customer_order_id.isnot(None))
# CustomerOrder.id.notin_(ya_facturados) → "esta orden NO tiene Sale"
```
Su propio docstring documenta el mismo hallazgo que este research: "la aprobación/confirmación de
un pago QR ahora factura en el mismo paso... pero deja el pedido en `status="abierta"` a
propósito". Esta spec **reutiliza el mismo patrón de consulta** (no lo reinventa) en un helper
nuevo, sin modificar `has_billable_orders` ni `table_sessions/service.py` — tocar esa función no es
necesario para ningún FR de esta spec, y hacerlo sería una refactorización no relacionada
(Principio V). El helper nuevo vive en el módulo `orders` (`app/api/v1/orders/service.py`), que es
lo que necesitan **ambos** consumidores nuevos (`kitchen.void_item`, D3, y la serialización de
`OrderResponse.paid` de este mismo D2) — `table_sessions/service.py` sigue con su propia copia
inline, intencionalmente sin tocar.

**Alternativas consideradas**:
- *Revisar `order.status === 'pagada'` en el frontend, sin campo nuevo*: descartada — es
  exactamente la señal que ya se demostró insuficiente (nunca se cumple para el camino
  QR/mostrador vigente); habría dejado el defecto reportado sin corregir.
- *Consultar `GET /invoices?order_id=` por cada pedido al pintar el listado de mesas*: descartada
  por costo — N pedidos visibles = N llamadas adicionales solo para una insignia; el campo
  computado en la misma respuesta que ya se carga es más simple y no agrega una petición.
  (`resolveSaleForOrder` sigue siendo el mecanismo correcto para *obtener el documento a
  imprimir*, no para decidir una insignia sobre el listado completo.)
- *Inventar un nuevo valor de `TableDisplayStatus` (p. ej. `'listo_para_cobrar'`) en vez de
  reutilizar `'pago_pendiente'`*: descartada — `'pago_pendiente'` ya describe exactamente esta
  situación ("cocina terminó, falta cobrar") y ya está en `NEEDS_STAFF`; una insignia adicional casi
  idéntica solo fragmentaría la interfaz sin agregar información.

## D3: Bloqueo de anulación tras pago — nuevo guard en `void_item`, reutilizando la misma señal de D2

**Decisión**: agregar en `kitchen.void_item` (`pos-backend/app/api/v1/orders/kitchen.py:93-176`) un
guard al inicio, antes de mutar nada: si la orden ya tiene una `Sale` asociada (misma consulta que
sustenta el campo `paid` de D2) **o** su `status == 'pagada'`, responder `409` con un mensaje
explícito ("El pedido ya fue pagado y no puede anularse") en vez de proceder. `transition_kitchen`
y `mark_order_ready` **no cambian** — siguen exactamente igual (ver Razón). En el frontend, el
botón "Anular" (`pos-order-panel.component.ts:99-102`, siempre visible hoy sin condición) se oculta
cuando el pedido seleccionado está `paid` (mismo campo de D2), y una llamada que igual llegara a
recibir el `409` del backend se maneja mostrando el mensaje del servidor (defensa en profundidad,
no solo ocultar el botón).

**Razón**: esta anomalía ya está documentada como **A-16** en
[`registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md#a-16--cocina-puede-mutar-y-anular-ítems-sin-validar-el-estado-del-pedido-padre)
— "ACCIDENTAL" para la porción que compara `void_item` contra `mark_order_ready` (que sí valida
parcialmente), con "tratamiento acordado: corregir... replicar en `transition_kitchen` y
`void_item` la misma validación de status que ya tiene `mark_order_ready`". Esta spec **decide
solo la mitad de eso**: aplica la corrección a `void_item` (autorizado explícitamente por esta
spec, Historia 1), pero **no** a `transition_kitchen` — porque bloquear transiciones de cocina
sobre un pedido ya pagado rompería el flujo principal vigente (QR/mostrador, "pagar primero,
cocina después"): ahí la cocina **debe** poder seguir moviendo ítems de `pendiente` a `listo`
**después** de que el pago ya se registró, ya que el pago ocurre primero por diseño (spec 026,
`checkout_and_send`/D3 de spec 028). Bloquear `transition_kitchen` ahí rompería la operación diaria,
no la protegería. La pregunta de negocio abierta que cita A-16 sobre `transition_kitchen`/
`mark_order_ready` y el estado `'bloqueada'` (RN-ORD-37) sigue **pendiente y fuera de esta spec**
— solo se decide aquí la anulación (`void_item`), no la preparación.

El guard reutiliza la misma señal de "pago existente" que D2 (`Sale.customer_order_id`) y no solo
`order.status == 'pagada'`, por el mismo motivo: el camino QR/mostrador nunca alcanza `'pagada'`, y
el reporte original del usuario ("cuando se paga la orden... no se puede anular") es precisamente
sobre ese camino — comprobar solo `status` habría dejado el defecto reportado sin corregir para el
caso que motivó la spec.

**Impacto sobre characterization tests (Principio III de la Constitución)**: el test existente
`test_transition_kitchen_y_void_item_no_validan_status_de_la_orden_a16`
(`pos-backend/app/characterization_tests/test_orders_kitchen.py:71-89`) congela **ambas**
funciones en una sola aserción conjunta. Debe **actualizarse explícitamente** (no en silencio):
- La aserción sobre `transition_kitchen` (primera mitad del test) se mantiene sin cambios — sigue
  siendo comportamiento protegido, esta spec no lo toca.
- La aserción sobre `void_item` (segunda mitad, `s2`) debe cambiar de "se ejecuta igual" a "lanza
  `HTTPException` 409" — documentando en el docstring del test que esta spec (029) autoriza el
  cambio, citando A-16 y la decisión de negocio de esta spec (Historia 1, FR-005/006/007).
  Recomendado dividir el test en dos métodos (uno por función) para que el nombre y el docstring de
  cada uno describan con precisión su propio comportamiento — evita que un test con dos funciones
  distintas quede con un nombre que ya no describe a ambas por igual.
- El test de contraste `test_mark_order_ready_409_si_orden_pagada_contraste_a16` (línea ~92 en
  adelante) no cambia — sigue siendo la referencia de que `mark_order_ready` ya validaba.

## D4: Descuento manual — retirar del frontend y rechazar en el único endpoint que lo recibe

**Decisión**: en el frontend, retirar por completo: el atajo `F4` (`table-sessions.component.ts:
206-211`), el botón "Aplicar descuento (F4)" y su popover (`pos-order-panel.component.ts:122,
151-167`), y las señales/métodos asociados en el store (`discountPanelOpen`, `discountType`,
`discountValue`, `discountReason`, `appliedDiscount`, `toggleDiscountPanel`/`setDiscountType`/
`applyDiscount`/`cancelDiscount`, `pos-terminal.store.ts:226-231,1181-1196`) — `totals()` deja de
sumar ningún descuento manual, solo el que ya calcula el motor de promociones. En el backend,
endurecer el campo `discount` de `CheckoutAndSendIn`
(`pos-backend/app/api/v1/orders/schemas.py:265`) de `Field(0, ge=0, ...)` a
`Field(0, ge=0, le=0, ...)` — Pydantic rechaza con `422` cualquier valor distinto de cero en la
validación del propio esquema, sin lógica adicional en el handler.

**Razón**: se confirmó que el descuento del popover F4 viaja **exclusivamente** hacia
`checkout_and_send` — es el único consumidor: `pos-terminal.store.ts:1319-1327` solo envía
`discount` (si `> 0`) dentro del payload de `store.checkoutAndSend()`, que llama a
`api.checkoutAndSend(...)` → `POST /orders/{id}/checkout-and-send`. Ocultar el control en el
frontend no es suficiente por sí solo para una prohibición verificable (SC-003/SC-004 del spec
exigen "sin ninguna entrada manual", no solo "sin botón visible") — un cliente desactualizado o una
llamada directa a la API podría seguir enviando un valor. Enforzar `le=0` en el esquema de ese
único endpoint es el cambio mínimo que lo hace verdaderamente imposible, sin tocar el campo
`discount` compartido de `sales/schemas.py` (usado también por `create_sale`/`pay_order` de
mostrador y cierre de mesa unificado/dividido) — ese campo pertenece al mecanismo que **spec 011**
ya es dueña de implementar para los otros tres caminos de cobro (A-11, FR-016 de esa spec); tocarlo
aquí mezclaría el alcance de dos specs distintas sobre el mismo campo compartido (Principio VI).

**Alternativas consideradas**:
- *Endurecer también el campo compartido `discount` de `sales/schemas.py` (afectando
  `create_sale`/`pay_order` de una vez)*: descartada — implementaría de facto la regla A-11 para
  mostrador/cierre unificado/dividido, que es explícitamente el alcance de spec 011, no de esta
  spec (fuera de alcance, ver spec.md); aunque la decisión de negocio ya está tomada en spec 011,
  su implementación y verificación es un incremento separado (Principio VI — no mezclar el mismo
  campo compartido bajo la autorización de dos specs distintas en un solo cambio).
- *Solo ocultar el control en el frontend, sin cambio de backend*: descartada — no hace la
  prohibición verificable server-side (SC-004), deja abierta la vía de una llamada directa a la
  API.
- *Rechazar en el handler de `checkout_and_send` con una validación explícita (`if data.discount >
  0: raise HTTPException(...)`) en vez de en el esquema*: descartada por más código para el mismo
  resultado — `Field(..., le=0)` ya lo hace declarativamente, con un `422` estándar de Pydantic
  consistente con el resto de validaciones del mismo esquema.

## Resumen de Technical Context

| Aspecto | Backend (`pos-backend`) | Frontend (`pos-heladeria`) |
|---|---|---|
| Lenguaje/versión | Python 3.12, FastAPI 0.136.3, Pydantic 2.13 | TypeScript, Angular 21.1 (standalone, signals) |
| Dependencias clave | SQLAlchemy 2.0.50, Redis 8 (streams), Celery 5.6.3 + APScheduler | Tailwind CSS 4.1, sin librería de componentes UI |
| Almacenamiento | PostgreSQL 16, schema-per-tenant — **sin migración**: `paid` es un campo computado en la respuesta (subconsulta `EXISTS` sobre `sales.customer_order_id`), no una columna nueva | N/A |
| Testing | `unittest` stdlib, `app/characterization_tests/*.py`, SQLite en memoria; el existente `test_orders_kitchen.py` requiere actualización explícita (ver D3) | Vitest vía `ng test`; specs existentes relevantes: `pos-terminal.store.spec.ts` (`describe('deriveTableStatus', ...)`, líneas 66-133) y `pos-checkout-panel.component.spec.ts` (test T033 de "Reimprimir Factura POS", línea 148) |
| Target | API HTTP (Linux server) | Navegador — mismo diseño ya vigente en tablet táctil y escritorio (spec 026), sin cambios de layout en esta spec |
| Proyecto | Servicio web (API) | Aplicación web (SPA) |
