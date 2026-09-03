# Data Model: Corrección — la Terminal de mesas cobra sin aplicar el descuento por promoción

**Spec**: [spec.md](./spec.md) | **Research**: [research.md](./research.md) (D1, D2, D7, D12; D13-D15 para US7)

**Adenda 2026-09-03 (US7 — quinta superficie)**: sin cambios de esquema. Ver la nota al final de
"Sin cambios en ningún otro modelo".

## Cambios de esquema (`pos-backend`, PostgreSQL 16, schema-per-tenant)

### `customer_orders` — columna nueva

| Columna | Tipo | Nullable | Default | Descripción |
|---|---|---|---|---|
| `promotion_evaluated_at` | `TIMESTAMP WITH TIME ZONE` (**aware UTC** — `DateTime(timezone=True)`; **distinto de `created_at`**) | Sí | ninguno | El instante contra el que se evalúa la vigencia **temporal** (fecha, día, franja) de las promociones de este pedido — FR-008. Se fija **una sola vez**, al crear el pedido, y nunca se actualiza. `NULL` en todo pedido creado antes de esta spec (FR-012) y en cualquier pedido creado por un flujo que no pase por los dos puntos de creación de research.md D2. |

**Por qué aware y no naive como `created_at`**: el valor se pasa a `auto_discount` →
`evaluate_variant_sets`/`_valid_now` → `local_now()` (`promotions/service.py:59`), que interpreta
un `datetime` **naive** como hora local del tenant. Un instante UTC guardado naive desplazaría la
evaluación de fecha/día/franja por el offset del tenant (−5 en Colombia): "hasta las 20:00" se
evaluaría como "hasta las 15:00". `created_at` no sufre esto porque nunca se pasa a `local_now()`
(research.md D1).

**Compatibilidad con datos existentes**: 100% de las filas actuales quedan en `NULL` tras la
migración — ningún backfill (research.md D1). El código que lee esta columna (research.md D3)
trata `NULL` como "usa la hora del cobro", que es exactamente el comportamiento que esas filas ya
tenían antes de esta spec. Cero cambio observable para pedidos históricos.

**Quién la escribe**: únicamente `orders/service.py::create_order` y el bloque de creación de
`CustomerOrder` en `cart/service.py` (research.md D2), ambos con
`datetime.now(timezone.utc)` (**aware**, sin `.replace(tzinfo=None)`) en el momento exacto de la
inserción. Ningún otro código escribe esta columna — en particular,
`void_item`/`kitchen_transition`/`move_order` (que modifican el pedido después de creado) no la
tocan: FR-010 exige que el descuento se recalcule sobre los ítems vigentes **manteniendo** el
instante congelado, no que el instante cambie.

### `sales` — columna nueva

| Columna | Tipo | Nullable | Default | Descripción |
|---|---|---|---|---|
| `promotion_evaluated_at` | `TIMESTAMP WITH TIME ZONE` (**aware UTC** — `DateTime(timezone=True)`, mismo criterio que `customer_orders` arriba) | Sí | ninguno | El instante que **efectivamente se usó** para evaluar las promociones de esta venta (FR-011a) — el mismo valor que decidió `promotion_evaluation_instant(...)` (research.md D3) en el momento de facturar, no necesariamente igual al `promotion_evaluated_at` del pedido si la cuenta agrupó varias rondas (FR-012a: manda la más antigua). `NULL` en toda venta emitida antes de esta spec y en cualquier venta que no derive de un `CustomerOrder` con el campo poblado (p. ej. venta de mostrador directa, research.md D2). |

**Compatibilidad con datos existentes**: mismo criterio — `NULL` para todo lo emitido antes de
esta spec, sin backfill, sin cambiar el importe ni la representación de ninguna venta ya emitida
(Principio VII / FR-011).

**Quién la escribe**: `build_sale` (`sales/builder.py`), vía el nuevo kwarg opcional
`promotion_evaluated_at` (research.md D7). Los call sites que ya tienen un instante congelado se
lo pasan explícitamente; el resto (ventas sin ningún pedido congelado detrás) simplemente no lo
pasan y queda `NULL`.

**Quién la lee (FR-011a / SC-009)**: `SaleResponse` (`sales/schemas.py`) la expone como
`UtcDatetime | None` — igual serialización que `sold_at` —, y `GET /sales/{id}` la devuelve sin
cambio de router. El detalle de venta del frontend (`sales-page.component.ts`) pinta una fila con
ese instante cuando la venta llevó descuento, para distinguir un descuento de una promoción hoy
vencida de una falla del sistema.

### Sin cambios en ningún otro modelo

- `OrderItem`, `PromotionRule`, `Promotion`: sin columnas nuevas. El motor de descuentos
  (`evaluate_variant_sets`/`_valid_now`/`active_variant_set_rules`) no cambia de firma ni de
  comportamiento — solo cambia qué valor de `now` le llega desde fuera (research.md D3, D8).
- `SaleItem`: sin cambios — el desglose agregado (Subtotal/Descuento/Domicilio/Total, FR-004) vive
  a nivel de `Sale`/`CustomerOrder`, no se prorratea por línea (decisión del dueño, ver
  "Assumptions" en spec.md).
- `OrderPaymentAttempt` / `PaymentAttemptResponse`: **sin cambios** (quinta superficie, US7 —
  FR-021 a FR-024). La revisión de pago del cajero para pedidos QR reusa `CheckoutPreview` (abajo)
  vía el endpoint ya existente y el instante congelado que ya viaja por `Sale.promotion_evaluated_at`
  — no persiste nada nuevo (research.md D13-D15). El único cambio de backend es que
  `confirm_cash_payment_attempt` calcula su chequeo previo con `compute_checkout_preview(...).total`
  en vez de la función `_order_total`, que se elimina.

## Migración Alembic

Una única revisión nueva, encadenada tras el head actual del repo
(`94144eaa60b5_categories_display_order.py`), siguiendo el esqueleto de
`d427cd419e79_domicilio_orden_manual.py` (research.md D12) **salvo el tipo de columna**:
`DateTime(timezone=True)` en vez de `DateTime`, por el motivo de zona horaria explicado arriba:

```python
"""congela el instante de vigencia de promociones al tomar el pedido

Spec 073 (FR-008, FR-011a): agrega `promotion_evaluated_at` a `customer_orders`
(instante congelado del pedido) y a `sales` (instante efectivamente usado al
facturar). Columnas puramente aditivas y nulables — sin backfill: los pedidos
y ventas existentes conservan su comportamiento actual (evaluar la vigencia
con la hora del cobro), research.md D1/D12.
"""

@for_each_tenant_schema
def upgrade(schema: str) -> None:
    # `DateTime(timezone=True)`: el instante se pasa a `local_now()`, que trata un
    # naive como hora local del tenant — un UTC naive desplazaría la franja por el
    # offset. Se desvía a propósito del esqueleto de `d427cd419e79` en este punto.
    if _has_table(schema, "customer_orders"):
        op.add_column(
            "customer_orders",
            sa.Column("promotion_evaluated_at", sa.DateTime(timezone=True), nullable=True),
            schema=schema,
        )
    if _has_table(schema, "sales"):
        op.add_column(
            "sales",
            sa.Column("promotion_evaluated_at", sa.DateTime(timezone=True), nullable=True),
            schema=schema,
        )

@for_each_tenant_schema
def downgrade(schema: str) -> None:
    if _has_table(schema, "sales"):
        op.drop_column("sales", "promotion_evaluated_at", schema=schema)
    if _has_table(schema, "customer_orders"):
        op.drop_column("customer_orders", "promotion_evaluated_at", schema=schema)
```

**Estrategia de rollback**: `downgrade` elimina ambas columnas sin condición — al ser puramente
aditivas y sin backfill, no hay ningún dato derivado que reconstruir ni ninguna otra migración
que dependa de ellas todavía. Revertir esta migración hace que todo pedido/venta vuelva a
comportarse exactamente como hoy (instante = hora del cobro), sin pérdida de ningún dato que
existiera antes de aplicarla.

## Value object nuevo (no persistido): `CheckoutPreview`

Forma común que devuelven los dos endpoints nuevos (research.md D4, D5), que consume también la
quinta superficie (revisión de pago del cajero para pedidos QR — research.md D13/D14) y que
reutiliza internamente `table_sessions.compute_bill` en su cálculo (aunque su respuesta pública,
`SessionBillResponse`, no cambia de forma — ver
[contracts/preview-cobro-pedido.md](./contracts/preview-cobro-pedido.md)):

| Campo | Tipo | Descripción |
|---|---|---|
| `subtotal` | `Decimal` | Suma de `unit_price * quantity` de las líneas cobrables, sin descuento. |
| `discount` | `Decimal` | Descuento automático por promoción (RF-012/spec 063), siempre `≥ 0`. |
| `delivery_fee` | `Decimal` | Valor del domicilio del pedido (`0` si no es `DELIVERY` o no tiene valor cargado). |
| `total` | `Decimal` | `max(0, subtotal - discount + delivery_fee)` — research.md D6, misma fórmula que `build_sale`. |
| `promotion_evaluated_at` | `datetime` (aware UTC; se serializa con `UtcDatetime`, igual que `SaleResponse.sold_at`) | El instante contra el que se evaluó la vigencia — expuesto para que el frontend pueda, si hace falta, explicarle al cajero por qué un descuento sigue o no aplicando. |

No es una tabla ni un modelo de SQLAlchemy — es un DTO de respuesta (`CheckoutPreviewResponse` en
`orders/schemas.py`), construido en cada llamada a partir de datos ya persistidos, nunca
almacenado.
