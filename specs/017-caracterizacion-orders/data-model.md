# Data Model: Red de characterization tests para `orders`

Esta spec no introduce ningún modelo de dominio nuevo: los cinco ficheros de producción en
alcance (`service.py`, `checkout.py`, `consolidation.py`, `kitchen.py`, `tables_advanced.py`) no
se tocan (FR-017), y sus modelos SQLAlchemy (`app/models/*.py`) tampoco. Lo que sigue son las
entidades de **infraestructura de test** que `orders_fixtures.py` crea sobre SQLite en memoria —
un subconjunto ya existente del esquema real de `pos-backend`, no un modelo nuevo — y las
entidades conceptuales que `spec.md` usa para describir el alcance de la cobertura (Key Entities).

## Entidades conceptuales de la spec (`spec.md` §Key Entities)

### Función pública en alcance
Una de las 23 funciones sin prefijo `_` repartidas entre los cinco ficheros. Unidad mínima de
cobertura exigida por FR-002.

| Fichero | Funciones públicas |
|---|---|
| `service.py` | `create_order` |
| `checkout.py` | `block_order`, `compute_bill`, `order_sale_lines`, `promo_lines_for`, `pay_order`, `confirm_order`, `cancel_order`, `close_participants`, `close_table_sessions`, `release_table` |
| `consolidation.py` | `active_table_session_id`, `get_or_create_table_session_id`, `get_or_create_open_order`, `consolidate_table`, `add_item_to_table` |
| `kitchen.py` | `transition_kitchen`, `mark_order_ready`, `void_item` |
| `tables_advanced.py` | `set_table_status`, `move_order`, `merge_orders`, `group_bill` |

### Caso de anomalía citado
Un characterization test cuyo propósito explícito es documentar el comportamiento de una anomalía
ya registrada en `specs/000-reconocimiento/registro-de-anomalias.md`, citada en su docstring o
nombre (SC-002): A-01 (caminos B y C), A-04, A-16, A-25 [PROTEGIDA], A-26 (tres hallazgos), A-29
(parcial), A-38 (parcial: RN-ORD-31, RN-ORD-32).

### Dependencia externa fijada
Un módulo que `orders` consume real y sin mocks, ejercitado con fixtures mínimos pero no
re-caracterizado en profundidad: `sales.builder.build_sale`/`ensure_open_shift`,
`promotions.service.evaluate`/`combo_discount_for_lines`/`expand_combo`,
`orders.consumption.deduct_order_items`/`reverse_order_items`, `invoices.service.issue_for_sale`
(invocado internamente por `build_sale`).

### Frontera con `cart`/`table_sessions`
La superficie de `checkout.py` (`cancel_order`, `TERMINAL`, `close_table_sessions`,
`order_sale_lines`, `promo_lines_for`) que las specs 015 y 016 ya consumían como dependencia real
sin profundizar — esta spec cierra esa cobertura pendiente sin reabrir esas dos specs.

## Esquema SQLite de `orders_fixtures.new_session()`

Cierre transitivo real de lo que las 23 funciones públicas y sus dependencias externas
(`orders.consumption`, `promotions.service`, `sales.builder`, `invoices.service`) tocan —
verificado leyendo los cinco ficheros de producción y sus tres dependencias directas.

### Reexportadas de `fixtures.py` (catálogo, sin cambios — 10 tablas)

`categories`, `products`, `product_variants`, `option_groups`, `options`,
`variant_option_groups`, `recipe_items`, `inventory_items`, `inventory_movements`,
`unit_measures`.

### Nuevas en `orders_fixtures.py` (replicadas de `cart_fixtures.py`/`table_sessions_fixtures.py`
más `order_item_void_logs`, nueva — 21 tablas)

| Tabla | Modelo | Por qué la necesita `orders` |
|---|---|---|
| `dining_tables` | `DiningTable` | Toda función que recibe `table_id` (`consolidation`, `checkout`, `tables_advanced`) |
| `table_sessions` | `TableSession` | `consolidation.get_or_create_table_session_id`, `checkout.close_table_sessions` |
| `session_participants` | `SessionParticipant` | `service.create_order` (vía `participant_id`), `checkout.order_sale_lines` (split), `close_participants` |
| `customer_orders` | `CustomerOrder` | Entidad central de las 23 funciones |
| `order_items` | `OrderItem` | Líneas de toda orden; `EN_CURSO`, `estado_cocina` |
| `order_item_options` | `OrderItemOption` | Selección de opciones por línea (A-04, `add_item_to_table`) |
| `order_item_void_logs` | `OrderItemVoidLog` | **Nueva** — `kitchen.void_item` (ni `cart_fixtures.py` ni `table_sessions_fixtures.py` la necesitaron) |
| `order_cancel_logs` | `OrderCancelLog` | `checkout.cancel_order` |
| `carts` | `Cart` | `consolidation.consolidate_table` (carritos del comensal), `close_participants` (abandono) |
| `cart_items` | `CartItem` | `consolidation.consolidate_table` (líneas a copiar) |
| `cart_item_options` | `CartItemOption` | Opciones de las líneas de carrito consolidadas |
| `promotions` | `Promotion` | `checkout.pay_order` (`promotions.evaluate`), `consolidation.add_item_to_table` (`combo_id`) |
| `promotion_targets` | `PromotionTarget` | Alcance de una promoción percent/fixed |
| `promotion_combo_items` | `PromotionComboItem` | `promotions.expand_combo` (componentes del combo) |
| `cash_registers` | `CashRegister` | FK de `cash_shifts` |
| `cash_shifts` | `CashShift` | `checkout.pay_order` vía `ensure_open_shift` |
| `payment_methods` | `PaymentMethod` | `build_sale` (validación de pago) |
| `payments` | `Payment` | `build_sale` |
| `sales` | `Sale` | `checkout.pay_order` (resultado) |
| `sale_items` | `SaleItem` | Líneas de la venta construida |
| `invoices` | `Invoice` | `build_sale` emite factura siempre, internamente (research.md §4) |
| `invoice_counters` | `InvoiceCounter` | Numeración de factura, requisito de `issue_for_sale` |
| `audit_logs` | `AuditLog` | `checkout.cancel_order` (`record_audit`, pérdida de inventario en cancelación) |

No se remueve ningún índice único parcial del metadata de test (`idx_active_session_per_table` en
`table_sessions`, `idx_open_cart_per_participant` en `carts`, `idx_open_shift_per_register` en
`cash_shifts`): ningún escenario de `spec.md` necesita sembrar dos filas activas que colisionen
con esos índices a la vez.

## Factories de `orders_fixtures.py`

Replican, con la misma firma y semántica, las que ya existen en `cart_fixtures.py`/
`table_sessions_fixtures.py` para las tablas que comparten, más una nueva:

- Reexportadas de `fixtures.py`: `make_category`, `make_product`, `make_variant`,
  `make_option_group`, `make_option`, `link_variant_group`, `make_recipe_item`,
  `make_inventory_item`, `make_unit`.
- Replicadas: `make_dining_table`, `make_table_session`, `make_participant`,
  `make_customer_order`, `make_order_item`, `make_cart`, `make_cart_item`, `make_promotion`,
  `make_promotion_target`, `make_combo_item`, `make_cash_register`, `make_cash_shift`,
  `make_payment_method`.
- **Nueva**: `make_order_item_void_log(db, order_item, **kw) -> OrderItemVoidLog` — mismo patrón
  `kw.setdefault` de las demás, con `motivo`/`user_id`/`user_name` por defecto.
- Dobles: `make_tenant_double`, `make_user_double` (mismos `SimpleNamespace` mínimos que ya
  definieron `cart_fixtures.py`/`table_sessions_fixtures.py`, reusados aquí porque `orders`
  también recibe `User` real en varias funciones — `void_item`, `cancel_order`,
  `consolidate_table`, `add_item_to_table`).
- Helper de forzado de excepción (A-26/RN-ORD-60, no una factory de fila):
  `force_flush_integrity_error(db)` — context manager que envuelve
  `unittest.mock.patch.object(db, "flush", side_effect=IntegrityError(...))` (research.md §2).

Ningún estado mutable compartido entre tests: cada test llama `orders_fixtures.new_session()` y
obtiene una base SQLite en memoria nueva y vacía.
