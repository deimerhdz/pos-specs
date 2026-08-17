# Feature Specification: Red de characterization tests para `table_sessions` (`router.py` + `service.py`)

**Feature Branch**: `016-caracterizacion-table-sessions`

**Created**: 2026-08-17

**Status**: Draft

**Input**: User description: "Construcción de la red de characterization tests para table_sessions
(app/api/v1/table_sessions/router.py + service.py) de pos-backend, como prerrequisito de una
futura extracción strangler fig."

**Naturaleza de esta spec**: **characterization, no extracción** (Principio II de la
[Constitución](../../.specify/memory/constitution.md)). No mueve ni reescribe una sola línea de
`table_sessions/router.py` ni de `table_sessions/service.py`: construye la red de tests que
**congela** su comportamiento observable actual — anomalías incluidas — como el prerrequisito que
el Principio III (Estrangulamiento antes que Reescritura) exige antes de que `table_sessions`
pueda entrar en una spec de extracción, igual que `specs/015-caracterizacion-cart/` exigió primero
su propia red de characterization tests antes de que `cart` pudiera moverse.

## Contexto — qué existe hoy y qué protege esta spec

`app/api/v1/table_sessions/` tiene dos ficheros de producción:

- **`service.py`** (671 líneas): 9 funciones públicas — `get_session`, `has_billable_orders`,
  `try_release_if_empty`, `list_sessions`, `compute_bill`, `close_session`, `add_participant`,
  `remove_participant`, `set_assignments` — más funciones privadas (`_load`, `_billable_orders`,
  `_assert_closable`, `_participantes_con_consumo`, `_unique_label`, `_notify_bill_changed`,
  `_nombre_cuenta`, `_close_unified`, `_close_split`) que solo se ejercitan a través de las
  públicas. `_unique_label` reimporta `unique_display_label` de
  `app/api/v1/cart/service.py`, ya congelado por `specs/015-caracterizacion-cart/`.
- **`router.py`** (138 líneas): 7 endpoints — `GET /table-sessions`,
  `GET /table-sessions/{id}`, `POST /table-sessions/{id}/participants`,
  `DELETE /table-sessions/{id}/participants/{participant_id}`,
  `PUT /table-sessions/{id}/assignments`, `GET /table-sessions/{id}/bill`,
  `POST /table-sessions/{id}/close`. La apertura de sesión (nace al escanear el QR) no tiene
  endpoint aquí: vive en `cart` y ya está congelada por la spec 015.

`table_sessions` no es una isla: consume `app.api.v1.orders.checkout` en varios puntos —
`checkout.TERMINAL` (para decidir qué pedidos siguen siendo cobrables en
`has_billable_orders`/`_billable_orders`), `checkout.close_table_sessions` (al liberar la mesa
en `try_release_if_empty` y `close_session`), `checkout.order_sale_lines` y
`checkout.promo_lines_for` (al armar las líneas de cobro en `compute_bill`, `_close_unified` y
`_close_split`). Ninguna de esas funciones de `checkout` se toca ni se re-caracteriza en esta
spec — solo se ejercitan en su comportamiento actual, tal como ya las consume `table_sessions`
hoy; tampoco se reordena esa dependencia (`table_sessions` sigue llamando a `checkout`, nunca al
revés). `promotions.evaluate` y `promotions.combo_discount_for_lines` se consumen igual, con
fixtures mínimos, sin caracterizar sus reglas propias.

`add_participant`, `remove_participant` y `set_assignments` llaman además a
`_notify_bill_changed` (`service.py:501-510`), que publica un evento `bill_changed` vía
`app.core.events` a un Redis Stream real (`XADD` con timeout de socket). A diferencia de
`checkout` y `promotions`, esta dependencia sí se neutraliza en los tests (ver Clarifications):
se intercepta la llamada real a Redis para mantener la suite determinista (FR-010), pero se
congela que se invoca con los argumentos correctos después de que la transacción hizo commit —
mismo patrón que ya usa la suite de `cart` para `order_created`.

Existen hoy tres scripts legado que ejercitan parte de este código sin seguir la convención
formal ni correr en CI (**A-27**): `app/scripts/test_table_sessions.py` (145 líneas),
`app/scripts/test_table_release.py` (217 líneas) y `app/scripts/test_split_blindaje.py` (475
líneas, el más extenso — cubre justamente el blindaje de A-15). Pueden usarse como punto de
partida de casos, pero deben **migrarse** al formato `unittest` de
`app/characterization_tests/` — no incorporarse tal cual ni ejecutarse como script aparte.

Cinco anomalías del registro caen, en su totalidad o en parte, dentro de `table_sessions` y esta
spec las cubre explícitamente:

- **A-01, camino A** (`table_sessions.compute_bill`, `service.py:139-181`): de las tres
  implementaciones que responden "cuánto se le debe cobrar a esta mesa", esta es la única
  descrita como **correcta** en el registro (excluye pedidos `cancelada`/`pagada`, aplica
  promociones y combos por comensal). Esta spec la deja como **caso base de referencia**: el
  comportamiento correcto contra el cual medir cualquier extracción futura, sin abrir preguntas
  nuevas sobre ella.
- **A-15, [PROTEGIDA]** (`service.py:590-632`, dentro de `_close_split`): cuatro huecos de
  seguridad ya reales una vez y cerrados el 2026-08-04 (commit `42b5dec3`) — comensales
  repetidos en el mismo split cobrados dos veces, importes de raíz (`discount`/`tax`/`tip`/
  `payments`) ignorados en silencio cuando `billing_mode='split'`, el bloque sin comensal
  asignado saliendo sin nombre en la venta/factura, y la cobertura exacta comensal-con-consumo
  ↔ comensal-en-el-split (ni falta ni sobra). Es el **invariante de mayor prioridad de toda esta
  spec**: ya se rompió una vez y el registro exige tratarlo como "especificar tal cual, no
  tocar" con el mayor número de casos de test posible.
- **A-17, parcial (R12)** (`service.py:38-55` — `_load` — y sus llamadas sin `lock=True` en
  `add_participant:335`, `remove_participant:370`, `set_assignments:403`): `close_session`
  carga la sesión con `_load(..., lock=True)` (`FOR UPDATE`), pero `add_participant`,
  `remove_participant` y `set_assignments` cargan la misma fila sin bloqueo. Un reparto de
  cuenta concurrente con un cierre de sesión en curso puede leer datos a medio comitear. Esta
  spec cubre solo esta porción de A-17 (R12); las demás piezas (caja, `cart`) viven en otros
  módulos y quedan fuera.
- **A-29, parcial** (`service.py:555-559` dentro de `_close_unified`, y su equivalente dentro de
  `_close_split`): cuando las líneas cobradas usan más de un combo distinto (o ninguno),
  `promotion_id` no registra ninguno — el descuento monetario se suma correctamente, pero se
  pierde la trazabilidad por promoción en reportes. Este es uno de los tres caminos de cobro que
  comparten el mismo mecanismo (los otros dos, `orders/checkout.py` y `sales/service.py`, quedan
  fuera de esta spec).
- **A-38, parcial** (`service.py:578-671` y `:362-388`): de los cinco hallazgos del clúster,
  esta spec cubre los dos que viven en `table_sessions`: **RN-MESA-13** (una mesa de un solo
  comensal puede cerrarse en `billing_mode='split'` sin restricción de mínimo, equivalente en la
  práctica a `unified` disfrazado) y **RN-MESA-24** (`remove_participant` no permite quitar un
  comensal que ya tiene productos asignados, aunque estén anulados o su pedido ya no sea
  cobrable). Los otros tres hallazgos del clúster (`RN-ORD-31`, `RN-ORD-32`, `RN-ORD-34`) viven
  en `orders/checkout.py` y quedan fuera de esta spec.

Y el motivo por el que esta red debe correr en CI, no ser otro script más: la misma razón que
ya documentó `specs/015-caracterizacion-cart/` sobre **A-27** — el pipeline de `pos-backend`
(`.github/workflows/deploy.yml`) solo ejecuta `test_promotions_rules.py` de los ~12 scripts
`app/scripts/test_*.py`. Esta spec suma sus ficheros al mismo paso
(`python -m unittest discover -s app/characterization_tests`) que ya ejecuta la red de `cart` y
el resto de `app/characterization_tests/`, sin repetir el patrón de A-27.

## Clarifications

### Session 2026-08-17

- Q: ¿Cómo debe la suite tratar la publicación de eventos por Redis (`events.bill_changed`,
  disparada por `add_participant`, `remove_participant` y `set_assignments`) para cumplir FR-010
  (nada de Redis real)? → A: Neutralizar la llamada real (interceptarla, sin abrir un socket a
  Redis) y además congelar que se invoca con los argumentos correctos después de que la
  transacción hizo commit — mismo patrón que ya usa la suite de `cart` para su evento
  `order_created`.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Congelar el blindaje de split de A-15, con el mayor número de casos (Priority: P1)

Como responsable de la modernización, escribo characterization tests que ejercitan, uno por uno,
los cuatro huecos de seguridad que A-15 cerró en `_close_split` — comensal repetido, importes de
raíz ignorados en modo split, bloque sin comensal sin nombre, cobertura exacta comensal-consumo
↔ comensal-split — de modo que cualquier regresión futura sobre este invariante [PROTEGIDA] la
detecte esta suite antes que un cliente real.

**Why this priority**: es el invariante de mayor prioridad de todo el módulo — el registro de
anomalías lo marca [PROTEGIDA] precisamente porque ya se rompió una vez en producción (commit
`42b5dec3`, 2026-08-04) y el criterio de aceptación de esta spec exige que reciba, de las cinco
anomalías cubiertas, el mayor número de casos de test.

**Independent Test**: se puede verificar de forma aislada corriendo
`python -m unittest app.characterization_tests.test_table_sessions_split_blindaje -v` (o el
nombre de fichero que se use), migrando y formalizando los casos ya presentes en
`app/scripts/test_split_blindaje.py` sin necesitar aún el resto de la suite.

**Acceptance Scenarios**:

1. **Given** un `CloseSessionIn` con `billing_mode='split'` y dos bloques de `splits` que
   comparten el mismo `participant_id`, **When** se llama a `close_session`, **Then** responde
   422 citando los comensales repetidos, sin crear ninguna venta — congelando que un comensal no
   puede pagar dos veces por el mismo consumo.
2. **Given** un `CloseSessionIn` con `billing_mode='split'` y cualquiera de `discount`, `tax`,
   `tip` o `payments` puestos en la raíz del payload (no dentro de un bloque de `splits`),
   **When** se llama a `close_session`, **Then** responde 422 en vez de aceptar el importe y
   perderlo en silencio.
3. **Given** una sesión con un bloque de `splits` sin `participant_id` (lo que tecleó el mesero
   directamente), **When** se cierra en `billing_mode='split'`, **Then** la venta resultante
   tiene como `customer_name` el nombre de la mesa (`"Mesa {number}"`), nunca un nombre vacío.
4. **Given** una sesión con comensales con consumo real, **When** el `data.splits` recibido no
   cubre exactamente ese conjunto (falta alguno, o sobra uno sin consumo), **Then** `close_session`
   responde 422 citando específicamente los comensales que faltan o sobran, sin cobrar nada.
5. **Given** un split válido que cubre exactamente a los comensales con consumo, sin repetidos y
   sin importes en la raíz, **When** se llama a `close_session`, **Then** se genera una venta por
   comensal, cada una con su propio `customer_name`, y la sesión queda `closed`.

---

### User Story 2 - Congelar las 9 funciones públicas de `table_sessions/service.py` (Priority: P1)

Como responsable de la modernización, escribo characterization tests que ejercitan cada una de
las 9 funciones públicas de `service.py` contra los modelos SQLAlchemy reales sobre SQLite en
memoria (mismo mecanismo que el resto de `app/characterization_tests/`), documentando el
comportamiento que observo hoy — incluyendo A-01 (caso base), A-17 (R12), A-29 y A-38 —
sin corregir nada.

**Why this priority**: junto con la Historia 1, es el núcleo de la spec — sin esta red no existe
línea base contra la cual verificar que una futura extracción de `table_sessions` no cambió
nada. Es P1 igual que la Historia 1 porque ambas cubren `service.py`; se numeran distinto solo
porque A-15 exige tratamiento propio por su prioridad de negocio.

**Independent Test**: se puede verificar de forma aislada corriendo
`python -m unittest app.characterization_tests.test_table_sessions_service -v`, reutilizando los
fixtures ya usados por la Historia 1 para sembrar sesiones, comensales y pedidos.

**Acceptance Scenarios**:

1. **Given** una sesión de mesa con pedidos en distintos estados (`recibida`, `en_preparacion`,
   `cancelada`, `pagada`) repartidos entre dos comensales, con una promoción y un combo activos,
   **When** se llama a `compute_bill`, **Then** el `total` y el desglose por comensal excluyen
   los pedidos `cancelada`/`pagada`, aplican el descuento de promoción/combo por comensal, y
   coinciden exactamente con lo que hoy produce el código — congelado como el caso base de A-01
   (camino A, correcto).
2. **Given** dos llamadas concurrentes simuladas a `add_participant`, `remove_participant` o
   `set_assignments` sobre la misma sesión mientras otra operación tiene la fila bloqueada (
   sembrado directamente, sin concurrencia real), **When** se ejercitan en ese orden, **Then** el
   test documenta que estas tres funciones cargan la sesión sin `FOR UPDATE` (a diferencia de
   `close_session`, que sí lo usa) — el mismo defecto que describe A-17 (R12), sin corregirlo.
3. **Given** una sesión cuyas líneas cobradas usan dos combos distintos (o ninguno), **When** se
   llama a `compute_bill` o se cierra la sesión en cualquiera de los dos `billing_mode`,
   **Then** el test observa que `promotion_id` no registra ningún combo — el mismo defecto que
   describe A-29, sin corregirlo, aunque el descuento monetario sí se sume correctamente.
4. **Given** una mesa con un único comensal y consumo propio, **When** se cierra la sesión con
   `billing_mode='split'` con un solo bloque para ese comensal, **Then** el cierre se acepta sin
   ninguna restricción de mínimo de comensales — congelando RN-MESA-13 (A-38), equivalente en la
   práctica a un `unified` disfrazado.
5. **Given** un comensal con al menos un producto asignado (incluso si ese ítem está `anulado` o
   pertenece a un pedido ya no cobrable), **When** se llama a `remove_participant`, **Then**
   responde 409 y no lo quita — congelando RN-MESA-24 (A-38) tal cual, sin distinguir si el
   producto asignado sigue siendo cobrable.
6. **Given** una sesión sin comensales activos y sin pedidos cobrables, **When** se llama a
   `try_release_if_empty`, **Then** libera la mesa (`status='libre'`) y cierra la sesión;
   **Given** la misma sesión pero con un pedido todavía cobrable, **When** se llama de nuevo,
   **Then** no libera nada — congelando las dos condiciones documentadas en su docstring.
7. **Given** la llamada real a Redis interceptada (sin abrir un socket real), **When** se llama a
   `add_participant`, `remove_participant` o `set_assignments` con éxito, **Then** el test
   observa que `app.core.events.bill_changed` se invoca exactamente una vez, con el `tenant_id` y
   el `table_session_id` correctos, después de que la transacción del `service` ya hizo commit —
   congelando el contrato del evento sin depender de una conexión Redis real.

---

### User Story 3 - Congelar los 7 endpoints de `table_sessions/router.py` (Priority: P2)

Como responsable de la modernización, escribo characterization tests que ejercitan cada uno de
los 7 endpoints de `router.py`, congelando el comportamiento observable que añade la capa de
router sobre `service.py` ya congelado en las Historias 1 y 2: códigos de estado, forma de la
respuesta y mapeo de errores.

**Why this priority**: depende de que las Historias 1 y 2 existan primero (el router delega toda
la lógica de negocio en `service.py`); es de menor prioridad porque, aislada, protege una capa
más delgada — pero sigue siendo parte del alcance que el encargo pide explícitamente ("los
endpoints de table_sessions/router.py").

**Independent Test**: se puede verificar de forma aislada corriendo
`python -m unittest app.characterization_tests.test_table_sessions_router -v`, reutilizando los
mismos fixtures de las Historias 1 y 2 para construir el contexto de usuario/tenant que cada
endpoint necesita.

**Acceptance Scenarios**:

1. **Given** una sesión de mesa existente, **When** se invoca `GET /table-sessions/{id}`,
   **Then** responde 200 con `TableSessionResponse` incluyendo sus comensales; **Given** un id
   que no existe, **When** se invoca el mismo endpoint, **Then** responde 404.
2. **Given** un `display_name` vacío o solo espacios, **When** se invoca
   `POST /table-sessions/{id}/participants`, **Then** responde 422 sin crear el comensal;
   **Given** un nombre válido, **Then** responde 201 con el `display_label` desambiguado.
3. **Given** un comensal con productos asignados, **When** se invoca
   `DELETE /table-sessions/{id}/participants/{participant_id}`, **Then** responde 409 con el
   detalle de cuántos productos tiene asignados, congelando el contrato de error del endpoint.
4. **Given** un lote de asignaciones válido, **When** se invoca
   `PUT /table-sessions/{id}/assignments`, **Then** responde 200 con la `SessionBillResponse` ya
   recalculada — congelando que el endpoint no exige una segunda llamada a `GET .../bill`.
5. **Given** un `CloseSessionIn` válido para `billing_mode='unified'`, **When** se invoca
   `POST /table-sessions/{id}/close`, **Then** responde 200 con `CloseSessionResponse`
   (`table_session` ya `closed` y `sale_ids` con exactamente una venta).

---

### User Story 4 - La suite corre en CI de forma determinista (Priority: P3)

Como responsable de la modernización, incorporo los ficheros de test de las historias 1 a 3 al
mismo paso de CI del backend que ya ejecuta la red de `cart`
(`.github/workflows/deploy.yml` o el que lo sustituya), de modo que se ejecuten automáticamente
en cada push, y verifico que la suite completa pasa de forma idéntica en ejecuciones repetidas,
sin flakiness.

**Why this priority**: sin esto, la red existe pero no es el árbitro que el Principio II
exige — sería solo otro script más que alguien tiene que acordarse de correr a mano, el mismo
patrón que A-27 ya identificó como riesgo real. Es P3 porque depende de que las historias
anteriores ya existan para tener algo que ejecutar.

**Independent Test**: se puede verificar abriendo un pull request trivial contra el branch de
esta spec y confirmando en la ejecución de CI que el paso existente recoge los ficheros nuevos,
sin depender de que ninguna otra spec futura exista todavía.

**Acceptance Scenarios**:

1. **Given** el workflow de CI del backend, **When** se inspecciona su definición, **Then** el
   paso `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` (o
   equivalente) cubre también los ficheros nuevos de esta spec, sin necesitar un paso adicional
   ni reemplazar el existente.
2. **Given** la suite de esta spec ejecutada tres veces seguidas sin ningún cambio de código,
   **When** se comparan los tres resultados, **Then** los tres son idénticos — ninguno depende
   del reloj real, de Redis, de Postgres real, ni de orden de ejecución.
3. **Given** un push que modifica `table_sessions/service.py` o `router.py` de forma que cambia
   su comportamiento observable, **When** corre CI, **Then** al menos un test de esta suite falla
   en rojo, demostrando que la red efectivamente detecta el cambio.

### Edge Cases

- ¿Qué pasa si, al escribir un test de esta suite, falla contra el código actual sin modificar?
  → El defecto está en el test (mal capturó lo que el código realmente hace), nunca en
  `table_sessions/service.py` ni `router.py`: se corrige el test hasta que refleje la realidad
  observada. Esta spec no autoriza ningún cambio en los dos ficheros de producción.
- ¿Qué pasa si aparece, mientras se escriben estos tests, una anomalía no listada hoy en
  `registro-de-anomalias.md`? → Se documenta ahí (Principio I) y se congela tal cual en el test;
  no se corrige como parte de esta spec.
- ¿Qué pasa con las reglas propias de `orders.checkout` (qué líneas arma `order_sale_lines`, qué
  hace `close_table_sessions` internamente) o de `promotions` (qué promoción/combo aplica)? →
  Fuera de alcance: esta spec ejercita esas funciones reales con fixtures mínimos controlados
  (pedidos con y sin promoción/combo activos) para congelar cómo `table_sessions` las usa, no
  para caracterizar exhaustivamente sus propias reglas — eso pertenece a specs futuras de esos
  módulos.
- ¿Qué pasa si un test necesita concurrencia real (dos hilos/procesos a la vez) para observar
  A-17 (R12)? → No se requiere: el escenario se reproduce sembrando directamente el estado que la
  condición de carrera produciría (o invocando las funciones en la secuencia que la expone) y
  observando la ausencia de `FOR UPDATE`, sin necesitar concurrencia real ni no determinismo —
  mismo enfoque que `specs/015-caracterizacion-cart/` usó para A-17 (R16).
- ¿Qué pasa con los tres scripts legado (`test_table_sessions.py`, `test_table_release.py`,
  `test_split_blindaje.py`)? → Se usan como fuente de casos ya pensados (en particular
  `test_split_blindaje.py` para la Historia 1), pero ningún test de esta spec se limita a
  copiarlos: cada caso se reescribe en la convención formal (`unittest`, docstring "CONGELA
  comportamiento actual:") y se verifica contra el código real antes de aceptarse.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE incluir, bajo `app/characterization_tests/`, uno o más ficheros de
  test (prefijo `test_table_sessions_`) que sigan la convención documentada en
  `app/characterization_tests/__init__.py` (docstring "CONGELA comportamiento actual:",
  Principio II) y usen exclusivamente `unittest` de la biblioteca estándar sobre los modelos
  SQLAlchemy reales del proyecto vía SQLite en memoria — cero dependencias nuevas (Principio IV).
- **FR-002**: La suite DEBE incluir al menos un characterization test por cada una de las 9
  funciones públicas de `table_sessions/service.py`: `get_session`, `has_billable_orders`,
  `try_release_if_empty`, `list_sessions`, `compute_bill`, `close_session`, `add_participant`,
  `remove_participant`, `set_assignments`.
- **FR-003**: La suite DEBE incluir al menos un characterization test por cada uno de los 7
  endpoints de `table_sessions/router.py`: `GET /table-sessions`, `GET /table-sessions/{id}`,
  `POST /table-sessions/{id}/participants`,
  `DELETE /table-sessions/{id}/participants/{participant_id}`,
  `PUT /table-sessions/{id}/assignments`, `GET /table-sessions/{id}/bill`,
  `POST /table-sessions/{id}/close`.
- **FR-004**: La suite DEBE incluir al menos un caso que congele A-01 (camino A,
  `compute_bill`) como caso base de referencia: excluye pedidos `cancelada`/`pagada` y aplica
  promociones/combos por comensal, sin abrir ninguna pregunta nueva sobre su corrección.
- **FR-005**: La suite DEBE incluir el mayor número de casos, de las cinco anomalías cubiertas
  por esta spec, para A-15 [PROTEGIDA] — como mínimo un caso por cada uno de sus cuatro huecos
  de seguridad cerrados: comensal repetido en el split, importes de raíz ignorados en modo
  split, bloque sin comensal sin nombre en la venta, y cobertura exacta comensal-consumo ↔
  comensal-split.
- **FR-006**: La suite DEBE incluir al menos un caso que congele A-17 (R12): a diferencia de
  `close_session` (que carga la sesión con `FOR UPDATE`), `add_participant`, `remove_participant`
  y `set_assignments` la cargan sin bloqueo — reproducido sin requerir concurrencia real.
- **FR-007**: La suite DEBE incluir al menos un caso que congele A-29: cuando las líneas
  cobradas usan más de un combo distinto (o ninguno), `promotion_id` no registra ningún combo,
  aunque el descuento monetario se sume correctamente.
- **FR-008**: La suite DEBE incluir al menos un caso para cada uno de los dos hallazgos de A-38
  que viven en `table_sessions`: RN-MESA-13 (cierre en `billing_mode='split'` de una mesa de un
  solo comensal, sin restricción de mínimo) y RN-MESA-24 (`remove_participant` rechaza quitar un
  comensal con productos asignados, aunque estén anulados o no sean ya cobrables).
- **FR-009**: La suite DEBE ejercitar `orders.checkout` (`TERMINAL`, `close_table_sessions`,
  `order_sale_lines`, `promo_lines_for`) y `promotions` (`evaluate`,
  `combo_discount_for_lines`) como dependencias reales sobre fixtures mínimos y controlados, sin
  mocks y sin re-derivar sus propias reglas de negocio, y sin reordenar la dependencia de
  `table_sessions` sobre `orders.checkout`.
- **FR-010**: Cada test DEBE ser determinista: ninguno puede depender del reloj real de la
  máquina que lo ejecuta, de una conexión Redis o Postgres real, ni del orden de ejecución de
  otros tests.
- **FR-010a**: Los tests que ejercitan `add_participant`, `remove_participant` o
  `set_assignments` DEBEN interceptar la llamada real a `app.core.events.bill_changed` (evitando
  el socket a Redis exigido por FR-010) y, además, congelar que se invoca exactamente una vez,
  con el `tenant_id` y el `table_session_id` correctos, después de que la transacción del
  `service` ya hizo commit.
- **FR-011**: La suite DEBE ejecutarse automáticamente como parte del mismo paso de CI del
  backend que ya ejecuta la red de `cart` (`.github/workflows/deploy.yml` o el que lo
  sustituya) en cada push, sin reemplazar ningún paso existente.
- **FR-012**: Ningún test de esta suite DEBE requerir una modificación de
  `app/api/v1/table_sessions/service.py` ni de `app/api/v1/table_sessions/router.py` para pasar:
  si un test recién escrito falla contra el código actual sin modificar, el defecto está en el
  test.
- **FR-013**: El sistema NO DEBE corregir, mitigar ni alterar A-01, A-15, A-17 (R12), A-29, ni
  A-38 (RN-MESA-13, RN-MESA-24) como parte de esta spec — cada test debe documentar el
  comportamiento observado, no el comportamiento deseado.
- **FR-014**: El sistema NO DEBE extraer, mover, ni reescribir código de
  `table_sessions/service.py` ni `router.py`, ni cambiar la interfaz (firmas de funciones, rutas
  de endpoints, esquemas de request/response) de ninguno de los dos ficheros, ni reordenar su
  dependencia de `orders.checkout`, como parte de esta spec.

### Key Entities *(include if feature involves data)*

- **Función pública de `table_sessions/service.py`**: una de las 9 funciones sin prefijo `_` —
  la unidad mínima de cobertura exigida por FR-002.
- **Endpoint de `table_sessions/router.py`**: una de las 7 rutas HTTP expuestas — la unidad
  mínima de cobertura exigida por FR-003.
- **Caso de anomalía citado (A-01 / A-15 / A-17 / A-29 / A-38)**: un characterization test cuyo
  propósito explícito es documentar el comportamiento de una anomalía ya registrada, con su cita
  correspondiente en el docstring o nombre del test.
- **Dependencia externa fijada**: un módulo que `table_sessions` consume (`orders.checkout`,
  `promotions`) cuyo comportamiento se ejercita real y sin mocks, pero no se re-caracteriza en
  profundidad dentro de esta spec.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las 9 funciones públicas de `table_sessions/service.py` y el 100% de
  los 7 endpoints de `table_sessions/router.py` tienen al menos un characterization test
  asociado.
- **SC-002**: A-01, A-15, A-17 (R12), A-29 y A-38 (RN-MESA-13, RN-MESA-24) están representados
  cada uno por al menos un caso, verificable por su cita explícita en el docstring o nombre del
  test correspondiente; A-15 tiene, de las cinco, el mayor número de casos (al menos cuatro, uno
  por cada hueco de seguridad cerrado).
- **SC-003**: La suite completa pasa en verde en tres ejecuciones consecutivas sin ningún cambio
  de código entre ellas, con exactamente el mismo resultado en las tres.
- **SC-004**: La suite se ejecuta automáticamente en el pipeline de CI del backend — verificable
  inspeccionando la definición del workflow — y no solo de forma manual/local.
- **SC-005**: `table_sessions/service.py` y `table_sessions/router.py` tienen cero líneas
  modificadas (diff vacío) al concluir esta spec, comparados contra su estado inmediatamente
  anterior a ella.
- **SC-006**: Cero dependencias nuevas añadidas a `requirements.txt` como resultado de esta
  spec.

## Assumptions

- `cart` ya tiene su propia red de characterization tests (`specs/015-caracterizacion-cart/`
  concluida), incluyendo `unique_display_label`, que `table_sessions._unique_label` reimporta.
  Esta spec no la vuelve a caracterizar: la consume real, tal como ya lo hace el código.
- `orders.checkout` y `promotions.service` no tienen todavía su propia red
  `characterization_tests` propia y dedicada (`promotions` solo tiene el script legado
  `test_promotions_rules.py`, que sí corre en CI pero fuera de esta convención; `checkout` no
  tiene ninguna). Esta spec no llena ese vacío: los ejercita como dependencias reales con
  fixtures mínimos, suficientes para congelar cómo `table_sessions` los usa, sin caracterizar
  sus reglas propias — ese trabajo, si se necesita, es una spec futura de esos módulos.
- El mecanismo concreto para congelar los endpoints de `router.py` (invocar las funciones de
  ruta directamente con un `User`/`Tenant`/`Session` construidos a mano, vs.
  `fastapi.testclient`) es una decisión de implementación que se resuelve en la fase de
  planificación, no en esta spec — el requisito aquí es el comportamiento observable congelado
  (FR-003, FR-010), no el mecanismo.
- El caso de A-17 (R12) se reproduce sembrando directamente el estado de datos o invocando las
  funciones en la secuencia que expone la ausencia de `FOR UPDATE`, no ejecutando
  hilos/procesos concurrentes reales — mismo enfoque, y misma justificación, que
  `specs/015-caracterizacion-cart/` usó para A-17 (R16).
- Los tres scripts legado (`test_table_sessions.py`, `test_table_release.py`,
  `test_split_blindaje.py`) se leen como fuente de casos ya pensados por el equipo, en particular
  `test_split_blindaje.py` para la Historia 1 (A-15), pero cada caso migrado se reescribe en la
  convención formal y se verifica contra el código real antes de aceptarse — no se importan ni
  se referencian directamente como dependencia de ejecución.
- `_notify_bill_changed` atrapa cualquier excepción y solo loguea un warning (`service.py:501-510`),
  así que dejarlo sin interceptar no rompería un test — pero sí introduciría una conexión de red
  real (con su propio timeout) en cada test de `add_participant`/`remove_participant`/
  `set_assignments`, violando el espíritu de FR-010. Por eso se intercepta explícitamente
  (FR-010a), siguiendo el mismo patrón que `specs/015-caracterizacion-cart/` usó para el evento
  `order_created` de `cart` y para el rate limiting dependiente de Redis
  (`settings.RATE_LIMIT_ENABLED`).
- El paso de CI de esta spec se suma al mismo paso ya usado por
  `specs/015-caracterizacion-cart/` en `.github/workflows/deploy.yml` (el único workflow del
  backend que existe hoy); si en el momento de implementar existe otro mecanismo de CI para el
  backend, se usa ese en su lugar, conservando el requisito sustantivo (FR-011): que la suite
  corra automáticamente en cada push, no solo a mano.
