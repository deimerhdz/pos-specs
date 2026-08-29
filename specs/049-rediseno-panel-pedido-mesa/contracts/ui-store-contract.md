# Contrato UI/Store: panel de pedido y panel de cuenta de la mesa

Sin API nueva (no hay backend en esta spec) — este documento fija el contrato entre
`PosTerminalStore` y los dos componentes que consume, para que `/speckit-tasks` pueda derivar tareas
verificables sin releer todo `research.md`/`data-model.md`.

## `PosTerminalStore` (`pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`)

### Se retira

| Miembro | Motivo |
|---|---|
| `newOrderOnTable(): void` | FR-001 — sin ningún control que lo invoque (research.md, D2) |

### Se modifica (misma firma pública, cambia el resultado o el alcance interno)

| Miembro | Antes | Después |
|---|---|---|
| `orderTabs` (computed) | `label: o.customer_name \|\| 'Pedido'` | `label: 'Pedido ' + (index + 1)` (data-model.md) |
| `marcarListo(orderId?: string): Promise<void>` | Sin parámetro, siempre `selectedOrder()` | Con `orderId` opcional; sin argumento preserva el comportamiento actual (research.md, D6) |
| `voidPersistedCombo(comboId: string): Promise<void>` | Busca el pedido dueño vía `selectedOrder()` | Busca el pedido dueño recorriendo `orders()` (research.md, D6) |
| `avanzarItem(key: string): Promise<void>` | Busca la línea en `cartView()` | Busca la línea en `tableItemsView` (nuevo, cubre todos los pedidos de la mesa) |
| `cartView` (computed) | Construye líneas persistidas inline | Delega en `persistedItemsView(selectedOrder())` (mismo resultado observable) |

### Se agrega

| Miembro | Tipo | Contrato |
|---|---|---|
| `showAllOrders` | `WritableSignal<boolean>` | `true` por defecto cuando la mesa tiene >1 pedido; se reinicia en `selectTable()` (data-model.md) |
| `ordersView` | `computed<OrderCardView[]>` | Una entrada por pedido de `ordersOfTable(selectedTableId())`, en el mismo orden que `orderTabs()` (data-model.md) |
| `selectedTableStatusMeta` | `computed<{label,chipClass} \| null>` | `null` sin mesa seleccionada; si no, `STATUS_META[deriveTableStatus(...)]` de la mesa seleccionada, independiente del filtro de la grilla (research.md, D7) |
| `persistedItemsView(order)` | método privado | Extracción sin cambio de comportamiento de la lógica ya existente en `cartView()` (research.md, D4) |
| `tableItemsView` | `computed<CartLine[]>` (privado o interno) | Todas las líneas persistidas de `ordersOfTable(selectedTableId())`; usado por `avanzarItem` |

### Sin cambios

`selectTable()`, `selectOrder()`, `customerName` (signal), `customerPlaceholder`, `totals()`,
`kitchenReady()`, `saveOrder()`, `openCatalog()`/`closeCatalog()`, `voidPersistedItem()`,
`hasDraft()`, `submitting()` — ninguno cambia de comportamiento; algunos simplemente dejan de
usarse desde el nuevo layout de cabecera (p. ej. el input que llamaba a `customerName.set(...)`
directamente desde el template se retira, pero el signal en sí sigue existiendo y sigue siendo la
fuente que lee `session-bill-panel`/`pos-checkout-panel`).

## `PosOrderPanelComponent` (`pos-order-panel.component.ts`)

**Cabecera** (reemplaza el bloque actual de líneas 36-70): una fila con número de mesa, chip de
`store.selectedTableStatusMeta()`, nombre de cliente en texto (`store.customerName() ||
store.customerPlaceholder()`, nunca un `<input>`) y el botón "← Volver" ya existente.

**Pestañas** (reemplaza `orderTabs()` + "+ Nuevo pedido" de las líneas 58-69): cuando
`store.orderTabs().length > 0`, una pestaña "Todos los pedidos (N)" (activa `showAllOrders()`)
seguida de una pestaña por `store.orderTabs()` (activa `!showAllOrders() && selectedOrderId ===
tab.id`, click → `store.showAllOrders.set(false); store.selectOrder(tab.id)`). Ningún control
"+ Nuevo pedido".

**Contenido**:
- `showAllOrders()` verdadero → itera `store.ordersView()`; cada tarjeta muestra `createdAtLabel`,
  la pastilla `pending` ("Pendiente"/"Listo"), sus `items` (pill de estado + acción "✓ Listo" por
  ítem, sin cambios de comportamiento) y, si `pending`, un botón "Marcar pedido listo" que llama
  `store.marcarListo(card.order.id)`. Sin "+ Agregar producto" en este modo (research.md, D5).
- `showAllOrders()` falso → una sola tarjeta para `store.selectedOrder()`, con el mismo contenido
  de hoy (catálogo embebido, "+ Agregar producto", "Guardar pedido"), **sin** las filas
  Subtotal/Descuento/Total (FR-002) — el botón "Marcar pedido listo" se mueve al pie de esa tarjeta
  (mismo comportamiento, ahora sin el contenedor de totales alrededor).

## `SessionBillPanelComponent` (`session-bill-panel.component.ts`)

Se agrega, entre el desglose por comensal (líneas 60-89) y la fila "Total" ya existente, dos filas
nuevas alimentadas por `billSummary()` (data-model.md): "Subtotal" y "Descuento" (esta última solo
si `billSummary()!.discount > 0`, mismo criterio de ocultar-en-cero que ya usaba el panel de
pedido). Ningún `@Input`/`@Output` cambia de forma; `readOnly`, `beforeCharge`, `charged`, etc.
siguen intactos (FR-005).
