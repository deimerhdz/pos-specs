# Data Model: Rediseño del panel de pedido de mesa — cliente, pedidos y cuenta

Esta spec **no agrega ni modifica entidades de backend** (spec.md, Assumptions): `DiningOrder`,
`DiningOrderItem` y `SessionBill`/`SessionBillLine` (`pos-heladeria/src/app/modules/tables/
interfaces/dining.interface.ts`) se reutilizan tal cual. Lo que sigue son **vistas derivadas**
(computeds de `PosTerminalStore` y de `SessionBillPanelComponent`) nuevas o modificadas para esta
pantalla — no persisten en ningún backend ni cambian ningún esquema.

## Vistas nuevas/modificadas en `pos-terminal.store.ts`

### `CartLine` (tipo ya existente, sin cambios de forma)

Sigue siendo el resultado de por-línea que hoy ya produce `cartView()` (`kind`, `key`, `comboId`,
`qty`, `name`, `bullets`, `unitPrice`, `subtotal`, `ready`, `kitchenStatus`, `pendingItemIds`). Su
construcción para líneas persistidas se extrae a `persistedItemsView(order)` (research.md, D4);
`cartView()` sigue devolviendo exactamente lo mismo que hoy para el pedido seleccionado.

### `OrderCardView` (nuevo)

| Campo | Tipo | Origen |
|---|---|---|
| `order` | `DiningOrder` | tal cual, de `ordersOfTable(tableId)` |
| `items` | `CartLine[]` | `persistedItemsView(order)` (D4) — sin líneas `draft` |
| `createdAtLabel` | `string` | formateo de `order.created_at` (hora local, mismo criterio que ya usa `elapsedLabel`/`fmt` en el store) |
| `pending` | `boolean` | `hasPendingKitchenWork(order)` (`order-status.util.ts`) — decide la pastilla "Pendiente"/"Listo" de la tarjeta (D3) |

`ordersView: OrderCardView[]` (computed) = `ordersOfTable(selectedTableId()).map(...)`, en el mismo
orden que ya devuelve `ordersOfTable`/`orderTabs()` hoy (cronológico de creación).

### Señales de estado de vista (solo UI, no persisten)

| Señal | Tipo | Default | Se reinicia en |
|---|---|---|---|
| `showAllOrders` | `signal<boolean>` | `true` cuando la mesa tiene >1 pedido activo | `selectTable()` (junto con `selectedOrderId`/`customerName`, igual que hoy) |

`selectedOrderId` (ya existente) sigue determinando cuál es "el pedido en edición" cuando
`showAllOrders() === false` — sin cambios de tipo ni de semántica.

### `orderTabs` (modificado)

Mismo computed ya existente (`pos-terminal.store.ts:500-507`), mismo criterio de visibilidad
(`length > 1`), pero la etiqueta pasa de `o.customer_name || 'Pedido'` a `Pedido ${i + 1}` (índice
1-based en el orden ya devuelto por `ordersOfTable`) — el nombre del cliente deja de repetirse por
pestaña porque ahora se muestra una sola vez en la cabecera (FR-008/FR-009).

### `selectedTableStatusMeta` (nuevo)

`{ label: string; chipClass: string } | null` — `null` sin mesa seleccionada; si no, el mismo par
`STATUS_META[deriveTableStatus(tableOrders(id), table.status)]` que ya usa `tablesView()`, pero
calculado directamente sobre la mesa seleccionada (research.md, D7).

## Vistas nuevas en `session-bill-panel.component.ts`

### `billSummary` (nuevo, computed sobre `currentBill`)

| Campo | Tipo | Cálculo |
|---|---|---|
| `subtotal` | `number` | `Σ Number(line.subtotal)` sobre `bill.split` |
| `discount` | `number` | `Σ Number(line.discount)` sobre `bill.split` |

`null` cuando `currentBill()` es `null` (mismo guard que ya usa el resto del template con `@if
(!bill)`). El `total` agregado sigue siendo `bill.total`, sin cambios — solo se le agregan estas
dos filas encima, no se reemplaza.

## Relaciones

```
DiningTable (seleccionada)
  └─ ordersOfTable(tableId): DiningOrder[]           (ya existente, sin cambios)
       ├─ orderTabs(): {id,label}[]                   (label cambia: "Pedido N")
       ├─ ordersView(): OrderCardView[]                (nuevo — una tarjeta por pedido)
       │    └─ items: CartLine[]                       (persistedItemsView(order), D4)
       └─ selectedOrder(): DiningOrder | null           (ya existente; vigente cuando
                                                          !showAllOrders())

SessionBill (bill.split: SessionBillLine[])
  └─ billSummary(): {subtotal, discount}                (nuevo, en SessionBillPanelComponent)
```

No hay migraciones, no hay estrategia de rollback de datos (Principio VIII, N/A) — todo lo anterior
es estado derivado en memoria del cliente, recalculado en cada render a partir de datos que el
backend ya entrega hoy.
