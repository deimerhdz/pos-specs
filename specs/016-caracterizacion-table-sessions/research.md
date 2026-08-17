# Research: Red de characterization tests para `table_sessions`

Todos los "NEEDS CLARIFICATION" del Technical Context se resuelven aquí antes de Fase 1. Ninguno
requiere clarificación de negocio adicional (`speckit-clarify`): la única pregunta de negocio de
esta spec ya se resolvió en `spec.md` (Clarifications, sesión 2026-08-17, sobre `bill_changed`);
lo que sigue son decisiones de implementación explícitamente delegadas a esta fase por las
Assumptions de `spec.md`.

## 1. Mecanismo para congelar los 7 endpoints de `router.py`

**Decisión**: invocar las funciones de endpoint de `router.py` directamente como funciones Python
(no vía `fastapi.testclient.TestClient` ni un servidor ASGI real), pasando la `Session` de SQLite
de `table_sessions_fixtures.new_session()` como `db`, y dobles mínimos `SimpleNamespace` para
`Tenant` (`id`, `invoice_prefix`) y `User` (`id`, `name`) donde el endpoint los recibe por
`Depends(get_tenant)`/`Depends(get_current_user)`.

**Rationale**: a diferencia de `cart/router.py`, ningún endpoint de `table_sessions/router.py`
abre su propio contexto (no hay equivalente a `open_qr_context`/`open_session_context`): los tres
`Depends` que usa (`get_db`, `get_tenant`, `get_current_user`) son objetos que el endpoint recibe
ya resueltos y de los que solo lee un puñado de atributos (`tenant.id`, `tenant.invoice_prefix`,
y en los cinco endpoints que reciben `User` casi ninguno lo usa — se bindea a `_` en 5 de los 7,
y en `close_session` se pasa tal cual como `cashier` a `service.close_session`, que solo lee
`.id`/`.name` — ver `service.py:_close_unified`/`_close_split` vía `build_sale`). Ni `Tenant` ni
`User` reales (`app.core.models`, schema `shared`) hace falta persistirlos en el SQLite de test:
basta un objeto con esos atributos, exactamente como ya decidió `cart`
(`specs/015-caracterizacion-cart/research.md` §1, "Tenant de prueba"). `Depends(...)` nunca se
resuelve al llamar la función de Python directamente (ignorado por completo), así que no hace
falta ningún `unittest.mock.patch` para los 7 endpoints — más simple que el harness de `cart`, que
sí necesitaba parchear `open_qr_context`/`open_session_context` para sus 2 endpoints que abrían
contexto propio.

Un detalle de firma a respetar: `list_sessions(only_active: bool = Query(True, ...))` tiene como
valor por defecto un objeto `fastapi.params.Query`, no `True` — resuelto solo cuando FastAPI
atiende una request real vía ASGI. El harness de test **siempre** pasa `only_active=` explícito
(nunca confía en el default) para no invocar la función con ese objeto `Query` sin resolver.

**Alternatives considered**:
- `fastapi.testclient.TestClient` sobre la app real: mismo argumento que descartó `cart`
  (`015-caracterizacion-cart/research.md` §1) — exigiría `app.dependency_overrides` para el árbol
  completo de dependencias de la aplicación y levantar la app entera para invocar 7 rutas, sin
  ganancia sobre lo que se quiere congelar (el comportamiento de la función de endpoint, no el
  routing HTTP de FastAPI en sí). Se descarta por complejidad injustificada.
- Reutilizar `cart_fixtures.build_session_context`/`SessionContext`: no aplica, `table_sessions`
  no tiene ni usa ese dataclass — es exclusivo del flujo de comensal anónimo por QR.

## 2. A-17 (R12): cómo congelar la ausencia de `FOR UPDATE` cuando SQLite no la puede reproducir

**Decisión**: en vez de intentar observar un bloqueo real, el test espía `service._load` con
`unittest.mock.patch("app.api.v1.table_sessions.service._load", wraps=service._load)` y, tras
invocar cada función pública, inspecciona `spy.call_args` (o la lista de llamadas) para congelar
con qué valor de `lock=` fue invocada: `close_session` con `lock=True`, y `add_participant`/
`remove_participant`/`set_assignments` con `lock=False` (el default, ni siquiera pasado
explícito hoy). El caso de "concurrencia simulada" de la Historia 2 (escenario 2 de `spec.md`)
se resuelve sembrando el estado directamente (una sesión con datos que una operación concurrente
habría dejado a medio comitear) e invocando las tres funciones sin lock sobre ese estado, sin
necesitar hilos ni procesos reales.

**Rationale**: se verificó en este entorno (SQLAlchemy 2.0.50, motor SQLite en memoria) que
`select(...).with_for_update()` compila, sobre el dialecto SQLite, a exactamente la misma
sentencia SQL que sin la cláusula — el dialecto de SQLite no emite `FOR UPDATE` (no lo soporta) y
SQLAlchemy no levanta error, simplemente lo omite en silencio. Eso significa que, en el motor de
test que exige FR-001 (SQLite en memoria, cero infraestructura nueva), **no existe ninguna
diferencia observable en el resultado de la consulta** entre `lock=True` y `lock=False`: cualquier
intento de "demostrar" A-17 (R12) ejecutando de verdad las cinco funciones y comparando su
resultado terminaría demostrando que las cinco se comportan igual, lo cual sería **falso** respecto
al comportamiento real contra Postgres (donde `FOR UPDATE` sí bloquea). Espiar la llamada a `_load`
y congelar su argumento es lo único que captura el contrato real (qué función pide el lock y cuál
no) sin depender de un motor que sí lo implemente — coherente con la Edge Case de `spec.md` que ya
autoriza sembrar el estado o invocar en secuencia "sin necesitar concurrencia real ni no
determinismo", y con el precedente de A-17 (R16) en `cart`
(`015-caracterizacion-cart/research.md` §3, sobre el mismo tipo de limitación de SQLite frente a
índices parciales de Postgres).

**Alternatives considered**:
- Levantar Postgres real (contenedor) en CI para observar el bloqueo de verdad con dos conexiones
  concurrentes: contradice FR-010 (determinismo, cero infraestructura externa) y el precedente de
  toda la red existente. Se descarta.
- Inspeccionar el SQL compilado de cada llamada real a la base de datos (p. ej. con un event
  listener de SQLAlchemy) buscando la subcadena `FOR UPDATE`: no serviría porque, como se verificó
  arriba, el compilador de SQLite nunca la emite — ni siquiera cuando `lock=True` sí se pidió. Se
  descarta por no ser una señal válida en este motor.
- No cubrir A-17 (R12) en absoluto, dejándolo solo documentado en prosa: viola FR-006/SC-002, que
  exigen al menos un caso de test citándolo explícitamente. Se descarta.

## 3. Redis: `bill_changed` (FR-010a) vs. los otros tres eventos de `close_session`

**Decisión**: para `add_participant`/`remove_participant`/`set_assignments`, interceptar
`app.core.events.bill_changed` con `unittest.mock.patch("app.core.events.bill_changed",
side_effect=_spy)` y, dentro de `_spy`, consultar el propio `db` de test para confirmar que la
sesión ya ve el estado posterior al commit (mismo patrón exacto que usó `cart` para
`order_created`, `015-caracterizacion-cart/test_cart_router.py:241`). Para los otros tres eventos
que dispara `close_session` cuando `tenant_id is not None` (`payment_completed`, `session_closed`,
`table_status_changed`), no se provee ningún doble: se confía en que `app.core.events.publish`
(`app/core/events.py:106-135`) ya es *fail-open* — atrapa cualquier excepción, incluida la que
lanza un intento de conexión a un Redis inexistente en el entorno de CI/test, y no vuelve a
lanzarla — exactamente el mismo argumento que ya aceptó `cart`
(`015-caracterizacion-cart/research.md` §2) para el resto de sus eventos fuera del único que sí
necesitaba doblar.

**Rationale**: FR-010a exige explícitamente, y solo, que se congele el contrato de
`bill_changed` (invocado una vez, con `tenant_id`/`table_session_id` correctos, después del
commit) — es la única aserción de negocio sobre un evento que pide esta spec. Los otros tres no
tienen ningún Acceptance Scenario ni Functional Requirement que dependa de su payload; exigirles
un doble solo para no tocar Redis sería sobre-ingeniería frente al patrón fail-open que el propio
código de producción ya implementa y que `cart` ya usó como precedente aceptado. `REALTIME_ENABLED`
por defecto es `True` (`app/core/config.py:74`) y no se cambia en los tests: si el entorno de CI/
desarrollador no tiene Redis escuchando en `REDIS_URL`, el intento de `XADD` falla rápido
(conexión rechazada, no timeout) y `publish()` lo atrapa; si por el contrario hay un Redis real
disponible (p. ej. un desarrollador con su entorno completo levantado), los tres eventos se
publican de verdad — comportamiento aceptado, no una falla de test, igual que ya lo aceptó `cart`.

**Alternatives considered**:
- Doblar los tres eventos igual que `bill_changed`, por simetría: más código sin ningún requisito
  que lo pida: ninguna aserción de esta spec depende de su contenido. Se descarta por
  desproporcionado.
- Poner `settings.REALTIME_ENABLED = False` globalmente en `table_sessions_fixtures.new_session()`
  para que **ningún** evento (ni siquiera `bill_changed`) intente publicar: rompería FR-010a, que
  exige observar que `bill_changed` sí se invoca (con `mock.patch` como espía, no que esté
  deshabilitado). Se descarta.

## 4. `JSONB` sobre SQLite: `sale_items.options`

**Decisión**: registrar en `table_sessions_fixtures.py` el mismo compilador de SQLAlchemy que ya
usa `cart_fixtures.py` para `audit_logs.payload`:
`@compiles(JSONB, "sqlite")` → `"JSON"`, aplicado antes de `create_all()`.

**Rationale**: `SaleItem.options` (`app/models/sale.py`) es `postgresql.JSONB`, la misma situación
exacta que ya resolvió `cart_fixtures.py` para `audit_logs.payload`
(`015-caracterizacion-cart/research.md` §4, nota de fixture: la versión de SQLAlchemy de este
entorno no compila `JSONB` a un tipo genérico sobre SQLite por sí sola, y sin este shim
`create_all()` falla con `UnsupportedCompilationError` antes de crear una sola tabla). Se registra
de forma independiente en `table_sessions_fixtures.py` (no se importa desde `cart_fixtures.py`,
ver plan.md §Structure Decision) para que este fichero sea autónomo si se ejecuta solo; el
registro de `@compiles` es una directiva global de SQLAlchemy, así que registrarla dos veces con
la misma implementación (una por cada fixture module, si ambos se importan en el mismo proceso de
test) es idempotente y no produce ningún conflicto.

**Alternatives considered**: ninguna — es la misma decisión ya validada por `cart`, sin motivo
para reabrirla.

## 5. Por qué esta spec NO necesita un reloj fijado (`frozen_now`), a diferencia de `cart`

**Decisión**: los fixtures de promoción/combo que necesiten los tests de A-01 y A-29
(`table_sessions_fixtures.make_promotion`) se siembran **sin** ventana horaria (`start_time`/
`end_time`/fechas de vigencia en `None`, siempre válidas) — nunca se congela `datetime.now()`
dentro de `table_sessions/service.py`.

**Rationale**: `compute_bill`, `_close_unified` y `_close_split` sí leen el reloj real
(`datetime.now(timezone.utc)`, `service.py:164,...`) para pasarlo a
`promotions.evaluate`/`promotions.combo_discount_for_lines`, pero esa lectura solo importa para el
resultado si la promoción/combo sembrada tiene una ventana horaria o de fechas que ese instante
concreto puede o no cubrir. A diferencia de A-08 en `cart` (una anomalía de zona horaria que
**exige** fijar un instante exacto para observarla), ninguna de las cinco anomalías que esta spec
cubre depende del reloj — A-29 es sobre `promotion_id` perdiéndose con combos múltiples,
independiente de cuándo corre el test. Sembrar promociones/combos sin restricción horaria hace que
el resultado sea el mismo sin importar el instante real en que corra la suite, cumpliendo FR-010
sin necesitar ningún parche de `datetime`.

**Alternatives considered**:
- Replicar `cart_fixtures.frozen_now` de todas formas, "por si acaso": código muerto sin ningún
  test que lo necesite — viola la guía del proyecto de no añadir infraestructura sin un requisito
  que la use. Se descarta.

## 6. Alcance de tablas nuevas para `table_sessions_fixtures.new_session()`

**Decisión**: además de las 10 tablas ya cubiertas por `fixtures.py`, `table_sessions_fixtures.
new_session()` crea: `dining_tables, table_sessions, session_participants, customer_orders,
order_items, order_item_options, carts, promotions, promotion_targets, promotion_combo_items,
cash_registers, cash_shifts, payment_methods, payments, sales, sale_items, invoices,
invoice_counters`.

**Rationale**: es el cierre transitivo real de lo que las 9 funciones de `service.py` y sus
dependencias externas (`orders.checkout`, `promotions.service`, `sales.builder`,
`invoices.service`) tocan — verificado leyendo los cuatro módulos:
- `checkout.close_table_sessions` → `close_participants` consulta `Cart` (`WHERE participant_id=…
  AND status='abierto'`) para abandonarlos: se necesita la tabla `carts`, aunque ningún test de
  esta spec siembre filas ahí a propósito (la consulta debe poder ejecutarse sobre una tabla
  vacía). `cart_items` **no** se necesita: nada en el camino de `table_sessions` la consulta.
- `sales.builder.build_sale` → `Sale`, `SaleItem`, `Payment`, `PaymentMethod`, `CashShift` (vía
  `ensure_open_shift`, que a su vez exige `CashRegister` por la FK de `CashShift`), y
  `invoices.service.issue_for_sale` → `Invoice`, `InvoiceCounter`. Esta es la diferencia principal
  frente al fixture de `cart` (que nunca llega al cobro): `table_sessions.close_session` sí cierra
  el ciclo completo de venta y factura.
- `promotions.service.evaluate`/`combo_discount_for_lines` → `Promotion`, `PromotionTarget`,
  `PromotionComboItem`, igual que ya resolvió `cart_fixtures.py` para el mismo módulo.

A diferencia de `cart`, esta spec **no** necesita remover ningún índice único parcial del
metadata de test: ni `idx_active_session_per_table` (ningún test de esta spec siembra dos
`TableSession` `active` para la misma mesa) ni `idx_open_shift_per_register` (ningún test necesita
dos `CashShift` `open` en la misma caja) — cada escenario de `spec.md` opera sobre una sola sesión
y un solo turno de caja a la vez.

**Alternatives considered**: limitar el fixture a solo lo que exige cada escenario de aceptación
literal — se descarta por la misma razón que ya documentó `cart`
(`015-caracterizacion-cart/research.md` §4): duplicaría setup entre los ~35 métodos de test, y
FR-009 pide fixtures "mínimos pero controlados" y reutilizables, no ad-hoc por test.
