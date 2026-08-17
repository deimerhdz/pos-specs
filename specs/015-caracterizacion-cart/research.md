# Research: Red de characterization tests para `cart`

Todos los "NEEDS CLARIFICATION" del Technical Context se resuelven aquí antes de Fase 1. Ninguno
requiere clarificación de negocio (`speckit-clarify`): son decisiones de implementación explícitamente
delegadas a esta fase por las Assumptions de `spec.md`.

## 1. Mecanismo para congelar los 9 endpoints de `router.py`

**Decisión**: invocar las funciones de endpoint de `router.py` directamente como funciones Python
(no vía `fastapi.testclient.TestClient` ni un servidor ASGI real), pasando un `SessionContext`
construido a mano donde el endpoint lo recibe por `Depends(get_session_context)`, y sustituyendo con
un doble de prueba los dos endpoints que abren su propio contexto **dentro** del cuerpo de la función
(`open_session` vía `open_qr_context`, `leave` vía `open_session_context`).

**Rationale**: `get_session_context`/`open_session_context`/`open_qr_context`
(`app/core/qr_context.py`) y, transitivamente, `resolve_tenant_by_id`/`with_db`
(`app/core/db.py:33,152`) usan el `engine` de módulo de `app.core.db`, creado con
`settings.DATABASE_URL` — una conexión Postgres real. Levantar Postgres real rompería FR-008
(determinismo, cero infraestructura externa) y el precedente ya establecido por el resto de
`app/characterization_tests/`, que corre exclusivamente contra SQLite en memoria. `Depends(...)` solo
se resuelve cuando FastAPI atiende una request real (vía ASGI); llamar la función de Python
directamente lo ignora por completo, así que para los 7 endpoints que reciben `ctx` como parámetro
inyectado basta con construirlo y pasarlo — cero mocking. Los 2 endpoints que abren el contexto ellos
mismos (`open_session`: `with open_qr_context(body.qr_token) as ctx:`; `leave`:
`with open_session_context(x_session_token) as ctx:`) necesitan `unittest.mock.patch` sobre el nombre
importado en `app.api.v1.cart.router` (`open_qr_context`, `open_session_context`), devolviendo un
context manager que produce un `QrContext`/`SessionContext` de prueba en vez de resolver un tenant
real. El caso `leave` sin token (Historia 2, escenario 2) ni siquiera necesita el parche: el código
retorna antes de llamar a `open_session_context` cuando `x_session_token` es `None`
(`router.py:122-123`).

**Alternatives considered**:
- `fastapi.testclient.TestClient` sobre la app real: exigiría además overridear
  `app.dependency_overrides` para el resto del árbol de dependencias de la aplicación (rate limit,
  DB de otros módulos que ni siquiera tocan `cart`) y levantar la app completa solo para invocar 9
  rutas — más superficie de fallo no relacionado con `cart` por characterization test escrito, y
  ninguna ganancia: lo que se quiere congelar es el comportamiento de la función de endpoint, no el
  routing HTTP de FastAPI en sí (eso ya lo prueba el framework). Se descarta por complejidad
  injustificada.
- Levantar un Postgres real (contenedor) en CI para que `resolve_tenant_by_id`/`with_db` funcionen
  sin parches: contradice FR-008 y el precedente de determinismo de toda la red existente; además
  introduciría infraestructura de CI nueva no cubierta por Principio IV sin justificación de negocio.
  Se descarta.

## 2. Cómo neutralizar Redis (rate limiting y eventos) sin mocks de librería externa

**Decisión**: aprovechar que ambos caminos de producción que tocan Redis ya son *fail-open*
(`app/core/rate_limit.py:53-59` atrapa cualquier excepción y dejA pasar; `app/core/events.py:106-135`
atrapa cualquier excepción y devuelve `None` con un warning), así que en el camino feliz de la mayoría
de los tests **no hace falta ningún doble**: sin Redis real disponible, ambos módulos fallan
silenciosamente y el test observa el mismo comportamiento que producción tendría ante una caída de
Redis — que además es un camino que la propia spec pide congelar (`settings.RATE_LIMIT_ENABLED`, ya
existente, se deja en `False` para los tests que no ejercitan el límite en sí). Para el único
escenario que sí necesita observar el 429 con `Retry-After` (Historia 2, escenario 5), `cart_fixtures`
provee `FakeRedisBucket`, un objeto mínimo (`incr`/`expire` async, contador en memoria por proceso)
que se inyecta con `unittest.mock.patch("app.core.rate_limit.redis", FakeRedisBucket())` — incrementa
localmente en vez de golpear un socket real.

**Rationale**: cumple FR-001 (cero dependencias nuevas) y FR-008 (determinismo) sin necesitar un
Redis real en CI ni una librería como `fakeredis`. El comportamiento fail-open ya es parte del
contrato observable de `rate_limit.enforce`/`events.publish` (está documentado en sus propios
docstrings), así que ejercitarlo con Redis ausente no es un mock que oculte lógica: es el
comportamiento real del código ante la ausencia del servicio, uno de los casos que ya contempla.

**Alternatives considered**:
- `fakeredis` (dependencia nueva): resolvería lo mismo con menos código propio, pero exige
  justificación explícita en la spec y aprobación (Principio IV) para un caso que el propio código de
  producción ya maneja sin Redis. Se descarta por innecesario.
- Contenedor Redis real en CI (`services: redis:` en el workflow): funciona, pero es infraestructura
  de CI nueva para un solo escenario de un solo test, cuando el propio diseño fail-open del código ya
  da un camino sin dependencias. Se descarta por desproporcionado.

## 3. Índices únicos parciales que SQLite compila distinto a Postgres

**Decisión**: en `cart_fixtures.new_session()`, antes de `Base.metadata.create_all(...)`, remover de
`Table.indexes` los dos índices que usan el kwarg `postgresql_where` sobre tablas que esta spec
necesita: `idx_active_session_per_table` (`table_sessions`, `dining_table_id`) e
`idx_open_cart_per_participant` (`carts`, `participant_id`).

**Rationale**: `postgresql_where` es un kwarg de dialecto (`app/models/table_session.py:65-70`,
`app/models/cart.py:39-45`) que SQLAlchemy solo interpreta al compilar DDL para el dialecto
Postgres; sobre SQLite el índice se crea igual pero **sin** el predicado parcial, es decir, como
`UNIQUE` incondicional sobre la columna. Esto es más estricto que producción y rompe dos cosas
distintas si no se corrige:
- El caso de A-17/R16 (Historia 1, escenario 2) necesita sembrar **dos** `TableSession` con
  `status='active'` para la misma mesa — imposible de insertar si SQLite trata `dining_table_id` como
  único sin condición.
- El flujo normal de `submit_cart` → `_get_or_create_open_cart` (Historia 1, casi todos los
  escenarios que envían más de un pedido) crea un carrito `'abierto'` nuevo después de dejar el
  anterior en `'confirmado'` para el mismo `participant_id` — en Postgres es válido (el índice es
  parcial, solo cuenta `'abierto'`); en SQLite sin corregir, el segundo `INSERT` violaría el único
  incondicional aunque el primero ya no esté `'abierto'`.

Sin esta corrección, la mayoría de los tests de Historia 1 fallarían por un artefacto del motor de
test, no por el comportamiento real de `cart` — exactamente el caso que la Edge Case de spec.md
("el defecto está en el test, nunca en el código de producción") pide resolver ajustando el test/
fixture, nunca el modelo de producción. La remoción ocurre sobre el objeto `Table` en memoria
después de importar los modelos y antes de `create_all`; no se toca `table_session.py` ni `cart.py`
(cero líneas de producción modificadas, SC-005) — es una mutación exclusiva del `Base.metadata` vivo
del proceso de test, igual de aislada que el resto de lo que ya hace `fixtures.py`.

**Alternatives considered**:
- No sembrar el estado directamente y en su lugar recrear la condición de carrera con hilos/procesos
  reales: la propia spec (Edge Cases, Assumptions) ya descarta esto explícitamente por no
  determinismo.
- Insertar con SQL crudo (`db.execute(text("INSERT ..."))`) saltándose el ORM: no evita el problema,
  porque la restricción vive en el índice de la tabla, no en la capa ORM — SQLite la aplicaría igual.
- Recrear `table_sessions`/`carts` en un `MetaData` aparte sin copiar esos dos índices: más código y
  más riesgo de que el esquema de test diverja de los modelos reales en algo más que el índice
  parcial. Se prefiere remover puntualmente los dos índices conocidos sobre el metadata real.

## 4. Alcance de tablas nuevas para `cart_fixtures.new_session()`

**Decisión**: además de las 10 tablas ya cubiertas por `fixtures.py`
(`categories, products, product_variants, option_groups, options, variant_option_groups,
recipe_items, inventory_items, inventory_movements, unit_measures`), `cart_fixtures.new_session()`
crea: `dining_tables, table_sessions, session_participants, carts, cart_items, cart_item_options,
customer_orders, order_items, order_item_options, promotions, promotion_targets,
promotion_combo_items, order_cancel_logs, audit_logs`.

**Rationale**: es el cierre transitivo real de lo que las 11 funciones de `service.py` y sus tres
dependencias externas (motor de catálogo, promociones, checkout) tocan — verificado leyendo
`service.py`, `promotions/service.py` (`expand_combo` necesita `promotion_combo_items`;
`active_discount_promotions`/`best_line_discount` necesitan `promotions`/`promotion_targets`) y
`orders/checkout.py` (`cancel_order` necesita `order_cancel_logs` vía `record_audit` →
`audit_logs`). `audit_logs.payload` es `postgresql.JSONB`
(`app/models/audit_log.py:2,22`): SQLAlchemy compila `JSONB` sobre SQLite como el tipo genérico
`JSON` (no falla `create_all`; se verifica en Fase 1/implementación con un test de humo del propio
fixture antes de escribir los tests de negocio).

**Alternatives considered**: limitar el fixture a solo lo que exige cada escenario de aceptación
literal (menos tablas por test) — se descarta porque duplicaría setup entre los ~40 métodos de test y
porque el propio FR-007 pide fixtures "mínimos pero controlados" reutilizables, no ad-hoc por test.

## 5. A-08: cómo fijar el reloj sin tocar `service.py`

**Decisión**: `cart_fixtures.frozen_now(instant: datetime)` es un context manager que hace
`unittest.mock.patch("app.api.v1.cart.service.datetime", FixedDatetime)`, donde `FixedDatetime` es una
subclase mínima de `datetime.datetime` cuyo único método sobrescrito es `now(tz=None)` (devuelve
`instant` tal cual, ya con o sin tz según lo que el test necesite reproducir). `timedelta` y
`timezone`, importados por separado en `service.py:7`, no se tocan.

**Rationale**: `_now()` (`service.py:52-53`) llama `datetime.now(timezone.utc)` sobre el símbolo
`datetime` importado al nivel de módulo de `cart/service.py` — parchear ese símbolo puntual (no el
`datetime` global del intérprete) es el punto de inyección estándar de `unittest.mock` para código que
no acepta un reloj como parámetro, y no requiere ningún cambio de firma en `service.py`. Como
`_add_combo` (`service.py:320-321`) también llama `datetime.now(timezone.utc)` directamente (no vía
`_now()`), el mismo parche cubre ambos puntos de lectura del reloj dentro del módulo. El test fija un
instante en el que `TENANT_TIMEZONE=America/Bogota` (UTC-5) diverge de UTC, y compara lo que
`promotions.local_now()` interpretaría de ese naive contra la hora Bogotá real, dejando la aserción en
rojo-esperado si algún día alguien "corrige" A-08 sin pasar por el registro de anomalías (Principio
II).

**Alternatives considered**: variar `TZ` del proceso de test en vez de fijar el reloj — no sirve
porque `_now()` construye el valor a partir de `datetime.now(timezone.utc)` (explícitamente UTC, no
depende de `TZ` del SO); lo que hay que fijar es el instante, no la zona horaria del proceso.

## 6. CI: por qué el paso actual no corre ni siquiera los characterization tests que ya existen

**Decisión**: cambiar el paso `Install deps` de `.github/workflows/deploy.yml` de
`pip install sqlalchemy pydantic pydantic-settings fastapi` a `pip install -r requirements.txt`, y
sumar (no reemplazar) un paso `python -m unittest discover -s app/characterization_tests
-p 'test_*.py' -v` después del paso existente `Reglas de promociones`.

**Rationale**: el import chain de cualquier fichero de `app/characterization_tests/` pasa por
`app.core.models`/`app.models`, que arrastran `app/core/db.py` (requiere `alembic`, `psycopg`,
`bcrypt`) y, para lo que esta spec añade, `app/core/qr_token.py` (requiere `PyJWT`) y
`app/core/rate_limit.py`/`app/core/events.py` (requieren `redis`) — ninguno de esos 5 paquetes está
en la lista de 4 que el workflow instala hoy. Es la causa raíz técnica de A-27: no es que alguien haya
olvidado añadir un paso, es que el paso que existe no podría siquiera importar los ficheros de test ya
presentes en el repo. `requirements.txt` ya está congelado en el repo (mismas versiones que usa
producción), así que instalarlo completo no introduce ninguna dependencia nueva ni ningún riesgo de
versión.

**Alternatives considered**: listar a mano solo los paquetes adicionales que esta spec necesita
(`alembic psycopg[binary] bcrypt PyJWT redis`) en vez de `requirements.txt` completo — más frágil
(hay que mantenerlo sincronizado a mano cada vez que cambie una dependencia transitiva) y no arregla
A-27 para el resto de `app/characterization_tests/` que hoy tampoco corre en CI, que es exactamente lo
que FR-009/SC-004 piden que deje de pasar. Se prefiere instalar el fichero congelado completo.
