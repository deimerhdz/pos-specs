# Data Model: Red de characterization tests para `table_sessions`

Esta spec no introduce entidades de negocio nuevas — `table_sessions/service.py` y
`table_sessions/router.py` quedan sin tocar (FR-014). Lo que este documento modela son las
**entidades de infraestructura de test** que `table_sessions_fixtures.py` crea para poder
ejercitar las 9 funciones públicas y los 7 endpoints contra datos reales (SQLite en memoria), y su
relación con los modelos SQLAlchemy de producción que representan.

## 1. Modelos SQLAlchemy de producción reutilizados (sin modificar)

Ver research.md §6 para el porqué de cada tabla.

### Ya cubiertas por `fixtures.py` (reutilizadas tal cual, sin duplicar factory)

| Modelo | Tabla | Rol en los tests de `table_sessions` |
|---|---|---|
| `Category` / `Product` / `ProductVariant` | `categories` / `products` / `product_variants` | Variante que consume cada `OrderItem`; `price` alimenta `order_sale_lines`/`compute_bill`. |
| `OptionGroup` / `Option` | `option_groups` / `options` | Solo si un test de `set_assignments` ejercita la duplicación de `OrderItemOption` al partir una línea con opciones (no obligatorio para el resto de casos). |
| `VariantOptionGroup` / `RecipeItem` / `InventoryItem` / `UnitMeasure` | — | Se crean porque `import app.models` las registra en `Base.metadata`, pero ningún camino de `table_sessions` las consulta directamente — no necesitan factory propia en este fixture más allá de las ya heredadas de `fixtures.py`. |

### Nuevas en `table_sessions_fixtures.py` (Fase 1 de esta spec)

| Modelo | Tabla | Rol en los tests de `table_sessions` | Nota de fixture |
|---|---|---|---|
| `DiningTable` | `dining_tables` | Mesa de la sesión; `status`/`number` los lee `_nombre_cuenta`/`_close_split` (nombre de cuenta sin comensal) y `try_release_if_empty`/`close_session` (libera a `'libre'`). | — |
| `TableSession` | `table_sessions` | La entidad central; `status`, `billing_mode`, `closed_by_user_*` los mutan `close_session`/`try_release_if_empty`. | Ningún test siembra dos filas `active` para la misma mesa — el índice parcial `idx_active_session_per_table` no se toca (research.md §6). |
| `SessionParticipant` | `session_participants` | Comensal; `add_participant`/`remove_participant` lo crean/borran, `set_assignments`/`compute_bill`/`_close_split` lo leen para etiquetar líneas. | — |
| `CustomerOrder` | `customer_orders` | Pedido de la sesión; `status` (`recibida`/`en_preparacion`/`cancelada`/`pagada`) gobierna `_billable_orders`/A-01. | — |
| `OrderItem` / `OrderItemOption` | `order_items` / `order_item_options` | Línea cobrable; `participant_id`, `estado_cocina`, `combo_id` son los campos que ejercitan A-01, A-29, A-38 (RN-MESA-24) y el reparto de `set_assignments`. | `set_assignments` puede insertar filas nuevas de `OrderItem` (partir una línea) — el fixture no precrea eso, lo hace el propio `service.py` bajo prueba. |
| `Cart` | `carts` | Solo para que `checkout.close_participants` (invocada por `close_table_sessions`) pueda consultarla sin error; ningún test siembra un carrito abierto a propósito. | Tabla creada vacía; no hace falta factory dedicada más allá de reexportar (o no) `make_cart` si algún caso futuro la necesita. |
| `Promotion` / `PromotionTarget` / `PromotionComboItem` | `promotions` / `promotion_targets` / `promotion_combo_items` | Fixture mínimo para A-01 (descuento por comensal) y A-29 (combos múltiples/ninguno). | Sin ventana horaria (`start_time`/`end_time`/fechas en `None`) — ver research.md §5, por qué esta spec no necesita reloj fijado. |
| `CashRegister` / `CashShift` | `cash_registers` / `cash_shifts` | `close_session` exige un turno `open` (`ensure_open_shift`). | Ningún test necesita dos turnos `open` en la misma caja — el índice parcial `idx_open_shift_per_register` no se toca. |
| `PaymentMethod` / `Payment` | `payment_methods` / `payments` | Método de pago (`is_cash` gobierna la validación de vuelto en `build_sale`) y los pagos que cierran cada venta. | — |
| `Sale` / `SaleItem` | `sales` / `sale_items` | Venta(s) que produce `close_session`; `customer_name`, `promotion_id`, `participant_id` son justo lo que A-15/A-29 congelan. | `SaleItem.options` es `JSONB` — requiere el shim de compilador para SQLite (research.md §4). |
| `Invoice` / `InvoiceCounter` | `invoices` / `invoice_counters` | `build_sale` emite factura dentro de la misma transacción (`invoices.service.issue_for_sale`) — necesaria para que `close_session` no falle, aunque esta spec no caracteriza el formato de factura (ya cubierto por `test_invoices_full_number.py`). | — |

## 2. Entidades de infraestructura de test (nuevas, solo en `table_sessions_fixtures.py`)

Estas no son modelos SQLAlchemy: son los objetos Python que el harness de router/servicio
construye para sustituir lo que en producción resuelve `Depends` o para observar contratos que
SQLite no puede reproducir de forma nativa (research.md §1-3).

- **`Tenant` de prueba**: `SimpleNamespace(id, invoice_prefix)` — no requiere el modelo `Tenant`
  real de `app.core.models` (schema `shared`, fuera del alcance de tablas creadas). Se pasa como
  `tenant=` a los 5 endpoints de router que lo reciben vía `Depends(get_tenant)`, y su `.id` se
  usa como `tenant_id=` al invocar las funciones de `service.py` directamente en los tests de
  Historia 1 y 2.
- **`User` de prueba (cajero)**: `SimpleNamespace(id, name)` — no requiere el modelo `User` real
  (mismo motivo). `close_session` (y, transitivamente, `build_sale`/`issue_for_sale`) solo leen
  `.id`/`.name`; los otros 6 endpoints lo reciben pero lo ignoran (`_: User = Depends(...)`).
- **Espía de `_load` (A-17/R12)**: `unittest.mock.patch("app.api.v1.table_sessions.service._load",
  wraps=service._load)` — no reemplaza el comportamiento (usa `wraps`, así que la función real se
  sigue ejecutando), solo permite inspeccionar con qué `lock=` fue invocada cada llamada, ver
  research.md §2.
- **Doble de `events.bill_changed` (FR-010a)**: función `_spy` pasada como `side_effect` a
  `unittest.mock.patch("app.core.events.bill_changed", side_effect=_spy)`, que registra los
  argumentos recibidos y opcionalmente consulta `db` para confirmar que el commit ya ocurrió —
  mismo patrón que `cart_fixtures`/`test_cart_router.py` usó para `order_created`.

## 3. Relaciones y ciclo de vida relevantes para los tests

```text
DiningTable (1) ──< TableSession (1 activa por mesa; esta spec nunca siembra una segunda)
TableSession (1) ──< SessionParticipant (N comensales)
TableSession (1) ──< CustomerOrder (N pedidos) ──< OrderItem (N líneas) ──< OrderItemOption
SessionParticipant (1) ──< OrderItem (0..N líneas asignadas, vía participant_id nullable)
CashRegister (1) ──< CashShift (1 open a la vez; esta spec nunca siembra un segundo)
CashShift (1) ──< Sale (N ventas del turno)
Sale (1) ──< SaleItem (N líneas) / Payment (N pagos) / Invoice (1, emitida por build_sale)
Promotion (1) ──< PromotionTarget (0..N alcance) / PromotionComboItem (0..N componentes de combo)
```

No hay transiciones de estado nuevas que documentar: las que importan (`TableSession.status`,
`SessionParticipant.status`, `CustomerOrder.status`, `OrderItem.estado_cocina`, `Sale.status`) ya
están descritas en los docstrings de sus modelos de producción y en `spec.md` (Acceptance
Scenarios); esta spec las observa, no las redefine.
