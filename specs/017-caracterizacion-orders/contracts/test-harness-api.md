# Contrato: `app/characterization_tests/orders_fixtures.py`

No es una API de producción — es el contrato interno que deben respetar los cinco ficheros de
test nuevos (`test_orders_consolidation.py`, `test_orders_checkout.py`, `test_orders_kitchen.py`,
`test_orders_tables_advanced.py`, `test_orders_service.py`) al consumir el fixture module de esta
spec. Sigue la misma forma que `contracts/test-harness-api.md` de
`specs/016-caracterizacion-table-sessions/`, adaptada al superconjunto de tablas que `orders`
necesita (data-model.md).

## Sesión y base de datos

```python
def new_session() -> Session
```

Devuelve una `sqlalchemy.orm.Session` real sobre un motor SQLite en memoria nuevo, con las 10
tablas de catálogo (reexportadas de `fixtures.py`) más las 22 tablas nuevas de esta spec
(data-model.md §Esquema SQLite) ya creadas (`Base.metadata.create_all`). Cada llamada crea un
motor y una base nuevos — sin estado compartido entre tests. Registra `@compiles(JSONB,
"sqlite")` → `"JSON"` antes de `create_all()` (idempotente si convive en el mismo proceso con
`cart_fixtures.py`/`table_sessions_fixtures.py`) y aplica el mismo parche de
`server_default` de `sale_items.options` que ya documentó `table_sessions_fixtures.py`
(`_patch_sqlite_incompatible_server_defaults`, solo sobre el metadata en memoria, no toca
`app/models/sale.py`).

Los tests nunca llaman `Base.metadata.create_all` ni abren un `engine` propio: siempre pasan por
`new_session()`.

## Factories reexportadas de `fixtures.py` (sin cambios)

`make_category`, `make_product`, `make_variant`, `make_option_group`, `make_option`,
`link_variant_group`, `make_recipe_item`, `make_inventory_item`, `make_unit` — mismas firmas que
documenta `fixtures.py`, reexportadas tal cual.

## Factories nuevas de `orders_fixtures.py`

Todas siguen el mismo patrón: reciben la `Session`, argumentos opcionales de las entidades padre
(creando una por defecto si no se pasa), `**kw` con `setdefault` para cada columna no obligatoria,
`db.add` + `db.flush()` (nunca `commit()` — el commit lo hace la función de producción bajo
prueba, o el propio test si necesita aislar el fixture del `try/except` de la función).

```python
def make_dining_table(db: Session, **kw) -> DiningTable
def make_table_session(db: Session, table: DiningTable | None = None, **kw) -> TableSession
def make_participant(db: Session, table_session: TableSession | None = None, **kw) -> SessionParticipant
def make_customer_order(db: Session, table_session: TableSession, participant: SessionParticipant | None = None, **kw) -> CustomerOrder
def make_order_item(db: Session, order: CustomerOrder, variant: ProductVariant, **kw) -> OrderItem
def make_order_item_void_log(db: Session, order_item: OrderItem, **kw) -> OrderItemVoidLog
def make_cart(db: Session, participant: SessionParticipant | None = None, **kw) -> Cart
def make_cart_item(db: Session, cart: Cart, variant: ProductVariant, **kw) -> CartItem
def make_promotion(db: Session, **kw) -> Promotion
def make_promotion_target(db: Session, promotion: Promotion, **kw) -> PromotionTarget
def make_combo_item(db: Session, promotion: Promotion, variant: ProductVariant, **kw) -> PromotionComboItem
def make_cash_register(db: Session, **kw) -> CashRegister
def make_cash_shift(db: Session, register: CashRegister | None = None, **kw) -> CashShift
def make_payment_method(db: Session, **kw) -> PaymentMethod
```

Defaults que un test puede necesitar conocer sin leer el código del fixture:

- `make_order_item(..., estado_cocina="listo")` por defecto — cobrable de inmediato; los tests
  que ejercitan cocina en curso (A-16, `block_order`) o anulados lo sobreescriben explícitamente.
- `make_customer_order(..., status="abierta")` por defecto — los tests que ejercitan `recibida`,
  `bloqueada`, `pagada` o `cancelada` (A-01, A-16, Historia 2) lo sobreescriben.
- `make_promotion(..., start_time=None, end_time=None)` — sin ventana horaria, siempre válida sin
  importar el reloj real (mismo criterio de `table_sessions_fixtures.py`, research.md §5 de la
  spec 016).
- `make_payment_method(..., is_cash=True)` — evita el chequeo "no_efectivo > total" de
  `build_sale` por defecto; los tests de pago mixto lo sobreescriben.
- `make_order_item_void_log(..., motivo="prueba", user_id=<uuid4 nuevo>, user_name="Cajero de
  prueba")` — no es una factory que ningún test de esta spec necesite llamar directamente para
  sembrar estado previo (`void_item` la crea internamente); existe solo para que la tabla del
  esquema quede completa y disponible si algún caso de aceptación necesitara verificar una fila
  ya existente.

## Dobles de `Tenant`/`User`

```python
def make_tenant_double(*, id: int = 1, invoice_prefix: str = "") -> SimpleNamespace
def make_user_double(*, id=None, name: str = "Cajero de prueba") -> SimpleNamespace
```

No son los modelos reales (schema `shared`, fuera de las tablas creadas por este fixture): bastan
los atributos que las 23 funciones leen (`user.id`, `user.name`; `tenant.id`,
`tenant.invoice_prefix` si algún test necesita `pay_order` con prefijo — hoy ninguno lo pasa
explícito, `checkout.pay_order` usa el default `""` de `build_sale`, ver research.md §4). Mismo
patrón exacto que `cart_fixtures.py`/`table_sessions_fixtures.py`; `orders/router.py` está fuera
de alcance (Assumptions de `spec.md`), así que estos dobles se usan aquí solo para invocar las
funciones de `service.py`/`checkout.py`/`consolidation.py`/`kitchen.py`/`tables_advanced.py`
directamente, nunca para resolver un `Depends`.

## Forzado de `IntegrityError` (A-26/RN-ORD-60)

```python
class force_flush_integrity_error:
    """Context manager: parchea `db.flush` (de instancia, no de clase) con
    `side_effect=IntegrityError(...)` durante el bloque `with`, para ejercitar el
    `except IntegrityError` huérfano de `move_order` (research.md §2). Fuera del
    bloque, `db.flush` vuelve a su comportamiento real."""
```

Uso:

```python
with orders_fixtures.force_flush_integrity_error(db):
    with self.assertRaises(HTTPException) as ctx:
        tables_advanced.move_order(db, order.id, target_table.id)
assert ctx.exception.status_code == 409
```

No es una factory de fila — no crea ninguna entidad. Es el único mock que introduce esta spec:
todo lo demás se ejercita contra código real, sin dobles (FR-010, FR-011, FR-012).

## Qué NO expone este fixture

- Ningún espía de evento (`app.core.events`): `orders` no dispara ningún evento de
  `app.core.events` desde ninguna de sus 23 funciones públicas (research.md, plan.md Technical
  Context) — a diferencia de `table_sessions`, que sí necesitó `spy_bill_changed`.
- Ningún espía de `_load`/lock (`spy_load` de `table_sessions_fixtures.py`): ninguna anomalía en
  alcance de esta spec depende de observar el argumento de `with_for_update` (research.md §6).
- Ningún helper de reloj fijado (`frozen_now`): ninguna anomalía en alcance depende del reloj
  real (mismo criterio documentado por `table_sessions`, research.md §5 de la spec 016).
- Nada de `orders/router.py`: fuera de alcance (Assumptions de `spec.md`) — las 23 funciones se
  invocan directamente como funciones Python, nunca vía `fastapi.testclient` ni resolviendo
  `Depends`.
