# Contrato: API pública de `table_sessions_fixtures.py`

`app/characterization_tests/table_sessions_fixtures.py` es infraestructura de test, no código de
producción, pero los tres ficheros de esta spec dependen de su forma exacta — este contrato fija
esa forma antes de implementarla, igual que `contracts/test-harness-api.md` de
`specs/015-caracterizacion-cart/` fijó la de `cart_fixtures.py`.

## Sesión y base de datos

```python
def new_session() -> Session:
    """Sesión SQLAlchemy real sobre SQLite en memoria, con el esquema ampliado
    (catálogo, reexportado de fixtures.py, más mesas/sesión/comensales/pedidos/
    promociones/caja/ventas/factura). Registra el compilador JSONB->JSON para
    SQLite antes de create_all (research.md §4). No remueve ningún índice único
    parcial (research.md §6: esta spec no lo necesita)."""
```

Reexporta tal cual (sin duplicar) desde `fixtures.py`: `make_unit`, `make_category`,
`make_product`, `make_variant`, `make_inventory_item`, `make_option_group`, `make_option`,
`link_variant_group`, `make_recipe_item`.

## Factories nuevas (mesas, sesión, comensales, pedidos, promociones, caja, ventas)

Mismo patrón que las factories existentes de `fixtures.py`/`cart_fixtures.py`:
`kw.setdefault(...)` para cada columna con valor por defecto razonable, `db.add` + `db.flush()`,
devuelven el objeto ORM ya persistido (no comiteado salvo que la propia función de servicio bajo
prueba lo requiera).

```python
def make_dining_table(db: Session, **kw) -> DiningTable: ...
def make_table_session(db: Session, table: DiningTable | None = None, **kw) -> TableSession: ...
def make_participant(
    db: Session, table_session: TableSession | None = None, **kw
) -> SessionParticipant: ...
def make_customer_order(
    db: Session, table_session: TableSession, participant: SessionParticipant | None = None, **kw
) -> CustomerOrder: ...
def make_order_item(
    db: Session, order: CustomerOrder, variant: ProductVariant, **kw
) -> OrderItem: ...
def make_promotion(db: Session, **kw) -> Promotion:
    """kw.setdefault fuerza start_time=None, end_time=None (research.md §5):
    sin ventana horaria, siempre válida sin importar el reloj real."""
def make_promotion_target(db: Session, promotion: Promotion, **kw) -> PromotionTarget: ...
def make_combo_item(
    db: Session, promotion: Promotion, variant: ProductVariant, **kw
) -> PromotionComboItem: ...
def make_cash_register(db: Session, **kw) -> CashRegister: ...
def make_cash_shift(
    db: Session, register: CashRegister | None = None, **kw
) -> CashShift: ...
def make_payment_method(db: Session, **kw) -> PaymentMethod:
    """kw.setdefault(is_cash=True): la mayoría de los tests de cierre no
    necesita distinguir método; is_cash=True evita el chequeo de
    'no_efectivo > total' de build_sale por defecto."""
```

Cada factory usa los mismos defaults "inertes" que ya establecen `fixtures.py`/`cart_fixtures.py`
(nombres derivados del `id` generado, `active=True`/`status` operativo por defecto) para que un
test que no le importa un campo concreto no tenga que pasarlo.

## Dobles de prueba para `Tenant`/`User` (router)

```python
def make_tenant_double(*, id: int = 1, invoice_prefix: str = "") -> SimpleNamespace:
    """No es el modelo Tenant real (schema shared, fuera de las tablas creadas
    por este fixture) — basta con los dos atributos que table_sessions/
    router.py y service.py leen de él (research.md §1)."""

def make_user_double(*, id=None, name: str = "Cajero de prueba") -> SimpleNamespace:
    """No es el modelo User real, por el mismo motivo. id por defecto es un
    uuid4() nuevo si no se pasa."""
```

## Espía de `_load` (A-17/R12)

```python
class spy_load:
    """Context manager: parchea app.api.v1.table_sessions.service._load con
    wraps=service._load (research.md §2) — la función real se sigue
    ejecutando, pero .calls queda disponible como lista de
    (table_session_id, lock) por cada invocación durante el bloque `with`.

    Uso:
        with table_sessions_fixtures.spy_load() as spy:
            service.add_participant(db, ts.id, "Ana", tenant_id=1)
        assert spy.calls[-1].lock is False
    """
```

## Doble de `events.bill_changed` (FR-010a)

```python
class spy_bill_changed:
    """Context manager: parchea app.core.events.bill_changed
    (unittest.mock.patch con side_effect) para no abrir un socket real a
    Redis y registrar cada invocación. .calls queda disponible como lista de
    (tenant_id, table_session_id) en el orden en que ocurrieron.

    Uso:
        with table_sessions_fixtures.spy_bill_changed() as spy:
            service.add_participant(db, ts.id, "Ana", tenant_id=1)
        spy.assert_called_once_with(tenant_id=1, table_session_id=ts.id)
    """
```

Los otros tres eventos que dispara `close_session` (`payment_completed`, `session_closed`,
`table_status_changed`) no tienen doble: se apoyan en el diseño *fail-open* ya existente de
`app.core.events.publish` (research.md §3) — ningún test de esta spec necesita interceptarlos.

## Qué NO expone este módulo

- No reexporta ni depende de nada de `cart_fixtures.py` (`SessionContext`, `QrContext`,
  `patched_qr_context`, `patched_session_context`, `FakeRedisBucket`, `frozen_now`): ninguno
  aplica a `table_sessions/router.py` ni a `table_sessions/service.py` (research.md §1, §5).
- No provee ningún mecanismo de reloj fijado (`frozen_now`): esta spec no lo necesita
  (research.md §5). Si una spec futura de `table_sessions` lo necesitara, se añadiría entonces,
  replicando (no importando) el patrón de `cart_fixtures.frozen_now` — mismo motivo por el que
  este módulo no importa de `cart_fixtures.py`.
- No doble de `orders.checkout`, `promotions.service`, `sales.builder` ni `invoices.service`: se
  ejercitan reales, sin mocks (FR-009).
