# Research: Corrección de zona horaria en vigencia de promociones del menú y carrito QR (A-08)

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — se resolvió por completo
leyendo `pos-backend` (el patrón correcto ya existe en cuatro ficheros distintos:
`checkout.py`/`table_sessions/service.py`/`sales/service.py`) y el propio "tratamiento acordado" de
A-08, que no deja ambigüedad sobre qué hacer. Este documento registra las decisiones de diseño.

## Decisión 1 — Corregir el punto de invocación, no `_now()` en sí

- **Decisión**: en `cart/service.py:205` (`serialize_cart`), reemplazar `now = _now()` por
  `now = datetime.now(timezone.utc)` (aware) — una asignación local, sin modificar la función
  compartida `_now()` (líneas 52-53) ni su otro uso en `open_session:107` (`expires_at`). En
  `menu/router.py:82` (`_build_menu`), reemplazar `now = datetime.now(timezone.utc).replace(
  tzinfo=None)` por `now = datetime.now(timezone.utc)`.
- **Rationale**: `_now()` de `cart/service.py` se usa en dos sitios con requisitos distintos: (1)
  `serialize_cart` lo pasa a `active_discount_promotions`, que necesita un `datetime` aware para
  convertirse correctamente vía `local_now()`; (2) `open_session` lo usa para calcular
  `expires_at`, un `DateTime` **naive** en base de datos (`SessionParticipant.expires_at`) que se
  compara en `qr_context.py`/`scheduler.py` contra otro `datetime` naive
  (`datetime.now(timezone.utc).replace(tzinfo=None)`). Si `_now()` empezara a devolver un
  `datetime` aware, `expires_at` quedaría con tzinfo mientras el resto del sistema lo sigue
  tratando como naive — riesgo real de `TypeError` (comparar naive vs. aware) o de un desfase
  silencioso en el TTL de sesión, una anomalía completamente distinta a A-08 (Principio III: un
  módulo a la vez). Cambiar el `datetime` únicamente en el punto de invocación de `serialize_cart`
  resuelve el 100% de A-08 sin tocar ese segundo uso.
- **Alternatives considered**: convertir `_now()` en sí a aware y actualizar también
  `open_session`/`qr_context.py`/`scheduler.py` para que comparen aware contra aware — descartado
  para esta delta: expandiría el alcance de A-08 a una anomalía distinta (TTL de sesión) no
  autorizada por el registro de anomalías, violando Principio I (ningún cambio de comportamiento
  sin decisión de negocio que lo cite) y Principio III.

## Decisión 2 — Cómo fijar el reloj para el test nuevo de `menu`

- **Decisión**: generalizar `cart_fixtures.frozen_now` para aceptar un parámetro `module: str`
  (nombre completo del módulo cuyo `datetime` se parchea), con valor por defecto
  `"app.api.v1.cart.service"` — el mismo comportamiento de hoy para los tests existentes de `cart`,
  sin tocarlos. El test nuevo de `menu` lo invoca con
  `module="app.api.v1.menu.router"`.
- **Rationale**: `frozen_now` ya resuelve exactamente el problema de fijar `datetime.now(tz)` sin
  importar el `tz` recibido (`cart_fixtures.py:292-320`), parcheando la clase `datetime` importada
  en el módulo de producción indicado. Generalizarlo con un parámetro es un cambio aditivo y
  retrocompatible (mismo default), evita duplicar ~15 líneas de lógica de mock delicada en un
  segundo fichero, y mantiene un solo lugar de verdad para "cómo se fija el reloj en los tests de
  characterization de zona horaria" — consistente con que ambos puntos corregidos (`cart`, `menu`)
  pertenecen a la misma anomalía A-08.
- **Alternatives considered**: escribir una clase `frozen_now` duplicada dentro de
  `test_menu_router.py` — descartado, duplica lógica de mock ya escrita y probada en spec 015 sin
  ganancia real, y dificultaría mantener el mismo comportamiento si `frozen_now` cambia en el
  futuro.

## Decisión 3 — Reutilizar el harness de `cart_fixtures.py` para `test_menu_router.py`

- **Decisión**: `test_menu_router.py` importa sus factories (`make_category`, `make_product`,
  `make_variant`, `make_promotion`, `new_session`, etc.) desde `cart_fixtures.py`, en vez de crear
  un `menu_fixtures.py` nuevo.
- **Rationale**: `cart_fixtures.py` ya extiende el motor SQLite-en-memoria de `fixtures.py` con las
  14 tablas de mesas/carrito/pedidos/**promociones** que `menu/router.py` también necesita
  (`Category`, `Product`, `ProductVariant`, `Promotion` y sus targets) — no hay ninguna tabla que
  `menu` necesite y `cart_fixtures.py` no tenga ya. Crear un tercer módulo de fixtures solo para
  duplicar ese mismo armado de tablas no aporta aislamiento real (ambos ejercitan el mismo motor de
  promociones) y sí más superficie de mantenimiento.
- **Alternatives considered**: crear `menu_fixtures.py` dedicado — descartado por duplicación
  innecesaria; usar directamente `fixtures.py` (el motor base, sin promociones) y añadir ahí las
  tablas de `Promotion` — descartado porque `fixtures.py` está documentado como "no se toca"
  (contrato de spec 015, `contracts/test-harness-api.md`) para no arriesgar los characterization
  tests que ya dependen de su forma actual.

## Decisión 4 — Alcance sobre los endpoints que exponen `menu` y `cart`

- **Decisión**: la corrección de `menu` se aplica en `_build_menu`, la función privada que
  construyen ambos endpoints públicos (`GET /menu` y `GET /menu/qr-token/{token}`); la de `cart` se
  aplica en `serialize_cart`, invocada por los endpoints de `cart/router.py` que devuelven el
  carrito serializado (`GET /cart`, `POST /cart/items`, `PATCH /cart/items/{item_id}`, etc.).
- **Rationale**: verificado en `menu/router.py` y `cart/service.py` — ambos endpoints de menú
  (`GET /menu`, `GET /menu/qr-token/{token}`) llaman a `_build_menu`; `add_item`, `update_item`,
  `remove_item` y `_add_combo` (`cart/service.py:312,348,395,541`) terminan devolviendo
  `get_cart(db, participant_id)`, que a su vez llama `serialize_cart` (línea 271) — así que todos
  los endpoints de carrito que devuelven `CartResponse` pasan, directa o indirectamente vía
  `get_cart`, por `serialize_cart`; corregir la función compartida una sola vez en cada módulo
  corrige todos sus callers sin lógica adicional ni duplicada.
- **Alternatives considered**: ninguna — no hay una alternativa razonable a corregir cada función
  compartida una sola vez.

## Decisión 5 — Shim de `bool_or` para SQLite (descubierta durante implementación, no anticipada en la planificación)

- **Decisión**: registrar `bool_or` como función agregada de SQLite
  (`sqlite3.Connection.create_aggregate`) sobre la conexión que abre
  `cart_fixtures.new_session()`, junto al shim ya existente de `JSONB` → `JSON`.
- **Rationale**: al ejecutar el primer test de `menu/router._build_menu` contra SQLite
  (T005 de `tasks.md`), `_option_availability` (`menu/router.py`) falló con
  `OperationalError: no such function: bool_or` — usa `func.bool_or(...)` (agregado de
  Postgres) para decidir si algún grupo de opciones no exige cantidad. El error ocurre en
  la compilación/ejecución del SQL, no por falta de filas, así que ningún seedeo de datos
  lo evita. Es exactamente el mismo tipo de gap que ya documentó y resolvió spec 015 para
  `JSONB` (`cart_fixtures.py:105-114`) — un shim de test, no una migración de modelo ni un
  cambio de producción.
- **Alternatives considered**: reescribir la consulta de `_option_availability` para no usar
  `bool_or` — descartado, tocaría producción fuera del alcance de A-08 (Principio III);
  mockear `_option_availability` en el test — descartado, dejaría de ejercitar el código real
  del endpoint que el test dice caracterizar.
