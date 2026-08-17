# Contrato: API pública de `cart_fixtures.py`

`app/characterization_tests/cart_fixtures.py` es infraestructura de test, no código de producción,
pero `test_cart_service.py` y `test_cart_router.py` dependen de su forma exacta — este contrato fija
esa forma antes de implementarla, igual que `014-extraccion-motor-catalogo/contracts/module-api.md`
fijó la API del paquete que esa spec extrajo.

## Sesión y base de datos

```python
def new_session() -> Session:
    """Sesión SQLAlchemy real sobre SQLite en memoria, con el esquema ampliado
    (catálogo + inventario, reexportado de fixtures.py, más mesas/carrito/
    pedidos/promociones). Remueve idx_active_session_per_table e
    idx_open_cart_per_participant del metadata antes de create_all (ver
    research.md §3) — cada llamada es idempotente respecto a esa remoción."""
```

Reexporta tal cual (sin duplicar) desde `fixtures.py`: `make_unit`, `make_category`, `make_product`,
`make_variant`, `make_inventory_item`, `make_option_group`, `make_option`, `link_variant_group`,
`make_recipe_item`.

## Factories nuevas (mesas, sesión, carrito, pedidos, promociones)

Mismo patrón que las factories existentes de `fixtures.py`: `kw.setdefault(...)` para cada columna
con valor por defecto razonable, `db.add` + `db.flush()`, devuelven el objeto ORM ya persistido
(no comiteado salvo que la propia función de servicio bajo prueba lo requiera).

```python
def make_dining_table(db: Session, **kw) -> DiningTable: ...
def make_table_session(db: Session, table: DiningTable | None = None, **kw) -> TableSession: ...
def make_participant(
    db: Session, table_session: TableSession | None = None, **kw
) -> SessionParticipant: ...
def make_cart(db: Session, participant: SessionParticipant | None = None, **kw) -> Cart: ...
def make_cart_item(
    db: Session, cart: Cart, variant: ProductVariant, **kw
) -> CartItem: ...
def make_customer_order(
    db: Session, participant: SessionParticipant, **kw
) -> CustomerOrder: ...
def make_promotion(db: Session, **kw) -> Promotion: ...
def make_promotion_target(db: Session, promotion: Promotion, **kw) -> PromotionTarget: ...
def make_combo_item(
    db: Session, promotion: Promotion, variant: ProductVariant, **kw
) -> PromotionComboItem: ...
```

Cada factory usa los mismos defaults "inertes" que ya establece `fixtures.py` (nombres derivados del
`id` generado, `active=True` donde aplique) para que un test que no le importa un campo concreto no
tenga que pasarlo.

## Contexto de router (sustituye `Depends`/context managers de `app.core.qr_context`)

```python
def build_session_context(
    db: Session, *, tenant_id: int = 1,
    table: DiningTable, table_session: TableSession, participant: SessionParticipant,
) -> "app.core.qr_context.SessionContext":
    """SessionContext real (mismo dataclass de producción) poblado a mano. Se
    pasa como ctx= directamente a las 7 funciones de endpoint que lo reciben
    vía Depends(get_session_context) — Depends no se resuelve al llamar la
    función Python directamente, así que no hace falta ningún parche para
    estas 7."""


class patched_qr_context:
    """Context manager: parchea app.api.v1.cart.router.open_qr_context para
    que, en vez de resolver el tenant vía Postgres real, entregue un
    QrContext de prueba fijo (o levante la misma HTTPException 401 que el
    código real levantaría con un token inválido, si el test lo pide).
    Uso: with patched_qr_context(ctx=mi_qr_context): service.open_session(...)."""


class patched_session_context:
    """Análogo para app.api.v1.cart.router.open_session_context, usado por el
    test de POST /cart/leave con token válido (el caso sin token no lo
    necesita: el endpoint retorna antes de abrir el contexto)."""
```

## Reloj fijado (A-08)

```python
class frozen_now:
    """Context manager: parchea app.api.v1.cart.service.datetime por una
    subclase cuyo .now(tz) devuelve `instant` fijo. timedelta/timezone del
    módulo no se tocan. Uso:
        with frozen_now(instant=datetime(2026, 1, 15, 20, 0, tzinfo=timezone.utc)):
            resp = service.open_session(db, tenant_id, table, "Ana")
    """
    def __init__(self, instant: datetime): ...
```

## Doble de Redis (solo Historia 2, escenario 5)

```python
class FakeRedisBucket:
    """Doble mínimo de app.core.redis.token_blocklist para
    app.core.rate_limit.enforce: incr/expire async, contador en memoria por
    instancia. No implementa el resto de la API de redis.asyncio.Redis.
    Uso: with mock.patch("app.core.rate_limit.redis", FakeRedisBucket()): ...
    junto con mock.patch.object(settings, "RATE_LIMIT_ENABLED", True)."""
    async def incr(self, key: str) -> int: ...
    async def expire(self, key: str, ttl: int) -> None: ...
```

## Qué NO expone este módulo

- No reexporta ni envuelve `mint_qr_token`/`verify_qr_token`/`mint_session_token`/
  `verify_session_token` (`app.core.qr_token`): el harness de router no pasa por la verificación real
  del token porque sustituye el punto que la invoca (`open_qr_context`/`open_session_context`), así
  que un test que solo quiera congelar la respuesta 401 por token inválido puede seguir llamando a
  `verify_qr_token` directamente si lo necesita, sin que este módulo tenga que envolverlo.
- No provee un doble de `app.core.db.with_db`/`resolve_tenant_by_id`: estos dos nunca se ejecutan en
  la ruta de test (ver research.md §1), así que no hace falta interceptarlos.
