---

description: "Task list template for feature implementation"
---

# Tasks: Corrección — "Liberar Mesa" bloqueada por un pedido ya cancelado

**Input**: Design documents from `/specs/050-correccion-liberar-mesa-pedido-cancelado/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/api-contract.md](./contracts/api-contract.md), [quickstart.md](./quickstart.md)

**Tests**: Se incluyen tareas de test. La Constitución (Principio X, Verificación Obligatoria) exige
verificar toda corrección de comportamiento, y research.md ya identificó dos huecos de cobertura
reales: (a) ningún test combina un pedido `'cancelada'` con ítems sin terminar en cocina para
`release_paid_session`; (b) `ToastService` no tiene ningún archivo de test hoy.

**Organization**: Dos historias de usuario independientes (spec.md: US1 en P1, US2 en P2), cada una
en un repositorio distinto. Las rutas de US1 son relativas a `../pos-backend`; las de US2, a
`../pos-heladeria` (ambos repositorios hermanos de este `pos-specs`).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de tareas incompletas)
- **[Story]**: Historia de usuario a la que pertenece la tarea (US1, US2)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

US1 usa como raíz `pos-backend/` y US2 usa como raíz `pos-heladeria/` (repositorios hermanos de
este `pos-specs`), según la `Project Structure` de [plan.md](./plan.md). Al ser repositorios y
archivos completamente distintos, US1 y US2 son independientes entre sí y pueden implementarse en
cualquier orden o en paralelo.

---

## Phase 1: Setup

**Purpose**: Confirmar el estado base de ambos repositorios antes de tocar cualquier archivo

- [X] T001 Ejecutar la suite de tests existente en `pos-backend` (`pytest -q`) y registrar el estado real como línea base de regresión (Principio X)
- [X] T002 [P] Ejecutar la suite de tests existente en `pos-heladeria` (`ng test`) y registrar el estado real como línea base de regresión (Principio X)
- [X] T003 [P] Confirmar que `ng build` compila sin errores en `pos-heladeria`, como referencia antes del cambio

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Prerrequisitos bloqueantes compartidos por todas las historias

**Nota**: esta feature no tiene prerrequisitos fundacionales bloqueantes — US1 y US2 son dos
correcciones independientes en repositorios y archivos distintos, sin ningún acoplamiento entre
ellas.

**Checkpoint**: Fase 1 completa — ambas historias de usuario pueden comenzar, en cualquier orden.

---

## Phase 3: User Story 1 - Liberar una mesa con un pedido ya cancelado (Priority: P1) 🎯 MVP

**Goal**: `release_paid_session` deja de contar los ítems de un pedido `'cancelada'` al validar si
la mesa tiene "ítems sin terminar en cocina" — una mesa cuyo único obstáculo es un pedido cancelado
se libera sin error.

**Independent Test**: Crear un pedido manual sobre una mesa, dejar al menos un ítem sin terminar en
cocina, cancelar ese pedido, y verificar que "Liberar Mesa" ahora libera la mesa sin el error "Hay
ítems sin terminar en cocina...". Verificar además que una mesa con un pedido `'pagada'` cuya
cocina siga en curso sigue rechazando la liberación (comportamiento protegido).

### Tests for User Story 1 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T004 [US1] En `pos-backend/app/characterization_tests/test_table_sessions_service.py`, junto a los demás `test_release_paid_session_*` (~línea 647), agregar un test que arme una mesa cuya sesión tenga **un único** pedido `status="cancelada"` (`fx.make_customer_order(db, ts, status="cancelada")`) con un ítem `estado_cocina="pendiente"` (`fx.make_order_item(db, order, variant, estado_cocina="pendiente")`) y confirme que `service.release_paid_session(db, ts.id, cashier)` **no** lanza `HTTPException` — libera la mesa (`ts.status == "closed"`, `table.status == "libre"`), igual que el patrón ya usado por `test_release_paid_session_libera_la_mesa_cuando_todo_esta_pagado_y_listo` (~línea 670) (contracts/api-contract.md, fila 1 de la tabla de comportamiento)

### Implementation for User Story 1

- [X] T005 [US1] En `pos-backend/app/api/v1/table_sessions/service.py`, dentro de `release_paid_session` (~líneas 366-369), agregar `CustomerOrder.status != "cancelada"` al `where(...)` del query que arma `orders` antes de `_assert_closable(db, orders)` — mismo criterio que ya usa `_billable_orders` (línea ~153); ajustar el docstring de `release_paid_session` para aclarar que "todos los pedidos de la sesión" significa "todos salvo los cancelados" (research.md, D1)

**Checkpoint**: En este punto, la Historia 1 debe funcionar y probarse de forma completa e
independiente — correr T004 antes de T005 (debe fallar), y después de T005 (debe pasar), más el
test existente de "pagada" con cocina en curso (debe seguir pasando sin cambios).

---

## Phase 4: User Story 2 - Un error que se repite no apila avisos idénticos (Priority: P2)

**Goal**: `ToastService` deja de apilar una tarjeta nueva cuando ya hay una visible con el mismo
tipo y el mismo texto.

**Independent Test**: Provocar dos veces seguidas, sin resolver la causa entre intentos, el mismo
error (mismo tipo y texto) y verificar que solo queda una tarjeta de aviso visible, no dos; provocar
un error de texto o tipo distinto y verificar que sí aparece como tarjeta independiente.

### Tests for User Story 2 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T006 [P] [US2] Crear `pos-heladeria/src/app/shared/feedback/toast.service.spec.ts` (no existe hoy ninguna cobertura de `ToastService`) con tests que confirmen: (a) dos llamadas seguidas a `error()` con el mismo texto dejan un único toast en `toasts()`; (b) `error()` con un texto distinto sí agrega una segunda entrada; (c) `success()`/`info()` con el mismo texto que un `error()` ya visible sí agregan una entrada aparte (mismo texto, distinto `kind`); (d) tras `dismiss(id)` del primero, una nueva llamada con el mismo texto y tipo vuelve a mostrarse (contracts/api-contract.md, tabla de `ToastService`)

### Implementation for User Story 2

- [X] T007 [US2] En `pos-heladeria/src/app/shared/feedback/toast.service.ts`, dentro de `push()` (~líneas 36-42), agregar al inicio: si `this.toasts()` ya contiene una entrada con el mismo `kind` y el mismo `text`, retornar sin agregar nada — sin cambiar la firma pública de `success`/`error`/`info` (research.md, D2)

**Checkpoint**: En este punto, ambas historias deben funcionar juntas de forma independiente.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final

- [X] T008 [P] Ejecutar manualmente los tres escenarios de [quickstart.md](./quickstart.md) contra un entorno local con `pos-backend` y `pos-heladeria` corriendo — **parcial**: sin navegador disponible en este entorno de implementación no se pudo hacer el recorrido manual real (crear/cancelar un pedido y hacer clic en "Liberar Mesa"). Se verificó en su lugar la cobertura automatizada equivalente: el test nuevo de T004 ejercita exactamente el Escenario 1 (libera con pedido cancelado) y el Escenario 2 (sigue bloqueando con `'pagada'`, sin cambios); los tests nuevos de T006 ejercitan exactamente el Escenario 3 (deduplicación). **Queda pendiente que el usuario/QA reproduzca el recorrido real** (crear pedido manual → cancelar → Liberar Mesa) antes de dar la spec por verificada en producción (Principio X)
- [X] T009 Ejecutar la suite completa de tests de `pos-backend` (`pytest -q`, en la práctica `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` — comando real de este repo, ver `README.md`) y confirmar que no hay regresiones más allá del test nuevo de T004, comparando contra la línea base de T001 — **457/457 tests pasan** (línea base 456 + 1 nuevo), cero regresiones
- [X] T010 Ejecutar la suite completa de tests de `pos-heladeria` (`ng test`) y confirmar que no hay regresiones más allá de los tests nuevos de T006, comparando contra la línea base de T002 — **490/501 tests pasan** (línea base 486 + 4 nuevos), mismos 11 fallos preexistentes, cero regresiones nuevas
- [X] T011 [P] Ejecutar `ng build` en `pos-heladeria` y confirmar que compila sin errores nuevos respecto a la línea base de T003 — confirmado, mismo warning preexistente (`qrcode` CommonJS), sin errores nuevos

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: Sin tareas bloqueantes.
- **User Story 1 (Phase 3)**: Puede empezar tras la Fase 1 — repositorio y archivos propios
  (`pos-backend`), sin depender de la Historia 2.
- **User Story 2 (Phase 4)**: Puede empezar tras la Fase 1 — repositorio y archivos propios
  (`pos-heladeria`), sin depender de la Historia 1. Puede implementarse en paralelo con la Historia
  1 o en cualquier orden respecto a ella.
- **Polish (Phase 5)**: Depende de que ambas historias estén completas.

### Within Each User Story

- T004 (test) se escribe y debe fallar antes de T005 (implementación) — ejercita exactamente el
  filtro que T005 agrega.
- T006 (tests) se escribe y debe fallar antes de T007 (implementación) — ejercita exactamente la
  deduplicación que T007 agrega.

### Parallel Opportunities

- T001/T002/T003 (Setup) en paralelo — repositorios y comandos distintos.
- La Historia 1 completa (T004-T005) y la Historia 2 completa (T006-T007) pueden ejecutarse en
  paralelo entre sí — repositorios y archivos completamente distintos.
- T008/T011 (Polish) pueden iniciarse en paralelo; T009/T010 son las validaciones finales de cada
  repositorio.

---

## Parallel Example: Ambas historias a la vez

```bash
# Historia 1 (pos-backend) e Historia 2 (pos-heladeria) son independientes:
Task: "Agregar test de release_paid_session con pedido cancelado en test_table_sessions_service.py, luego el fix del query en service.py"
Task: "Crear toast.service.spec.ts con los tests de deduplicación, luego el fix de push() en toast.service.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (sin tareas).
3. Completar Fase 3: User Story 1 (T004-T005) — resuelve el bug reportado (mesa bloqueada).
4. **DETENERSE Y VALIDAR**: probar la Historia 1 de forma independiente.
5. Desplegar/demostrar si está listo — ya resuelve el problema principal reportado.

### Incremental Delivery

1. Completar Setup + Foundational → base lista en ambos repositorios.
2. Agregar Historia 1 → probar de forma independiente → Desplegar/Demo (MVP, resuelve el bug de
   fondo).
3. Agregar Historia 2 → probar de forma independiente → Desplegar/Demo (mejora de UX general).
4. Completar Fase 5: Polish (T008-T011).

---

## Notes

- [P] = archivos/repositorios distintos o bloques independientes sin dependencias entre sí.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- Verificar que los tests fallan antes de implementar.
- Commit tras cada tarea o grupo lógico — recordar que US1 y US2 son commits en repositorios
  distintos (`pos-backend` vs. `pos-heladeria`), separados del propio commit de esta spec en
  `pos-specs`.
