---

description: "Task list template for feature implementation"
---

# Tasks: Eliminación de Carritos al Liberarse la Mesa

**Input**: Design documents from `/specs/039-eliminacion-carritos-cierre-mesa/`

**Prerequisites**: [plan.md](./plan.md) (required), [spec.md](./spec.md) (requerido para historias de
usuario), [research.md](./research.md), [data-model.md](./data-model.md),
[contracts/liberacion-mesa.md](./contracts/liberacion-mesa.md), [quickstart.md](./quickstart.md)

**Tests**: Esta spec modifica un comportamiento protegido por un characterization test `CONGELA`
(Principio III de la Constitución) y su plan/quickstart exigen explícitamente reescribirlo y agregar
tests nuevos por historia de usuario, incluyendo la primera cobertura `unittest` de `_sweep_schema`
(research.md Decisión 8) — **los tests no son opcionales en este feature**, están incluidos abajo
como tareas de implementación.

**Organization**: Las tareas están agrupadas por historia de usuario para permitir implementación y
prueba independiente de cada una. Todo el código de este feature vive en el repo sibling
`../pos-backend` (rutas relativas a la raíz de ese repo, ver plan.md §Project Structure). Sin cambios
en `../pos-heladeria` (contracts/liberacion-mesa.md).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivos distintos, sin dependencias entre sí)
- **[Story]**: A qué historia de usuario pertenece la tarea (US1, US2, US3, US4)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

- **Backend** (`pos-backend/`): `app/api/v1/orders/checkout.py`,
  `app/api/v1/table_sessions/service.py`, `app/core/scheduler.py`,
  `app/characterization_tests/*.py`
- **Documentación de gobierno** (`pos-specs/`): `specs/000-reconocimiento/registro-de-anomalias.md`

---

## Phase 1: Setup

**Purpose**: Cerrar el bloqueo administrativo obligatorio y confirmar la línea base antes de tocar
código (plan.md §Constitution Check Principio II; quickstart.md Paso 0)

- [X] T001 Agregar la entrada **A-54** a
  `pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` (última entrada hoy: A-53, spec
  038), siguiendo exactamente el mismo formato de esa entrada (título con
  `[DECISIÓN DE NEGOCIO — spec 039]`, y las secciones **Qué cambia**, **Por qué cambia**, **Quién
  tomó la decisión y cuándo** (propietario del repositorio, leonardogomez306@gmail.com, 2026-08-26,
  citando `specs/039-eliminacion-carritos-cierre-mesa/spec.md` §"Naturaleza de esta spec"),
  **Funcionalidades afectadas** (los 5 caminos de liberación de mesa y el único test `CONGELA`
  `test_leave_session_cierra_participante_abandona_carrito_y_libera_mesa`), **Clasificación**
  (DECISIÓN DE NEGOCIO) y **Tratamiento acordado** (implementar según
  `specs/039-eliminacion-carritos-cierre-mesa/plan.md`/`tasks.md`). **Bloqueante sin condicional**:
  según el Principio II de la Constitución, `/speckit-implement` no debe avanzar a la Fase 2 hasta
  que esta entrada exista (plan.md §Constitution Check, Principio II)

- [X] T002 Ejecutar la suite de characterization tests de `table_sessions`/`orders`/`cart` en
  `pos-backend` y confirmar que todo está en verde, incluido el único test `CONGELA` que esta spec
  va a modificar y los tres que **no** se tocan (research.md Decisión 5):
  `python -m unittest app.characterization_tests.test_cart_service -v`,
  `python -m unittest app.characterization_tests.test_table_sessions_service -v`,
  `python -m unittest app.characterization_tests.test_orders_checkout -v`
  (quickstart.md Paso 0)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Infraestructura de test compartida por varias historias de usuario — no hay
infraestructura de producción foundational-only más allá de lo que ya existe en el repo (sin
dependencias nuevas, sin módulos nuevos — Technical Context del plan). La función de producción
(`delete_orphan_carts`) queda etiquetada como parte de la Fase 3 (US1), que es donde se usa por
primera vez, igual que hizo `specs/038-vaciado-carrito-pedido/tasks.md` con su cambio de
comportamiento equivalente.

**⚠️ CRITICAL**: Ninguna historia puede agregar sus tests nuevos hasta que esta tarea esté completa.

- [X] T003 Agregar el helper `make_cart(db: Session, participant: SessionParticipant | None = None,
  **kw) -> Cart` a `pos-backend/app/characterization_tests/table_sessions_fixtures.py`, espejo
  directo de `cart_fixtures.make_cart` (`pos-backend/app/characterization_tests/cart_fixtures.py:248-
  258`): si `participant` es `None` llama a `make_participant(db)`; `kw.setdefault("id", _uid())`,
  `kw.setdefault("participant_id", participant.id)`, `kw.setdefault("status", "abierto")`;
  `db.add(obj)` + `db.flush()`. Requiere agregar `from app.models.cart import Cart` a los imports del
  módulo y `"make_cart"` a la lista `__all__` (líneas 48-59, agregar antes del corchete de cierre en
  la línea 59). No se modifica `cart_fixtures.py` ni se
  importa desde ahí (research.md Decisión 9) — los tests nuevos de US1, US2 (escenarios 1-2), US3
  (rollback) y US4 son su único consumidor

---

## Phase 3: User Story 1 - La mesa se libera sola y sus carritos desaparecen con ella (Priority: P1) 🎯 MVP

**Goal**: cuando el último comensal activo sale de la sesión (o su token expira, o cancela su último
pedido vivo) y no queda nada por cobrar, la mesa se libera automáticamente **y**, en la misma
operación, todos los `Cart` de los participantes de esa sesión —sin importar su `status`— se
eliminan físicamente de la base de datos.

**Independent Test**: abrir una sesión de mesa, agregar ítems a un carrito sin confirmar pedido,
hacer que el comensal salga siendo el último activo sin nada por cobrar, y verificar que (a) la mesa
queda `libre` y (b) no existe ninguna fila de `Cart` para los participantes de esa sesión.

**Depende de**: Phase 1 (Setup) y Phase 2 (Foundational). Introduce la función compartida
(`delete_orphan_carts`) de la que dependen US2 y US3; US4 la ejercita sin volver a tocarla.

### Implementation for User Story 1

- [X] T004 [US1] Implementar `delete_orphan_carts(db: Session, sessions: list[TableSession]) -> None`
  en `pos-backend/app/api/v1/orders/checkout.py`, junto a `close_participants`/`close_table_sessions`
  (líneas 633-663 hoy): si `sessions` está vacío, retorna sin hacer nada; si no, resuelve
  `SessionParticipant.id` cuyo `table_session_id` esté en `[s.id for s in sessions]` (sin filtrar por
  `status` del participante) y ejecuta una única sentencia
  `db.execute(delete(Cart).where(Cart.participant_id.in_(subquery_participant_ids)))` — `DELETE`
  masivo, no un `for` con `db.delete()` por fila (research.md Decisión 2). Requiere agregar `delete`
  al import ya existente `from sqlalchemy import select` (línea 14, pasa a
  `from sqlalchemy import delete, select`); `Cart` y `SessionParticipant` ya están importados
  (líneas 23-24). No hace `commit()` ni `flush()` propio — se une a la transacción del caller
  (research.md Decisión 3, FR-001, FR-002, FR-004)

- [X] T005 [US1] En `try_release_if_empty`
  (`pos-backend/app/api/v1/table_sessions/service.py:88-130`), capturar el `list[TableSession]` que
  ya devuelve `checkout.close_table_sessions(db, ts.dining_table_id, closed_by=None)` (línea 126,
  hoy sin capturar) en una variable `sessions`, y llamar `checkout.delete_orphan_carts(db, sessions)`
  justo después de `table.status = "libre"` (línea 129), dentro del mismo `if table is not None:`.
  Sin import nuevo (`checkout` ya está importado). Ningún `if`/`return` existente de la función se
  modifica (FR-005)

- [X] T006 [US1] Reescribir el test `CONGELA`
  `test_leave_session_cierra_participante_abandona_carrito_y_libera_mesa` en
  `pos-backend/app/characterization_tests/test_cart_service.py` (líneas 466-481): mantener el mismo
  nombre (research.md Decisión 6 — el paso intermedio que describe sigue ocurriendo), reemplazar
  `cart = db.get(Cart, resp.cart_id); self.assertEqual(cart.status, "abandonado")` por
  `self.assertIsNone(db.get(Cart, resp.cart_id))`, y citar esta spec (039) en el docstring en vez de
  (o junto a) la cita actual de comportamiento heredado (depende de T004, T005)

- [X] T007 [US1] Agregar un test junto a `test_try_release_if_empty_libera_y_no_libera`
  (`pos-backend/app/characterization_tests/test_table_sessions_service.py:126`): dos comensales de la
  misma sesión, uno con `fx.make_cart(db, participant=..., status="abandonado")` (simulando que salió
  antes) y otro con `fx.make_cart(db, participant=..., status="confirmado")` (simulando la
  consolidación del mesero) → el segundo sale siendo el último activo sin nada por cobrar
  (`service.try_release_if_empty`) → verificar `table.status == "libre"` y que **ninguno** de los dos
  `Cart` sigue existiendo (`db.get(Cart, ...) is None` para ambos) — confirma que el borrado no
  distingue por `status` (Acceptance Scenario 2 de US1, edge case de spec.md, depende de T003, T004,
  T005)

- [X] T008 [US1] Agregar un test junto al de T007, en el mismo fichero: una sesión con un único
  comensal activo y un `Cart` `'abierto'` con un pedido `CustomerOrder` vivo asociado; simular la
  cancelación de ese último pedido activo dejando la sesión sin nada por cobrar (vía
  `service.has_billable_orders` en `False`, igual que hace `cancel_my_order` antes de llamar
  `try_release_if_empty`) y llamar `service.try_release_if_empty(db, ts.id)` → verificar que la mesa
  se libera y que el `Cart` de ese comensal se elimina en la misma operación (Acceptance Scenario 3
  de US1, depende de T003, T004, T005)

**Checkpoint**: en este punto, User Story 1 es completamente funcional y verificable de forma
independiente — el camino de liberación más frecuente ya limpia sus carritos, sin importar cuántos
comensales ni en qué `status` estuvieran sus carritos.

---

## Phase 4: User Story 2 - La liberación manual de staff también limpia los carritos (Priority: P1)

**Goal**: cuando el cajero cobra y cierra la mesa, cuando staff libera una sesión ya pagada, cuando
usa "Liberar Mesa", o cuando el barrido automático encuentra una mesa vencida sin nada pendiente, la
mesa queda `libre` exactamente igual que hoy y, además, los carritos de esa sesión dejan de existir.

**Independent Test**: para cada uno de los tres caminos de staff/barrido, preparar una sesión con al
menos un carrito huérfano y verificar que, tras la liberación, la mesa queda `libre` y el carrito ya
no existe.

**Depende de**: Phase 3 (US1) — reutiliza `checkout.delete_orphan_carts` introducida ahí; no la
reimplementa.

### Implementation for User Story 2

- [X] T009 [US2] En `close_session`
  (`pos-backend/app/api/v1/table_sessions/service.py:244-298`), capturar el retorno de
  `checkout.close_table_sessions(db, ts.dining_table_id, closed_by=cashier)` (línea 284, hoy sin
  capturar) en `sessions`, y llamar `checkout.delete_orphan_carts(db, sessions)` justo después de
  `table.status = "libre"` (línea 288), antes del `db.commit()` de la línea 290 (FR-002). Ningún
  `if`/`return` de la validación de cobro (`_assert_closable`, el 409 de sesión no `active`) se
  modifica (FR-005)

- [X] T010 [US2] En `release_paid_session`
  (`pos-backend/app/api/v1/table_sessions/service.py:335-398`), capturar el retorno de
  `checkout.close_table_sessions(db, ts.dining_table_id, closed_by=cashier)` (línea 373, hoy sin
  capturar) en `sessions`, y llamar `checkout.delete_orphan_carts(db, sessions)` justo después de
  `table.status = "libre"` (línea 377), antes del `db.commit()` de la línea 379 (FR-002). El 409 por
  "algo por cobrar" (`has_billable_orders`) no cambia (FR-005)

- [X] T011 [P] [US2] En `release_table`
  (`pos-backend/app/api/v1/orders/checkout.py:697-736`), capturar el retorno de
  `close_table_sessions(db, table.id, closed_by=closed_by)` (línea 728, hoy sin capturar) en
  `sessions`, y llamar `delete_orphan_carts(db, sessions)` justo después, antes del `db.commit()` de
  la línea 729 — cubre explícitamente el edge case de spec.md "dos `TableSession` activas sobre la
  misma mesa cerrándose en la misma operación": se borran los carritos de participantes de **ambas**
  porque `sessions` las incluye a las dos. El 409 por órdenes activas no bloqueantes no cambia
  (FR-005)

- [X] T012 [P] [US2] En `_sweep_schema` (`pos-backend/app/core/scheduler.py:88-165`): agregar
  `delete_orphan_carts` al import ya existente `from app.api.v1.orders.checkout import (TERMINAL,
  close_participants, close_table_sessions)` (líneas 93-95); capturar el retorno de
  `close_table_sessions(db, table_id, closed_by=None)` (línea 140, hoy sin capturar) en `sessions`; y
  llamar `delete_orphan_carts(db, sessions)` **solo** dentro de la rama `if quedo_libre:` (línea
  156-157), justo después de `table.status = "libre"` — nunca en la rama `elif pendientes is not
  None:` ni en la rama temprana `if has_billable_orders(...)` (línea 129-137, que ni siquiera llama
  `close_table_sessions`). Esto es lo que garantiza FR-003 en el único camino donde
  `close_table_sessions` puede ejecutarse sin que la mesa termine `libre` (research.md Decisión 1)

- [X] T013 [US2] Agregar un test junto a `test_close_session_unified_camino_feliz`
  (`pos-backend/app/characterization_tests/test_table_sessions_service.py:260`): sobre
  `self._seed_billable_session()`, agregar con `fx.make_cart` un `Cart` huérfano de un comensal ya
  `closed` antes del cobro (además de los participantes activos de la sesión) → `service.close_session`
  cobra y cierra → verificar que la mesa queda `'libre'` y que ese `Cart` ya no existe (Acceptance
  Scenario 1 de US2, depende de T003, T004, T009)

- [X] T014 [US2] Agregar un test junto a
  `test_release_paid_session_libera_la_mesa_cuando_todo_esta_pagado_y_listo`
  (`pos-backend/app/characterization_tests/test_table_sessions_service.py:528`): sobre
  `self._seed_bare_session()` con un pedido ya `'pagada'` y `'listo'` en cocina, agregar con
  `fx.make_cart` un `Cart` huérfano → `service.release_paid_session` → verificar que la mesa queda
  `'libre'` y que el `Cart` ya no existe (Acceptance Scenario 2 de US2, depende de T003, T004, T010)

- [X] T015 [P] [US2] Agregar un test junto a
  `test_release_table_409_con_ordenes_activas_y_libera_sin_ellas`
  (`pos-backend/app/characterization_tests/test_orders_checkout.py:494`): mesa con dos
  `TableSession` `'active'` (o una, según se necesite para aislar el escenario del edge case), cada
  una con un participante con `fx.make_cart` (`orders_fixtures.make_cart`, ya existente en ese
  módulo) sin órdenes bloqueantes → `checkout.release_table(db, table.id)` → verificar que la mesa
  queda `'libre'` y que **ninguno** de los `Cart` de ambas sesiones sigue existiendo (Acceptance
  Scenario 2 de US2, edge case de spec.md "dos TableSession cerrándose en la misma operación",
  depende de T004, T011)

- [X] T016 [P] [US2] Crear `pos-backend/app/characterization_tests/test_scheduler.py` (fichero
  nuevo, primera cobertura `unittest` de `_sweep_schema` — research.md Decisión 8): `TestCase` que
  monta una sesión SQLite en memoria vía `table_sessions_fixtures.new_session()`, siembra una mesa
  `'ocupada'` con una `TableSession` vencida (`opened_at` anterior al `corte` que se le pasa a
  `_sweep_schema`, o un comensal sin actividad reciente vía `expires_at`) sin nada por cobrar y con un
  `Cart` huérfano (`fx.make_cart`) de un comensal ya `closed`, y llama
  `app.core.scheduler._sweep_schema(schema, corte)` parcheando `with_db` para que abra la sesión de
  prueba (mismo patrón de parche que ya usan `test_invitations_resend_cancel.py`,
  `test_orders_payment_gate.py` y `fixtures.py` en este repo — `_sweep_schema(schema, corte,
  tenant_id=None)` no acepta ningún parámetro `db`/`Session`, así que parchear `with_db` es el
  único camino viable) → verificar que la mesa
  queda `'libre'` y que el `Cart` huérfano ya no existe (Acceptance Scenario 3 de US2 / escenario 3,
  depende de T003, T004, T012)

**Checkpoint**: en este punto, User Stories 1 y 2 funcionan juntas de forma independiente — los
cinco caminos de liberación de mesa limpian sus carritos.

---

## Phase 5: User Story 3 - Si la mesa no queda libre, ningún carrito se elimina (Priority: P1)

**Goal**: cuando una sesión se cierra (o sus comensales se cierran) pero la mesa **no** vuelve a
`libre` —porque queda algo por cobrar o porque persiste un `CustomerOrder` huérfano de la misma mesa
física—, ningún carrito se elimina; y si cualquier intento de liberar una mesa falla y hace
`rollback()`, ningún carrito queda eliminado ni modificado.

**Independent Test**: forzar cada uno de los dos frenos existentes (pedido facturable pendiente;
`CustomerOrder` huérfano no terminal de la misma mesa) y verificar que, aunque la sesión se cierre o
se eche a los comensales, la mesa sigue `ocupada` y ningún carrito de esos comensales se elimina.

**Depende de**: Phase 4 (US2) — T017/T018 extienden el fichero `test_scheduler.py` creado en T016;
la garantía negativa en sí ya quedó implementada por la ubicación de la llamada en T012 (dentro de
`if quedo_libre:`) y por el `try/except: db.rollback()` que cada call-site ya tenía antes de esta
spec — **esta historia no agrega código de producción nuevo**, solo lo verifica explícitamente
(Principio X).

### Tests para User Story 3

- [X] T017 [US3] Agregar un test a `pos-backend/app/characterization_tests/test_scheduler.py`
  (creado en T016): sesión vencida con un pedido `'abierta'` sin cobrar (`has_billable_orders` en
  `True`) y un `Cart` huérfano de un comensal ya cerrado → correr `_sweep_schema` → verificar que
  solo se llamó `close_participants` (la mesa sigue `'ocupada'`, `close_table_sessions` no corrió
  para esa sesión) y que el `Cart` huérfano **sigue existiendo** (con el mismo comportamiento de hoy:
  pasa a `'abandonado'` si estaba `'abierto'`, pero no se borra) — RN-SCHED-03 (Acceptance Scenario 1
  de US3, depende de T016)

- [X] T018 [US3] Agregar un test junto al de T017, en el mismo fichero: sesión vencida sin nada por
  cobrar pero con un `CustomerOrder` no terminal huérfano de la misma mesa física (sin
  `table_session_id`) → correr `_sweep_schema` → verificar que la sesión se cierra
  (`close_table_sessions` sí corre) pero la mesa **no** vuelve a `'libre'` (RN-SCHED-04,
  `quedo_libre is False`) y que ningún `Cart` de esa sesión se elimina (Acceptance Scenario 2 de US3,
  depende de T016)

- [X] T019 [P] [US3] Agregar un test de rollback en
  `pos-backend/app/characterization_tests/test_table_sessions_service.py`: sobre
  `self._seed_billable_session()`, agregar con `fx.make_cart` un `Cart` huérfano; mockear
  `_close_unified` (o `_close_split`, según el `billing_mode` del escenario) para que lance una
  excepción genérica dentro de `service.close_session`; verificar que, tras el `rollback()` del
  `except Exception:` ya existente (línea 294-297), el `Cart` sigue existiendo con el mismo `id` y
  `status` que tenía antes del intento — ninguna eliminación parcial (Acceptance Scenario 3 de US3,
  FR-002, depende de T003, T004, T009)

**Checkpoint**: la garantía negativa queda verificada en los tres frenos existentes
(RN-SCHED-03, RN-SCHED-04, rollback genérico) sin que ninguno de ellos borre un carrito que el
cajero todavía necesita.

---

## Phase 6: User Story 4 - La liberación de una mesa nunca toca los carritos de otra (Priority: P2)

**Goal**: en un local con varias mesas activas al mismo tiempo, cuando una de ellas se libera, los
carritos huérfanos de las demás mesas —todavía ocupadas, o ya libres desde antes— no se ven
afectados en absoluto.

**Independent Test**: con dos mesas activas, cada una con su propio carrito huérfano, liberar una de
las dos y verificar que el carrito de la otra mesa sigue existiendo sin cambios.

**Depende de**: Phase 3 (US1) — verifica el aislamiento de `delete_orphan_carts` introducida ahí; no
agrega código de producción nuevo (FR-004 ya está garantizado porque `delete_orphan_carts` solo
recibe las `sessions` de la mesa que efectivamente se libera, research.md Decisión 3).

### Tests para User Story 4

- [X] T020 [US4] Agregar un test en
  `pos-backend/app/characterization_tests/test_table_sessions_service.py`: dos mesas activas
  independientes, cada una con su propia `TableSession`/participante y un `Cart` huérfano
  (`fx.make_cart`) de un comensal ya cerrado → liberar solo una de las dos (vía
  `service.try_release_if_empty` o `service.close_session`) → verificar que el `Cart` de la mesa que
  sigue ocupada permanece exactamente igual: mismo `id`, mismo `status`, sin eliminarse (Acceptance
  Scenario 1 de US4, FR-004, depende de T003, T004, T005)

**Checkpoint**: todas las historias de usuario son funcionales de forma independiente — la limpieza
de carritos está aislada por sesión/mesa/tenant.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de no-regresión sobre toda la suite

- [X] T021 Ejecutar la suite completa de `pos-backend`
  (`python -m unittest discover -s app/characterization_tests -v`) y confirmar que los 3 tests
  `CONGELA` de `test_orders_checkout.py` examinados en research.md Decisión 5
  (`test_close_participants_cierra_activos_y_devuelve_conteo`,
  `test_close_table_sessions_no_valida_pendientes_rn_ord_31`,
  `test_release_table_409_con_ordenes_activas_y_libera_sin_ellas`) siguen en verde **sin ninguna
  modificación**, que `test_orders_consolidation.py`/`test_cart_router.py` (ajenos a este cambio,
  research.md Decisión 4) no se ven afectados, y que ningún otro test `"CONGELA comportamiento
  actual:"` quedó en rojo sin autorización (Principio III, quickstart.md §Verificación final,
  depende de T006, T009, T010, T011, T012)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato. T001 (registro A-54) es
  bloqueante sin condicional (Principio II): ninguna tarea de Phase 2 en adelante debe iniciarse
  antes de que exista esa entrada
- **Foundational (Phase 2)**: depende de Setup. Bloquea las tareas de test de US1-US4 (todas usan
  `make_cart`), aunque no bloquea T004/T005 (implementación pura, sin fixtures de test)
- **User Story 1 (Phase 3)**: depende de Setup y Foundational. Es la base técnica
  (`delete_orphan_carts`) de la que dependen US2, US3 y US4
- **User Story 2 (Phase 4)**: depende de que Phase 3 esté completa (reusa `delete_orphan_carts` de
  T004; T016 crea el fichero que extienden T017/T018 de US3)
- **User Story 3 (Phase 5)**: depende de que Phase 4 esté completa (T017/T018 extienden
  `test_scheduler.py`, creado en T016 de US2)
- **User Story 4 (Phase 6)**: depende de que Phase 3 esté completa (verifica el aislamiento que US1
  introduce); no depende de US2 ni US3
- **Polish (Phase 7)**: depende de que todas las historias deseadas estén completas

### User Story Dependencies

- **User Story 1 (P1)**: sin dependencia de otras historias — es la base de todas las demás
- **User Story 2 (P1)**: depende de User Story 1 (reusa `checkout.delete_orphan_carts`)
- **User Story 3 (P1)**: depende de User Story 1 (garantía negativa sobre el mismo mecanismo) y,
  para sus tareas concretas, de User Story 2 (extiende `test_scheduler.py`)
- **User Story 4 (P2)**: depende de User Story 1

**Nota sobre archivos compartidos**: T009 y T010 modifican el mismo archivo
(`pos-backend/app/api/v1/table_sessions/service.py`) — secuenciales entre sí, no paralelas. T013,
T014, T019 y T020 también comparten `test_table_sessions_service.py` con T007/T008 (US1) — todas
secuenciales dentro de ese archivo, aunque pertenezcan a historias distintas.

### Within Each User Story

- La función compartida (T004) antes de cualquier call-site que la invoque (T005, T009-T012)
- Cada call-site modificado (T005, T009, T010, T011, T012) antes de los tests que lo verifican
  (T006-T008, T013-T020)
- El fixture `make_cart` (T003) antes de cualquier test nuevo que lo use
- Historia completa antes de pasar a la siguiente en prioridad

### Parallel Opportunities

- T011 (`checkout.py::release_table`) y T012 (`scheduler.py::_sweep_schema`) en paralelo — archivos
  distintos entre sí y frente a T009/T010 (`table_sessions/service.py`)
- T015 (`test_orders_checkout.py`) y T016 (`test_scheduler.py`, nuevo) en paralelo — archivos
  distintos, ambos dependen solo de T004 y de su propio call-site (T011, T012)
- T019 (rollback, US3) puede ejecutarse en paralelo a T017/T018 (ambos en `test_scheduler.py`) —
  archivo distinto

---

## Parallel Example: User Story 2

```bash
# T011 y T012 en paralelo (call-sites en archivos distintos):
Task: "Enganchar delete_orphan_carts en release_table, pos-backend/app/api/v1/orders/checkout.py"
Task: "Enganchar delete_orphan_carts en _sweep_schema, pos-backend/app/core/scheduler.py"

# Tras T011/T012, T015 y T016 en paralelo (tests en archivos distintos):
Task: "Test de release_table con carrito huérfano en pos-backend/app/characterization_tests/test_orders_checkout.py"
Task: "Crear test_scheduler.py con el escenario positivo del barrido en pos-backend/app/characterization_tests/test_scheduler.py"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup (incluye el registro A-54, bloqueante)
2. Completar Phase 2: Foundational (`make_cart`)
3. Completar Phase 3: User Story 1 (T004-T008)
4. **DETENER y VALIDAR**: probar User Story 1 de forma independiente (el camino automático más
   frecuente ya limpia sus carritos, sin importar `status` ni cantidad de comensales)
5. Desplegar/demostrar si está listo — resuelve ya el camino que más huérfanos acumula (spec.md
   "Why this priority" de US1)

### Incremental Delivery

1. Setup + Foundational → Phase 3 (US1) → probar de forma independiente → Demo (MVP)
2. Agregar Phase 4 (US2) → probar de forma independiente → Demo (cierra los tres caminos de
   staff/barrido que faltaban)
3. Agregar Phase 5 (US3) → probar de forma independiente → Demo (garantía negativa verificada)
4. Agregar Phase 6 (US4) → probar de forma independiente → Demo (aislamiento entre mesas verificado)
5. Phase 7: Polish — no-regresión completa sobre toda la suite

---

## Notes

- [P] = archivos distintos, sin dependencias entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- FR-005 (ninguna condición de liberación existente cambia) no genera tareas propias — es una
  restricción negativa verificada por omisión: ningún task de este documento toca un `if`/`return`
  de los 5 caminos, solo agrega una llamada nueva en la rama donde la mesa ya queda `libre`
  (verificable en T005, T009-T012)
- El edge case "comensal agregado por staff sin carrito propio" (spec.md) no genera tarea propia —
  `delete_orphan_carts` (T004) ya no tiene ninguna fila que tocar para ese participante, sin
  necesitar ningún `if` especial
- El edge case "carritos huérfanos de mesas ya `libre` desde antes de desplegar esta spec" (Fuera de
  Alcance) tampoco genera tarea — esta spec actúa solo hacia adelante, ningún backfill
- Confirmar que T002 (línea base) pasa en verde antes de tocar cualquier código de producción
- Detenerse en cada checkpoint para validar la historia de forma independiente
