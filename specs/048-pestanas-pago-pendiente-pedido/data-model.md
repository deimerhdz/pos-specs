# Data Model: Pestañas para Ver el Pedido Pagado Junto al Pago Pendiente de la Misma Mesa

Esta spec **no agrega ni modifica entidades de backend** (spec.md, Key Entities). No hay migración
de base de datos, ni columna, ni endpoint nuevo. Este documento describe únicamente el estado de
**frontend** que se agrega — sigue siendo estado de presentación, no un modelo de datos persistente.

## Campos de dominio reutilizados (sin cambios)

| Campo/Computed | Origen | Uso en esta spec |
|---|---|---|
| `pendingOfSelectedTable` | `pos-terminal.store.ts` (computed, ya existente) | Insumo de `hasPendingAndActiveOrders` — sin cambios propios |
| `ordersOfTable(tableId)` (privado) | `pos-terminal.store.ts` (método, ya existente) | Insumo de `hasPendingAndActiveOrders` — sin cambios propios |
| `centralState` | `pos-terminal.store.ts` (computed, ya existente) | Insumo de `effectiveCentralView` cuando no hay ambos tipos de pedido — sin cambios propios |

## Estado nuevo de presentación (frontend, `pos-terminal.store.ts`)

| Signal / Computed | Tipo | Descripción | Regla |
|---|---|---|---|
| `centralPanelTab` | `signal<'validar-pago' \| 'pedido'>` | Pestaña elegida por el cajero cuando la mesa tiene ambos tipos de pedido a la vez (FR-001/FR-004) | Por defecto `'validar-pago'`; único punto de escritura: los dos botones de pestaña del encabezado; se reinicia a `'validar-pago'` dentro de `resetTransient()` (al seleccionar otra mesa o deseleccionar) |
| `hasPendingAndActiveOrders` | `computed<boolean>` | ¿La mesa seleccionada tiene a la vez algún pago pendiente de confirmar y algún pedido pagado/activo? (FR-001/FR-005) | `true` solo cuando `pendingOfSelectedTable().length > 0 && ordersOfTable(tableId).length > 0`; determina si aparecen las pestañas |
| `effectiveCentralView` | `computed<'validar-pago' \| 'mesa-libre' \| 'pedido'>` | Qué debe renderizar el panel central (FR-002/FR-003/FR-005) | `centralPanelTab()` cuando `hasPendingAndActiveOrders()` es `true`; `centralState()` sin cambios en cualquier otro caso |

## Transiciones de estado relevantes

- **`centralPanelTab`**: `'validar-pago' ⇄ 'pedido'`, a elección manual del cajero (clic en el
  botón correspondiente) mientras `hasPendingAndActiveOrders()` sea `true`; vuelve a `'validar-pago'`
  automáticamente al cambiar de mesa o deseleccionar (`resetTransient()`), nunca mientras la mesa
  siga siendo la misma (FR-006, Edge Cases — un pago nuevo que llegue no saca al cajero de "Pedido
  de la mesa" si ya estaba ahí).
- **`hasPendingAndActiveOrders`**: `false → true` en cuanto la mesa gana simultáneamente un pago
  pendiente y un pedido pagado/activo (o viceversa, al perder uno de los dos); es puramente
  reactivo a `orders()`, sin ningún efecto secundario ni escritura.
- **`effectiveCentralView`**: sigue mecánicamente a `centralPanelTab()` o a `centralState()` según
  el punto anterior — no introduce ningún estado propio, es una función pura de los otros dos.
