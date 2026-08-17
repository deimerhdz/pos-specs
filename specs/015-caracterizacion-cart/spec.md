# Feature Specification: Red de characterization tests para `cart` (`router.py` + `service.py`)

**Feature Branch**: `015-caracterizacion-cart`

**Created**: 2026-08-17

**Status**: Draft

**Input**: User description: "Construcción de la red de characterization tests para cart
(app/api/v1/cart/router.py + service.py) de pos-backend, como prerrequisito de una futura
extracción strangler fig."

**Naturaleza de esta spec**: **characterization, no extracción** (Principio II de la
[Constitución](../../.specify/memory/constitution.md)). No mueve ni reescribe una sola línea de
`cart/router.py` ni de `cart/service.py`: construye la red de tests que **congela** su
comportamiento observable actual — anomalías incluidas — como el prerrequisito que el Principio
III (Estrangulamiento antes que Reescritura) exige antes de que `cart` pueda entrar en una spec de
extracción, igual que `specs/014-extraccion-motor-catalogo/` exigió primero los 41
characterization tests + golden master del motor de catálogo antes de moverlo.

## Contexto — qué existe hoy y qué protege esta spec

`app/api/v1/cart/` tiene dos ficheros de producción:

- **`service.py`** (541 líneas): 11 funciones públicas — `unique_display_label`, `open_session`,
  `serialize_cart`, `get_cart`, `add_item`, `update_item`, `list_my_orders`, `cancel_my_order`,
  `leave_session`, `submit_cart`, `remove_item` — más 6 funciones privadas (`_now`,
  `_get_or_create_table_session`, `_load_open_cart`, `_get_or_create_open_cart`,
  `_cart_consumption`, `_add_combo`) que solo se ejercitan a través de las públicas.
  `unique_display_label` además la reimporta `app/api/v1/table_sessions/service.py:317-319`, así
  que es superficie pública real, no solo del módulo `cart`.
- **`router.py`** (193 líneas): 9 endpoints — `POST /cart/sessions`, `GET /cart`,
  `POST /cart/items`, `PATCH /cart/items/{item_id}`, `DELETE /cart/items/{item_id}`,
  `POST /cart/leave`, `POST /cart/submit`, `GET /cart/orders`,
  `POST /cart/orders/{order_id}/cancel`.

`cart` no es una isla: consume tres motores externos, ninguno de los cuales se toca ni se
re-caracteriza en esta spec — solo se ejercitan en su comportamiento actual, tal como ya lo
consume `cart` hoy:

1. **Motor de catálogo** (`app.api.v1.catalog.line_pricing`, hoy fachada de
   `app.catalog_engine` tras `specs/014-extraccion-motor-catalogo/`): `compute_line_price`,
   `load_valid_options`, `check_availability` —de las siete módulos consumidores del motor,
   `cart` es el único que además usa `required_consumption` vía el reexport de
   `line_pricing.py:31-36`—. Ya tiene su propia red (`test_catalog_line_pricing.py`,
   `test_catalog_consumption_plan.py`, golden master); esta spec no la duplica.
2. **Promociones** (`app.api.v1.promotions.service`): `active_discount_promotions` y
   `best_line_discount` (en `serialize_cart`, para `discounted_total`) y `expand_combo` (en
   `_add_combo`). Solo tiene hoy `app/scripts/test_promotions_rules.py` (legado, sí corre en CI,
   pero no sigue la convención `characterization_tests`).
3. **Checkout de pedidos** (`app.api.v1.orders.checkout.cancel_order`, en `cancel_my_order`): sin
   red de characterization propia todavía.

Dos anomalías del registro caen, en parte o por completo, dentro de `cart/service.py` y esta spec
las cubre explícitamente:

- **A-08**: `_now()` (`service.py:52-53`) usa `datetime.now(timezone.utc).replace(tzinfo=None)`
  — un naive que en realidad es UTC — y `serialize_cart` (`service.py:205-206`) lo pasa a
  `promotions.active_discount_promotions`, cuya función `local_now()` asume que un naive **ya
  está** en hora local. Con `TENANT_TIMEZONE=America/Bogota` (UTC-5) esto reproduce, en `cart`, el
  mismo bug que A-07 corrigió en el resto del sistema. `open_session` también usa `_now()` para
  calcular `expires_at`.
- **A-17, parcial (R16)**: `_get_or_create_table_session` (`service.py:58-74`) lee-y-decide sin
  `SELECT...FOR UPDATE`, a diferencia del resto del código de mesas; `open_session`
  (`service.py:93-123`) envuelve la creación en un `except Exception` genérico. Dos comensales
  escaneando el QR de la misma mesa a la vez pueden violar la invariante de "una sola sesión
  `active` por mesa" y recibir un 500 genérico en vez de un error controlado. Esta spec cubre solo
  esta porción de A-17 (R16); las demás piezas de A-17 (caja, reparto de mesa) viven en otros
  módulos y quedan fuera.

Y el motivo por el que esta red debe correr en CI, no ser otro script más: **A-27** documenta que
el pipeline de `pos-backend` (`.github/workflows/deploy.yml`) ejecuta hoy un único script,
`test_promotions_rules.py`, de los ~12 `app/scripts/test_*.py` existentes — los otros 11 no
corren nunca de forma automatizada. Los tests de esta spec, para contar como el árbitro que exige
el Principio II, no pueden repetir ese patrón: tienen que ejecutarse en CI en cada push, con el
mismo mecanismo (`python -m unittest discover -s app/characterization_tests`) que ya usa el resto
de `app/characterization_tests/`.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Congelar las 11 funciones públicas de `cart/service.py` (Priority: P1)

Como responsable de la modernización, escribo characterization tests que ejercitan cada una de
las 11 funciones públicas de `cart/service.py` contra los modelos SQLAlchemy reales sobre SQLite
en memoria (mismo mecanismo que el resto de `app/characterization_tests/`), documentando el
comportamiento que observo hoy — incluyendo A-08 y A-17 (R16) — sin corregir nada.

**Why this priority**: es el núcleo de la spec. Sin esta red, no existe línea base contra la cual
verificar que una futura extracción de `cart` no cambió nada; las historias 2 y 3 dependen de que
esta exista primero.

**Independent Test**: se puede verificar de forma aislada corriendo
`python -m unittest app.characterization_tests.test_cart_service -v` (o el nombre de fichero que
se use) sin que existan aún los tests de router ni el wiring de CI.

**Acceptance Scenarios**:

1. **Given** un carrito vacío recién creado para un comensal, **When** se llama a `add_item` con
   una variante activa y opciones válidas, **Then** el precio de línea, las opciones guardadas y
   el `CartResponse` devuelto coinciden con lo que hoy produce el código (delegando el cálculo del
   motor de catálogo real, no un mock).
2. **Given** dos `TableSession` con `status='active'` sembradas directamente para la misma mesa
   (estado que la invariante de aplicación normalmente impide, pero que puede darse ante una
   condición de carrera real — A-17/R16), **When** se llama a `open_session` para esa mesa,
   **Then** el test documenta que hoy se propaga una excepción no controlada (no un 4xx
   controlado) — el mismo defecto que describe A-17 (R16), sin corregirlo.
3. **Given** un reloj del sistema fijado/inyectado a un instante conocido en el que
   `TENANT_TIMEZONE=America/Bogota` difiere de UTC, **When** se llama a `open_session` o
   `serialize_cart` (ambas usan `_now()`), **Then** el test observa el desfase de A-08 (el naive
   que `_now()` produce se trata como si ya fuera hora local) de forma determinista, sin depender
   del reloj real de la máquina que ejecuta el test.
4. **Given** un carrito con una promoción de descuento activa y otro sin ninguna, **When** se
   llama a `serialize_cart` en ambos casos, **Then** `discounted_total` refleja exactamente lo que
   hoy calcula el código (incluida la regla de no aplicar descuento percent/fixed a líneas de
   combo, que ya llevan su propio ahorro).
5. **Given** un pedido del comensal en estado `recibida` (sin ítems en cocina), **When** se llama
   a `cancel_my_order`, **Then** se cancela; **Given** el mismo pedido ya en `en_preparacion`,
   **When** se llama a `cancel_my_order`, **Then** responde 409 — congelando la política descrita
   en el docstring de la función tal cual existe hoy.

---

### User Story 2 - Congelar los 9 endpoints de `cart/router.py` (Priority: P2)

Como responsable de la modernización, escribo characterization tests que ejercitan cada uno de los
9 endpoints de `cart/router.py`, congelando el comportamiento observable que añade la capa de
router sobre `service.py` ya congelado en la Historia 1: códigos de estado, forma de la respuesta,
mapeo de errores, y los efectos de borde que solo viven en el router (rate limiting, emisión de
eventos después del commit, caché `ETag`/304, y el contrato "nunca falla con 401" de
`POST /cart/leave`).

**Why this priority**: depende de que la Historia 1 exista primero (el router delega toda la
lógica de negocio en `service.py`); es de menor prioridad porque, aislada, protege una capa más
delgada — pero sigue siendo parte del alcance que el encargo pide explícitamente ("los endpoints
de cart/router.py").

**Independent Test**: se puede verificar de forma aislada corriendo
`python -m unittest app.characterization_tests.test_cart_router -v`, reutilizando los mismos
fixtures de la Historia 1 para construir el contexto de sesión que cada endpoint necesita.

**Acceptance Scenarios**:

1. **Given** un token de QR firmado válido para una mesa activa, **When** se invoca el endpoint de
   `POST /cart/sessions`, **Then** responde 201 con `SessionOpenResponse` y un `session_token`
   utilizable; **Given** la mesa no existe o está inactiva, **When** se invoca el mismo endpoint,
   **Then** responde 404.
2. **Given** ningún `x-session-token` en la petición, **When** se invoca `POST /cart/leave`,
   **Then** responde 204 sin lanzar 401 — el contrato "nunca falla" descrito en el docstring del
   endpoint, congelado tal cual.
3. **Given** un carrito con ítems, **When** se invoca `POST /cart/submit` con éxito, **Then** el
   evento `order_created` se publica después de que la transacción del `service` ya hizo commit
   (no antes), congelando el orden actual.
4. **Given** una segunda petición a `GET /cart/orders` sin cambios desde la anterior, **When**
   se envía el mismo `ETag` recibido, **Then** responde 304, congelando el mecanismo de caché
   documentado en el endpoint.
5. **Given** el límite de tasa configurado para el bucket `cart_items`, **When** se supera desde
   la misma mesa, **Then** `POST /cart/items` responde 429 con `Retry-After` — congelando que el
   límite se aplica antes de tocar `service.add_item`.

---

### User Story 3 - La suite corre en CI de forma determinista (Priority: P3)

Como responsable de la modernización, incorporo los ficheros de test de las historias 1 y 2 al
job de test del pipeline de CI del backend (`.github/workflows/deploy.yml` o el que lo sustituya),
de modo que se ejecuten automáticamente en cada push — a diferencia del patrón que A-27 documenta
para los scripts legado — y verifico que la suite completa pasa de forma idéntica en ejecuciones
repetidas, sin flakiness.

**Why this priority**: sin esto, la red existe pero no es el árbitro que el Principio II exige —
sería solo otro script más que alguien tiene que acordarse de correr a mano, exactamente el
patrón que A-27 ya identificó como riesgo real (el precedente del spec roto de `ReportsService`
en el frontend). Es P3 porque depende de que las historias 1 y 2 ya existan para tener algo que
ejecutar.

**Independent Test**: se puede verificar abriendo un pull request trivial contra el branch de esta
spec y confirmando en la ejecución de CI que el paso nuevo corre y pasa, sin depender de que
ninguna otra spec futura exista todavía.

**Acceptance Scenarios**:

1. **Given** el workflow de CI del backend actualizado, **When** se inspecciona su definición,
   **Then** incluye un paso que ejecuta
   `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` (o equivalente que
   cubra al menos los ficheros nuevos de las historias 1 y 2), sin reemplazar el paso existente de
   `test_promotions_rules`.
2. **Given** la suite de esta spec ejecutada tres veces seguidas sin ningún cambio de código,
   **When** se comparan los tres resultados, **Then** los tres son idénticos (mismos tests en
   verde) — ninguno depende del reloj real, de Redis, de Postgres real, ni de orden de ejecución.
3. **Given** un push que modifica `cart/service.py` o `cart/router.py` de forma que cambia su
   comportamiento observable, **When** corre CI, **Then** al menos un test de esta suite falla en
   rojo, demostrando que la red efectivamente detecta el cambio (no es un test que siempre pasa
   sin importar el código).

### Edge Cases

- ¿Qué pasa si, al escribir un test de esta suite, falla contra el código actual sin modificar?
  → El defecto está en el test (mal capturó lo que el código realmente hace), nunca en
  `cart/service.py` ni `cart/router.py`: se corrige el test hasta que refleje la realidad
  observada. Esta spec no autoriza ningún cambio en los dos ficheros de producción.
- ¿Qué pasa si aparece, mientras se escriben estos tests, una anomalía no listada hoy en
  `registro-de-anomalias.md`? → Se documenta ahí (Principio I) y se congela tal cual en el test;
  no se corrige como parte de esta spec.
- ¿Qué pasa con las reglas propias de promociones (qué promoción aplica, cómo se calcula un
  descuento) o de checkout (qué revierte `cancel_order`)? → Fuera de alcance: esta spec ejercita
  esas funciones reales con fixtures mínimos controlados (por ejemplo, "sin promoción activa" como
  caso por defecto, más un caso simple con una promoción activa) para congelar cómo `cart` las
  usa, no para caracterizar exhaustivamente sus propias reglas — eso pertenece a una spec futura
  de esos módulos.
- ¿Qué pasa con la dependencia de Redis del rate limiting (`app/core/rate_limit.py`) en los tests
  de router? → Se controla vía `settings.RATE_LIMIT_ENABLED` (ya existente, sin dependencia
  nueva) para los casos que no ejercitan el límite en sí, y se activa deliberadamente solo en el
  caso que sí lo congela (Historia 2, escenario 5).
- ¿Qué pasa si un test de esta suite necesita concurrencia real (dos hilos/procesos a la vez) para
  observar A-17 (R16)? → No se requiere: el escenario se reproduce sembrando directamente el
  estado de datos que la condición de carrera produciría (dos `TableSession` `active` para la
  misma mesa) y observando cómo reacciona el código ante ese estado, sin necesitar concurrencia
  real ni no determinismo.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE incluir, bajo `app/characterization_tests/`, uno o más ficheros de
  test (prefijo `test_cart_`) que sigan la convención documentada en
  `app/characterization_tests/__init__.py:1-39` (docstring "CONGELA comportamiento actual:",
  Principio II) y usen exclusivamente `unittest` de la biblioteca estándar sobre los modelos
  SQLAlchemy reales del proyecto vía SQLite en memoria — cero dependencias nuevas (Principio IV).
- **FR-002**: La suite DEBE incluir al menos un characterization test por cada una de las 11
  funciones públicas de `cart/service.py`: `unique_display_label`, `open_session`,
  `serialize_cart`, `get_cart`, `add_item`, `update_item`, `list_my_orders`, `cancel_my_order`,
  `leave_session`, `submit_cart`, `remove_item`.
- **FR-003**: La suite DEBE incluir al menos un characterization test por cada uno de los 9
  endpoints de `cart/router.py`: `POST /cart/sessions`, `GET /cart`, `POST /cart/items`,
  `PATCH /cart/items/{item_id}`, `DELETE /cart/items/{item_id}`, `POST /cart/leave`,
  `POST /cart/submit`, `GET /cart/orders`, `POST /cart/orders/{order_id}/cancel`.
- **FR-004**: La suite DEBE incluir al menos un caso que congele el comportamiento de A-08 (el
  naive UTC que produce `_now()` tratado como hora local por `promotions.active_discount_promotions`
  al calcularse desde `open_session` y/o `serialize_cart`), usando un reloj fijado o inyectado
  para que el resultado sea determinista — sin corregir el desfase que documenta.
- **FR-005**: La suite DEBE incluir al menos un caso que congele el comportamiento de A-17 (R16):
  ante dos `TableSession` en estado `active` sembradas para la misma mesa, `open_session` (o
  `_get_or_create_table_session` directamente) propaga hoy una excepción no controlada en vez de
  un error HTTP explícito — reproducido sin requerir concurrencia real.
- **FR-006**: La suite DEBE ejercitar el motor de catálogo (`compute_line_price`,
  `load_valid_options`, `check_availability`, `required_consumption`) a través de las funciones
  reales importadas por `cart/service.py` (vía la fachada `line_pricing.py`), sin mocks y sin
  re-derivar sus propias reglas de negocio — esas ya están cubiertas por
  `test_catalog_line_pricing.py`, `test_catalog_consumption_plan.py` y el golden master de
  `specs/014-extraccion-motor-catalogo/`.
- **FR-007**: La suite DEBE ejercitar `promotions.active_discount_promotions`,
  `promotions.best_line_discount`, `promotions.expand_combo` y
  `orders.checkout.cancel_order` como dependencias reales sobre fixtures mínimos y controlados
  (por ejemplo, "sin promoción activa" como caso por defecto), sin asumir la responsabilidad de
  caracterizar exhaustivamente las reglas propias de esos módulos.
- **FR-008**: Cada test DEBE ser determinista: ninguno puede depender del reloj real de la
  máquina que lo ejecuta, de una conexión Redis o Postgres real, ni del orden de ejecución de
  otros tests.
- **FR-009**: La suite DEBE ejecutarse automáticamente como parte del pipeline de CI del backend
  (`.github/workflows/deploy.yml` o el que lo sustituya) en cada push, sumándose al paso
  existente de `test_promotions_rules` sin reemplazarlo — a diferencia del patrón que A-27
  documenta para los 11 scripts `app/scripts/test_*.py` que no corren en CI.
- **FR-010**: Ningún test de esta suite DEBE requerir una modificación de
  `app/api/v1/cart/service.py` ni de `app/api/v1/cart/router.py` para pasar: si un test recién
  escrito falla contra el código actual sin modificar, el defecto está en el test.
- **FR-011**: El sistema NO DEBE corregir, mitigar ni alterar A-08, A-17 (R16), ni ninguna otra
  anomalía de `cart` como parte de esta spec — cada test debe documentar el comportamiento
  observado, no el comportamiento deseado.
- **FR-012**: El sistema NO DEBE extraer, mover, ni reescribir código de `cart/service.py` ni
  `cart/router.py`, ni cambiar la interfaz (firmas de funciones, rutas de endpoints, esquemas de
  request/response) de ninguno de los dos ficheros, como parte de esta spec.

### Key Entities *(include if feature involves data)*

- **Función pública de `cart/service.py`**: una de las 11 funciones sin prefijo `_` — la unidad
  mínima de cobertura exigida por FR-002.
- **Endpoint de `cart/router.py`**: una de las 9 rutas HTTP expuestas — la unidad mínima de
  cobertura exigida por FR-003.
- **Caso de anomalía citado (A-08 / A-17 R16)**: un characterization test cuyo propósito
  explícito es documentar el comportamiento de una anomalía ya registrada, con su cita
  correspondiente en el docstring del test.
- **Dependencia externa fijada**: un módulo que `cart` consume (motor de catálogo, promociones,
  checkout) cuyo comportamiento se ejercita real y sin mocks, pero no se re-caracteriza en
  profundidad dentro de esta spec.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las 11 funciones públicas de `cart/service.py` y el 100% de los 9
  endpoints de `cart/router.py` tienen al menos un characterization test asociado.
- **SC-002**: A-08 y A-17 (R16) están representados en al menos un caso cada uno, verificable por
  su cita explícita en el docstring o nombre del test correspondiente.
- **SC-003**: La suite completa pasa en verde en tres ejecuciones consecutivas sin ningún cambio
  de código entre ellas, con exactamente el mismo resultado en las tres.
- **SC-004**: La suite se ejecuta automáticamente en el pipeline de CI del backend — verificable
  inspeccionando la definición del workflow — y no solo de forma manual/local.
- **SC-005**: `cart/service.py` y `cart/router.py` tienen cero líneas modificadas (diff vacío) al
  concluir esta spec, comparados contra su estado inmediatamente anterior a ella.
- **SC-006**: Cero dependencias nuevas añadidas a `requirements.txt` como resultado de esta spec.

## Assumptions

- El motor de catálogo ya fue extraído (`specs/014-extraccion-motor-catalogo/` está concluida:
  `app/api/v1/catalog/line_pricing.py` es hoy una fachada de `app.catalog_engine`, verificado
  contra el repositorio al escribir esta spec). Por eso esta spec fija el comportamiento del
  motor a través de la fachada real (mismo camino de import que usa `cart/service.py:31-36` hoy),
  en vez de mockearlo — ya tiene su propia red de regresión y su salida es determinista.
- `promotions.service` y `orders.checkout` no tienen todavía su propia red
  `characterization_tests` (`promotions` solo tiene el script legado
  `test_promotions_rules.py`, que sí corre en CI pero fuera de esta convención; `checkout` no
  tiene ninguna). Esta spec no llena ese vacío: los ejercita como dependencias reales con
  fixtures mínimos, suficientes para congelar cómo `cart` los usa, sin caracterizar sus reglas
  propias — ese trabajo, si se necesita, es una spec futura de esos módulos.
- El mecanismo concreto para congelar los endpoints de `router.py` (invocar las funciones de ruta
  directamente con un `SessionContext`/`Request` construido a mano, vs. `fastapi.testclient`) es
  una decisión de implementación que se resuelve en la fase de planificación, no en esta spec —
  el requisito aquí es el comportamiento observable congelado (FR-003, FR-008), no el mecanismo.
- El rate limiting (`app/core/rate_limit.py`) depende de Redis en producción; los tests lo
  controlan vía el flag ya existente `settings.RATE_LIMIT_ENABLED` en vez de levantar un Redis
  real, salvo en el caso puntual que necesita congelar el límite en sí (Historia 2, escenario 5),
  donde se usa la infraestructura de test que ya exista para eso en el proyecto.
- El caso de A-17 (R16) se reproduce sembrando directamente el estado de datos de la condición de
  carrera (dos `TableSession` `active` para la misma mesa), no ejecutando hilos/procesos
  concurrentes reales — ambos enfoques observan el mismo comportamiento del código, pero el
  primero es determinista y el segundo no.
- El paso de CI de esta spec se agrega a `.github/workflows/deploy.yml` (el único workflow del
  backend que existe hoy); si en el momento de implementar existe otro mecanismo de CI para el
  backend, se usa ese en su lugar, conservando el requisito sustantivo (FR-009): que la suite
  corra automáticamente en cada push, no solo a mano.
