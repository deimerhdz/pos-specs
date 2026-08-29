# Data Model: Habilitación del tipo de orden "Domicilio" en la creación manual de pedidos

Las decisiones detrás de cada elección están en [research.md](./research.md); este documento se
limita a columnas, constraints, migración y validaciones.

## `CustomerOrder` (`customer_orders`, schema `tenant`) — MODIFICADA

### Columnas nuevas

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `delivery_address` | `String(255)` | Sí | ninguno | Dirección de entrega. Obligatoria solo cuando `order_type = 'DELIVERY'` (validado en aplicación, research.md Decisión 4, no en la columna — igual que `order_type` en spec 055, la ausencia solo es válida para pedidos que no son de este tipo). |
| `delivery_phone` | `String(30)` | Sí | ninguno | Teléfono de contacto. Siempre opcional (FR-008), incluso para `DELIVERY`. |
| `delivery_fee` | `Numeric(12, 2)` | Sí | ninguno | Valor del domicilio. Obligatorio y no negativo solo cuando `order_type = 'DELIVERY'` (validado en aplicación); `NULL` en cualquier otro tipo de orden y en todo pedido histórico anterior a esta mejora. |

- `CheckConstraint` nuevo: `ck_customer_order_delivery_fee_non_negative`:
  `"delivery_fee IS NULL OR delivery_fee >= 0"` (research.md Decisión 3).
- Sin índice nuevo — a diferencia de `channel`/`order_type` (spec 055), estos tres campos no son
  catálogos de filtrado, son datos de un pedido puntual; no hay ningún requisito de reporte por
  dirección/teléfono/valor de domicilio en spec.md.
- Los tres campos quedan siempre `NULL` para pedidos con `order_type` distinto de `DELIVERY`
  (`DINE_IN`, `TAKEAWAY`, o el histórico sin tipo asignado de spec 055) — ningún cambio de
  comportamiento para esos pedidos (FR-012).

### Columnas sin cambio (confirmadas contra `app/models/customer_order.py`)

`id`, `table_session_id`, `participant_id`, `dining_table_id`, `customer_name`, `channel`,
`order_type`, `is_consolidation_order`, `status`, `version`, `user_id`, `merged_group_id`, `notes`,
`created_at`; relaciones `items`, `cancel_logs`, `payment_attempts`; los `CheckConstraint`/`Index`
ya existentes de `status`/`channel`/`order_type` (spec 055).

## `Sale` (`sales`, schema `tenant`) — MODIFICADA

### Columna nueva

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `delivery_fee` | `Numeric(12, 2)` | Sí | ninguno | Valor del domicilio efectivamente incluido en el total de esta venta, copiado de `CustomerOrder.delivery_fee` al momento de facturar (research.md Decisión 5). `NULL`/`0` en cualquier venta que no provenga de una orden `DELIVERY`, incluidas todas las ventas ya emitidas antes de esta mejora (Principio VII — ninguna se recalcula). |

- Mismo patrón que las columnas ya existentes `discount`/`tax`/`tip` (`app/models/sale.py:54-58`,
  todas `Numeric(12,2)`): cada término del total tiene su propia columna, no solo el agregado.
- `total` (columna existente, sin cambio de tipo) pasa a calcularse como `subtotal - discount + tax
  + tip + delivery_fee` únicamente para ventas nuevas generadas a partir de esta mejora — ninguna
  venta ya emitida se recalcula (Principio VII, FR-013).

### Columnas sin cambio

`id`, `customer_order_id` (FK nulable a `customer_orders.id`, ya existente — confirma que `Sale`
puede leer `delivery_fee` de la orden asociada sin necesidad de una relación ORM nueva),
`subtotal`, `discount`, `tax`, `tip`, `total` (mismo tipo, fórmula extendida), y el resto de
columnas de pago/factura no relacionadas con esta mejora.

## Migración de datos existentes

Ver research.md, Decisión 10, para la migración Alembic completa. Resumen (sin backfill — columnas
puramente aditivas y nulables, ningún pedido histórico es `DELIVERY`):

```sql
-- customer_orders
ALTER TABLE customer_orders ADD COLUMN delivery_address varchar(255);
ALTER TABLE customer_orders ADD COLUMN delivery_phone varchar(30);
ALTER TABLE customer_orders ADD COLUMN delivery_fee numeric(12, 2);
ALTER TABLE customer_orders ADD CONSTRAINT ck_customer_order_delivery_fee_non_negative
  CHECK (delivery_fee IS NULL OR delivery_fee >= 0);

-- sales
ALTER TABLE sales ADD COLUMN delivery_fee numeric(12, 2);
```

## Reglas de validación (FR-006, FR-007)

- Obligatorios únicamente cuando `order_type = 'DELIVERY'` (validado en
  `orders/service.py::create_order`, research.md Decisión 4): `customer_name` (no vacío tras
  `.strip()`), `delivery_address` (no vacío tras `.strip()`), `delivery_fee` (no `None`).
  `422 Unprocessable Entity` si falta cualquiera de los tres.
- `delivery_phone` nunca es obligatorio, ni siquiera para `DELIVERY` (FR-008).
- `delivery_fee` no negativo — reforzado en dos capas: `CheckConstraint` de base de datos (defensa
  en profundidad) y, en la práctica, nunca se envía un valor negativo desde
  `manual-order-page.component.ts` (input numérico `min="0"`, research.md Decisión 8).
- Sin cambios a la matriz canal×tipo de orden ni al guard de mesa de spec 055 — `POS`+`DELIVERY` ya
  es una combinación válida, y `dining_table_id` ya se rechaza para `DELIVERY` desde spec 055
  (`orders/service.py:140-146`).

## Contrato de request/response (`OrderCreate` / `OrderResponse`, resumen)

Ver [contracts/orders-create.md](./contracts/orders-create.md) para el detalle completo de
creación, y [contracts/orders-checkout-total.md](./contracts/orders-checkout-total.md) para el
detalle del impacto en el total de la venta al facturar.

- `OrderCreate.delivery_address: str | None = None` (nuevo).
- `OrderCreate.delivery_phone: str | None = None` (nuevo).
- `OrderCreate.delivery_fee: Decimal | None = None` (nuevo).
- `OrderResponse.delivery_address: str | None` (nuevo).
- `OrderResponse.delivery_phone: str | None` (nuevo).
- `OrderResponse.delivery_fee: Decimal | None` (nuevo).

## Frontend (`pos-heladeria`) — tipos espejo

`src/app/modules/tables/interfaces/dining.interface.ts`:

- `OrderType`: sin cambio, ya incluye `'DELIVERY'` (spec 055).
- `OrderCreatePayload`: gana `delivery_address?: string | null; delivery_phone?: string | null;
  delivery_fee?: number | null;`.
- `DiningOrder`: gana los mismos tres campos como opcionales de respuesta.

`src/app/modules/tables/services/pos-terminal.store.ts`: tres signals nuevos (`deliveryAddress`,
`deliveryPhone`, `deliveryFee`), `totals` extendido con el término `deliveryFee`, y
`createManualOrderFromDraft()` con la tercera rama `esDomicilio`. Ver research.md Decisión 7 para
el detalle completo.

Ver research.md Decisión 8 para el detalle completo de los cambios en
`manual-order-page.component.ts` (pestaña habilitada, campos nuevos, hallazgo de
`applyDefaultCustomerName()`).
