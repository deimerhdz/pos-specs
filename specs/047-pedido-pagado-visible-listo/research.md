# Research: Pedido de Mostrador Pagado Sigue Visible Hasta Liberar la Mesa

Todos los puntos de la Technical Context del plan quedaron resueltos por investigación directa del
código de `pos-heladeria` y `pos-backend`, hecha **antes** de escribir la spec (ver spec.md,
Input/Assumptions) — no había ninguna ambigüedad de negocio pendiente. Este documento consolida esa
investigación en el formato Decisión/Justificación/Alternativas.

## 1. Causa raíz exacta y por qué el fix es simplemente quitar una condición

- **Decisión**: en `pos-terminal.store.ts`, quitar el conjunto `(o.status !== 'pagada' ||
  hasPendingKitchenWork(o))` tanto de `activeOrders` (computed privado, líneas 377-384) como de
  `tableOrders(tableId)` (método privado, líneas 401-408). Con el fix:

  ```ts
  private readonly activeOrders = computed(() =>
    this.orders().filter(
      (o) => o.status !== 'cancelada' && (o.status !== 'recibida' || o.channel !== 'qr'),
    ),
  );

  private tableOrders(tableId: string): DiningOrder[] {
    return this.orders().filter(
      (o) => o.dining_table_id === tableId && o.status !== 'cancelada',
    );
  }
  ```

- **Justificación**: `hasPendingKitchenWork(order)` (`order-status.util.ts`, líneas 114-116)
  devuelve `true` mientras algún ítem siga `'pendiente'`/`'en_preparacion'`, y `false` en cuanto
  todos quedan `'listo'`. La condición actual, `(status !== 'pagada' OR hasPendingKitchenWork)`,
  significa: "incluir mientras NO esté pagado, O esté pagado pero cocina siga trabajando" — es
  decir, excluye exactamente el momento en que un pedido pagado por adelantado (`checkout_and_send`,
  spec 035/028) termina de cocinarse. Esa condición se agregó en spec 035 (A-52) con el objetivo
  correcto de que un pedido `'pagada'` no desapareciera **mientras cocina todavía trabaja** (su
  propio docblock lo dice: "de lo contrario la mesa se vería libre con el pedido aún en
  preparación"), pero nunca contempló que, al escribirla como una disyunción con
  `hasPendingKitchenWork`, el pedido queda excluido justo al terminar de cocinar — el vacío que
  reporta el usuario.
- **Quién decide cuándo un pedido pagado deja de contar de verdad**: el backend, no el frontend.
  `pos-backend/app/api/v1/orders/service.py::list_orders(active_sessions_only=True)` (líneas
  77-108) solo excluye un pedido pagado una vez que su `TableSession.status != 'active'` — es decir,
  después de que `TableSessionService.release()` (`Liberar Mesa`) cierra la sesión de verdad. El
  frontend siempre pide `GET /orders?active_sessions_only=true` en `reload()` (confirmado en
  `dining-session.service.ts:39-42` y ejercitado por `pos-terminal.store.spec.ts:753`), así que no
  necesita duplicar ningún criterio adicional mirando el estado de cocina — el pedido ya
  desaparecerá de `orders()` por sí solo en cuanto la mesa se libere.
- **Alternativas consideradas**: reemplazar la condición por una más angosta (p. ej. mantenerla
  excluida solo si además la mesa ya fue liberada) — rechazada porque es exactamente lo que ya hace
  el backend vía `active_sessions_only`; agregar esa lógica también en el frontend duplicaría una
  fuente de verdad sin necesidad, y sería el mismo tipo de condición frágil que causó este bug.

## 2. Import de `hasPendingKitchenWork` y `deriveTableStatus`

- **Decisión**: retirar `hasPendingKitchenWork` del import de la línea 34
  (`import { KITCHEN_NOT_READY, hasPendingKitchenWork } from '../../orders/order-status.util';` →
  queda solo `KITCHEN_NOT_READY`). Confirmado por grep que, tras el fix, `hasPendingKitchenWork` no
  tiene más usos en `pos-terminal.store.ts` (los dos únicos eran las líneas 381 y 406, ambas
  retiradas); `KITCHEN_NOT_READY` sigue vivo en `kitchenReady()` (línea ~631) y `ensureReadyToCharge`
  (línea ~1250), sin cambios. No se toca `order-status.util.ts` — `hasPendingKitchenWork` se queda
  definida ahí (no tiene otro consumidor en `pos-heladeria/src/`, pero retirar la función misma no
  es necesario ni parte de este bug: es una utilidad de dominio genérica, no exclusiva de este
  filtro).
- **`deriveTableStatus()`** (mismo archivo, líneas 154-190) **no cambia su código** — su rama
  `'listo'` (líneas 165-179) ya estaba correctamente escrita, exigiendo `paid === true` en todas las
  órdenes con consumo. El problema nunca fue esa función: era que `tableOrders()` nunca le entregaba
  la orden pagada-y-lista para evaluar. Con el fix, esa rama por fin es alcanzable en producción. Su
  comentario (líneas 168-176) sí queda desactualizado — cita `hasPendingKitchenWork` como la razón
  por la que puede llegar una orden `'pagada'` con ítems `'listo'`; se recomienda refrescarlo en el
  mismo cambio (texto, sin riesgo) para no inducir a error al siguiente que lea el archivo.

## 3. Cada consumidor de `activeOrders`/`tableOrders`/`ordersOfTable` — verificado uno por uno

- **`ordersOfTable(tableId)`** (líneas 387-389, filtra `activeOrders()` por mesa): con el fix, un
  pedido `'pagada'` de la mesa entra sin importar la cocina — es exactamente lo que necesita para
  seguir siendo editable/seleccionable.
- **`orderTabs`** (líneas 454-461, pestañas cuando la mesa tiene >1 pedido activo): usa
  `ordersOfTable`. Antes, si el pedido pagado desaparecía, dejaba de contar como pestaña; con el fix
  sigue apareciendo — correcto, sigue siendo un pedido real de la mesa.
- **`resyncSelectedOrder()`** (líneas 832-841, spec 044): usa `ordersOfTable(tableId)` para decidir
  si `selectedOrderId` sigue vigente tras un `reload()`. Es el mecanismo exacto que produce el
  síntoma reportado: si el pedido seleccionado era `'pagada'` y cocina terminaba durante el mismo
  `reload()` (el que dispara `marcarListo()`), `list.some((o) => o.id === current)` daba `false` y
  la función reseteaba la selección — a otro pedido de la mesa, o a `null` si no quedaba ninguno. Con
  el fix, `current` sigue en la lista y la selección no se toca.
- **`selectTable(tableId)`** (líneas 849-864): mismo mecanismo — con el fix selecciona
  correctamente el pedido pagado en vez de caer a "sin pedido".
- **`ordersToCharge()`** (líneas 1288-1296): con `sessionBill()` cargado, filtra `orders()`
  directamente por `bill.order_ids` — no pasa por `tableOrders`/`activeOrders`, no cambia. En el
  fallback sin `bill`, usa `ordersOfTable(tableId)`; con el fix puede incluir una orden `'pagada'`
  ya lista, pero `ensureReadyToCharge()` (línea 1249) filtra después por
  `KITCHEN_NOT_READY.includes(...)` — una orden pagada con todo `'listo'` no tiene ítems en ese
  conjunto, así que no genera ningún llamado extra a `markOrderReady` ni cambia el flujo de cobro.
- **`billOrphan`** (`loadSessionBill`, línea 1353): `!session && this.tableOrders(tableId).some((o)
  => !o.paid)`. Indiferente a este fix: la condición solo mira órdenes **sin pagar** (`!o.paid`);
  este cambio únicamente afecta si las órdenes **pagadas** se incluyen o no en `tableOrders()`. Una
  orden pagada nunca satisface `!o.paid`, así que no contribuye a `billOrphan` ni antes ni después.
- **`centralState`** (líneas 432-441): usa `tableOrders(tableId).length > 0`. Antes, con el pedido
  pagado ya listo, `tableOrders` devolvía `[]` → `centralState` caía a `'mesa-libre'` (el bug). Con
  el fix, la orden sigue en la lista → `centralState` se queda en `'pedido'`.
- **`tablesView`** (líneas 463-505): usa `tableOrders(t.id)` tanto para `deriveTableStatus` como
  para el conteo de ítems/subtotal/hora más antigua de la tarjeta. Con el fix, la tarjeta vuelve a
  reflejar la cuenta pendiente de liberar, y su estado puede llegar a `'listo'` (badge "Listo") en
  vez de quedarse en `'ocupada'`.
- **`pendingOrders`/`pendingOfSelectedTable`**: no dependen de `activeOrders`/`tableOrders`, sin
  cambios — el flujo de pedidos QR pendientes de confirmar no se ve afectado.

No se encontró ningún consumidor donde la desaparición de un pedido pagado-y-listo fuera el
comportamiento deseado — en todos los casos revisados era exactamente el bug reportado.

## 4. Cobertura de tests

- **Decisión**: `pos-terminal.store.spec.ts` ya prueba la rama `'listo'` de `deriveTableStatus()`
  directamente (`describe('"Listo" exige pago (spec 029)')`, líneas 150-174), pero con arrays de
  pedidos armados a mano usando `status: 'abierta'` — el supuesto previo a spec 035, que nunca pasa
  por `tableOrders()`/`activeOrders()`. Esos tests **no se tocan** (siguen siendo válidos como
  prueba pura de `deriveTableStatus`). Se agregan dos bloques nuevos, siguiendo los patrones ya
  usados en el mismo archivo:
  1. Integración `centralState()`/`tablesView()` sin HTTP (mismo patrón que
     `describe('PosTerminalStore.pendingOrders — solo canal qr')`, líneas 282-308): una orden
     `status: 'pagada'`, `paid: true`, todos los ítems `'listo'`, canal `counter`, puesta
     directamente vía `store.orders.set(...)` + `store.selectedTableId.set(...)` — asegura que
     `centralState()` sea `'pedido'` (no `'mesa-libre'`) y que la fila de `tablesView()` para esa
     mesa tenga `statusLabel: 'Listo'` (no `'Ocupada'`). Requiere inyectar `TableService` y poner
     `tables.set([...])` con una fila mínima para la mesa de prueba.
  2. `marcarListo()` de punta a punta (mismo patrón HTTP que
     `describe('PosTerminalStore.ensureReadyToCharge')`, líneas 707-770, y el mismo `tick()` que
     `describe('PosTerminalStore.reload — resincroniza la selección...')`, líneas 626-696): una
     orden `'pagada'` con un ítem `'pendiente'`, se llama `store.marcarListo()`, se responde
     `POST /orders/{id}/ready` y el `reload()` subsiguiente (`/orders/tables`,
     `/orders?active_sessions_only=true`, `/table-sessions`) con la orden ya `'listo'` — se
     confirma que `selectedOrderId()` sigue en el mismo pedido (no lo resetea
     `resyncSelectedOrder()`) y que `centralState()` sigue en `'pedido'`.
- **Justificación**: Principio X (verificación obligatoria) exige cerrar precisamente el gap que
  dejó pasar este bug — ningún test existente ejercitaba `status: 'pagada'` a través de
  `tableOrders()`/`activeOrders()`/`centralState()`/`marcarListo()` juntos. No se necesita ningún
  test de backend nuevo: `pos-backend` no se toca en esta spec.
