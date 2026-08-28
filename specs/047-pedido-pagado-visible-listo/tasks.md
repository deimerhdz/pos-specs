---

description: "Task list template for feature implementation"
---

# Tasks: Pedido de Mostrador Pagado Sigue Visible Hasta Liberar la Mesa

**Input**: Design documents from `/specs/047-pedido-pagado-visible-listo/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [quickstart.md](./quickstart.md)

**Tests**: Se incluyen tareas de test. La Constitución (Principio X, Verificación Obligatoria) exige
verificar toda corrección de comportamiento, y research.md §4 ya identificó el gap exacto de
cobertura que dejó pasar este bug (ningún test ejercitaba `status: 'pagada'` a través de
`tableOrders()`/`activeOrders()`/`centralState()`/`marcarListo()` juntos). Los `describe` existentes
de `deriveTableStatus` (con `status: 'abierta'`) deben permanecer en verde sin ningún cambio.

**Organization**: Una única historia de usuario (spec.md solo tiene P1). Todas las rutas de archivo
son relativas al repositorio de la aplicación `../pos-heladeria` (el código no vive en este
repositorio de specs). No hay ninguna tarea sobre `pos-backend` — esta spec no cambia backend
(research.md §1).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de tareas incompletas)
- **[Story]**: Historia de usuario a la que pertenece la tarea (US1)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

Todas las rutas usan como raíz `pos-heladeria/` (repositorio hermano de este `pos-specs`), según la
`Project Structure` de [plan.md](./plan.md).

---

## Phase 1: Setup

**Purpose**: Confirmar el estado base del entorno antes de tocar cualquier archivo

- [X] T001 Ejecutar la suite de tests existente en `pos-heladeria` (`ng test`) y registrar el estado real como línea base de regresión (Principio X) — confirmado **449/460** (53/58 archivos), igual a la línea base ya conocida tras spec 046
- [X] T002 [P] Confirmar que `ng build` compila sin errores en `pos-heladeria`, como referencia antes del cambio — confirmado, solo warnings preexistentes de budget/CommonJS

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Prerrequisitos bloqueantes compartidos por todas las historias

**Nota**: esta feature no tiene prerrequisitos fundacionales bloqueantes ni cambios de backend
(research.md §1) — es una única historia sobre un único archivo de producción.

**Checkpoint**: Fase 1 completa — la historia de usuario puede comenzar.

---

## Phase 3: User Story 1 - El pedido pagado sigue visible hasta que el cajero libera la mesa (Priority: P1) 🎯 MVP

**Goal**: Un pedido `'pagada'` sigue contando como consumo vivo de su mesa sin importar el estado de
la cocina — deja de mostrarse únicamente cuando el cajero libera la mesa explícitamente.

**Independent Test**: Cobrar por adelantado un pedido de mostrador, marcarlo como listo antes de
liberar la mesa, y verificar que el pedido sigue viéndose (con su resumen) en el panel central y
derecho, y que la tarjeta de la mesa refleja el estado "Listo" sin volverse "mesa libre".

### Tests for User Story 1 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T003 [P] [US1] Agregar en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` un bloque de test que ponga una orden `status: 'pagada'`, `paid: true`, todos los ítems `'listo'`, canal `'counter'`, directamente vía `store.orders.set(...)` + `store.selectedTableId.set(...)` (inyectando `TableService` y poniendo `tables.set([...])` con una fila mínima para la mesa de prueba) y confirme que `store.centralState()` es `'pedido'` (no `'mesa-libre'`) y que la fila de `store.tablesView()` para esa mesa tiene `statusLabel: 'Listo'` (no `'Ocupada'`) — mismo patrón de setup que `describe('PosTerminalStore.pendingOrders — solo canal qr')` (líneas 282-308) (research.md §4.1)
- [X] T004 [P] [US1] Agregar en el mismo archivo un bloque de test que ejercite `store.marcarListo()` de punta a punta: una orden `'pagada'` con un ítem `'pendiente'`, `store.selectedTableId.set(...)`/`store.selectedOrderId.set(...)`, se llama `marcarListo()`, se responde `POST /orders/{id}/ready` y el `reload()` subsiguiente (`/orders/tables`, `/orders?active_sessions_only=true`, `/table-sessions`) con la orden ya `'listo'` — confirmar que `store.selectedOrderId()` sigue en el mismo pedido y que `store.centralState()` sigue en `'pedido'`; mismo patrón HTTP/`tick()` que `describe('PosTerminalStore.ensureReadyToCharge')` (líneas 707-770) y `describe('PosTerminalStore.reload — resincroniza la selección...')` (líneas 626-696) (research.md §4.2)

### Implementation for User Story 1

- [X] T005 [US1] En `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`, quitar el conjunto `(o.status !== 'pagada' || hasPendingKitchenWork(o))` de `activeOrders` (líneas ~377-384) y de `tableOrders(tableId)` (líneas ~401-408); actualizar los docblocks de ambas explicando el fix (cita el gap de spec 035, A-52) en vez de la razón vieja e incompleta; retirar `hasPendingKitchenWork` del import de la línea 34 (queda solo `KITCHEN_NOT_READY`, que sigue usándose en `kitchenReady()`/`ensureReadyToCharge()`) (research.md §1-§2)
- [X] T006 [US1] En el mismo archivo, refrescar el comentario de la rama `'listo'` de `deriveTableStatus()` (líneas ~168-179), que hoy cita `hasPendingKitchenWork`/"tableOrders() ya deja pasar esas órdenes mientras les quede comida en preparación" como la razón por la que puede llegar una orden `'pagada'` con ítems `'listo'` — con el fix de T005 esa premisa deja de ser la razón real; no cambia el código de `deriveTableStatus()`, solo el texto (research.md §2)

**Checkpoint**: La historia de usuario debe funcionar y probarse de forma completa en este punto.

---

## Phase 4: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final

- [X] T007 [P] Ejecutar manualmente el escenario de [quickstart.md](./quickstart.md) contra un entorno local con `pos-heladeria` y `pos-backend` corriendo — **parcial**: sin navegador disponible en este entorno de implementación no se pudo ejecutar la QA manual completa (cobrar por adelantado, marcar listo, verificar visualmente el panel/badge). Se verificó en su lugar: `ng build` sin errores y `ng serve` sirviendo `HTTP 200` sin errores de consola en el arranque, más la cobertura automatizada de T003-T004 que ejercita exactamente el mismo comportamiento a nivel de store. **Queda pendiente que el usuario/QA ejecute el recorrido manual real** antes de dar la spec por verificada en producción (Principio X)
- [X] T008 Ejecutar la suite completa de tests de `pos-heladeria` (`ng test`) y confirmar que no hay regresiones más allá de los dos tests nuevos de T003-T004; confirmar que los `describe` existentes de `deriveTableStatus` (con `status: 'abierta'`) siguen en verde sin cambios de intención — **452/463 tests pasan** (53/58 archivos); los 11 que fallan son exactamente los mismos preexistentes ya documentados en spec 046 (`app.spec.ts`, `auth.service.spec.ts`, `sidebar.component.spec.ts`, regresión de `MoneyInputComponent`); cero regresiones nuevas; los 4 `describe` de `deriveTableStatus` (líneas 150-174) siguen en verde sin cambios

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: Sin tareas bloqueantes.
- **User Story 1 (Phase 3)**: Puede empezar tras la Fase 1.
- **Polish (Phase 4)**: Depende de que la Fase 3 esté completa.

### Within User Story 1

- T003 y T004 (tests) se escriben y deben fallar antes de T005 (implementación) — ambos ejercitan exactamente la condición que T005 corrige, así que deben fallar contra el código actual y pasar después.
- T005 y T006 tocan el mismo archivo (`pos-terminal.store.ts`) — secuenciales, no en paralelo. T006 es de bajo riesgo (solo texto) y depende de que T005 ya haya cambiado la razón real.

### Parallel Opportunities

- T001/T002 (Setup) en paralelo.
- T003/T004 (tests US1) en paralelo entre sí — mismo archivo pero bloques de test independientes.
- T007/T008 (Polish) pueden iniciarse en paralelo, aunque T008 es la validación final tras T007.

---

## Parallel Example: User Story 1

```bash
# Lanzar juntos los dos tests de la Historia 1 (mismo archivo, bloques independientes):
Task: "Agregar test de integración centralState/tablesView con orden 'pagada' ya lista"
Task: "Agregar test end-to-end de marcarListo() con orden 'pagada'"
```

---

## Implementation Strategy

### MVP First (única historia)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (sin tareas).
3. Completar Fase 3: User Story 1 (T003-T006).
4. **DETENERSE Y VALIDAR**: probar la historia de forma independiente (pedido pagado sigue visible
   tras marcarlo listo, hasta liberar la mesa).
5. Completar Fase 4: Polish (T007-T008) y desplegar/demostrar.

---

## Notes

- [P] = archivos distintos o bloques independientes sin dependencias entre sí.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- No hay ninguna tarea de backend — esta spec no cambia `pos-backend` (research.md §1, Constitution Check Principio VI/VII).
- Verificar que los tests fallan antes de implementar.
- Commit tras cada tarea o grupo lógico.
