# Quickstart: validar la corrección de zona horaria en menú y carrito (A-08)

Guía de ejecución para comprobar que la corrección cumple su contrato. No repite firmas ni tablas
ya detalladas en [data-model.md](./data-model.md),
[contracts/menu-endpoint.md](./contracts/menu-endpoint.md) y
[contracts/cart-endpoint.md](./contracts/cart-endpoint.md) — solo enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` activado (Python 3.14), ejecutado desde la raíz
de `../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL real: `app/characterization_tests/cart_fixtures.py` crea SQLite en
memoria y ya trae `make_category`, `make_product`, `make_variant`, `make_promotion`, `new_session`
y el reloj fijado `frozen_now` (research.md Decisión 2/3).

## Paso 1 — Confirmar el defecto actual del lado `cart` (antes de tocar código)

```bash
python3 -m unittest app.characterization_tests.test_cart_service -v
```

**Resultado esperado antes del fix**:
`test_open_session_y_serialize_cart_a08_zona_horaria_no_aplicada`
(`test_cart_service.py:137-160`) pasa en verde **congelando el defecto** — con el reloj fijado a
las 20:00 UTC (15:00 Bogotá, fuera de una ventana 20:00-21:00), el carrito sí aplica el descuento.

## Paso 2 — Generalizar `frozen_now` para aceptar el módulo a parchear

Editar `app/characterization_tests/cart_fixtures.py` según research.md Decisión 2:

```python
class frozen_now:
    def __init__(self, instant: datetime, module: str = "app.api.v1.cart.service"):
        self._instant = instant
        self._module = module
        self._patcher = None

    def __enter__(self) -> "frozen_now":
        instant = self._instant

        class FixedDatetime(datetime):
            @classmethod
            def now(cls, tz=None):
                return instant

        self._patcher = mock.patch(f"{self._module}.datetime", FixedDatetime)
        self._patcher.start()
        return self
    # __exit__ sin cambios
```

Ningún test existente cambia de comportamiento: el default sigue siendo `cart.service`.

## Paso 3 — Escribir el test nuevo del lado `menu` (RED, antes del fix)

Crear `app/characterization_tests/test_menu_router.py`:

```python
"""CONGELA comportamiento actual → corregido: app/api/v1/menu/router.py:82
(_build_menu) — A-08 (registro-de-anomalias.md, contraste con A-07).

Ejecutar solo este módulo:

    python -m unittest app.characterization_tests.test_menu_router -v
"""
from datetime import datetime, time, timezone
from decimal import Decimal
import unittest

from app.characterization_tests import cart_fixtures as fx
from app.api.v1.menu.router import _build_menu


class TestBuildMenuA08(unittest.TestCase):
    def test_a08_fuera_de_ventana_en_hora_local_no_descuenta(self):
        db = fx.new_session()
        category = fx.make_category(db)
        product = fx.make_product(db, category=category)
        fx.make_variant(db, product=product, price=Decimal("10000"))
        fx.make_promotion(
            db, type="percent", value=Decimal("20"), status="active",
            start_time=time(20, 0), end_time=time(21, 0),
        )
        db.commit()

        instant = datetime(2026, 1, 15, 20, 0, tzinfo=timezone.utc)  # 15:00 Bogotá
        with fx.frozen_now(instant, module="app.api.v1.menu.router"):
            menu = _build_menu(db)

        variant_resp = menu[0].products[0].variants[0]
        self.assertIsNone(variant_resp.discounted_price)  # tras el fix: sin descuento
```

```bash
python3 -m unittest app.characterization_tests.test_menu_router -v
```

**Resultado esperado antes del fix**: el test **falla** (`discounted_price` no es `None` — el
código actual sí aplica el descuento fuera de la ventana real). Confirma A-08 en `menu` con datos
reales.

## Paso 4 — Aplicar la corrección

Editar `app/api/v1/menu/router.py:82` según research.md Decisión 1:

```python
# Antes (A-08):
now = datetime.now(timezone.utc).replace(tzinfo=None)
# Después:
now = datetime.now(timezone.utc)
```

Editar `app/api/v1/cart/service.py:205` (`serialize_cart`):

```python
# Antes (A-08):
now = _now()
# Después:
now = datetime.now(timezone.utc)
```

Sin import nuevo — `datetime`/`timezone` ya están importados en ambos ficheros. `_now()` (líneas
52-53 de `cart/service.py`) y su uso en `open_session:107` quedan intactos (FR-004).

## Paso 5 — Modificar el test `CONGELA` existente de `cart` (Principio II)

Actualizar `test_open_session_y_serialize_cart_a08_zona_horaria_no_aplicada` en
`test_cart_service.py:137-160` para verificar el comportamiento corregido, **citando A-08 en el
mismo commit**:

```python
def test_serialize_cart_a08_zona_horaria_aplicada_tras_la_correccion(self):
    """CONGELA comportamiento corregido — A-08 (`cart/service.py:205`): a las
    20:00 UTC (15:00 Bogotá), fuera de una ventana 20:00-21:00 local, el
    carrito NO aplica el descuento (cita: registro-de-anomalias.md, A-08)."""
    db, table, ts, participant = self._seed_session()
    variant, product, category = self._seed_variant(db)
    cart_fixtures.make_promotion(
        db, type="percent", value=Decimal("20"), status="active",
        start_time=time(20, 0), end_time=time(21, 0),
    )

    instant = datetime(2026, 1, 15, 20, 0, tzinfo=timezone.utc)
    with cart_fixtures.frozen_now(instant):
        resp = service.add_item(
            db, participant.id,
            CartItemIn(product_variant_id=variant.id, quantity=1),
        )

    self.assertIsNone(resp.discounted_total)
```

## Paso 6 — Confirmar la corrección

```bash
python3 -m unittest app.characterization_tests.test_cart_service -v
python3 -m unittest app.characterization_tests.test_menu_router -v
```

**Resultado esperado tras el fix**: ambos tests en verde — el descuento deja de aplicarse fuera de
la ventana horaria real (FR-001/FR-002, CA1/CA2), y sigue aplicándose dentro de ella sin regresión
(FR-003, CA3, ver el segundo escenario de cada historia en `spec.md`).

## Paso 7 — No regresión en `expires_at` (FR-004, Historia 3)

```bash
python3 -m unittest app.characterization_tests.test_table_sessions_service -v
python3 -m unittest app.characterization_tests.test_cart_router -v
```

**Resultado esperado**: sin cambios — ningún test que dependa de `expires_at`/`open_session`
cambia de resultado, confirmando que `_now()` y su uso para el TTL de sesión quedaron intactos.

## Paso 8 — No regresión en el motor de promociones (A-07 protegida)

```bash
python3 -m unittest app.scripts.test_promotions_rules -v
```

**Resultado esperado**: sin cambios — este script (único que corre en CI) no importa `cart` ni
`menu`; confirma que `active_discount_promotions`/`local_now`/`best_line_discount` no se tocaron.

## Verificación final — SC-001 a SC-005

```bash
python3 -m unittest discover -s app/characterization_tests -p "test_*.py"
```

Suite completa en verde, incluidos los dos tests nuevos/modificados de esta spec y todos los
preexistentes sin cambios, es la señal de que la corrección está completa y no introdujo ninguna
regresión (Principio II).
