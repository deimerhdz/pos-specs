# Data Model: Red de characterization tests para `cart`

Esta spec no introduce entidades de negocio nuevas — `cart/service.py` y `cart/router.py` quedan sin
tocar (FR-012). Lo que este documento modela son las **entidades de infraestructura de test** que
`cart_fixtures.py` crea para poder ejercitar esas 11 funciones y 9 endpoints contra datos reales
(SQLite en memoria), y su relación con los modelos SQLAlchemy de producción que representan.

## 1. Modelos SQLAlchemy de producción reutilizados (sin modificar)

Tablas que `cart_fixtures.new_session()` crea, agrupadas por origen y con la relación relevante para
`cart`. Ver research.md §4 para el porqué de cada una.

### Ya cubiertas por `fixtures.py` (reutilizadas tal cual, sin duplicar factory)

| Modelo | Tabla | Rol en los tests de `cart` |
|---|---|---|
| `Category` | `categories` | Padre de `Product`; necesario para `discounted_total` por categoría (promociones). |
| `Product` | `products` | Padre de `ProductVariant`; `preparation_type` no afecta a `cart`. |
| `ProductVariant` | `product_variants` | Línea del carrito referencia `product_variant_id`; `price` alimenta `compute_line_price`. |
| `OptionGroup` / `Option` | `option_groups` / `options` | Selección de opciones de una línea (`load_valid_options`, `CartItemOption`). |
| `VariantOptionGroup` | `variant_option_groups` | Regla min/max de selección que `load_valid_options` valida. |
| `RecipeItem` | `recipe_items` | Consumo de insumo por variante, usado por `required_consumption`/`check_availability`. |
| `InventoryItem` / `UnitMeasure` | `inventory_items` / `unit_measures` | Disponibilidad que `check_availability` verifica antes de `add_item`/`submit_cart`. |

### Nuevas en `cart_fixtures.py` (Fase 1 de esta spec)

| Modelo | Tabla | Rol en los tests de `cart` | Nota de fixture |
|---|---|---|---|
| `DiningTable` | `dining_tables` | Mesa que resuelve el QR; `status`/`active` gobiernan el 404 de `POST /cart/sessions`. | — |
| `TableSession` | `table_sessions` | Sesión de mesa que `_get_or_create_table_session` busca/crea. | Índice `idx_active_session_per_table` se remueve del metadata de test (research.md §3) para poder sembrar dos filas `active` de la misma mesa (A-17/R16). |
| `SessionParticipant` | `session_participants` | Comensal; `open_session` lo crea, `cancel_my_order`/`leave_session` lo reciben ya resuelto. | `expires_at` se calcula con `_now()` — depende del reloj fijado (research.md §5) cuando el test lo congela. |
| `Cart` | `carts` | Carrito abierto/confirmado/abandonado de un comensal. | Índice `idx_open_cart_per_participant` se remueve del metadata de test (research.md §3) para permitir el segundo carrito tras `submit_cart`. |
| `CartItem` / `CartItemOption` | `cart_items` / `cart_item_options` | Líneas y opciones del carrito; lo que `add_item`/`update_item`/`remove_item` mutan. | — |
| `CustomerOrder` / `OrderItem` / `OrderItemOption` | `customer_orders` / `order_items` / `order_item_options` | Pedido que resulta de `submit_cart`; lo que `list_my_orders`/`cancel_my_order` leen. | — |
| `Promotion` / `PromotionTarget` / `PromotionComboItem` | `promotions` / `promotion_targets` / `promotion_combo_items` | Fixture mínimo para `active_discount_promotions`/`best_line_discount` (descuento simple) y `expand_combo` (combo con 1-2 componentes). | Un caso "sin promoción activa" (default) y un caso "una promoción activa" bastan (FR-007) — no se caracterizan las reglas propias del módulo. |
| `OrderCancelLog` | `order_cancel_logs` | Auditoría que `checkout.cancel_order` escribe al cancelar (vía `record_audit`). | — |
| `AuditLog` | `audit_logs` | Tabla destino de `record_audit`; `payload` es `JSONB`, compila como `JSON` genérico en SQLite. | Se verifica con un test de humo del fixture antes de escribir los tests de negocio (research.md §4). |

## 2. Entidades de infraestructura de test (nuevas, solo en `cart_fixtures.py`)

Estas no son modelos SQLAlchemy: son los objetos Python que el harness de router construye para
sustituir lo que en producción resuelve `Depends`/los context managers de `app.core.qr_context`.

- **`SessionContext` de prueba**: instancia real del dataclass `app.core.qr_context.SessionContext`
  (`db`, `tenant`, `table_id`, `participant`, `table_session`), poblada con los objetos creados por
  las factories de este fixture en vez de resueltos desde un token+Postgres real. Se pasa como
  argumento `ctx=` directamente a las 7 funciones de endpoint que hoy lo reciben vía
  `Depends(get_session_context)`.
- **`QrContext` de prueba**: análogo para `app.core.qr_context.QrContext` (`db`, `tenant`, `table_id`),
  usado por el doble que sustituye `open_qr_context` en el test de `POST /cart/sessions`.
- **Tenant de prueba**: no requiere el modelo `Tenant` real de `app.core.models` (que vive en el
  schema `shared`, fuera del alcance de tablas creadas); basta un objeto mínimo con los atributos que
  `cart`/`router.py` leen de él (`id`; `schema` no se usa porque el fixture no pasa por `with_db`).
  Se documenta explícitamente en `contracts/test-harness-api.md` cuáles atributos son necesarios.
- **`FakeRedisBucket`**: doble mínimo con `incr(key)`/`expire(key, ttl)` async, contador en memoria
  por instancia, usado solo en el test que congela el 429 de rate limiting (Historia 2, escenario 5).
  No implementa el resto de la API de `redis.asyncio.Redis` — el harness no lo necesita.
- **`FixedDatetime`**: subclase de `datetime.datetime` con `now(tz=None)` sobrescrito para devolver un
  instante fijo, usada por `frozen_now()` para el caso A-08 (Historia 1, escenario 3) y por cualquier
  otro test que necesite un `expires_at`/`created_at` determinista.

## 3. Relaciones y ciclo de vida relevantes para los tests

```text
DiningTable (1) ──< TableSession (1 activa por mesa en Postgres;
                     sin esa restricción en el fixture de test, ver research.md §3)
TableSession (1) ──< SessionParticipant (N comensales por sesión)
SessionParticipant (1) ──< Cart (1 abierto a la vez en Postgres;
                     sin esa restricción en el fixture de test)
Cart (1) ──< CartItem (N líneas) ──< CartItemOption (N opciones por línea)
SessionParticipant (1) ──< CustomerOrder (N pedidos enviados)
CustomerOrder (1) ──< OrderItem (N líneas) ──< OrderItemOption
CustomerOrder (1) ──< OrderCancelLog (0..1 por cancelación)
Promotion (1) ──< PromotionTarget (0..N alcance) / PromotionComboItem (0..N componentes de combo)
```

No hay transiciones de estado nuevas que documentar: las que importan (`Cart.status`,
`TableSession.status`, `CustomerOrder.status`, `SessionParticipant.status`) ya están descritas en los
docstrings de sus modelos de producción y en `spec.md` (Acceptance Scenarios); esta spec las observa,
no las redefine.
