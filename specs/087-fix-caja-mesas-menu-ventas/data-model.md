# Data Model: Correcciones de Caja, Terminal de Mesas, Menú QR y Ventas

**Spec**: [spec.md](./spec.md) | **Research**: [research.md](./research.md)

Todas las tablas viven en el schema `tenant` (schema-per-tenant, `pos-backend`). Las
migraciones siguen el patrón `@for_each_tenant_schema` ya establecido (ver
`alembic/versions/f5a6b7c8d9e0_availability_change_partial_count.py` como referencia
directa).

## 1. `CustomerOrder` (`tenant.customer_orders`) — modificada

Modelo actual: `app/models/customer_order.py`.

### Campos nuevos

| Campo | Tipo | Nullable | Default | Notas |
|---|---|---|---|---|
| `cash_shift_id` | `UUID` | Sí | `NULL` | FK a `cash_shifts.id`, `ondelete="SET NULL"`. Se puebla solo cuando `order_type = 'DINE_IN'`, con el turno abierto vigente del tenant en el momento de creación (`cash_shifts` con `status='open'`; si excepcionalmente hay más de uno, el de `opened_at` más reciente — ver decisión confirmada en `research.md`). `NULL` para `TAKEAWAY`/`DELIVERY` y para todo pedido creado antes de esta migración. |
| `table_order_number` | `Integer` | Sí | `NULL` | Asignado **una sola vez**, en creación, solo para `order_type = 'DINE_IN'`. Valor = `COUNT(*) + 1` de pedidos `DINE_IN` existentes con el mismo `cash_shift_id`, calculado dentro de la misma transacción de creación (con bloqueo/`SELECT ... FOR UPDATE` sobre el turno, o constraint `UNIQUE(cash_shift_id, table_order_number)` con reintento, para evitar que dos creaciones casi simultáneas obtengan el mismo número). Inmutable tras asignarse: ningún flujo posterior lo recalcula ni lo desplaza. |

### Campo existente cuya semántica cambia (sin cambio de esquema)

| Campo | Cambio de semántica |
|---|---|
| `customer_name` | Sigue siendo `String(255) NULL` a nivel de columna (no se toca el esquema — los pedidos históricos deben seguir siendo consultables sin el dato). Pasa a ser **obligatorio a nivel de validación de aplicación** al crear cualquier pedido nuevo (`DINE_IN`/`TAKEAWAY`/`DELIVERY`), en el servicio de creación de pedido (backend) y en el formulario (frontend). Los pedidos existentes con `customer_name IS NULL` se muestran con un placeholder ("Cliente sin nombre"), nunca se bloquean ni se migran. |

### Migración (Alembic, `@for_each_tenant_schema`)

```python
def upgrade(schema: str) -> None:
    if not _has_table(schema, "customer_orders"):
        return
    op.add_column("customer_orders", sa.Column("cash_shift_id", sa.UUID(), nullable=True), schema=schema)
    op.add_column("customer_orders", sa.Column("table_order_number", sa.Integer(), nullable=True), schema=schema)
    op.create_foreign_key(
        op.f("fk__customer_orders__cash_shift_id__cash_shifts"),
        "customer_orders", "cash_shifts", ["cash_shift_id"], ["id"],
        source_schema=schema, referent_schema=schema, ondelete="SET NULL",
    )
    op.create_index(op.f("ix__customer_orders__cash_shift_id"), "customer_orders",
                     ["cash_shift_id"], schema=schema)
    # Unicidad del número dentro del turno, solo para filas que sí lo tienen (DINE_IN):
    op.create_index(
        "uq_customer_orders_shift_table_order_number", "customer_orders",
        ["cash_shift_id", "table_order_number"], unique=True, schema=schema,
        postgresql_where=sa.text("table_order_number IS NOT NULL"),
    )

def downgrade(schema: str) -> None:
    if not _has_table(schema, "customer_orders"):
        return
    op.drop_index("uq_customer_orders_shift_table_order_number", "customer_orders", schema=schema)
    op.drop_index(op.f("ix__customer_orders__cash_shift_id"), "customer_orders", schema=schema)
    op.drop_constraint(op.f("fk__customer_orders__cash_shift_id__cash_shifts"),
                        "customer_orders", schema=schema, type_="foreignkey")
    op.drop_column("customer_orders", "table_order_number", schema=schema)
    op.drop_column("customer_orders", "cash_shift_id", schema=schema)
```

**Compatibilidad con datos existentes**: additive-only, ambas columnas nullable sin
`server_default` obligatorio — ningún pedido histórico se toca; nace con `NULL` en ambos
campos nuevos, exactamente el mismo estado que representa hoy la ausencia de numeración.
**Rollback**: `downgrade()` elimina limpiamente ambas columnas, el índice único y la FK; no
hay pérdida de información distinta de la que la propia migración introdujo (no se borra
ningún dato preexistente).

## 2. `CashPartialCount` (`tenant.cash_partial_counts`) — eliminada

Modelo actual: `app/models/cash_partial_count.py`. Se elimina la clase del modelo y se
elimina la tabla físicamente.

### Migración (Alembic, `@for_each_tenant_schema`)

```python
def upgrade(schema: str) -> None:
    if not _has_table(schema, "cash_partial_counts"):
        return
    op.drop_index(op.f("ix__cash_partial_counts__cash_shift_id"), "cash_partial_counts", schema=schema)
    op.drop_table("cash_partial_counts", schema=schema)

def downgrade(schema: str) -> None:
    # Recrea la tabla vacía (estructura idéntica a f5a6b7c8d9e0) si se necesita revertir;
    # los datos borrados en upgrade() NO se recuperan — el borrado es intencionalmente
    # permanente (FR-001, decisión de negocio).
    ...
```

**Compatibilidad con datos existentes**: **destructiva e intencional** — FR-001 exige borrado
físico permanente de todo el historial de arqueo parcial, no archivado. Confirmado sin FKs
entrantes hacia esta tabla (solo tiene una FK saliente hacia `cash_shifts`), por lo que el
`DROP TABLE` no requiere ninguna limpieza en cascada adicional. **Rollback**: solo estructural
(recrear la tabla vacía); los datos borrados no son recuperables por diseño — cualquier
necesidad de recuperación depende de un backup de base de datos externo a esta migración, no
de la migración misma.

**Nota de gobernanza**: requiere entrada en `specs/000-reconocimiento/registro-de-anomalias.md`
(Principio II) antes de ejecutar esta migración en producción.

## 3. `Sale` (`tenant.sales`) — sin cambios de esquema

`Sale.paid_amount` y `Sale.change_given` ya existen (migración `f5a6b7c8d9e0`). Este spec solo
verifica/ajusta su serialización en `SaleResponse` y su presentación en el frontend — ningún
cambio de modelo de datos.

## 4. `OrderItem` / `OrderItemOption` — sin cambios de esquema

El fix de cálculo (FR-011) y el formato "Nombre xN" (FR-012) son correcciones de lógica de
cálculo/presentación sobre datos que ya existen (precio de línea, opciones seleccionadas con
cantidad) — ningún campo nuevo.

## 5. `SaleItem` — sin cambios de esquema

`description` sigue siendo un snapshot de texto libre (D8 en `research.md`); si falta incluir
la presentación se ajusta el builder que arma ese texto, no se añade una columna estructurada
nueva.

## 6. Adenda US8–US11 (2026-09-29) — sin cambios de esquema

- **US8 (presentación en el modal), US11 (estilo de notas)**: solo plantilla/estilos del
  frontend; ningún dato cambia.
- **US10 (total al editar pedido manual)**: el pedido **no** persiste `subtotal`/`total`
  (se derivan de los ítems vigentes), por lo que no hay campo que migrar ni recalcular. La
  corrección es de cálculo/estado en el frontend (D13 en `research.md`). Ninguna venta ya
  emitida se toca (Principio VII).
- **US9 (adicionales en el panel)**: `OrderItemOption` sigue guardando solo `option_id` +
  `quantity`. **No se agrega columna de snapshot**: el nombre y el grupo se resuelven en
  lectura con un JOIN a `options`/`option_groups` y se exponen como campos **derivados** de la
  respuesta (`contracts/api-changes.md` §7). Es seguro porque `order_item_options.option_id`
  es FK a `options.id` (sin `ON DELETE`): una opción referenciada por un pedido no puede
  borrarse, así que el JOIN siempre encuentra el nombre, aunque su producto ya no esté en el
  menú vigente.

Compatibilidad hacia atrás: campos nuevos opcionales en la respuesta; pedidos históricos se
benefician sin backfill. Rollback: quitar los dos campos del schema de respuesta.

## Resumen de impacto en migraciones

| Migración | Tipo | Reversible sin pérdida de datos |
|---|---|---|
| `customer_orders.cash_shift_id` + `table_order_number` | Additive (2 columnas + FK + 2 índices) | Sí |
| `DROP TABLE cash_partial_counts` | Destructiva, intencional (FR-001) | No (por diseño) |

Las historias US8–US11 no añaden migraciones: el resumen de arriba no cambia.
