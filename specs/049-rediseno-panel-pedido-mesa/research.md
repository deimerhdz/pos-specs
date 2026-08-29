# Research: Rediseño del panel de pedido de mesa — cliente, pedidos y cuenta

Todas las incógnitas de esta spec son de diseño técnico dentro del frontend `pos-heladeria`
(Angular, módulo `tables`); no hay dependencias externas nuevas que investigar. Cada decisión cita
el código real leído para tomarla.

## D1 — Cómo mostrar el resumen Subtotal/Descuento/Total en el panel de cuenta

**Decision**: Agregar dos `computed` (o un único `computed` que devuelva `{subtotal, discount}`)
dentro de `SessionBillPanelComponent` que sumen `bill.split[].subtotal` y `bill.split[].discount`
(ambos ya presentes como `string` en `SessionBillLine`, `dining.interface.ts:289-296`), y renderizar
dos filas nuevas ("Subtotal", "Descuento") justo antes de la fila "Total" ya existente
(`session-bill-panel.component.ts:83-88`), con el mismo formato `number: '1.2-2'` que ya usa el
resto del panel.

**Rationale**: `SessionBill` (`dining.interface.ts:300-305`) no expone un subtotal/descuento
agregado a nivel de cuenta — solo el desglose por comensal (`split`) y el `total` final. Sumar esas
dos columnas client-side es aritmética pura sobre datos que el backend ya entrega; no cambia ninguna
regla de cálculo (Principio VII/XI: esto es visualización, no una decisión de negocio nueva sobre
cómo se calcula el descuento).

**Alternatives considered**:
- *Pedir al backend un campo agregado nuevo en `SessionBill`*: rechazado — exige un cambio de
  contrato de API para un dato que ya se puede derivar sin red adicional; violaría Principio VI
  (mezclar un cambio de backend con uno de UI en el mismo incremento) sin necesidad.
- *Reusar `store.totals()` (el mismo computed que hoy alimenta el panel de pedido)*: rechazado —
  `store.totals()` deriva de `cartView()`, que solo refleja el pedido **seleccionado**, no la mesa
  completa; el panel de cuenta necesita el total de **todos** los pedidos de la sesión, que es
  justamente lo que ya agrega `bill` (el mismo dato que hoy arma su único "Total").

## D2 — Cómo dejar de ofrecer "+ Nuevo pedido"

**Decision**: Quitar el botón (`pos-order-panel.component.ts:67`) y el método
`newOrderOnTable()` (`pos-terminal.store.ts:918-922`) por completo, en vez de dejarlo sin usar.

**Rationale**: `newOrderOnTable()` no tiene ningún otro llamador en `pos-heladeria/src` (verificado
por búsqueda exhaustiva) — dejar el método muerto violaría el criterio de higiene ya aplicado en
specs previas (p. ej. 045 retiró por completo el carrito embebido de mesa libre en vez de dejarlo
inalcanzable). Es limpieza directamente causada por este FR, no una refactorización oportunista no
relacionada (Principio V).

**Alternatives considered**: dejar el método sin usar "por si se necesita después" — rechazado,
código muerto sin spec que lo respalde no se justifica solo por posible reutilización futura.

## D3 — Estado agregado por tarjeta de pedido ("Entregado"/"Pendiente" en la imagen de referencia)

**Decision**: La pastilla de estado que corona cada tarjeta de pedido reutiliza
`hasPendingKitchenWork(order)` (`order-status.util.ts:114-116`, ya usado por spec 035/A-52) y las
etiquetas/clases ya existentes en `KITCHEN_STATUS` (`order-status.util.ts:61-66`): "Pendiente"
(ámbar) si `hasPendingKitchenWork(order)` es verdadero, "Listo" (verde) si es falso. No se introduce
la palabra "Entregado" ni ningún estado nuevo.

**Rationale**: la imagen de referencia es un mockup de layout, no una fuente de vocabulario de
negocio — introducir un cuarto rótulo ("Entregado") sin backend que lo respalde crearía un
significado nuevo no verificable contra ningún dato real. `hasPendingKitchenWork` ya resuelve
exactamente la pregunta que la tarjeta necesita responder ("¿a este pedido todavía le falta algo
por preparar?") para *cualquier* pedido de la mesa, no solo el seleccionado — es la generalización
que ya pedía spec.md, Assumptions ("el detalle exacto... se resuelve en la fase de planeación
técnica").

**Alternatives considered**:
- *Reusar `displayOrderStatus`/`orderStatusLabel` (Por confirmar/Abierta/Pagada/Cancelada)*:
  rechazado como estado principal de la tarjeta — responde "¿se cobró?", no "¿está listo en
  cocina?", que es lo que la imagen de referencia comunica visualmente (verde = ya no requiere
  acción de cocina).
- *Introducir "Entregado" como cuarto estado de `KitchenStatus`*: rechazado — cambiaría el modelo
  de datos de cocina (Principio VIII) sin ninguna necesidad de negocio declarada; "listo" ya
  significa exactamente eso para el backend.

## D4 — Ver los ítems de **todos** los pedidos de la mesa a la vez (pestaña "Todos los pedidos")

**Decision**: Extraer la construcción de líneas de ítems persistidos que hoy vive inline dentro de
`cartView()` (`pos-terminal.store.ts:566-617`: filtro de anulados, `plainItems`, `comboGroups`,
`persistedCombos`) a un método privado `persistedItemsView(order: DiningOrder): CartLine[]`, sin
tocar su lógica de precios/descuento/promoción. `cartView()` pasa a ser
`persistedItemsView(selectedOrder) + draft` (mismo resultado que hoy, cero cambio observable).
Se agrega un `computed` nuevo, `ordersView`, que mapea cada pedido de
`ordersOfTable(selectedTableId)` a `{ order, items: persistedItemsView(order), createdAtLabel,
pending: hasPendingKitchenWork(order) }`, para alimentar tanto la pestaña "Todos los pedidos" (itera
`ordersView()` completo) como cada pestaña individual "Pedido N" (filtra `ordersView()` por id).

**Rationale**: `cartView()` está atado a `selectedOrder()` a propósito (una sola orden a la vez);
la vista agregada necesita la misma lógica de líneas (incluyendo agrupación de combos y precio con
descuento de promoción vigente) pero para **varios** pedidos simultáneamente. Duplicar esa lógica
en el componente sería el mismo riesgo que ya evita `session-bill-panel.component.ts` al no
recalcular precios por su cuenta: la lógica de combos/promociones (comentarios A-09, spec 029) es
frágil y ya vive en un solo lugar; extraerla a un método privado reutilizado mantiene esa propiedad.

**Alternatives considered**:
- *Duplicar la lógica de `plainItems`/`persistedCombos` dentro del componente para la vista
  agregada*: rechazado — duplica lógica de descuento por promoción ya señalada como delicada
  (comentarios A-09 en el propio archivo), doblando el riesgo de que ambas copias diverjan.
- *Pedir al backend un endpoint que devuelva los pedidos ya combinados*: rechazado — no hay ninguna
  necesidad de negocio nueva que lo justifique (Principio IX/XI); toda la información ya llega hoy
  vía `orders()`/`ordersOfTable()`.

## D5 — Qué pestaña ("Todos los pedidos" vs. "Pedido N") controla qué acciones

**Decision**: Se agrega un signal de solo-UI, `showAllOrders`, además del ya existente
`selectedOrderId`. Cuando la mesa tiene más de un pedido, `showAllOrders` inicia en `true`
(coincide con el mockup: "Todos los pedidos" activa por defecto). Elegir "Todos los pedidos" pone
`showAllOrders.set(true)`; elegir una pestaña "Pedido N" hace `showAllOrders.set(false)` y reusa
`store.selectOrder(id)` tal cual existe hoy (sin cambios de comportamiento en `selectOrder`).
`showAllOrders` se reinicia junto con el resto del estado transitorio en `selectTable()`
(`pos-terminal.store.ts:895-910`), igual que hace spec 048 con `centralPanelTab` en
`resetTransient()`.

Las acciones de **agregar** ítems ("+ Agregar producto", catálogo embebido, "Guardar pedido") solo
se ofrecen dentro de una pestaña "Pedido N" concreta (`!showAllOrders()`) — necesitan un pedido
único e inequívoco al que agregar líneas, igual que hoy. Las acciones que ya identifican su target
por id propio — marcar listo un ítem (`avanzarItem`), anular ítem/combo
(`voidPersistedItem`/`voidPersistedCombo`) y marcar el pedido completo como listo
(`marcarListo`) — se ofrecen en **ambas** vistas, una vez por tarjeta.

**Rationale**: coincide con la Clarification "se mantiene por ítem" de spec.md (FR-014): las
acciones de preparación no dependen de cuál pestaña esté activa, solo de a qué pedido pertenece
cada tarjeta. Restringir "+ Agregar producto" a una pestaña individual evita la ambigüedad de "a
cuál de los N pedidos visibles se le está agregando este producto" cuando "Todos los pedidos" está
activa.

**Alternatives considered**:
- *Permitir agregar productos también desde "Todos los pedidos", eligiendo el pedido destino con un
  selector adicional*: rechazado — no lo pidió el usuario, agrega una decisión de UX no solicitada
  (Principio V, no anticipar requisitos hipotéticos) y complica el flujo actual de un solo clic.

## D6 — Generalizar `marcarListo`/`voidPersistedCombo`/`avanzarItem` para operar sobre cualquier pedido de la tarjeta, no solo el seleccionado

**Decision**:
- `marcarListo(orderId?: string)`: si se pasa `orderId`, opera sobre ese pedido
  (`this.orders().find(o => o.id === orderId)`); sin argumento, preserva el comportamiento actual
  (`this.selectedOrder()`). Cada tarjeta de pedido invoca `store.marcarListo(card.order.id)`.
- `voidPersistedCombo(comboId)`: busca el pedido dueño del combo recorriendo `this.orders()`
  (`items.some(i => i.combo_id === comboId)`) en vez de asumir `this.selectedOrder()`. Un
  `combo_id` es único dentro de su pedido, así que la búsqueda es igual de segura.
- `avanzarItem(key)`: busca la línea en un nuevo computed `tableItemsView` (todas las líneas
  persistidas de `ordersOfTable(selectedTableId)`, construido con `persistedItemsView` de D4) en
  vez de limitarse a `cartView()` (que solo cubre el pedido seleccionado).

**Rationale**: en la pestaña "Todos los pedidos" hay tarjetas de pedidos que **no** son
`selectedOrder()`; sin esta generalización, sus botones "✓ Listo"/"Anular"/"Marcar pedido listo"
operarían silenciosamente sobre el pedido equivocado (el seleccionado) en vez del pedido de la
tarjeta donde se hizo clic — un bug de origen, no un caso límite.

**Alternatives considered**: hacer que un clic en cualquier acción de una tarjeta ajena primero
llame a `store.selectOrder(card.order.id)` y luego a la acción sin argumento — rechazado: son dos
pasos con un estado intermedio observable (parpadeo de "pedido seleccionado") para lo que debería
ser una sola operación atómica; pasar el id directamente es más simple y no introduce estado
intermedio.

## D7 — Encabezado con estado de la mesa ("Ocupada") en modo lectura

**Decision**: Nuevo `computed` `selectedTableStatusMeta` en `pos-terminal.store.ts`, construido con
las mismas funciones ya usadas por `tablesView()` (`deriveTableStatus` + `STATUS_META`,
`pos-terminal.store.ts:101-109` y `:524-525`) pero aplicado directamente a `selectedTable()` /
`tableOrders(id)`, sin pasar por el `filter()`/`search()` de la grilla (que podría excluir la mesa
seleccionada de `tablesView()` si el cajero tiene un filtro activo).

**Rationale**: `STATUS_META` ya tiene exactamente la etiqueta ("Ocupada") y la clase de color que
pide la imagen de referencia; reconstruirlo desde cero duplicaría una tabla de mapeo que ya existe.
Depender de `tablesView()` sería frágil porque ese computed está pensado para la grilla filtrada,
no para "la mesa actualmente seleccionada, sin importar el filtro".

**Alternatives considered**: filtrar `tablesView()` por `t.id === selectedTableId()` — rechazado
por la razón de arriba (puede no estar en la lista filtrada).

## Resumen de impacto en tests existentes (Principio X)

0 tests con prefijo `"CONGELA comportamiento actual:"` en `pos-heladeria/src/` (mismo hallazgo que
specs 045/046/047/048) — ningún characterization test protegido bloquea este cambio.

`pos-order-panel.component.spec.ts` sí tiene un test no protegido que deja de aplicar tal cual está
escrito: `'el descuento mostrado en el total es siempre $0 sin promociones activas'`
(línea ~121, dentro de `describe('PosOrderPanelComponent — sin descuento manual')`), porque afirma
sobre una fila "Descuento" que este spec retira de este componente (FR-002). Se debe mover/adaptar
ese caso a `session-bill-panel.component.spec.ts` (donde la fila "Descuento" pasa a vivir, FR-003)
al momento de implementar — detalle para `/speckit-tasks`, no bloquea la planeación.
