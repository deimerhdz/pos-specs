# Data Model: Rediseño responsive de la Terminal de Mesas

Ninguna entidad nueva, ninguna columna nueva, ninguna migración (ver spec.md, Out of Scope y
Assumptions; Constitution Check, Principio VIII — no aplica). Este documento describe las entidades
ya existentes que esta feature lee/expone de forma distinta, y los view-models nuevos que solo
existen en memoria (frontend), derivados de datos ya cargados.

## Entidades ya existentes (sin cambios de esquema)

### `DiningOrder` (backend: `CustomerOrder`, `app/models/customer_order.py`)

| Campo | Tipo | Nota |
|---|---|---|
| `id` | UUID | sin cambios |
| `user_id` | UUID \| null | **ya existe** (línea 115); referencia blanda a `shared.users.id`; nulo solo si el pedido lo envió el cliente por QR. Esta feature lo **expone por primera vez** hacia el frontend (hoy no viaja en `OrderResponse`). |
| `order_type`, `status`, `channel`, `customer_name`, `delivery_*`, `notes`, `created_at`, `items` | — | sin cambios (spec 055/056) |

**Regla de exposición (FR-021/FR-023)**: se resuelve `user_id` → nombre de usuario mediante un
lookup a `shared.users`, sin condicionarlo al rol de ese usuario (Cajero o Mesero, ver
Clarifications de spec.md). Si `user_id` es `null`, el campo expuesto también es `null` — no se
inventa ningún valor por defecto.

### `CashShift` (frontend: `cash.interface.ts:12-23`; backend: módulo de Caja, sin cambios)

| Campo | Tipo | Nota |
|---|---|---|
| `id`, `cash_register_id` | UUID | sin cambios |
| `user_id`, `user_name` | UUID / string \| null | ya existente y ya expuesto — se **lee** desde la Terminal de Mesas (Historia 3), no se modifica su modelo. |
| `opening_amount`, `opened_at`, `closed_at`, `counted_amount`, `status`, `close_note` | — | sin cambios |

**Regla de lectura (FR-015/FR-017)**: `status === 'open'` determina si la barra superior muestra el
botón de abrir turno (ausente) o el nombre del cajero de turno (presente). El segmento de horario
("Mañana"/"Tarde"/"Noche" — ver research.md §5) es un valor **derivado en el frontend** de
`opened_at`, no un campo persistido.

### `Table` / `TableDisplayStatus` (`pos-terminal.store.ts`, `deriveTableStatus()`)

Sin cambios de modelo ni de lógica de derivación. Esta feature solo agrega una vista agregada sobre
el conjunto ya existente (ver `OccupancySummary` abajo) y cambia el contenedor visual de sus
tarjetas.

## View-models nuevos (solo en memoria, frontend — no persistidos)

### `OccupancySummary` (Historia 2)

Derivado (`computed()`) de la misma señal que ya alimenta `tablesView()` en `pos-terminal.store.ts`.
No es una entidad de backend ni requiere endpoint nuevo.

| Campo | Tipo | Origen |
|---|---|---|
| `total` | number | cuenta de todas las mesas visibles |
| `ocupadas` | number | cuenta de mesas con `TableDisplayStatus` ∈ {`ocupada`, `en_preparacion`, `listo`} |
| `libres` | number | cuenta de mesas con `TableDisplayStatus === 'libre'` |
| `pendientes` | number | cuenta de mesas con `TableDisplayStatus` ∈ {`por_confirmar`, `pago_pendiente`} |

**Invariante (FR-011)**: `total`, `ocupadas`, `libres`, `pendientes` son la única fuente que
alimenta tanto el resumen superior (FR-009) como los contadores de cada filtro (FR-010) — nunca se
calculan dos veces de forma independiente.

### `TableCardViewModel` (extensión del view-model de tarjeta ya existente)

Extiende el objeto que ya arma `tablesView()`/`ordersByType()` para alimentar
`order-summary-card.component.ts`. Campos nuevos respecto a hoy:

| Campo nuevo | Tipo | Regla |
|---|---|---|
| `atendidoPor` | string \| null | `null` si la mesa está libre o el pedido no tiene `user_id` (FR-023); en otro caso, el nombre resuelto por el backend (ver `DiningOrder.user_id` arriba). Para una mesa con más de un pedido activo, se toma del mismo pedido principal que ya determina el resto de la tarjeta (FR-024). |

Campos ya existentes que **no cambian** (`title`/`statusLabel`/`statusClass`/`secondaryLabel`/
`elapsedLabel`/`totalLabel`/`ordersCount`/`selected`) — FR-006 exige que ninguno se pierda al migrar
al nuevo layout de cuadrícula.

### `SyncStatusViewModel` (Historia 3)

| Campo | Tipo | Origen |
|---|---|---|
| `online` | boolean | `navigator.onLine` + eventos `online`/`offline` |
| `lastLoadOk` | boolean | `store.error() === null` tras la última carga |

No se persiste; se recalcula en el cliente (ver research.md §3).

## Relaciones

Sin cambios: `DiningOrder.user_id → shared.users.id` ya existía; esta feature no agrega ninguna
relación nueva, solo la vuelve visible en la respuesta ya existente y en la UI.
