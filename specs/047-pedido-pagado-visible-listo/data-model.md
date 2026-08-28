# Data Model: Pedido de Mostrador Pagado Sigue Visible Hasta Liberar la Mesa

Esta spec **no agrega ni modifica entidades de backend ni de frontend** (spec.md, Key Entities). No
hay migración de base de datos, ni columna, ni endpoint nuevo. Este documento describe únicamente
qué campos ya existentes intervienen en el filtro que se corrige.

## Campos de dominio reutilizados (sin cambios)

| Campo | Origen | Uso en esta spec |
|---|---|---|
| `DiningOrder.status` | `dining.interface.ts` | Se sigue leyendo tal cual (`'pagada'`, `'cancelada'`, `'recibida'`); el fix solo cambia **cómo se combina** con el estado de cocina, no el campo en sí |
| `DiningOrder.items[].estado_cocina` | `dining.interface.ts` | Sigue alimentando `hasPendingKitchenWork()`/`KITCHEN_NOT_READY` allí donde ya se usaban (`deriveTableStatus`, `kitchenReady`, `ensureReadyToCharge`); deja de combinarse con `status === 'pagada'` dentro de `activeOrders`/`tableOrders` |
| `DiningOrder.paid` | `dining.interface.ts`, calculado en el backend a partir de si existe una `Sale` para el pedido (`order_has_sale`/`paid_order_ids`, `pos-backend/app/api/v1/orders/router.py:55,536-538`) | Sin cambios — sigue siendo la señal explícita que usa `deriveTableStatus()` para la rama `'listo'` |
| `TableSession.status` | Backend (`pos-backend/app/models/table_session.py`) | Sin cambios — sigue siendo la autoridad real sobre cuándo un pedido pagado deja de pertenecer a una mesa, vía el filtro `active_sessions_only` de `GET /orders` |

## Estado de presentación que se simplifica (frontend)

| Elemento | Ubicación hoy | Cambio |
|---|---|---|
| `activeOrders` (computed privado) | `pos-terminal.store.ts:377-384` | Se retira el conjunto `(o.status !== 'pagada' \|\| hasPendingKitchenWork(o))` — la condición de exclusión ya no depende del estado de cocina |
| `tableOrders(tableId)` (método privado) | `pos-terminal.store.ts:401-408` | Mismo cambio — un pedido `'pagada'` cuenta como consumo vivo de la mesa sin importar la cocina |

No se agrega ningún signal, computed ni método nuevo — es estrictamente una simplificación de una
condición ya existente. Ningún consumidor (`centralState`, `tablesView`, `orderTabs`,
`resyncSelectedOrder`, `selectTable`, `ordersToCharge`, `billOrphan` — ver research.md §3) gana ni
pierde ninguna firma pública.

## Transiciones de estado relevantes

- **`activeOrders()`/`tableOrders(tableId)`**: antes, un pedido `'pagada'` transicionaba de
  "incluido" a "excluido" en el momento en que su último ítem pasaba a `estado_cocina: 'listo'`.
  Con el fix, esa transición ya no existe — un pedido `'pagada'` permanece incluido desde que se
  cobra hasta que su `TableSession` se cierra (backend, tras "Liberar Mesa"), momento en el que el
  backend deja de devolverlo en `GET /orders?active_sessions_only=true` y por lo tanto desaparece
  de `orders()` (la fuente de la que se derivan `activeOrders`/`tableOrders`) sin necesidad de
  ningún filtro adicional del lado del frontend.
- **`deriveTableStatus()` → `'listo'`**: sin cambios de código; ahora es alcanzable en producción
  para el caso "pagado + toda la cocina lista", porque `tableOrders()` por fin le entrega esa orden
  (antes la excluía justo en ese momento).
