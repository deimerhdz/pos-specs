---

description: "Task list for feature implementation"
---

# Tasks: Red de characterization tests para `orders` (`service.py`, `checkout.py`, `consolidation.py`, `kitchen.py`, `tables_advanced.py`)

**Input**: Design documents from `/specs/017-caracterizacion-orders/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/test-harness-api.md](./contracts/test-harness-api.md), [quickstart.md](./quickstart.md)

**Path convention**: todas las rutas de código son relativas a la raíz del repositorio sibling
`../pos-backend` (ejecutado desde `pos-specs/`), tal como establece `plan.md` §Project Structure.
Las rutas bajo `specs/` o `.specify/` son relativas a este repositorio (`pos-specs`).

**Tests**: esta spec es, en sí misma, la escritura de una red de characterization tests (Principio
II) — no hay código de producción nuevo que probar con TDD. Cada tarea de Historia 1-5 crea
directamente los tests que congelan `orders/{service,checkout,consolidation,kitchen,
tables_advanced}.py`; no hay una fase separada de "tests antes de implementación" porque no hay
implementación de producción en esta spec (FR-015, FR-017).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias pendientes)
- **[Story]**: A qué historia de usuario pertenece (US1-US6)

---

## Phase 1: Setup

**Purpose**: preparar el entorno de `pos-backend` para escribir la red nueva

- [X] T001 Activar el entorno virtual de `pos-backend` (`cd ../pos-backend && source env/bin/activate`, o `pip install -r requirements.txt` en uno nuevo) y confirmar que `app/characterization_tests/orders_fixtures.py`, `app/characterization_tests/test_orders_consolidation.py`, `test_orders_checkout.py`, `test_orders_kitchen.py`, `test_orders_tables_advanced.py` y `test_orders_service.py` todavía no existen (`ls app/characterization_tests/`), sin colisión de nombre con `fixtures.py`/`cart_fixtures.py`/`table_sessions_fixtures.py`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: construir `app/characterization_tests/orders_fixtures.py` completo — esquema SQLite
ampliado (10 tablas de catálogo + 21 nuevas), factories nuevas, dobles de `Tenant`/`User` y el
helper de forzado de `IntegrityError` (A-26/RN-ORD-60) — ver
[contracts/test-harness-api.md](./contracts/test-harness-api.md) y
[data-model.md](./data-model.md). Módulo autónomo desde el principio: no importa de
`cart_fixtures.py` ni de `table_sessions_fixtures.py` (research.md §1).

**⚠️ CRITICAL**: ninguna de las cinco historias de test puede escribirse sin este fixture completo

- [X] T002 En `app/characterization_tests/orders_fixtures.py` (nuevo fichero): implementar `new_session()` reexportando tal cual las factories de catálogo de `fixtures.py` (`make_category`, `make_product`, `make_variant`, `make_option_group`, `make_option`, `link_variant_group`, `make_recipe_item`, `make_inventory_item`, `make_unit`), registrando el compilador `@compiles(JSONB, "sqlite")` → `"JSON"` (idempotente si convive en el mismo proceso con `cart_fixtures.py`/`table_sessions_fixtures.py`, research.md §1) antes de `create_all()`, aplicando el mismo parche de `server_default` de `sale_items.options` que ya documentó `table_sessions_fixtures.py` (`_patch_sqlite_incompatible_server_defaults`), y creando las 21 tablas nuevas (`dining_tables, table_sessions, session_participants, customer_orders, order_items, order_item_options, order_item_void_logs, order_cancel_logs, carts, cart_items, cart_item_options, promotions, promotion_targets, promotion_combo_items, cash_registers, cash_shifts, payment_methods, payments, sales, sale_items, invoices, invoice_counters`) sin remover ningún índice único parcial (`idx_active_session_per_table`, `idx_open_cart_per_participant`, `idx_open_shift_per_register`, data-model.md); `fixtures.py` no se modifica

- [X] T003 [P] En `orders_fixtures.py`: añadir `make_dining_table(db, **kw)`, `make_table_session(db, table=None, **kw)`, `make_participant(db, table_session=None, **kw)` siguiendo el patrón `kw.setdefault(...)` + `db.add` + `db.flush()`, con defaults inertes (`status='libre'`/`'active'`/`'open'` según corresponda) (contracts/test-harness-api.md §Factories nuevas; depende de T002)

- [X] T004 En `orders_fixtures.py`: añadir `make_customer_order(db, table_session, participant=None, **kw)` con `kw.setdefault(status="abierta")` por defecto (contracts/test-harness-api.md) y `make_order_item(db, order, variant, **kw)` con `kw.setdefault(estado_cocina="listo")` por defecto, cubriendo los estados relevantes de A-01/A-16/Historia 2 (`recibida`, `bloqueada`, `pagada`, `cancelada`, `pendiente`, `en_preparacion`, `anulado`) (depende de T002)

- [X] T005 [P] En `orders_fixtures.py`: añadir `make_order_item_void_log(db, order_item, **kw) -> OrderItemVoidLog` — **nueva**, ni `cart_fixtures.py` ni `table_sessions_fixtures.py` la necesitaron — con `motivo="prueba"`, `user_id=<uuid4 nuevo>`, `user_name="Cajero de prueba"` por defecto (data-model.md, contracts/test-harness-api.md; depende de T002)

- [X] T006 En `orders_fixtures.py`: añadir `make_cart(db, participant=None, **kw)`, `make_cart_item(db, cart, variant, **kw)` para `consolidation.consolidate_table` (data-model.md; depende de T002)

- [X] T007 En `orders_fixtures.py`: añadir `make_promotion(db, **kw)` forzando `kw.setdefault(start_time=None, end_time=None)` (sin ventana horaria, siempre válida), `make_promotion_target(db, promotion, **kw)`, `make_combo_item(db, promotion, variant, **kw)` (contracts/test-harness-api.md; depende de T002)

- [X] T008 En `orders_fixtures.py`: añadir `make_cash_register(db, **kw)`, `make_cash_shift(db, register=None, **kw)` con `status='open'` por defecto, y `make_payment_method(db, **kw)` con `kw.setdefault(is_cash=True)` (evita el chequeo de "no_efectivo > total" de `build_sale` por defecto) (contracts/test-harness-api.md, research.md §4; depende de T002)

- [X] T009 [P] En `orders_fixtures.py`: añadir `make_tenant_double(*, id=1, invoice_prefix="")` y `make_user_double(*, id=None, name="Cajero de prueba")` como `SimpleNamespace`, sin persistir ningún modelo `Tenant`/`User` real (contracts/test-harness-api.md §Dobles; depende de T002)

- [X] T010 En `orders_fixtures.py`: añadir la clase `force_flush_integrity_error(db)` — context manager que envuelve `unittest.mock.patch.object(db, "flush", side_effect=IntegrityError("stmt", {}, Exception("...")))` de **instancia**, no de clase (research.md §2, contracts/test-harness-api.md §Forzado de `IntegrityError`), única para el manejador huérfano de `move_order` (A-26/RN-ORD-60); depende de T002

- [X] T011 Test de humo del fixture (dentro de `orders_fixtures.py` o en un bloque `if __name__ == "__main__":` temporal): abrir `new_session()`, ejercitar `make_dining_table` → `make_table_session` → `make_participant` → `make_customer_order` → `make_order_item` → `make_order_item_void_log` → `make_cart` → `make_cart_item` → `make_promotion`/`make_promotion_target`/`make_combo_item` → `make_cash_register` → `make_cash_shift` → `make_payment_method`, confirmar que `sale_items.options` (JSONB→JSON en SQLite) acepta un `dict` insertando una `Sale`/`SaleItem` mínima a mano, y confirmar que `force_flush_integrity_error` intercepta `db.flush()` dentro del bloque `with` y lo restaura fuera de él (depende de T003-T010)

**Checkpoint**: `orders_fixtures.py` crea las 21 tablas nuevas, sus factories, los dobles y el
helper de forzado de excepción, verificado por el test de humo — las Historias 1-5 pueden empezar

---

## Phase 3: User Story 1 - Congelar A-04 y las 5 funciones públicas de `consolidation.py` (Priority: P1) 🎯 MVP

**Goal**: characterization tests que ejercitan `active_table_session_id`,
`get_or_create_table_session_id`, `get_or_create_open_order`, `consolidate_table` y
`add_item_to_table`, documentando con el mayor cuidado de la spec el defecto de A-04: que
`add_item_to_table` omite `variant` en `load_valid_options` y se salta la validación de selección
de opciones en el camino real del mesero, en contraste directo con `service.create_order`.

**Independent Test**: `python -m unittest app.characterization_tests.test_orders_consolidation -v`
pasa en verde sin que existan aún los otros cuatro ficheros de test (spec.md, Historia 1).

### Implementation for User Story 1

- [X] T012 [US1] En `app/characterization_tests/test_orders_consolidation.py` (nuevo fichero, docstring "CONGELA comportamiento actual:" per `__init__.py`, citando A-04 en el docstring del módulo): test — variante con grupo de opciones `min_select=1` obligatorio y ninguna opción seleccionada → `add_item_to_table` sin `combo_id` agrega el ítem igualmente sin error de validación, citando A-04 (`consolidation.py:199`) en el docstring (spec.md Historia 1 Acceptance Scenario 1, FR-003)

- [X] T013 [US1] Test de contraste — mismo escenario del T012 pero llamando a `service.create_order` en su lugar → la llamada sí falla con el error de validación de opciones, documentando el contraste exacto entre los dos caminos que motiva A-04 (spec.md Historia 1 Acceptance Scenario 2, FR-003; puede vivir en este fichero o en `test_orders_service.py`, T033 lo referencia)

- [X] T014 [US1] Test de `active_table_session_id` — mesa con sesión `active` devuelve su id; mesa sin sesión devuelve `None` (`consolidation.py:34-44`, FR-002)

- [X] T015 [US1] Test de `get_or_create_table_session_id` — mesa sin sesión abierta: al llamar, se crea la sesión de mesa nueva y su id se devuelve; mesa con sesión ya `active`: devuelve la existente sin duplicar (`consolidation.py:46-67`, FR-002)

- [X] T016 [US1] Test de `get_or_create_open_order` — mesa sin orden abierta crea una nueva vía `get_or_create_table_session_id`; mesa con orden `abierta` ya existente la reutiliza sin duplicar (`consolidation.py:69-104`, FR-002)

- [X] T017 [US1] Test — mesa sin sesión de mesa abierta ni orden abierta → `add_item_to_table` crea la sesión de mesa y la orden abierta necesarias y el ítem queda asociado a ellas, congelando la regla de "abrir sobre la marcha" tal como existe hoy (spec.md Historia 1 Acceptance Scenario 3, FR-002)

- [X] T018 [US1] Test de `consolidate_table` — orden ya abierta en la mesa con ítems previos, comensales con carritos → al llamar, consolida los ítems de los carritos en la orden existente, documentando el comportamiento observado de idempotencia (o su ausencia) al invocarlo dos veces seguidas con el mismo carrito (spec.md Historia 1 Acceptance Scenario 4, FR-002)

- [X] T019 [US1] Test — `combo_id` válido en `add_item_to_table` → el combo se expande en sus componentes reales a precio normal (sin el ahorro del combo, que se calcula al cobrar), congelando la regla documentada en el docstring de la función (spec.md Historia 1 Acceptance Scenario 5, FR-011)

- [X] T020 [US1] Test — variante sin receta asociada vía `add_item_to_table` → la guarda de `deduct_order_items` rechaza la creación, migrando el caso de `test_receta_obligatoria.py` correspondiente a este camino (research.md §5, FR-012, SC-007)

- [X] T021 [US1] Ejecutar `python -m unittest app.characterization_tests.test_orders_consolidation -v` desde `../pos-backend` y confirmar que las pruebas (T012, T014-T020) pasan en verde, cubriendo las 5 funciones públicas (SC-001) y citando A-04 explícitamente en al menos un test (SC-002; depende de T012, T014-T020)

**Checkpoint**: `consolidation.py` está congelado, con A-04 documentado con el mayor detalle de la
spec — Historia 2 puede empezar

---

## Phase 4: User Story 2 - Congelar las 10 funciones públicas de `checkout.py`, incluyendo la integración con `build_sale` (Priority: P1)

**Goal**: characterization tests que ejercitan `block_order`, `compute_bill`, `order_sale_lines`,
`promo_lines_for`, `pay_order`, `confirm_order`, `cancel_order`, `close_participants`,
`close_table_sessions`, `release_table`, incluyendo el camino real hacia
`sales.builder.build_sale`/`ensure_open_shift` en `pay_order` sin mocks, A-01 (camino B), A-29
(parcial) y A-38 (parcial: RN-ORD-31, RN-ORD-32).

**Independent Test**: `python -m unittest app.characterization_tests.test_orders_checkout -v` pasa
en verde, sembrando una orden, una sesión de mesa y un turno de caja abierto directamente, sin
depender de `cart` ni `table_sessions` (spec.md, Historia 2).

### Implementation for User Story 2

- [X] T022 [US2] En `app/characterization_tests/test_orders_checkout.py` (nuevo fichero, docstring "CONGELA comportamiento actual:"): test — orden `abierta` sin ítems pendientes en cocina → `block_order` con la versión correcta pasa a `bloqueada`; misma orden con al menos un ítem `pendiente`/`en_preparacion` → responde 409 con el detalle de los ítems sin terminar (`checkout.py:71-125`, spec.md Historia 2 Acceptance Scenario 1, FR-002)

- [X] T023 [US2] Test — tabla con órdenes en distintos status (`abierta`, `pagada`, `cancelada`) → `checkout.compute_bill` (camino B de A-01, sin caller de producción conocido) incluye las `pagada` en el total y no aplica ningún descuento, citando A-01 (camino B) en el docstring (`checkout.py:127-190`, spec.md Historia 2 Acceptance Scenario 8, FR-004)

- [X] T024 [US2] Test de `order_sale_lines` — camino feliz con producto y variante intactos produce `"Producto - Variante"`; ítem cuyo producto o variante fue borrado antes de cobrar → la descripción queda incompleta o vacía según cuál falte, citando RN-ORD-32/A-38 en el docstring (`checkout.py:191-230`, spec.md Historia 2 Acceptance Scenario 7, FR-009)

- [X] T025 [US2] Test de `promo_lines_for` — camino feliz con una promoción activa aplicable a las líneas dadas devuelve el descuento esperado; sin promoción aplicable devuelve la lista sin descuento (`checkout.py:231-249`, FR-002, FR-011)

- [X] T026 [US2] Test — orden `bloqueada` y turno de caja abierto, promoción activa y sin combos → `pay_order` construye el `Sale` vía `build_sale` real (sin mock), con el descuento de la promoción sumado y el turno de caja correcto, ejercitando `ensure_open_shift` (`checkout.py:250-306`, spec.md Historia 2 Acceptance Scenario 2, FR-010)

- [X] T027 [US2] Test — líneas cobradas que usan dos combos distintos → `pay_order` produce un `Sale` con `promotion_id=None` aunque el descuento monetario de ambos se sume correctamente, citando A-29 (parcial) en el docstring (`checkout.py:268-269`, spec.md Historia 2 Acceptance Scenario 3, FR-008)

- [X] T028 [US2] Test de `confirm_order` — pedido `recibida` con ítems válidos → pasa a `abierta` y descuenta inventario exactamente una vez (único punto de descuento del flujo QR); stock insuficiente de un insumo → la transacción entera revierte y el pedido sigue `recibida`, migrando el caso de `test_receta_obligatoria.py` correspondiente a este camino (`checkout.py:307-355`, spec.md Historia 2 Acceptance Scenario 4, FR-002, FR-012, SC-007)

- [X] T029 [US2] Test de `cancel_order` — orden con ítems en `pendiente`, `en_preparacion`, `listo`, `anulado` → solo los `pendiente` generan una entrada real de inventario; los `en_preparacion`/`listo` no vuelven al stock (se registran como pérdida en `audit_logs`), migrando el escenario de reversa de `test_cancel_inventory.py` verificado contra el código real (`checkout.py:357-464`, spec.md Historia 2 Acceptance Scenario 5, FR-002, research.md §5, SC-007)

- [X] T030 [US2] Test de `close_participants` — comensales activos de una sesión de mesa se cierran, devolviendo el conteo esperado (`checkout.py:466-493`, FR-002)

- [X] T031 [US2] Test — `close_table_sessions` no valida por sí mismo que no haya órdenes pendientes (delega esa responsabilidad al llamador), citando RN-ORD-31/A-38 en el docstring; incluye el escenario migrado de `test_session_ttl.py` que invoca `close_table_sessions`/`close_participants` bajo el disparador del barrido automático (`closed_by=None`, `scheduler.py:140`) (`checkout.py:495-527`, spec.md Historia 2 Acceptance Scenario 6, FR-009, research.md §5, SC-007)

- [X] T032 [US2] Test de `release_table` — mesa con órdenes activas sin cerrar → responde 409 con el detalle de las órdenes bloqueantes sin liberar la mesa; misma mesa sin órdenes activas → la mesa queda `libre` y sus sesiones `active` se cierran en cascada vía `close_table_sessions`/`close_participants` (`checkout.py:528+`, spec.md Historia 2 Acceptance Scenario 6, FR-002)

- [X] T033 [US2] En `test_orders_service.py` o `test_orders_consolidation.py` (según se decidió en T013): confirmar que el caso de contraste directo de A-04 contra `service.create_order` queda escrito y pasa en verde (referencia cruzada de T013, FR-003)

- [X] T034 [US2] Ejecutar `python -m unittest app.characterization_tests.test_orders_checkout -v` desde `../pos-backend` y confirmar que las pruebas (T022-T032) pasan en verde, cubriendo las 10 funciones públicas (SC-001) y citando A-01 (camino B), A-29 y A-38 (RN-ORD-31, RN-ORD-32) cada una en al menos un test (SC-002; depende de T022-T032)

**Checkpoint**: `checkout.py` está congelado en profundidad por primera vez, cerrando el hueco que
dejaron pendiente las specs 015 y 016 — Historia 3 puede empezar

---

## Phase 5: User Story 3 - Congelar las 3 funciones públicas de `kitchen.py`, incluyendo A-16 y A-25 (Priority: P2)

**Goal**: characterization tests que ejercitan `transition_kitchen`, `mark_order_ready` y
`void_item`, documentando que ninguna de las tres valida el `status` de la `CustomerOrder` padre
salvo la validación parcial de `mark_order_ready` (A-16), y que sus transiciones internas están
cerradas a una lista blanca sin vía genérica de asignación libre (A-25 [PROTEGIDA]).

**Independent Test**: `python -m unittest app.characterization_tests.test_orders_kitchen -v` pasa
en verde, sembrando una orden y sus ítems con distintos `estado_cocina` directamente (spec.md,
Historia 3).

### Implementation for User Story 3

- [X] T035 [US3] En `app/characterization_tests/test_orders_kitchen.py` (nuevo fichero, docstring "CONGELA comportamiento actual:"): test de `transition_kitchen` — ítem `pendiente` → transición directa hacia `listo` (el salto de un toque) se acepta; ítem `listo` → intento de retroceso hacia `pendiente` responde 409, congelando la lista blanca `_ALLOWED` tal cual, citando A-25 [PROTEGIDA] en el docstring (`kitchen.py:43-60`, spec.md Historia 3 Acceptance Scenario 1, FR-006)

- [X] T036 [US3] Test — orden `pagada` con ítems aún `pendiente` (estado producible sembrando datos directamente) → `transition_kitchen` y `void_item` se ejecutan igual, sin ningún error por el status de la orden padre, citando A-16 en el docstring (`kitchen.py:43-60,93-176`, spec.md Historia 3 Acceptance Scenario 2, FR-005)

- [X] T037 [US3] Test — misma orden `pagada` con ítems `pendiente` → `mark_order_ready` responde 409 citando que la orden ya es terminal, congelando el contraste con T036 que documenta A-16 (`kitchen.py:63-90`, spec.md Historia 3 Acceptance Scenario 3, FR-005)

- [X] T038 [US3] Test — orden `bloqueada` (no terminal de pago) con ítems `en_curso` → `mark_order_ready` pasa los ítems a `listo` sin ningún error, congelando que la validación bloquea solo `pagada`/`cancelada`, no `bloqueada` (la porción pendiente de A-16) (`kitchen.py:63-90`, spec.md Historia 3 Acceptance Scenario 4, FR-005)

- [X] T039 [US3] Test de `void_item` — ítem `pendiente` con `data.replacement` válido → el ítem original queda `anulado` (con reversa de inventario, por ser `pendiente`) y se crea uno nuevo `pendiente` con `void_de` apuntando al original y su propio descuento de inventario, congelando el ciclo completo de anular-y-reemplazar (`kitchen.py:93-176`, spec.md Historia 3 Acceptance Scenario 5, FR-002, FR-012)

- [X] T040 [US3] Test — inspección de las siete funciones públicas de los cinco ficheros que mutan algún estado (`block_order`, `confirm_order`, `pay_order`, `cancel_order`, `transition_kitchen`, `mark_order_ready`, `void_item`) → cada una impone su propia transición validada, ninguna acepta un `status`/`estado_cocina` arbitrario sin pasar por su propia guarda; citando A-25 [PROTEGIDA] como invariante de referencia, reutilizando los casos ya congelados en T022, T028, T026, T029, T035-T038 (spec.md Historia 3 Acceptance Scenario 6, FR-006)

- [X] T041 [US3] Ejecutar `python -m unittest app.characterization_tests.test_orders_kitchen -v` desde `../pos-backend` y confirmar que las pruebas (T035-T040) pasan en verde, cubriendo las 3 funciones públicas (SC-001) y citando A-16 y A-25 [PROTEGIDA] cada una en al menos un test (SC-002; depende de T035-T040)

**Checkpoint**: `kitchen.py` está congelado, con el invariante [PROTEGIDA] de A-25 documentado como
caso de referencia — Historia 4 puede empezar

---

## Phase 6: User Story 4 - Congelar las 4 funciones públicas de `tables_advanced.py`, incluyendo A-26 y A-01 (camino C) (Priority: P2)

**Goal**: characterization tests que ejercitan `set_table_status`, `move_order`, `merge_orders` y
`group_bill`, documentando los tres hallazgos de A-26 (estrictez de `move_order`, manejador
huérfano de `IntegrityError`, no-determinismo de `merge_orders`) y el camino C de A-01
(`group_bill` sin filtro de status ni descuentos, en uso real para mesas fusionadas).

**Independent Test**: `python -m unittest app.characterization_tests.test_orders_tables_advanced -v`
pasa en verde, sembrando órdenes en distintas mesas y grupos fusionados directamente (spec.md,
Historia 4).

### Implementation for User Story 4

- [X] T042 [US4] En `app/characterization_tests/test_orders_tables_advanced.py` (nuevo fichero, docstring "CONGELA comportamiento actual:"): test de `set_table_status` — mesa con al menos una orden activa → `new_status='libre'` o `'reservada'` responde 409; misma mesa sin órdenes activas → el cambio se acepta (`tables_advanced.py:30-43`, spec.md Historia 4 Acceptance Scenario 5, FR-002)

- [X] T043 [US4] Test — mesa destino con una orden activa ya presente → `move_order` hacia esa mesa responde 409, congelando que exige la mesa destino completamente libre de órdenes activas, más estricto que el modelo general de "varias órdenes por mesa", citando RN-ORD-58/A-26 en el docstring (`tables_advanced.py:45-73`, spec.md Historia 4 Acceptance Scenario 1, FR-007)

- [X] T044 [US4] Test — usando `orders_fixtures.force_flush_integrity_error(db)` (T010) alrededor de la llamada a `move_order`: se fuerza artificialmente `IntegrityError` en `db.flush()`, y el manejador `except IntegrityError` sigue presente y la traduce a 409, documentando explícitamente en el docstring que ya no hay ningún camino de datos reales que lo alcance (índice único retirado del modelo), citando RN-ORD-60/A-26 (`tables_advanced.py:56-63`, spec.md Historia 4 Acceptance Scenario 2, FR-007, research.md §2)

- [X] T045 [US4] Test — dos órdenes que ya pertenecen a dos `merged_group_id` distintos preexistentes (cada grupo con al menos otra orden ya asociada) → `merge_orders` invocado una sola vez con ambas produce un resultado `in {group_a, group_b}`, nunca fijando un valor específico, documentando explícitamente el no-determinismo (`SELECT` sin `ORDER BY`), citando RN-ORD-63/A-26 (`tables_advanced.py:75-89`, spec.md Historia 4 Acceptance Scenario 3, FR-007, research.md §3)

- [X] T046 [US4] Test de `group_bill` — grupo fusionado con órdenes en status `abierta`, `pagada` y `cancelada` → el total agregado incluye las tres sin excluir ninguna por status, y no aplica ningún descuento, citando A-01 (camino C) en el docstring (`tables_advanced.py:92-114`, spec.md Historia 4 Acceptance Scenario 4, FR-004)

- [X] T047 [US4] Ejecutar `python -m unittest app.characterization_tests.test_orders_tables_advanced -v` desde `../pos-backend` y confirmar que las pruebas (T042-T046) pasan en verde, cubriendo las 4 funciones públicas (SC-001) y citando A-01 (camino C) y los tres hallazgos de A-26 (RN-ORD-58, RN-ORD-60, RN-ORD-63) cada uno en al menos un test (SC-002; depende de T042-T046)

**Checkpoint**: `tables_advanced.py` está congelado, con los tres hallazgos de A-26 y el camino C
de A-01 documentados — Historia 5 puede empezar

---

## Phase 7: User Story 5 - Congelar `create_order`, la única función pública de `service.py` (Priority: P3)

**Goal**: characterization tests que ejercitan `create_order` — la comanda directa de
mostrador/mesero que nace ya `abierta` y descuenta inventario al crearse — documentando su
contraste explícito con A-04 (sí pasa `variant` a `load_valid_options`, a diferencia de
`add_item_to_table`).

**Independent Test**: `python -m unittest app.characterization_tests.test_orders_service -v` pasa
en verde (spec.md, Historia 5).

### Implementation for User Story 5

- [X] T048 [US5] En `app/characterization_tests/test_orders_service.py` (nuevo fichero, docstring "CONGELA comportamiento actual:"): test — variante con receta y opciones válidas → `create_order` nace en status `abierta` (no `recibida`) con el inventario ya descontado, congelando que este camino no vuelve a pasar por `confirm_order` (`service.py:37+`, spec.md Historia 5 Acceptance Scenario 1, FR-002)

- [X] T049 [US5] Test — variante sin receta asociada → la guarda de `deduct_order_items` rechaza la creación, migrando el caso de `test_receta_obligatoria.py` correspondiente a este camino, verificado contra el código real (spec.md Historia 5 Acceptance Scenario 2, FR-012, research.md §5, SC-007)

- [X] T050 [US5] Test — mesa sin sesión de mesa activa → `create_order` con `dining_table_id` crea la sesión de mesa vía `consolidation.get_or_create_table_session_id` antes de crear la orden, congelando la dependencia real entre `service.py` y `consolidation.py` (spec.md Historia 5 Acceptance Scenario 3, FR-002)

- [X] T051 [US5] Si no se escribió ya en T013: test de contraste directo — misma variante y grupo de opciones `min_select=1` sin selección de T012/T013 → `create_order` sí falla con el error de validación de opciones (a diferencia de `add_item_to_table`), citando A-04 en el docstring (spec.md Historia 1 Acceptance Scenario 2, FR-003)

- [X] T052 [US5] Ejecutar `python -m unittest app.characterization_tests.test_orders_service -v` desde `../pos-backend` y confirmar que las pruebas (T048-T051) pasan en verde, cubriendo la única función pública (SC-001) y el caso de contraste de A-04 (SC-002; depende de T048-T051)

**Checkpoint**: las cinco historias de caracterización están completas — las 23 funciones públicas
de los cinco ficheros de `orders` están congeladas — Historia 6 puede empezar

---

## Phase 8: User Story 6 - La suite corre en CI de forma determinista (Priority: P3)

**Goal**: verificar que las cinco historias anteriores quedan cubiertas por el mismo paso de CI del
backend que ya ejecuta las redes de `cart` y `table_sessions`
(`.github/workflows/deploy.yml`), sin necesitar ningún cambio al workflow — esta spec, igual que
`016-caracterizacion-table-sessions`, solo confirma que el paso `python -m unittest discover
-s app/characterization_tests -p 'test_*.py'` ya instalado descubre los cinco ficheros nuevos por
convención de nombre.

**Independent Test**: abrir un PR trivial contra esta rama y confirmar en la ejecución de CI que el
paso existente recoge los ficheros nuevos, sin que `.github/workflows/deploy.yml` tenga ningún
diff (spec.md, Historia 6).

**Prerequisito**: Historias 1-5 en verde — sin ellas no hay nada nuevo que el paso de CI recoja.

### Implementation for User Story 6

- [X] T053 [US6] Inspeccionar `.github/workflows/deploy.yml` y confirmar que el paso `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` (instalado por `specs/015-caracterizacion-cart/`) cubre por convención de nombre los cinco ficheros nuevos de esta spec, sin necesitar ningún paso adicional ni reemplazar el existente (spec.md Historia 6 Acceptance Scenario 1, FR-014)

- [X] T054 [US6] Verificar localmente el comando que CI ejecutará (quickstart.md Escenario 2): `python -m unittest discover -s app/characterization_tests -p 'test_orders_*.py' -v` desde `../pos-backend`, confirmando en verde e incluyendo explícitamente los cinco ficheros nuevos en la salida verbose (SC-004; depende de T021, T034, T041, T047, T052)

- [X] T055 [US6] Verificar determinismo (SC-003, quickstart.md Escenario 5): correr `python -m unittest discover -s app/characterization_tests -p 'test_orders_*.py'` tres veces seguidas sin ningún cambio de código, redirigiendo cada salida a un log, y confirmar que las tres son idénticas (`diff`), incluyendo que el test de `merge_orders` (T045) pasa las tres veces aunque el grupo ganador observado internamente pueda variar (depende de T054)

- [X] T056 [US6] Verificar que la red detecta un cambio real (spec.md Historia 6 Acceptance Scenario 3): modificar temporalmente una línea de uno de los cinco ficheros de `orders` (por ejemplo, invertir la condición de la lista blanca `_ALLOWED` en `kitchen.py` o comentar el chequeo de mesa destino libre en `tables_advanced.move_order`), confirmar con `git diff` que el cambio es visible, correr el fichero de test correspondiente y confirmar que al menos un test falla en rojo, luego revertir con `git checkout -- <fichero>` antes de continuar (FR-013; depende de T055)

**Checkpoint**: las seis historias están completas — la suite corre en el mismo paso de CI ya
existente, es determinista y detecta cambios reales, sin ningún diff en `deploy.yml`

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de que la spec no tocó producción y cerró el alcance

- [X] T057 [P] Confirmar diff vacío en `app/api/v1/orders/service.py`, `checkout.py`, `consolidation.py`, `kitchen.py` y `tables_advanced.py` respecto a su estado inmediatamente anterior a esta spec (`git diff --stat main -- app/api/v1/orders/service.py app/api/v1/orders/checkout.py app/api/v1/orders/consolidation.py app/api/v1/orders/kitchen.py app/api/v1/orders/tables_advanced.py` sin salida) (SC-005)

- [X] T058 [P] Confirmar cero dependencias nuevas en `requirements.txt` (`git diff --stat main -- requirements.txt` sin salida) (SC-006)

- [X] T059 [P] Confirmar que los tres scripts legado no corren ya como paso aparte (quickstart.md Escenario 8): `grep -rln "test_cancel_inventory\|test_receta_obligatoria\|test_session_ttl" .github/workflows/` sin coincidencias, y `grep -c "def test_" app/characterization_tests/test_orders_checkout.py app/characterization_tests/test_orders_consolidation.py app/characterization_tests/test_orders_service.py` con al menos un método por cada camino migrado (SC-007)

- [X] T060 Recorrer [quickstart.md](./quickstart.md) de punta a punta (Escenarios 1 → 8) como pase de sanidad final sobre el estado ya implementado del repositorio (depende de T056, T057, T058, T059)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA las seis historias
- **User Story 1 (Phase 3)**: depende de Foundational; no depende de Historias 2-5
- **User Story 2 (Phase 4)**: depende de Foundational; independiente de Historia 1 en fichero, aunque T033 referencia el caso de contraste de A-04 que puede vivir en Historia 1 o Historia 5
- **User Story 3 (Phase 5)**: depende de Foundational; reutiliza los casos ya congelados de Historia 2 en T040 (referencia, no bloqueo de escritura)
- **User Story 4 (Phase 6)**: depende de Foundational; independiente de Historias 1, 2 y 3
- **User Story 5 (Phase 7)**: depende de Foundational; su T051 referencia el mismo caso de contraste de A-04 que T013/T033
- **User Story 6 (Phase 8)**: depende de que Historias 1-5 estén en verde
- **Polish (Phase 9)**: depende de que las seis historias estén completas

### User Story Dependencies

- **User Story 1 (P1)**: depende solo de Foundational — punto de partida, junto con US2
- **User Story 2 (P1)**: depende solo de Foundational — independiente de US1 en fichero (`test_orders_checkout.py` vs `test_orders_consolidation.py`); comparte con US1 el caso de contraste de A-04 (T013/T033), que solo necesita escribirse una vez
- **User Story 3 (P2)**: depende solo de Foundational — funcionalmente independiente de cocina respecto a US1/US2, pero T040 reutiliza casos ya escritos en US2 como referencia
- **User Story 4 (P2)**: depende solo de Foundational — independiente de US1, US2, US3
- **User Story 5 (P3)**: depende solo de Foundational — su caso de contraste de A-04 puede reutilizar T013 en vez de duplicarlo (T051 es condicional)
- **User Story 6 (P3)**: depende de que US1-US5 estén en verde

### Parallel Opportunities

- Dentro de Foundational, T003, T005 y T009 tocan el mismo fichero (`orders_fixtures.py`) que T002/T004/T006/T007/T008/T010 pero sobre secciones independientes — marcados [P] porque no dependen entre sí más allá de T002; en la práctica, como comparten fichero, conviene escribirlos en secuencia salvo que se trabaje en ramas distintas
- Una vez completada Foundational, **Historias 1, 2, 4 y 5 pueden avanzar en paralelo** (ficheros de test distintos, sin dependencia entre sí salvo el caso de contraste de A-04 compartido entre US1/US2/US5, que basta escribir una sola vez); Historia 3 puede avanzar en paralelo también, aunque su T040 referencia casos de US2 ya congelados
- Dentro de cada historia, las tareas de un mismo fichero de test se ejecutan en secuencia (mismo fichero: T012-T020, T022-T032, T035-T040, T042-T046, T048-T051)
- T057, T058 y T059 son de solo lectura/verificación sobre ficheros distintos y pueden ejecutarse en paralelo

---

## Parallel Example: tras completar Foundational

```bash
# Historias 1, 2, 4 y 5 pueden avanzar en paralelo (ficheros de test distintos):
Task: "Escribir test_orders_consolidation.py (T012-T020)"
Task: "Escribir test_orders_checkout.py (T022-T032)"
Task: "Escribir test_orders_tables_advanced.py (T042-T046)"
Task: "Escribir test_orders_service.py (T048-T050)"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (fixture completo + factories + dobles + helper de excepción, verificado por el test de humo T011)
3. Completar Phase 3: User Story 1
4. **PARAR y VALIDAR**: `test_orders_consolidation.py` en verde, con A-04 citado explícitamente y su caso de contraste contra `create_order` (T013)
5. En este punto existe línea base para el hallazgo central de la spec (A-04), aunque `checkout.py`, `kitchen.py`, `tables_advanced.py` y `service.py` no estén cubiertos todavía — seguro parar aquí si hace falta

### Incremental Delivery

1. Setup + Foundational → fixture completo listo
2. Historia 1 → A-04 congelada con el mayor detalle (MVP técnico del hallazgo central)
3. Historia 2 → `checkout.py` congelado en profundidad por primera vez, cerrando el hueco que dejaron pendiente `cart` y `table_sessions`
4. Historia 3 → `kitchen.py` congelado, A-16 y A-25 [PROTEGIDA] documentados
5. Historia 4 → `tables_advanced.py` congelado, los tres hallazgos de A-26 y el camino C de A-01 documentados
6. Historia 5 → `service.py` congelado, cerrando el contraste de A-04 desde el lado correcto
7. Historia 6 → la suite corre en el mismo paso de CI ya existente, sin ningún cambio a `deploy.yml`, de forma determinista
8. Cada historia añade una señal de verificación sin romper la anterior — Historias 1, 2, 4 y 5 pueden entregarse en cualquier orden entre sí, pero Historia 6 exige las cinco completas

### Parallel Team Strategy

Con más de una persona disponible:

1. El equipo completa Setup + Foundational junto (bloqueante para todo lo demás)
2. Una vez completa Foundational:
   - Persona A: Historia 1 (`test_orders_consolidation.py`) — coordina con Persona B el caso de contraste de A-04 para no duplicarlo
   - Persona B: Historia 2 (`test_orders_checkout.py`), el fichero más grande
   - Persona C: Historia 4 (`test_orders_tables_advanced.py`) y luego Historia 3 (`test_orders_kitchen.py`)
   - Persona D: Historia 5 (`test_orders_service.py`)
3. Historia 6 cierra la spec una vez las cinco anteriores terminan

---

## Notes

- [P] = ficheros distintos o secciones independientes, sin dependencias pendientes entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- Ningún test de esta suite debe requerir modificar `orders/service.py`, `checkout.py`, `consolidation.py`, `kitchen.py` ni `tables_advanced.py` para pasar (FR-015): si una tarea de test falla contra el código actual sin modificar, el defecto está en el test, no en producción — corregir el test, nunca el código
- Esta spec no autoriza corregir A-01, A-04, A-16, A-25, A-26, A-29 ni A-38: cada test documenta el comportamiento observado, no el deseado (FR-016), incluyendo el fix de una línea de A-04 (`variant=variant` en `consolidation.py:199`), que queda explícitamente fuera
- El caso de contraste de A-04 (T013/T033/T051) solo necesita escribirse una vez — decidir en qué fichero vive (`test_orders_consolidation.py` o `test_orders_service.py`) al implementar Historia 1, y las historias posteriores lo referencian sin duplicarlo
- El no-determinismo de `merge_orders` (T045) se congela con una aserción de conjunto (`in {group_a, group_b}`), nunca un valor fijo — es el comportamiento en sí lo que se congela (research.md §3)
- El forzado de `IntegrityError` en `move_order` (T044) usa `orders_fixtures.force_flush_integrity_error` (T010), el único mock que introduce esta spec — todo lo demás se ejercita contra código real, sin dobles
- A diferencia de `specs/015-caracterizacion-cart/`, esta spec no toca `.github/workflows/deploy.yml` — Historia 6 es puramente verificación (FR-014 ya satisfecho desde `specs/015-caracterizacion-cart/`)
- Commitear tras cada tarea o grupo lógico
- Parar en cada checkpoint para validar la historia de forma independiente antes de seguir
