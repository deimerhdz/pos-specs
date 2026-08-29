# Data Model: Estandarización de canal y tipo de orden — habilitación de pedidos "Para Llevar"

Las decisiones detrás de cada elección están en [research.md](./research.md); este documento se
limita a columnas, constraints, migración y validaciones.

## `CustomerOrder` (`customer_orders`, schema `tenant`) — MODIFICADA

### Columna existente, valores reemplazados: `channel`

| Antes | Después | Origen |
|---|---|---|
| `qr` | `QR_MENU` | creado por un cliente desde el menú QR |
| `counter` | `POS` | creado directamente desde el cajero (pantalla de creación de orden manual) |
| `waiter` | `POS` | creado directamente por el mesero (Terminal de Mesas, modo híbrido — spec 028) |
| *(no existía)* | `WHATSAPP` | creado desde WhatsApp — sin punto de creación real todavía (research.md, Decisión 4) |
| *(no existía)* | `API` | creado desde una integración externa — sin punto de creación real todavía |

- Tipo de columna sin cambio: `String`, se amplía a `String(10)` si hace falta por el largo de
  `QR_MENU`/`WHATSAPP` (`String(10)` actual ya alcanza justo para `QR_MENU`; confirmar largo exacto
  en implementación).
- `CheckConstraint` reemplazado: `ck_customer_order_channel` pasa de
  `"channel IN ('qr', 'counter', 'waiter')"` a
  `"channel IN ('POS', 'QR_MENU', 'WHATSAPP', 'API')"`.
- Índice nuevo (FR-002): `Index("idx_customer_orders_channel", "channel")`.
- Espejo en `OrderChannel(str, Enum)` (`orders/schemas.py:12-15`): `POS`, `QR_MENU`, `WHATSAPP`,
  `API`.

### Columna nueva: `order_type`

| Columna | Tipo | Nulable | Default (app) | Notas |
|---|---|---|---|---|
| `order_type` | `String(10)` | Sí | `DINE_IN` (en `OrderCreate`, no a nivel de columna) | Cómo se atiende el pedido. `NULL` únicamente en pedidos históricos sin mesa (research.md Decisión 3). |

- `CheckConstraint` nuevo: `"order_type IN ('DINE_IN', 'TAKEAWAY', 'DELIVERY')"` (permite `NULL`).
- Índice nuevo (FR-004): `Index("idx_customer_orders_order_type", "order_type")`.
- Espejo en `OrderType(str, Enum)` (nuevo, `orders/schemas.py`): `DINE_IN`, `TAKEAWAY`, `DELIVERY`.

### Columna nueva, técnica (no forma parte del catálogo de negocio de la spec): `is_consolidation_order`

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `is_consolidation_order` | `Boolean` | No | `false` | `true` únicamente en las órdenes que crea/reusa `orders/consolidation.py::get_or_create_open_order` (antes distinguibles por `channel == 'waiter'`). Preserva ese comportamiento sin exponer la distinción `counter`/`waiter` en el canal estandarizado (research.md Decisión 2). No se expone en `OrderResponse`. |

### Columnas sin cambio (confirmadas contra `app/models/customer_order.py`)

`id`, `table_session_id`, `participant_id`, `dining_table_id`, `customer_name`, `status`,
`version`, `user_id`, `merged_group_id`, `notes`, `created_at`; relaciones `items`, `cancel_logs`,
`payment_attempts`; `CheckConstraint` de `status`; `Index` de `idx_active_order_per_participant`.

## Migración de datos existentes

Ver research.md, Decisión 7, para la migración Alembic completa. Resumen del backfill:

```sql
-- 1) columnas nuevas
ALTER TABLE customer_orders ADD COLUMN order_type varchar(10);
ALTER TABLE customer_orders ADD COLUMN is_consolidation_order boolean NOT NULL DEFAULT false;

-- 2) backfill (clarificación spec.md, sesión 2026-08-29 + research.md Decisión 2)
UPDATE customer_orders SET order_type = 'DINE_IN' WHERE dining_table_id IS NOT NULL;
UPDATE customer_orders SET is_consolidation_order = true WHERE channel = 'waiter';

-- 3) remapeo de channel
UPDATE customer_orders SET channel = CASE channel
  WHEN 'qr' THEN 'QR_MENU'
  WHEN 'counter' THEN 'POS'
  WHEN 'waiter' THEN 'POS'
END;

-- 4) constraints/índices nuevos (drop + create del constraint de channel, create del de
--    order_type, create de ambos índices) — ver research.md Decisión 7 para el orden exacto.
```

## Reglas de validación (FR-006, FR-007, y Decisión 5 de research.md)

| Canal | Tipos de orden permitidos |
|---|---|
| `POS` | `DINE_IN`, `TAKEAWAY`, `DELIVERY` |
| `QR_MENU` | `DINE_IN` únicamente |
| `WHATSAPP` | `TAKEAWAY`, `DELIVERY` (nunca `DINE_IN`) |
| `API` | `TAKEAWAY`, `DELIVERY` (nunca `DINE_IN`) |

- Validada únicamente en `orders/service.py::create_order` (research.md Decisión 4) — `400` si la
  combinación recibida no está en la tabla de arriba.
- `422` si `order_type` es `TAKEAWAY`/`DELIVERY` y `dining_table_id` no es `null` (research.md
  Decisión 5) — un pedido para llevar o a domicilio nunca lleva mesa.
- `submit_cart` (QR) y `get_or_create_open_order` (consolidación) construyen su propia combinación
  fija y ya válida por construcción (`QR_MENU`+`DINE_IN`, `POS`+`DINE_IN`
  respectivamente) — no pasan por esta validación.

## Contrato de request/response (`OrderCreate` / `OrderResponse`, resumen)

Ver [contracts/orders-create.md](./contracts/orders-create.md) para el detalle completo.

- `OrderCreate.channel: OrderChannel = OrderChannel.POS` (antes `OrderChannel.COUNTER`).
- `OrderCreate.order_type: OrderType = OrderType.DINE_IN` (nuevo).
- `OrderResponse.channel: str` (sin cambio de forma, cambian los valores posibles).
- `OrderResponse.order_type: str | None` (nuevo).
- `is_consolidation_order` no se agrega a ningún schema de respuesta — es un detalle interno.

## Frontend (`pos-heladeria`) — tipos espejo

`src/app/modules/tables/interfaces/dining.interface.ts`:

- `OrderChannel`: de `'qr' | 'counter' | 'waiter'` a `'POS' | 'QR_MENU' | 'WHATSAPP' | 'API'`.
- `OrderType` (nuevo): `'DINE_IN' | 'TAKEAWAY' | 'DELIVERY'`.
- `OrderCreatePayload.order_type?: OrderType` (nuevo).
- `DiningOrder.channel`/`order_type`: sin cambio de forma más allá del nuevo campo opcional
  `order_type?: string`.

Ver research.md Decisión 6 para el detalle completo de los cambios en
`manual-order-page.component.ts` y `pos-terminal.store.ts` (incluye los 4 literales `'qr'` que
pasan a `'QR_MENU'`).
