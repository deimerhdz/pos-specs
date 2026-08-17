---

description: "Task list for feature implementation"
---

# Tasks: Red de characterization tests para `table_sessions` (`router.py` + `service.py`)

**Input**: Design documents from `/specs/016-caracterizacion-table-sessions/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/test-harness-api.md](./contracts/test-harness-api.md), [quickstart.md](./quickstart.md)

**Path convention**: todas las rutas de código son relativas a la raíz del repositorio sibling
`../pos-backend` (ejecutado desde `pos-specs/`), tal como establece `plan.md` §Project Structure.
Las rutas bajo `specs/` o `.specify/` son relativas a este repositorio (`pos-specs`).

**Tests**: esta spec es, en sí misma, la escritura de una red de characterization tests (Principio
II) — no hay código de producción nuevo que probar con TDD. Cada tarea de Historia 1/2/3 crea
directamente los tests que congelan `table_sessions/service.py`/`router.py`; no hay una fase
separada de "tests antes de implementación" porque no hay implementación de producción en esta
spec (FR-012).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias pendientes)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3, US4)

---

## Phase 1: Setup

**Purpose**: preparar el entorno de `pos-backend` para escribir la red nueva

- [X] T001 Activar el entorno virtual de `pos-backend` (`cd ../pos-backend && source env/bin/activate`, o `pip install -r requirements.txt` en uno nuevo) y confirmar que `app/characterization_tests/table_sessions_fixtures.py`, `app/characterization_tests/test_table_sessions_split_blindaje.py`, `app/characterization_tests/test_table_sessions_service.py` y `app/characterization_tests/test_table_sessions_router.py` todavía no existen (`ls app/characterization_tests/`), sin colisión de nombre con `cart_fixtures.py`/`fixtures.py`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: construir `app/characterization_tests/table_sessions_fixtures.py` completo — esquema
SQLite ampliado, factories nuevas, dobles de `Tenant`/`User`, espía de `_load` (A-17/R12) y doble
de `events.bill_changed` (FR-010a) — ver [contracts/test-harness-api.md](./contracts/test-harness-api.md).
A diferencia de `cart_fixtures.py`, este módulo es autónomo desde el principio: las tres historias
de test (US1, US2, US3) lo consumen completo, no hay una porción exclusiva de router que se añada
después.

**⚠️ CRITICAL**: ningún test de Historia 1, 2 ni 3 puede escribirse sin este fixture completo

- [X] T002 En `app/characterization_tests/table_sessions_fixtures.py` (nuevo fichero): implementar `new_session()` reexportando tal cual las factories de `fixtures.py` (`make_unit`, `make_category`, `make_product`, `make_variant`, `make_inventory_item`, `make_option_group`, `make_option`, `link_variant_group`, `make_recipe_item`), registrando el compilador `@compiles(JSONB, "sqlite")` → `"JSON"` (research.md §4, mismo shim que `cart_fixtures.py` pero registrado de forma independiente) antes de `create_all()`, y creando las 18 tablas nuevas (`dining_tables, table_sessions, session_participants, customer_orders, order_items, order_item_options, carts, promotions, promotion_targets, promotion_combo_items, cash_registers, cash_shifts, payment_methods, payments, sales, sale_items, invoices, invoice_counters`) sin remover ningún índice único parcial (research.md §6: esta spec no siembra dos sesiones `active` para la misma mesa ni dos turnos `open` en la misma caja); `fixtures.py` no se modifica

- [X] T003 [P] En `table_sessions_fixtures.py`: añadir `make_dining_table(db, **kw)`, `make_table_session(db, table=None, **kw)`, `make_participant(db, table_session=None, **kw)` siguiendo el patrón `kw.setdefault(...)` + `db.add` + `db.flush()` de `fixtures.py`, con defaults inertes (`status='libre'`/`'active'`/`'open'` según corresponda, nombres derivados del `id`) (contracts/test-harness-api.md §Factories nuevas, data-model.md §1; depende de T002)

- [X] T004 En `table_sessions_fixtures.py`: añadir `make_customer_order(db, table_session, participant=None, **kw)` y `make_order_item(db, order, variant, **kw)` con el mismo patrón, cubriendo los estados relevantes para A-01 (`recibida`, `en_preparacion`, `cancelada`, `pagada`) y el `participant_id` nullable que ejercitan A-38 (RN-MESA-24) y `set_assignments` (contracts/test-harness-api.md, data-model.md §1; depende de T002)

- [X] T005 En `table_sessions_fixtures.py`: añadir `make_promotion(db, **kw)` forzando `kw.setdefault(start_time=None, end_time=None)` (research.md §5 — sin ventana horaria, siempre válida sin importar el reloj real), `make_promotion_target(db, promotion, **kw)`, `make_combo_item(db, promotion, variant, **kw)` con el mismo patrón (contracts/test-harness-api.md; depende de T002)

- [X] T006 En `table_sessions_fixtures.py`: añadir `make_cash_register(db, **kw)`, `make_cash_shift(db, register=None, **kw)` con `status='open'` por defecto, y `make_payment_method(db, **kw)` con `kw.setdefault(is_cash=True)` (evita el chequeo de "no_efectivo > total" de `build_sale` por defecto) (contracts/test-harness-api.md, data-model.md §1; depende de T002)

- [X] T007 [P] En `table_sessions_fixtures.py`: añadir `make_tenant_double(*, id=1, invoice_prefix="")` y `make_user_double(*, id=None, name="Cajero de prueba")` como `SimpleNamespace`, sin persistir ningún modelo `Tenant`/`User` real (research.md §1, contracts/test-harness-api.md §Dobles de prueba; depende de T002)

- [X] T008 En `table_sessions_fixtures.py`: añadir la clase `spy_load` — context manager que parchea `app.api.v1.table_sessions.service._load` con `unittest.mock.patch(..., wraps=service._load)` y expone `.calls` como lista de `(table_session_id, lock)` por cada invocación durante el bloque `with` (research.md §2, contracts/test-harness-api.md §Espía de `_load`; depende de T002)

- [X] T009 En `table_sessions_fixtures.py`: añadir la clase `spy_bill_changed` — context manager que parchea `app.core.events.bill_changed` con `unittest.mock.patch(..., side_effect=_spy)`, registrando en `.calls` cada `(tenant_id, table_session_id)` en el orden en que ocurrieron (research.md §3, contracts/test-harness-api.md §Doble de `events.bill_changed`; depende de T002)

- [X] T010 Test de humo del fixture (dentro de `table_sessions_fixtures.py` o en un bloque `if __name__ == "__main__":` temporal, según se prefiera al implementar): abrir `new_session()`, ejercitar `make_dining_table` → `make_table_session` → `make_participant` → `make_customer_order` → `make_order_item` → `make_cash_register` → `make_cash_shift` → `make_payment_method`, y confirmar que `sale_items.options` (JSONB→JSON en SQLite) acepta un `dict` insertando una `Sale`/`SaleItem` mínima a mano; confirmar además que `spy_load` y `spy_bill_changed` interceptan sin error alrededor de una llamada trivial a `service.get_session` (research.md §2-4; depende de T003, T004, T005, T006, T007, T008, T009)

**Checkpoint**: `table_sessions_fixtures.py` crea las 18 tablas nuevas, sus factories, los dobles de
router, el espía de `_load` y el doble de `bill_changed`, verificado por el test de humo — las
Historias 1, 2 y 3 pueden empezar

---

## Phase 3: User Story 1 - Congelar el blindaje de split de A-15, con el mayor número de casos (Priority: P1) 🎯 MVP

**Goal**: characterization tests que ejercitan, uno por uno, los cuatro huecos de seguridad que
A-15 [PROTEGIDA] cerró en `_close_split` (comensal repetido, importes de raíz ignorados en modo
split, bloque sin comensal sin nombre, cobertura exacta comensal-consumo ↔ comensal-split), más el
camino feliz — de las cinco anomalías de esta spec, la que más casos concentra (FR-005, SC-002).

**Independent Test**: `python -m unittest app.characterization_tests.test_table_sessions_split_blindaje -v`
pasa en verde sin que existan aún los otros dos ficheros de test ni ningún cambio de CI (spec.md,
Historia 1).

### Implementation for User Story 1

- [X] T011 [US1] En `app/characterization_tests/test_table_sessions_split_blindaje.py` (nuevo fichero, docstring "CONGELA comportamiento actual:" per `__init__.py:1-39`, citando A-15 [PROTEGIDA] en el docstring del módulo): test — `CloseSessionIn` con `billing_mode='split'` y dos bloques de `splits` que comparten el mismo `participant_id` → `close_session` responde 422 citando los comensales repetidos, sin crear ninguna venta (spec.md Historia 1 Acceptance Scenario 1, FR-005)

- [X] T012 [US1] Test — `CloseSessionIn` con `billing_mode='split'` y `discount`, `tax`, `tip` o `payments` puestos en la raíz del payload (no dentro de un bloque de `splits`) → `close_session` responde 422 en cada caso, en vez de aceptar el importe y perderlo en silencio (spec.md Historia 1 Acceptance Scenario 2, FR-005)

- [X] T013 [US1] Test — sesión con un bloque de `splits` sin `participant_id` → al cerrar en `billing_mode='split'`, la venta resultante tiene `customer_name == "Mesa {number}"`, nunca un nombre vacío (spec.md Historia 1 Acceptance Scenario 3, FR-005)

- [X] T014 [US1] Test — sesión con comensales con consumo real cuyo `data.splits` recibido no cubre exactamente ese conjunto (falta uno, o sobra uno sin consumo) → `close_session` responde 422 citando específicamente los comensales que faltan o sobran, sin cobrar nada (spec.md Historia 1 Acceptance Scenario 4, FR-005)

- [X] T015 [US1] Test — split válido que cubre exactamente a los comensales con consumo, sin repetidos y sin importes en la raíz → se genera una venta por comensal (cada una con su propio `customer_name`) y la sesión queda `closed` (spec.md Historia 1 Acceptance Scenario 5)

- [X] T016 [US1] Ejecutar `python -m unittest app.characterization_tests.test_table_sessions_split_blindaje -v` desde `../pos-backend` y confirmar que las 5 pruebas (T011-T015) pasan en verde, con al menos 4 de ellas citando A-15 explícitamente en el nombre/docstring — una por cada hueco de seguridad cerrado (SC-002; depende de T011-T015)

**Checkpoint**: el invariante [PROTEGIDA] de mayor prioridad del módulo está congelado y
verificable de forma independiente — Historia 2 puede empezar

---

## Phase 4: User Story 2 - Congelar las 9 funciones públicas de `table_sessions/service.py` (Priority: P1)

**Goal**: un characterization test por cada una de las 9 funciones públicas de `service.py`
(`get_session`, `has_billable_orders`, `try_release_if_empty`, `list_sessions`, `compute_bill`,
`close_session`, `add_participant`, `remove_participant`, `set_assignments`), incluyendo A-01
(caso base), A-17 (R12), A-29 y A-38 (RN-MESA-13, RN-MESA-24).

**Independent Test**: `python -m unittest app.characterization_tests.test_table_sessions_service -v`
pasa en verde, reutilizando `table_sessions_fixtures.py` (spec.md, Historia 2).

### Implementation for User Story 2

- [X] T017 [US2] En `app/characterization_tests/test_table_sessions_service.py` (nuevo fichero, docstring "CONGELA comportamiento actual:"): test de `get_session` — sesión existente devuelve el `TableSession` esperado; id inexistente propaga `HTTPException` 404 (`service.py:38-59`) (FR-002)

- [X] T018 [US2] Test de `has_billable_orders` — sesión con un pedido `recibida`/`en_preparacion` devuelve `True`; sesión con solo pedidos `cancelada`/`pagada` devuelve `False` (`service.py:64-71`) (FR-002)

- [X] T019 [US2] Test de `try_release_if_empty` — sesión sin comensales activos y sin pedidos cobrables libera la mesa (`status='libre'`) y cierra la sesión; la misma sesión con un pedido todavía cobrable no libera nada (spec.md Historia 2 Acceptance Scenario 6, FR-002)

- [X] T020 [US2] Test de `list_sessions` — con `only_active=True` (explícito, nunca el default `Query`, research.md §1) solo devuelve sesiones `active`; con `only_active=False` incluye también sesiones `closed` (`service.py`, FR-002)

- [X] T021 [US2] Test de `compute_bill` — A-01 caso base: sesión con pedidos en `recibida`, `en_preparacion`, `cancelada`, `pagada` repartidos entre dos comensales, con una promoción y un combo activos (vía `make_promotion`/`make_promotion_target`/`make_combo_item`) → el `total` y el desglose por comensal excluyen `cancelada`/`pagada` y aplican el descuento por comensal, citando A-01 (camino A, correcto) en el docstring (spec.md Historia 2 Acceptance Scenario 1, FR-004)

- [X] T022 [US2] Test de `close_session` en `billing_mode='unified'` — camino feliz: una sola venta agrupa todo lo cobrable de la sesión, la sesión queda `closed` y la mesa se libera (`service.py:_close_unified`) (FR-002)

- [X] T023 [US2] Test — A-17 (R12) vía `spy_load` (T008): invocar `add_participant`, `remove_participant` y `set_assignments` sobre la misma sesión y confirmar en `spy.calls` que las tres cargan con `lock=False` (el default, ni siquiera pasado explícito hoy), a diferencia de `close_session` (invocado también dentro de este test o del de T022) que carga con `lock=True`; documentar el escenario de "concurrencia simulada" sembrando el estado directamente, sin hilos reales, citando A-17 (R12) en el docstring (spec.md Historia 2 Acceptance Scenario 2, FR-006)

- [X] T024 [US2] Test — A-29: sesión cuyas líneas cobradas usan dos combos distintos (o ninguno) → al llamar `compute_bill` o cerrar la sesión en cualquiera de los dos `billing_mode`, `promotion_id` no registra ningún combo aunque el descuento monetario se sume correctamente, citando A-29 en el docstring (spec.md Historia 2 Acceptance Scenario 3, FR-007)

- [X] T025 [US2] Test — RN-MESA-13 (A-38): mesa con un único comensal y consumo propio, cerrada con `billing_mode='split'` con un solo bloque para ese comensal → el cierre se acepta sin ninguna restricción de mínimo de comensales, citando RN-MESA-13/A-38 en el docstring (spec.md Historia 2 Acceptance Scenario 4, FR-008)

- [X] T026 [US2] Test — RN-MESA-24 (A-38): comensal con al menos un producto asignado (incluso si ese ítem está `anulado` o su pedido ya no es cobrable) → `remove_participant` responde 409 y no lo quita, citando RN-MESA-24/A-38 en el docstring (spec.md Historia 2 Acceptance Scenario 5, FR-008)

- [X] T027 [US2] Test — evento `bill_changed` (FR-010a) vía `spy_bill_changed` (T009): al llamar `add_participant`, `remove_participant` y `set_assignments` con éxito, confirmar que `app.core.events.bill_changed` se invoca exactamente una vez por llamada, con el `tenant_id` y `table_session_id` correctos, después de que la transacción del `service` ya hizo commit (spec.md Historia 2 Acceptance Scenario 7, FR-010a)

- [X] T028 [US2] Test de `add_participant` — camino feliz (`display_name` desambiguado vía `_unique_label`/`unique_display_label` de `cart.service`, ya congelado por `specs/015-caracterizacion-cart/`) (FR-002)

- [X] T029 [US2] Ejecutar `python -m unittest app.characterization_tests.test_table_sessions_service -v` desde `../pos-backend` y confirmar que las pruebas (T017-T028) pasan en verde, cubriendo las 9 funciones públicas (SC-001) y citando A-01, A-17 (R12), A-29 y A-38 (RN-MESA-13, RN-MESA-24) cada una en al menos un test (SC-002; depende de T017-T028)

**Checkpoint**: las 9 funciones públicas de `service.py` están congeladas y verificables de forma
independiente — Historia 3 puede empezar

---

## Phase 5: User Story 3 - Congelar los 7 endpoints de `table_sessions/router.py` (Priority: P2)

**Goal**: un characterization test por cada uno de los 7 endpoints de `router.py`, congelando lo
que la capa de router añade sobre `service.py` ya congelado en las Historias 1 y 2: códigos de
estado, forma de la respuesta y mapeo de errores.

**Independent Test**: `python -m unittest app.characterization_tests.test_table_sessions_router -v`
pasa en verde, reutilizando `table_sessions_fixtures.py` para construir los dobles de `Tenant`/
`User` (spec.md, Historia 3).

**Prerequisito**: Historias 1 y 2 en verde — el router delega toda la lógica de negocio en
`service.py`, ya congelado por ellas.

### Implementation for User Story 3

- [X] T030 [US3] En `app/characterization_tests/test_table_sessions_router.py` (nuevo fichero, docstring "CONGELA comportamiento actual:"): test de `GET /table-sessions` (`list_sessions`) — invocado directamente como función Python con `only_active=` explícito (nunca el default `Query(True, ...)` sin resolver, research.md §1) y `db`/`_` (`User` doble de T007), devuelve la lista esperada (FR-003)

- [X] T031 [US3] Test de `GET /table-sessions/{table_session_id}` (`get_session`) — sesión existente responde con `TableSessionResponse` incluyendo sus comensales; id inexistente propaga la misma `HTTPException` 404 que congeló T017 (spec.md Historia 3 Acceptance Scenario 1, FR-003)

- [X] T032 [US3] Test de `POST /table-sessions/{table_session_id}/participants` (`add_participant`) — `display_name` vacío o solo espacios responde 422 sin crear el comensal; nombre válido responde 201 con `ParticipantResponse.display_label` desambiguado (spec.md Historia 3 Acceptance Scenario 2, FR-003)

- [X] T033 [US3] Test de `DELETE /table-sessions/{table_session_id}/participants/{participant_id}` (`remove_participant`) — comensal con productos asignados responde 409 con el detalle de cuántos productos tiene asignados, congelando el contrato de error del endpoint (spec.md Historia 3 Acceptance Scenario 3, FR-003)

- [X] T034 [US3] Test de `PUT /table-sessions/{table_session_id}/assignments` (`set_assignments`) — lote de asignaciones válido responde 200 con `SessionBillResponse` ya recalculada, sin exigir una segunda llamada a `GET .../bill` (spec.md Historia 3 Acceptance Scenario 4, FR-003)

- [X] T035 [US3] Test de `GET /table-sessions/{table_session_id}/bill` (`session_bill`) — delega en `service.compute_bill` y responde igual que la función ya congelada en T021 (FR-003)

- [X] T036 [US3] Test de `POST /table-sessions/{table_session_id}/close` (`close_session`) — `CloseSessionIn` válido para `billing_mode='unified'` responde 200 con `CloseSessionResponse` (`table_session` ya `closed`, `sale_ids` con exactamente una venta) (spec.md Historia 3 Acceptance Scenario 5, FR-003)

- [X] T037 [US3] Ejecutar `python -m unittest app.characterization_tests.test_table_sessions_router -v` desde `../pos-backend` y confirmar que las 7 pruebas (T030-T036) pasan en verde, cubriendo los 7 endpoints (SC-001; depende de T030-T036)

**Checkpoint**: los 7 endpoints de `router.py` están congelados y verificables de forma
independiente — Historia 4 puede empezar

---

## Phase 6: User Story 4 - La suite corre en CI de forma determinista (Priority: P3)

**Goal**: verificar que las tres historias anteriores quedan cubiertas por el mismo paso de CI del
backend que ya ejecuta la red de `cart` (`.github/workflows/deploy.yml`), sin necesitar ningún
cambio al workflow — a diferencia de `specs/015-caracterizacion-cart/`, que sí tuvo que instalarlo
por primera vez, esta spec solo confirma que el paso `python -m unittest discover
-s app/characterization_tests -p 'test_*.py'` ya instalado descubre los tres ficheros nuevos por
convención de nombre.

**Independent Test**: abrir un PR trivial contra esta rama y confirmar en la ejecución de CI que el
paso existente recoge los ficheros nuevos, sin que `.github/workflows/deploy.yml` tenga ningún
diff (spec.md, Historia 4).

**Prerequisito**: Historias 1, 2 y 3 en verde — sin ellas no hay nada nuevo que el paso de CI
recoja.

### Implementation for User Story 4

- [X] T038 [US4] Inspeccionar `.github/workflows/deploy.yml` y confirmar que el paso `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` (instalado por `specs/015-caracterizacion-cart/`) cubre por convención de nombre los tres ficheros nuevos de esta spec, sin necesitar ningún paso adicional ni reemplazar el existente (spec.md Historia 4 Acceptance Scenario 1, FR-011)

- [X] T039 [US4] Verificar localmente los dos comandos que CI ejecutará (quickstart.md Escenario 6): `python -m app.scripts.test_promotions_rules` y `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` desde `../pos-backend`, confirmando ambos en verde e incluyendo explícitamente los tres ficheros nuevos en la salida verbose (SC-004; depende de T016, T029, T037)

- [X] T040 [US4] Verificar determinismo (SC-003, quickstart.md Escenario 4): correr `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` tres veces seguidas sin ningún cambio de código, redirigiendo cada salida a un log, y confirmar que las tres son idénticas (`diff`) (depende de T039)

- [X] T041 [US4] Verificar que la red detecta un cambio real (spec.md Historia 4 Acceptance Scenario 3, quickstart.md Escenario 5): modificar temporalmente una línea de `app/api/v1/table_sessions/service.py` (por ejemplo, comentar el chequeo de comensales repetidos en `_close_split` o invertir una condición de `_assert_closable`), confirmar con `git diff` que el cambio es visible, correr `test_table_sessions_split_blindaje.py` y confirmar que al menos un test falla en rojo, luego revertir con `git checkout -- app/api/v1/table_sessions/service.py` antes de continuar (FR-012; depende de T040)

**Checkpoint**: las cuatro historias están completas — la suite corre en el mismo paso de CI ya
existente, es determinista y detecta cambios reales, sin ningún diff en `deploy.yml`

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de que la spec no tocó producción y cerró el alcance

- [X] T042 [P] Confirmar diff vacío en `app/api/v1/table_sessions/service.py` y `app/api/v1/table_sessions/router.py` respecto a su estado inmediatamente anterior a esta spec (`git diff --stat -- app/api/v1/table_sessions/service.py app/api/v1/table_sessions/router.py` sin salida) (SC-005)

- [X] T043 [P] Confirmar cero dependencias nuevas en `requirements.txt` (`git diff --stat -- requirements.txt` sin salida) (SC-006)

- [X] T044 Recorrer [quickstart.md](./quickstart.md) de punta a punta (Escenarios 1 → 6) como pase de sanidad final sobre el estado ya implementado del repositorio (depende de T016, T029, T037, T041)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA Historias 1, 2 y 3
- **User Story 1 (Phase 3)**: depende de Foundational; no depende de Historia 2 ni 3
- **User Story 2 (Phase 4)**: depende de Foundational; reutiliza los fixtures de la misma fase que Historia 1, pero es independiente de ella (ambas son P1 porque ambas cubren `service.py`, se numeran distinto solo por la prioridad de negocio de A-15)
- **User Story 3 (Phase 5)**: depende de que Historias 1 y 2 estén completas (el router delega toda la lógica en `service.py` ya congelado, spec.md lo declara explícitamente)
- **User Story 4 (Phase 6)**: depende de que Historias 1, 2 y 3 estén en verde (prerequisito explícito de spec.md)
- **Polish (Phase 7)**: depende de que las cuatro historias estén completas

### User Story Dependencies

- **User Story 1 (P1)**: depende solo de Foundational — es el punto de partida, junto con US2
- **User Story 2 (P1)**: depende solo de Foundational — independiente de US1 (ficheros distintos, `test_table_sessions_split_blindaje.py` vs `test_table_sessions_service.py`)
- **User Story 3 (P2)**: depende de User Story 1 y User Story 2 completas
- **User Story 4 (P3)**: depende de User Story 1, 2 y 3 en verde

### Parallel Opportunities

- Dentro de Foundational, T003 y T007 tocan el mismo fichero (`table_sessions_fixtures.py`) que T002/T004/T005/T006/T008/T009 pero sobre secciones independientes (factories de mesa/sesión/comensal vs. dobles de Tenant/User) — marcados [P] porque no dependen entre sí más allá de T002; en la práctica, como comparten fichero, conviene escribirlos en secuencia salvo que se trabaje en ramas distintas
- Una vez completada Foundational, **Historia 1 e Historia 2 pueden avanzar en paralelo** (ficheros de test distintos, sin dependencia entre sí) — a diferencia de `specs/015-caracterizacion-cart/`, donde Historia 2 dependía de Historia 1
- Dentro de Historia 1, T011-T015 tocan el mismo fichero (`test_table_sessions_split_blindaje.py`) y se ejecutan en secuencia
- Dentro de Historia 2, T017-T028 tocan el mismo fichero (`test_table_sessions_service.py`) y se ejecutan en secuencia
- Dentro de Historia 3, T030-T036 tocan el mismo fichero (`test_table_sessions_router.py`) y se ejecutan en secuencia
- T042 y T043 son de solo lectura/verificación sobre ficheros distintos y pueden ejecutarse en paralelo

---

## Parallel Example: tras completar Foundational

```bash
# Historia 1 e Historia 2 pueden avanzar en paralelo (ficheros de test distintos):
Task: "Escribir test_table_sessions_split_blindaje.py (T011-T015)"
Task: "Escribir test_table_sessions_service.py (T017-T028)"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (fixture completo + factories + dobles + espías, verificado por el test de humo T010)
3. Completar Phase 3: User Story 1
4. **PARAR y VALIDAR**: `test_table_sessions_split_blindaje.py` en verde, con ≥4 casos citando A-15 explícitamente
5. En este punto existe línea base para el invariante [PROTEGIDA] de mayor prioridad, aunque `service.py` no esté cubierto en su totalidad todavía — seguro parar aquí si hace falta

### Incremental Delivery

1. Setup + Foundational → fixture completo listo
2. Historia 1 → A-15 [PROTEGIDA] congelada (MVP técnico del invariante de mayor prioridad)
3. Historia 2 → las 9 funciones públicas de `service.py` congeladas (línea base completa de `service.py`, junto con Historia 1)
4. Historia 3 → `router.py` congelado, reutilizando la línea base de `service.py`
5. Historia 4 → la suite corre en el mismo paso de CI ya existente, sin ningún cambio a `deploy.yml`, de forma determinista
6. Cada historia añade una señal de verificación sin romper la anterior — Historia 1 e Historia 2 pueden entregarse en cualquier orden entre sí (ambas P1), pero Historia 3 exige ambas completas y Historia 4 exige las tres

### Parallel Team Strategy

Con más de una persona disponible:

1. El equipo completa Setup + Foundational junto (bloqueante para todo lo demás)
2. Una vez completa Foundational:
   - Persona A: Historia 1 (`test_table_sessions_split_blindaje.py`)
   - Persona B: Historia 2 (`test_table_sessions_service.py`)
3. Historia 3 empieza solo cuando ambas terminan (depende de las dos)
4. Historia 4 cierra la spec una vez las tres anteriores están en verde

---

## Notes

- [P] = ficheros distintos o secciones independientes, sin dependencias pendientes entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- Ningún test de esta suite debe requerir modificar `table_sessions/service.py` ni `router.py` para pasar (FR-012): si una tarea de test falla contra el código actual sin modificar, el defecto está en el test, no en producción — corregir el test, nunca el código
- Esta spec no autoriza corregir A-01, A-15, A-17 (R12), A-29 ni A-38 (RN-MESA-13, RN-MESA-24): cada test documenta el comportamiento observado, no el deseado (FR-013)
- A diferencia de `specs/015-caracterizacion-cart/`, esta spec no toca `.github/workflows/deploy.yml` — Historia 4 es puramente verificación (FR-011 ya satisfecho desde `specs/015-caracterizacion-cart/`)
- Commitear tras cada tarea o grupo lógico
- Parar en cada checkpoint para validar la historia de forma independiente antes de seguir
