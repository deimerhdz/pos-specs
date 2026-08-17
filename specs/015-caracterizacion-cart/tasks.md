---

description: "Task list for feature implementation"
---

# Tasks: Red de characterization tests para `cart` (`router.py` + `service.py`)

**Input**: Design documents from `/specs/015-caracterizacion-cart/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/test-harness-api.md](./contracts/test-harness-api.md), [quickstart.md](./quickstart.md)

**Path convention**: todas las rutas de código son relativas a la raíz del repositorio sibling
`../pos-backend` (ejecutado desde `pos-specs/`), tal como establece `plan.md` §Project Structure.
Las rutas bajo `specs/` o `.specify/` son relativas a este repositorio (`pos-specs`).

**Tests**: esta spec es, en sí misma, la escritura de una red de characterization tests (Principio
II) — no hay código de producción nuevo que probar con TDD. Cada tarea de Historia 1/2 crea
directamente los tests que congelan `cart/service.py`/`router.py`; no hay una fase separada de
"tests antes de implementación" porque no hay implementación de producción en esta spec (FR-012).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias pendientes)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3)

---

## Phase 1: Setup

**Purpose**: preparar el entorno de `pos-backend` para escribir la red nueva

- [X] T001 Activar el entorno virtual de `pos-backend` (`cd ../pos-backend && source env/bin/activate`, o `pip install -r requirements.txt` en uno nuevo) y confirmar que `app/characterization_tests/cart_fixtures.py`, `app/characterization_tests/test_cart_service.py` y `app/characterization_tests/test_cart_router.py` todavía no existen (`ls app/characterization_tests/`), sin colisión de nombre

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: construir `app/characterization_tests/cart_fixtures.py` con la porción que necesitan
**ambas** historias (esquema SQLite ampliado + factories + reloj fijado) — ver
[contracts/test-harness-api.md](./contracts/test-harness-api.md). La porción exclusiva del harness
de router (`build_session_context`, `patched_qr_context`, `patched_session_context`,
`FakeRedisBucket`) se añade dentro de la Historia 2 (T023), que es la única que la necesita.

**⚠️ CRITICAL**: ningún test de Historia 1 ni Historia 2 puede escribirse sin este fixture

- [X] T002 En `app/characterization_tests/cart_fixtures.py` (nuevo fichero): implementar `new_session()` reexportando tal cual las factories de `fixtures.py` (`make_unit`, `make_category`, `make_product`, `make_variant`, `make_inventory_item`, `make_option_group`, `make_option`, `link_variant_group`, `make_recipe_item`) y ampliando la lista de tablas a crear con las 14 tablas nuevas (`dining_tables, table_sessions, session_participants, carts, cart_items, cart_item_options, customer_orders, order_items, order_item_options, promotions, promotion_targets, promotion_combo_items, order_cancel_logs, audit_logs`), removiendo de `Table.indexes` los dos índices `postgresql_where` (`idx_active_session_per_table` en `table_sessions`, `idx_open_cart_per_participant` en `carts`) antes de `create_all` (research.md §3-4; `fixtures.py` no se modifica)

- [X] T003 En `cart_fixtures.py`: añadir `make_dining_table(db, **kw)`, `make_table_session(db, table=None, **kw)`, `make_participant(db, table_session=None, **kw)` siguiendo el patrón `kw.setdefault(...)` + `db.add` + `db.flush()` de `fixtures.py`, con defaults inertes (`active=True`, nombres derivados del `id`) (contracts/test-harness-api.md §Factories nuevas; depende de T002)

- [X] T004 En `cart_fixtures.py`: añadir `make_cart(db, participant=None, **kw)` y `make_cart_item(db, cart, variant, **kw)` con el mismo patrón (contracts/test-harness-api.md; depende de T002)

- [X] T005 En `cart_fixtures.py`: añadir `make_customer_order(db, participant, **kw)` con el mismo patrón (contracts/test-harness-api.md; depende de T002)

- [X] T006 En `cart_fixtures.py`: añadir `make_promotion(db, **kw)`, `make_promotion_target(db, promotion, **kw)`, `make_combo_item(db, promotion, variant, **kw)` con el mismo patrón (contracts/test-harness-api.md; depende de T002)

- [X] T007 En `cart_fixtures.py`: añadir `frozen_now(instant: datetime)` como context manager que hace `unittest.mock.patch("app.api.v1.cart.service.datetime", FixedDatetime)`, con `FixedDatetime` como subclase mínima de `datetime.datetime` que sobrescribe solo `now(tz=None)` para devolver `instant` (research.md §5, contracts/test-harness-api.md §Reloj fijado; depende de T002)

- [X] T008 Test de humo del fixture (dentro de `cart_fixtures.py` o en un bloque `if __name__ == "__main__":` temporal, según se prefiera al implementar): abrir `new_session()`, ejercitar `make_dining_table` → `make_table_session` (sembrar **dos** filas `active` para la misma mesa) → `make_participant` → `make_cart` → `make_cart_item`, y confirmar que ninguna inserción falla por el índice único incondicional que SQLite compilaría sin la remoción de T002, y que `audit_logs.payload` (JSONB→JSON en SQLite) acepta un `dict` (research.md §3-4; depende de T003, T004, T005, T006, T007)

**Checkpoint**: `cart_fixtures.py` crea las 14 tablas nuevas, sus factories básicas y el reloj
fijado, verificado por el test de humo — Historia 1 puede empezar

---

## Phase 3: User Story 1 - Congelar las 11 funciones públicas de `cart/service.py` (Priority: P1) 🎯 MVP

**Goal**: un characterization test por cada una de las 11 funciones públicas de
`cart/service.py`, incluyendo los casos A-08 y A-17 (R16), ejercitando el motor de catálogo, las
promociones y el checkout reales (sin mocks) contra SQLite en memoria.

**Independent Test**: `python -m unittest app.characterization_tests.test_cart_service -v` pasa en
verde sin que existan aún `test_cart_router.py` ni el wiring de CI (spec.md, Historia 1).

### Implementation for User Story 1

- [X] T009 [US1] En `app/characterization_tests/test_cart_service.py` (nuevo fichero, docstring "CONGELA comportamiento actual:" per `__init__.py:1-39`): test de `unique_display_label` — nombre sin colisión se devuelve tal cual; con un participante existente del mismo nombre en la sesión, se sufija de forma determinista (FR-002)

- [X] T010 [US1] Test de `open_session` — camino feliz: mesa activa sembrada con `make_dining_table`, crea `TableSession` + `SessionParticipant` + carrito inicial, `expires_at` calculado desde `_now()` (FR-002)

- [X] T011 [US1] Test de `open_session` — A-17 (R16): sembrar directamente dos `TableSession` con `status='active'` para la misma mesa (posible gracias a T002) y confirmar que `open_session`/`_get_or_create_table_session` propaga hoy una excepción no controlada, no un 4xx explícito, citando A-17 (R16) en el docstring del test (spec.md Historia 1 Acceptance Scenario 2, FR-005, SC-002)

- [X] T012 [US1] Test de `open_session` y `serialize_cart` — A-08: usando `frozen_now()` (T007) con un instante en el que `TENANT_TIMEZONE=America/Bogota` difiere de UTC, observar que el naive que `_now()` produce se trata como si ya fuera hora local por `promotions.active_discount_promotions`/`local_now()`, de forma determinista, citando A-08 en el docstring del test (spec.md Historia 1 Acceptance Scenario 3, FR-004, SC-002)

- [X] T013 [US1] Test de `get_cart` — carrito existente del participante se serializa correctamente; sin carrito abierto, documentar el comportamiento actual (crea uno nuevo o responde vacío, según lo que el código haga hoy) (FR-002)

- [X] T014 [US1] Test de `add_item` — variante activa + opciones válidas: el precio de línea (delegado a `compute_line_price`/`load_valid_options`/`check_availability` reales, sin mock), las opciones guardadas y el `CartResponse` devuelto coinciden con lo que produce el código hoy (spec.md Historia 1 Acceptance Scenario 1, FR-002, FR-006)

- [X] T015 [US1] Test de `update_item` — cambia cantidad/opciones de una línea existente y observa el recálculo de precio resultante (FR-002)

- [X] T016 [US1] Test de `remove_item` — elimina una línea del carrito y observa el `CartResponse` resultante, incluyendo el caso de eliminar la última línea (FR-002)

- [X] T017 [US1] Test de `serialize_cart` — `discounted_total`: un caso sin promoción activa (default, FR-007) y un caso con una promoción de descuento activa sembrada vía `make_promotion`/`make_promotion_target`, confirmando que las líneas de combo (sembradas vía `make_combo_item`) no reciben descuento adicional percent/fixed encima de su propio ahorro (spec.md Historia 1 Acceptance Scenario 4, FR-002, FR-007)

- [X] T018 [US1] Test de `list_my_orders` — pedidos previos del participante (creados vía `submit_cart` o sembrados con `make_customer_order`) se listan en el orden/forma que produce el código hoy (FR-002)

- [X] T019 [US1] Test de `cancel_my_order` — pedido en estado `recibida` (sin ítems en cocina) se cancela; el mismo pedido en `en_preparacion` responde 409, congelando la política del docstring de la función tal cual (spec.md Historia 1 Acceptance Scenario 5, FR-002)

- [X] T020 [US1] Test de `leave_session` — participante abandona la sesión y observa el efecto sobre `SessionParticipant`/`TableSession` que produce el código hoy (FR-002)

- [X] T021 [US1] Test de `submit_cart` — carrito con ítems se confirma en un `CustomerOrder`, el carrito queda `'confirmado'` y un carrito nuevo `'abierto'` puede crearse después para el mismo participante (posible gracias a la remoción del índice único en T002), ejercitando `orders.checkout` real donde aplique (FR-002, FR-007)

- [X] T022 [US1] Ejecutar `python -m unittest app.characterization_tests.test_cart_service -v` desde `../pos-backend` y confirmar que las 13 pruebas (T009-T021) pasan en verde, incluyendo al menos un test que cite A-08 y otro que cite A-17 (R16) en su nombre/docstring (SC-001, SC-002; depende de T009-T021)

**Checkpoint**: las 11 funciones públicas de `cart/service.py` están congeladas y verificables de
forma independiente — Historia 2 puede empezar

---

## Phase 4: User Story 2 - Congelar los 9 endpoints de `cart/router.py` (Priority: P2)

**Goal**: un characterization test por cada uno de los 9 endpoints de `cart/router.py`,
congelando lo que la capa de router añade sobre `service.py` (códigos de estado, forma de
respuesta, rate limiting, eventos post-commit, caché ETag/304, contrato "nunca falla con 401" de
`/cart/leave`).

**Independent Test**: `python -m unittest app.characterization_tests.test_cart_router -v` pasa en
verde reutilizando `cart_fixtures.py` (spec.md, Historia 2).

### Implementation for User Story 2

- [X] T023 [US2] Extender `cart_fixtures.py` con la porción exclusiva del harness de router (research.md §1-2, contracts/test-harness-api.md): `build_session_context(db, *, tenant_id=1, table, table_session, participant)` que puebla un `SessionContext` real de `app.core.qr_context`; `patched_qr_context`/`patched_session_context` como context managers que parchean `app.api.v1.cart.router.open_qr_context`/`open_session_context` con un doble que entrega un `QrContext`/`SessionContext` de prueba (o levanta la misma `HTTPException` 401 real); `FakeRedisBucket` con `incr`/`expire` async y contador en memoria (depende de T002)

- [X] T024 [US2] En `app/characterization_tests/test_cart_router.py` (nuevo fichero): test de `POST /cart/sessions` — token de QR válido para mesa activa responde 201 con `SessionOpenResponse` y `session_token` utilizable (con `patched_qr_context`); mesa inexistente/inactiva responde 404 (spec.md Historia 2 Acceptance Scenario 1, FR-003)

- [X] T025 [US2] Test de `GET /cart` — invocado con un `SessionContext` construido vía `build_session_context()` sobre un carrito ya sembrado, devuelve el `CartResponse` esperado (FR-003)

- [X] T026 [US2] Test de `POST /cart/items` — camino feliz: con `settings.RATE_LIMIT_ENABLED=False`, delega en `service.add_item` y responde igual que la función de servicio ya congelada en T014 (FR-003)

- [X] T027 [US2] Test de `POST /cart/items` — límite de tasa: con `settings.RATE_LIMIT_ENABLED=True` y `FakeRedisBucket` (T023) parcheado sobre `app.core.rate_limit.redis`, superar el límite del bucket `cart_items` desde la misma mesa responde 429 con `Retry-After`, congelando que el límite se aplica antes de tocar `service.add_item` (spec.md Historia 2 Acceptance Scenario 5, FR-003)

- [X] T028 [US2] Test de `PATCH /cart/items/{item_id}` — delega en `service.update_item` y responde igual que la función ya congelada en T015 (FR-003)

- [X] T029 [US2] Test de `DELETE /cart/items/{item_id}` — delega en `service.remove_item` y responde igual que la función ya congelada en T016 (FR-003)

- [X] T030 [US2] Test de `POST /cart/leave` — sin `x-session-token` en la petición responde 204 sin lanzar 401 (no necesita `patched_session_context`: el endpoint retorna antes de abrir el contexto, research.md §1); con token válido (`patched_session_context`), también responde 204 (spec.md Historia 2 Acceptance Scenario 2, FR-003)

- [X] T031 [US2] Test de `POST /cart/submit` — con éxito, el evento `order_created` se publica después de que la transacción de `service.submit_cart` ya hizo commit, congelando el orden actual (usar un doble/espía sobre `app.core.events.publish` o observar el estado ya comiteado en el momento de la llamada) (spec.md Historia 2 Acceptance Scenario 3, FR-003)

- [X] T032 [US2] Test de `GET /cart/orders` — primera petición devuelve 200 con `ETag`; segunda petición con el mismo `ETag` en la cabecera responde 304, congelando el mecanismo de caché documentado en el endpoint (spec.md Historia 2 Acceptance Scenario 4, FR-003)

- [X] T033 [US2] Test de `POST /cart/orders/{order_id}/cancel` — delega en `service.cancel_my_order` y responde igual que la función ya congelada en T019 (`recibida` cancela, `en_preparacion` 409) (FR-003)

- [X] T034 [US2] Ejecutar `python -m unittest app.characterization_tests.test_cart_router -v` desde `../pos-backend` y confirmar que las 10 pruebas (T024-T033) pasan en verde (SC-001; depende de T024-T033)

**Checkpoint**: los 9 endpoints de `cart/router.py` están congelados y verificables de forma
independiente — Historia 3 puede empezar

---

## Phase 5: User Story 3 - La suite corre en CI de forma determinista (Priority: P3)

**Goal**: sumar `test_cart_service.py` y `test_cart_router.py` al job `test` de
`.github/workflows/deploy.yml`, corrigiendo la causa raíz de A-27 (el paso `Install deps` no
instala lo necesario para importar ningún fichero de `app/characterization_tests/`).

**Independent Test**: abrir un PR trivial contra esta rama y confirmar en la ejecución de CI que
el paso nuevo corre y pasa (spec.md, Historia 3).

**Prerequisito**: Historias 1 y 2 en verde — sin ellas no hay nada que el wiring de CI ejecute.

### Implementation for User Story 3

- [X] T035 [US3] En `.github/workflows/deploy.yml`: cambiar el paso `Install deps` (hoy `pip install sqlalchemy pydantic pydantic-settings fastapi`) por `pip install -r requirements.txt` (research.md §6, FR-009)

- [X] T036 [US3] En `.github/workflows/deploy.yml`: sumar (sin reemplazar) un paso nuevo después del paso existente `Reglas de promociones` (`test_promotions_rules`) que ejecute `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` (research.md §6, FR-009; depende de T035)

- [X] T037 [US3] Verificar localmente los dos comandos que CI ejecutará: `python -m app.scripts.test_promotions_rules` y `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` desde `../pos-backend`, confirmando ambos en verde (quickstart.md Escenario 5, SC-004; depende de T036)

- [X] T038 [US3] Verificar determinismo (SC-003): correr `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` tres veces seguidas sin ningún cambio de código, redirigiendo cada salida a un log, y confirmar que las tres son idénticas (quickstart.md Escenario 3; depende de T037)

- [X] T039 [US3] Verificar que la red detecta un cambio real (spec.md Historia 3 Acceptance Scenario 3): modificar temporalmente una línea de `app/api/v1/cart/service.py` (por ejemplo invertir una condición en `cancel_my_order`), confirmar con `git diff` que el cambio es visible, correr `test_cart_service.py` y confirmar que al menos un test falla en rojo, luego revertir con `git checkout -- app/api/v1/cart/service.py` antes de continuar (quickstart.md Escenario 4, FR-012; depende de T038)

**Checkpoint**: las tres historias están completas — la suite corre en CI, es determinista y
detecta cambios reales

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de que la spec no tocó producción y cerró el alcance

- [X] T040 [P] Confirmar diff vacío en `app/api/v1/cart/service.py` y `app/api/v1/cart/router.py` respecto a su estado inmediatamente anterior a esta spec (`git diff --stat -- app/api/v1/cart/service.py app/api/v1/cart/router.py` sin salida) (SC-005)

- [X] T041 [P] Confirmar cero dependencias nuevas en `requirements.txt` (`git diff --stat -- requirements.txt` sin salida) (SC-006)

- [X] T042 Recorrer [quickstart.md](./quickstart.md) de punta a punta (Escenarios 1 → 5) como pase de sanidad final sobre el estado ya implementado del repositorio (depende de T022, T034, T039)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA Historias 1 y 2
- **User Story 1 (Phase 3)**: depende de Foundational; es la base de Historia 2 (el router delega en `service.py`)
- **User Story 2 (Phase 4)**: depende de que Historia 1 esté completa (reutiliza sus fixtures y su comportamiento ya congelado como línea base) — spec.md lo declara explícitamente, a diferencia del patrón habitual de historias independientes
- **User Story 3 (Phase 5)**: depende de que Historias 1 y 2 estén en verde (prerequisito explícito de spec.md)
- **Polish (Phase 6)**: depende de que las tres historias estén completas

### User Story Dependencies

- **User Story 1 (P1)**: depende solo de Foundational — es el punto de partida
- **User Story 2 (P2)**: depende de User Story 1 (reutiliza sus fixtures y su comportamiento congelado)
- **User Story 3 (P3)**: depende de User Story 1 y User Story 2 en verde

### Parallel Opportunities

- Dentro de Foundational, T003-T007 tocan el mismo fichero (`cart_fixtures.py`) y se ejecutan en secuencia, no en paralelo
- Dentro de Historia 1, T009-T021 tocan el mismo fichero (`test_cart_service.py`) y se ejecutan en secuencia; una vez escritos, pueden revisarse/ejecutarse como conjunto en T022
- Dentro de Historia 2, T024-T033 tocan el mismo fichero (`test_cart_router.py`) y se ejecutan en secuencia
- T040 y T041 son de solo lectura/verificación sobre ficheros distintos y pueden ejecutarse en paralelo

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (fixture base + factories + reloj fijado, verificado por el test de humo T008)
3. Completar Phase 3: User Story 1
4. **PARAR y VALIDAR**: `test_cart_service.py` en verde, citando A-08 y A-17 (R16) explícitamente
5. En este punto existe línea base para las 11 funciones públicas de `service.py`, aunque el router y el wiring de CI todavía no existan — seguro parar aquí si hace falta

### Incremental Delivery

1. Setup + Foundational → fixture base listo
2. Historia 1 → `service.py` congelado (MVP técnico)
3. Historia 2 → `router.py` congelado, reutilizando la línea base de Historia 1
4. Historia 3 → la suite corre en CI de forma determinista, cerrando A-27 para estos dos ficheros
5. Cada historia añade una señal de verificación sin romper la anterior — el orden P1→P2→P3 es también el orden de ejecución obligatorio (spec.md lo declara explícitamente para Historia 2), no solo de prioridad

---

## Notes

- [P] = ficheros distintos, sin dependencias pendientes entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- Ningún test de esta suite debe requerir modificar `cart/service.py` ni `cart/router.py` para pasar (FR-010): si una tarea de test falla contra el código actual sin modificar, el defecto está en el test, no en producción — corregir el test, nunca el código
- Esta spec no autoriza corregir A-08 ni A-17 (R16): los tests de T011/T012 documentan el comportamiento observado, no el deseado (FR-011)
- Commitear tras cada tarea o grupo lógico
- Parar en cada checkpoint para validar la historia de forma independiente antes de seguir
