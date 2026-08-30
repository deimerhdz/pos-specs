# Phase 1 Data Model: Carga diferida de datos y tarjetas de pedido de Domicilio/Para Llevar

Este feature **no agrega ni modifica ninguna entidad persistida** (spec, Out of Scope: "Cualquier
cambio de backend"). Los únicos modelos nuevos son de vista/estado en el frontend
(`pos-heladeria`), derivados de datos ya existentes en `DiningOrder`. Este documento describe esos
modelos derivados y cómo se relacionan con la entidad existente.

## Entidad existente reutilizada: `DiningOrder`

Sin cambios de forma. Campos relevantes para este feature (ya presentes desde spec 055/056,
`interfaces/dining.interface.ts:195-215`):

| Campo | Tipo | Uso en este feature |
|---|---|---|
| `id` | `string` (UUID) | Identidad de selección (`selectedOrderId`); nunca se muestra como "número de orden" (ver research.md §5). |
| `order_type` | `'DINE_IN' \| 'TAKEAWAY' \| 'DELIVERY' \| null` | Filtro de pertenencia a pestaña ("Domicilios" = `DELIVERY`, "Para llevar" = `TAKEAWAY`). |
| `dining_table_id` | `string \| null` | `null` para todo pedido `TAKEAWAY`/`DELIVERY` (spec 055/056) — señal de "pedido sin mesa". |
| `customer_name` | `string \| null` | Línea secundaria de la tarjeta y del panel de detalle. |
| `delivery_address` / `delivery_phone` / `delivery_fee` | `string \| null` / `string \| null` / `number \| null` | Solo con `order_type === 'DELIVERY'`; se muestran en el panel de detalle (FR-012), no en la tarjeta (no hay espacio ni lo pidió el spec). |
| `paid` | `boolean` | Determina si el pedido sigue "pendiente de cobro" (visible en su pestaña) o ya se cobró (se oculta, FR-008). |
| `status` | `DiningOrderStatus` | Junto con los `estado_cocina` de sus ítems, alimenta la insignia de estado reutilizada (`deriveTableStatus`). |
| `items` | `DiningOrderItem[]` | Cuenta de productos y total de la tarjeta (mismo cálculo que `tablesView()`, incluyendo descuentos por promoción). |

**Regla de visibilidad en su pestaña** (deriva de spec 036, ver research.md §4): un pedido
`TAKEAWAY`/`DELIVERY` es candidato a tarjeta cuando `order.order_type` coincide con la pestaña
activa **y** `!order.paid && order.status !== 'cancelada'`.

## Modelo de vista nuevo: `OrderSummaryCardView`

Contrato plano que consume el componente presentacional nuevo
(`order-summary-card.component.ts`, ver `contracts/ui-contracts.md`). Un mismo shape para las dos
fuentes de datos (mesas y pedidos sin mesa):

| Campo | Tipo | Origen para una **mesa** (`tablesView()`, sin cambios) | Origen para un **pedido sin mesa** (nuevo) |
|---|---|---|---|
| `id` | `string` | `table.id` | `order.id` |
| `title` | `string` | `"Mesa {number}"` | `"Domicilio"` / `"Para llevar"` (según `order_type`) |
| `statusLabel` / `statusClass` | `string` / `string` | `STATUS_META[deriveTableStatus(tableOrders, table.status)]` | `STATUS_META[deriveTableStatus([order], 'ocupada')]` (research.md §5) |
| `secondaryLabel` | `string` | `itemsLabel` ("N productos") | `customer_name \|\| 'Consumidor final'` |
| `elapsedLabel` | `string` | hora relativa desde el pedido más antiguo de la mesa | hora relativa desde `order.created_at` |
| `totalLabel` | `string` | suma de `orderSubtotal()` de los pedidos de la mesa | `orderSubtotal(order)` (mismo cálculo, un solo pedido) |
| `ordersCount` | `number \| undefined` | cantidad de pedidos activos de la mesa (solo si > 1) | `undefined` (un pedido sin mesa es siempre singular) |
| `selected` | `boolean` | `table.id === selectedTableId()` | `order.id === selectedOrderId() && !selectedTableId()` |

Este modelo **no se persiste** — es un `computed()` derivado en `PosTerminalStore`, recalculado a
partir de `tables()`/`orders()` ya cargados (mismo patrón que `tablesView()` hoy).

## Extensión al estado de selección de `PosTerminalStore`

No se agrega ninguna entidad — se documentan aquí los signals/computed que cambian de contrato de
comportamiento (implementación en tasks, no en este documento):

| Signal/Computed | Antes | Después |
|---|---|---|
| `selectedTableId` | Única fuente de "hay algo seleccionado" | Sigue significando "hay una mesa de por medio"; puede volver a `null` sin que eso implique "nada seleccionado" cuando hay un pedido sin mesa activo. |
| `selectedOrderId` | Se fija junto con `selectedTableId` (`selectTable()`) o al alternar entre pedidos de una misma mesa (`selectOrder()`) | Además puede fijarse **sin** ninguna mesa asociada, vía el nuevo punto de entrada para pedidos Domicilio/Para llevar (research.md §3). |
| `hasActiveSelection` *(nuevo)* | No existía (`hasActiveOrder` solo miraba `selectedTableId`) | `computed(() => !!selectedTableId() \|\| !!selectedOrderId())` — reemplaza a `hasActiveOrder` como condición de la que dependen `centralState()`/`effectiveCentralView()` y el placeholder de `pos-order-panel.component.ts`. |
| `ordersByType('domicilios' \| 'para-llevar')` *(nuevo)* | No existía | `computed()` que filtra `orders()` por `order_type` + "pendiente de cobro" (ver arriba) y los mapea a `OrderSummaryCardView[]`. |

## Diagrama de dependencias (derivación, no persistencia)

```text
orders() [signal, ya existente — reloadOrders()]
   │
   ├─▶ tableOrders(tableId) ─▶ tablesView() ──────────────┐
   │                                                        ├─▶ OrderSummaryCardView[]
   └─▶ ordersByType('domicilios' | 'para-llevar') [nuevo] ─┘      (consumidos por
                                                                    order-summary-card.component.ts)

selectedTableId + selectedOrderId ─▶ hasActiveSelection [nuevo] ─▶ centralState() / effectiveCentralView()
                                                                     └─▶ pos-order-panel.component.ts
                                                                          (detalle, con o sin mesa)
```

## Reglas de validación (heredadas, sin cambios)

Ninguna regla de validación de negocio nueva — este feature es de visualización y momento de
carga. Las reglas ya existentes que siguen aplicando sin modificación:
- Un pedido `DELIVERY` siempre tiene `delivery_address`/`delivery_fee` no nulos (garantizado por
  spec 056 al crearlo) — la tarjeta y el panel de detalle asumen esa garantía, no la revalidan.
- Un pedido `TAKEAWAY`/`DELIVERY` nunca tiene `dining_table_id` (garantizado por spec 055/056,
  reforzado en backend) — el mapeo a `OrderSummaryCardView` no necesita manejar el caso "pedido de
  Para llevar con mesa asignada".
