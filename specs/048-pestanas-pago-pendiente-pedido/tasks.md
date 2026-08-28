---

description: "Task list template for feature implementation"
---

# Tasks: Pestañas para Ver el Pedido Pagado Junto al Pago Pendiente de la Misma Mesa

**Input**: Design documents from `/specs/048-pestanas-pago-pendiente-pedido/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/ui-store-contract.md](./contracts/ui-store-contract.md), [quickstart.md](./quickstart.md)

**Tests**: Se incluyen tareas de test. La Constitución (Principio X, Verificación Obligatoria) exige
verificar toda corrección de comportamiento, y research.md §3 ya identificó qué cobertura falta:
ningún test existente ejercita una mesa con pago pendiente y pedido pagado/activo a la vez, ni el
mecanismo de pestañas nuevo.

**Organization**: Una única historia de usuario (spec.md solo tiene P1). Todas las rutas de archivo
son relativas al repositorio de la aplicación `../pos-heladeria` (el código no vive en este
repositorio de specs). No hay ninguna tarea sobre `pos-backend` — esta spec no cambia backend.

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

- [X] T001 Ejecutar la suite de tests existente en `pos-heladeria` (`ng test`) y registrar el estado real como línea base de regresión (Principio X) — confirmado **452/463** (53/58 archivos), igual a la línea base ya conocida tras spec 047
- [X] T002 [P] Confirmar que `ng build` compila sin errores en `pos-heladeria`, como referencia antes del cambio — confirmado, solo warnings preexistentes de budget/CommonJS

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Prerrequisitos bloqueantes compartidos por todas las historias

**Nota**: esta feature no tiene prerrequisitos fundacionales bloqueantes ni cambios de backend — es
una única historia sobre dos archivos de producción ya existentes.

**Checkpoint**: Fase 1 completa — la historia de usuario puede comenzar.

---

## Phase 3: User Story 1 - El cajero ve el pedido pagado junto al pago pendiente de confirmar de la misma mesa (Priority: P1) 🎯 MVP

**Goal**: Cuando una mesa tiene a la vez un pago pendiente de confirmar y un pedido pagado/activo,
el panel central ofrece dos pestañas ("Pagos por confirmar" / "Pedido de la mesa") para alternar
entre ambos, sin perder ninguno.

**Independent Test**: Con una mesa que tiene un pedido ya pagado y, a la vez, un nuevo pedido QR
pendiente de confirmar, seleccionarla y verificar que aparecen las dos pestañas, que "Pagos por
confirmar" está activa por defecto, y que cambiar a "Pedido de la mesa" muestra el mismo panel
completo de siempre (ítems, total, Imprimir Factura, Liberar Mesa).

### Tests for User Story 1 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T003 [P] [US1] Agregar en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` un bloque de test que confirme: (a) con una orden `'pagada'` y otra `'recibida'`+`qr` en la misma mesa, `store.hasPendingAndActiveOrders()` es `true` y `store.effectiveCentralView()` es `'validar-pago'` por defecto; (b) tras `store.centralPanelTab.set('pedido')`, `effectiveCentralView()` pasa a `'pedido'`; (c) con solo uno de los dos tipos de pedido, `hasPendingAndActiveOrders()` es `false` y `effectiveCentralView()` coincide con `centralState()`; (d) tras elegir la pestaña `'pedido'` y luego `selectTable()` de otra mesa, `centralPanelTab()` vuelve a `'validar-pago'` — mismo patrón de setup sin HTTP que `describe('PosTerminalStore.pendingOrders — solo canal qr')` (research.md §3)
- [X] T004 [P] [US1] Agregar en `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.spec.ts` un bloque de test que, con una mesa con ambos tipos de pedido, confirme que aparecen los dos botones de pestaña en el encabezado del panel central y que, al hacer clic en "Pedido de la mesa", el panel deja de mostrar `app-payment-validation-block` y pasa a mostrar `app-pos-order-panel` (y viceversa al volver a "Pagos por confirmar") — mismo patrón que `describe('TableSessionsComponent — diálogo de éxito sin botón duplicado (spec 029)')` (`store = fixture.componentInstance.store; vi.spyOn(store, 'init').mockResolvedValue(undefined);`, sin HTTP) (research.md §3); se agregaron también los dos casos de "solo un tipo → sin pestañas"

### Implementation for User Story 1

- [X] T005 [US1] En `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`, agregar el signal `centralPanelTab = signal<'validar-pago' | 'pedido'>('validar-pago')`, el computed `hasPendingAndActiveOrders` (`pendingOfSelectedTable().length > 0 && ordersOfTable(tableId).length > 0` para la mesa seleccionada) y el computed `effectiveCentralView` (`centralPanelTab()` cuando `hasPendingAndActiveOrders()`, si no `centralState()` tal cual) junto a `centralState` (~línea 447), sin modificar el cuerpo de `centralState` ni de `ordersOfTable`/`pendingOfSelectedTable`; agregar `this.centralPanelTab.set('validar-pago');` dentro de `resetTransient()` (~línea 949) (research.md §1)
- [X] T006 [US1] En `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts`, cambiar el `@switch` de contenido del panel central (~líneas 119-149) de `store.centralState()` a `store.effectiveCentralView()` (cuerpo de los tres `@case` sin cambios); en el encabezado (~líneas 104-110), envolver el `@switch` de solo texto existente en un `@if (store.hasPendingAndActiveOrders()) { <dos botones de pestaña, (click) escribe store.centralPanelTab.set(...)> } @else { <switch existente, sin cambios> }` — el botón de silenciar la campana no se toca (research.md §2)

**Checkpoint**: La historia de usuario debe funcionar y probarse de forma completa en este punto.

---

## Phase 4: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final

- [X] T007 [P] Ejecutar manualmente el escenario de [quickstart.md](./quickstart.md) contra un entorno local con `pos-heladeria` y `pos-backend` corriendo — **parcial**: sin navegador disponible en este entorno de implementación no se pudo ejecutar la QA manual completa (mesa con ambos tipos de pedido, clic entre pestañas). Se verificó en su lugar: `ng build` sin errores, `ng serve` sirviendo `HTTP 200` sin errores de consola en el arranque, y la cobertura automatizada de T003-T004 que ejercita exactamente el mismo comportamiento (aparición de pestañas, cambio de contenido al hacer clic). **Queda pendiente que el usuario/QA ejecute el recorrido manual real** antes de dar la spec por verificada en producción (Principio X)
- [X] T008 Ejecutar la suite completa de tests de `pos-heladeria` (`ng test`) y confirmar que no hay regresiones más allá de los dos tests nuevos de T003-T004 — **459/470 tests pasan** (53/58 archivos); los 11 que fallan son exactamente los mismos preexistentes ya documentados en specs 046/047 (`app.spec.ts`, `auth.service.spec.ts`, `sidebar.component.spec.ts`, regresión de `MoneyInputComponent`); cero regresiones nuevas — 459 = 452 (línea base) + 7 tests nuevos (4 en el store, 3 en el componente)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: Sin tareas bloqueantes.
- **User Story 1 (Phase 3)**: Puede empezar tras la Fase 1.
- **Polish (Phase 4)**: Depende de que la Fase 3 esté completa.

### Within User Story 1

- T003 y T004 (tests) se escriben y deben fallar antes de T005/T006 (implementación) — ambos ejercitan exactamente el mecanismo que T005/T006 agregan, así que deben fallar contra el código actual (sin pestañas) y pasar después.
- T005 (store) y T006 (componente) tocan archivos distintos, pero T006 depende de los miembros que T005 agrega al store (`hasPendingAndActiveOrders`, `centralPanelTab`, `effectiveCentralView`) — implementar T005 antes de T006.

### Parallel Opportunities

- T001/T002 (Setup) en paralelo.
- T003/T004 (tests US1) en paralelo entre sí — archivos distintos.
- T007/T008 (Polish) pueden iniciarse en paralelo, aunque T008 es la validación final tras T007.

---

## Parallel Example: User Story 1

```bash
# Lanzar juntos los dos tests de la Historia 1 (archivos distintos):
Task: "Agregar tests de hasPendingAndActiveOrders/effectiveCentralView/reinicio de centralPanelTab en pos-terminal.store.spec.ts"
Task: "Agregar test de renderizado y cambio de pestaña en table-sessions.component.spec.ts"
```

---

## Implementation Strategy

### MVP First (única historia)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (sin tareas).
3. Completar Fase 3: User Story 1 (T003-T006).
4. **DETENERSE Y VALIDAR**: probar la historia de forma independiente (mesa con ambos tipos de
   pedido muestra pestañas; mesas con un solo tipo no cambian).
5. Completar Fase 4: Polish (T007-T008) y desplegar/demostrar.

---

## Notes

- [P] = archivos distintos o bloques independientes sin dependencias entre sí.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- No hay ninguna tarea de backend — esta spec no cambia `pos-backend`.
- Verificar que los tests fallan antes de implementar.
- Commit tras cada tarea o grupo lógico.
