---

description: "Task list for feature implementation"
---

# Tasks: Corrección — la Terminal de mesas no detecta un turno de caja que sí está abierto

**Input**: Design documents from `/specs/072-fix-deteccion-turno-caja/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: incluidos — el plan y los contratos ya comprometen ficheros de test concretos
(Principio X, Verificación Obligatoria).

**Organización**: esta corrección es **100% frontend** (`pos-heladeria`); `pos-backend` no se
toca. `CashService.discoverOpenShift()` es el cambio compartido del que dependen las tres
historias (US2 y US3 son, en gran parte, consecuencia directa de su regla de resolución) — vive
en Foundational. US1 (P1) añade además el disparador reactivo específico que cierra el defecto
reportado.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: se puede hacer en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: US1, US2 o US3

---

## Phase 1: Setup

- [X] T001 Levantar `pos-heladeria` (`npm start`) y confirmar que `npm test` pasa en verde como línea base, siguiendo [quickstart.md §1-2](./quickstart.md)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: `CashService.discoverOpenShift()` — el mecanismo del que dependen las tres historias
para reflejar el turno de caja real sin depender de `localStorage`
([contracts/descubrimiento-turno-abierto.md](./contracts/descubrimiento-turno-abierto.md)).

**⚠️ CRITICAL**: ninguna historia puede verificarse de verdad sin esto — US1 necesita que el
descubrimiento encuentre el turno; US2/US3 necesitan su regla de resolución (0 → bloquea, 2+ →
ambiguo).

- [X] T002 [P] Escribir tests en `pos-heladeria/src/app/modules/cash-register/services/cash.service.spec.ts` para `discoverOpenShift()` cubriendo la tabla de casos de [contracts/descubrimiento-turno-abierto.md §2](./contracts/descubrimiento-turno-abierto.md#2-tabla-de-casos) (camino rápido con `localStorage` válido; sin `localStorage` y exactamente 1 turno abierto entre varias cajas → lo adopta y lo persiste; sin `localStorage` y 0 turnos → `shift` en `null`; sin `localStorage` y 2+ turnos → `shift` en `null`) — deben fallar antes de implementar
- [X] T003 Reemplazar `restoreShift()` por `discoverOpenShift()` en `pos-heladeria/src/app/modules/cash-register/services/cash.service.ts` (líneas ~50-63), implementando el algoritmo de [contracts/descubrimiento-turno-abierto.md §1](./contracts/descubrimiento-turno-abierto.md#1-algoritmo) (camino rápido con `localStorage`, y si no resuelve nada, `listRegisters()` + `getCurrentShift()` por cada caja en paralelo, mismo patrón que `CashSessionStore.loadOverview()`) — depende de T002
- [X] T004 [P] Actualizar el call site en `pos-heladeria/src/app/modules/cash-register/services/cash-session.store.ts` (línea ~251, rama admin de `init()`) de `this.api.restoreShift()` a `this.api.discoverOpenShift()` — depende de T003
- [X] T005 [P] Actualizar el call site en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` (línea ~969, dentro de `ensureCheckoutDataLoaded()`) de `this.cash.restoreShift()` a `this.cash.discoverOpenShift()` — depende de T003
- [X] T006 [P] Actualizar los dos *stubs* de mock que hoy exponen `restoreShift` a `discoverOpenShift`: `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts:52` y `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.spec.ts:65` — depende de T003

**Checkpoint**: `discoverOpenShift()` funciona; los casos de 0 y de 2+ turnos abiertos ya se
comportan según FR-003/FR-004 a nivel de `CashService` (US2/US3 se verifican con más detalle en
sus propias fases).

---

## Phase 3: User Story 1 - Cobrar el pedido que acaba de llegar a una mesa ya seleccionada (Priority: P1) 🎯 MVP

**Goal**: el chequeo de turno de caja se dispara también cuando el panel de cobro aparece de forma
reactiva (pedido nuevo en una mesa ya seleccionada), no solo al seleccionar activamente una mesa
que ya tenía pedido ([contracts/disparador-reactivo-cobro.md](./contracts/disparador-reactivo-cobro.md)).

**Independent Test**: con un turno de caja abierto (descubierto por Foundational, sin
`localStorage` previo), seleccionar una mesa libre y, sin volver a tocarla, hacer que le llegue un
pedido nuevo — el panel "Pagos por confirmar" debe habilitar "Confirmar efectivo" en menos de ~2
segundos.

### Tests for User Story 1

- [X] T007 [P] [US1] Escribir tests en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` (dentro del describe "carga diferida de datos de cobro (spec 059, Historia 1)", línea ~1449) para el caso reactivo: mesa vacía ya seleccionada (`selectTable('t1')` sin pedidos), luego `store.orders.set([...])` agrega un pedido `QR_MENU`/`recibida` para esa mesa simulando una recarga real (no una segunda llamada a `selectTable`), y se confirma que el flujo pide métodos de pago y turno de caja según la tabla de [contracts/disparador-reactivo-cobro.md §3](./contracts/disparador-reactivo-cobro.md#3-tabla-de-casos) — deben fallar antes de implementar

### Implementation for User Story 1

- [X] T008 [US1] Extraer el método privado `hasChargeableOrderNow()` en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`, reutilizando la condición que ya usan `selectTable()` (línea ~1153) y `selectStandaloneOrder()` (línea ~1182) para decidir si hay algo real que cobrar ([contracts/disparador-reactivo-cobro.md §1](./contracts/disparador-reactivo-cobro.md#1-dónde-y-cuándo-se-dispara)) — depende de T007
- [X] T009 [US1] Actualizar `selectTable()` y `selectStandaloneOrder()` para llamar a `hasChargeableOrderNow()` en vez de repetir la condición, y agregar en `reloadOrders()` (líneas 1074-1081) la llamada condicional `if (this.hasChargeableOrderNow()) void this.ensureCheckoutDataLoaded();` justo después de `this.announcePending(orders)` — depende de T008

**Checkpoint**: US1 funciona de forma independiente y es verificable por separado.

---

## Phase 4: User Story 2 - El bloqueo se mantiene cuando de verdad no hay turno abierto (Priority: P1)

**Goal**: confirmar que el nuevo disparador reactivo de US1, combinado con `discoverOpenShift()`,
no relaja el bloqueo cuando genuinamente no hay ningún turno de caja abierto — sin esto, la
corrección volvería un adorno el candado que introdujo la spec 028.

**Independent Test**: sin ningún turno de caja abierto en el tenant, repetir el flujo reactivo de
US1 (mesa ya seleccionada que recibe un pedido) y confirmar que "Confirmar efectivo" sigue
bloqueado con el mensaje de siempre.

- [X] T010 [US2] Escribir un test de integración en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` que combine el escenario reactivo de T007 con **cero** turnos de caja abiertos en el tenant (todas las cajas responden 404 en `getCurrentShift`), y confirme que `store.cashShiftId()` sigue en `null` después de que se dispara `ensureCheckoutDataLoaded()` — depende de T009

**Checkpoint**: US1 y US2 funcionan juntas y por separado.

---

## Phase 5: User Story 3 - Más de una caja abierta a la vez sigue pidiendo elegir cuál (Priority: P2)

**Goal**: confirmar que, con dos o más turnos abiertos simultáneamente, el disparador reactivo de
US1 tampoco elige uno al azar — sigue exigiendo la selección explícita ya existente ("Operar"),
para no atribuir un cobro en efectivo a la caja equivocada.

**Independent Test**: con dos turnos de caja abiertos a la vez (dos cajas distintas) y sin
`localStorage` apuntando a ninguna, repetir el flujo reactivo de US1 y confirmar que
`cashShiftId` sigue en `null` — el cajero debe seguir viendo el mensaje de bloqueo hasta operar
una caja explícitamente.

- [X] T011 [US3] Escribir un test de integración en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` que combine el escenario reactivo de T007 con **dos** cajas con turno abierto simultáneamente (dos `getCurrentShift` exitosos) y sin `localStorage`, y confirme que `store.cashShiftId()` sigue en `null` — depende de T009

**Checkpoint**: las tres historias funcionan juntas y por separado.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T012 [P] Correr la batería completa (`python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` en `pos-backend`, y `npm test` en `pos-heladeria`) y confirmar 0 fallos nuevos — en particular, que los 35 usos de `store.orders.set([...])` en `pos-terminal.store.spec.ts` siguen en verde sin peticiones HTTP inesperadas ([quickstart.md §2](./quickstart.md#2-batería-automatizada), research.md D1)
- [X] T013 Ejecutar el recorrido manual de [quickstart.md §3](./quickstart.md#3-validación-manual-por-historia) (US1, US2, US3) en un entorno real, confirmando el margen de ~2 segundos de las Clarifications

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: depende de Phase 1 — bloquea las tres historias.
- **US1 (Phase 3)**: depende de Phase 2. Es el único disparo de producción restante; sin él, US2
  y US3 no tienen ningún escenario reactivo que verificar.
- **US2 (Phase 4)**: depende de Phase 3 (T009) — su test combina el disparador reactivo de US1
  con cero turnos abiertos.
- **US3 (Phase 5)**: depende de Phase 3 (T009), independiente de US2 — su test combina el mismo
  disparador con dos turnos abiertos.
- **Polish (Phase 6)**: depende de que US1, US2 y US3 estén completas.

### Nota de paralelismo real

T004, T005 y T006 tocan archivos distintos y dependen solo de T003 (ya terminada) — se pueden
hacer en paralelo entre sí. T010 y T011 tocan el **mismo** archivo que T007
(`pos-terminal.store.spec.ts`) — aunque son historias independientes de probar, no se marcan
`[P]` entre sí para evitar conflictos de edición sobre el mismo fichero.

### Within Each User Story

- Tests antes que implementación (TDD): T007 debe fallar antes de T008/T009.
- US2 (T010) y US3 (T011) son tareas de **verificación** (tests de integración), no producen
  código nuevo — la regla que verifican ya la implementa `discoverOpenShift()` (Foundational);
  solo confirman que el disparador reactivo de US1 no la rompe.

---

## Parallel Example: Foundational

```bash
# T004, T005 y T006 dependen solo de T003 (ya terminada) — en paralelo:
Task: "Actualizar el call site de discoverOpenShift en cash-session.store.ts"
Task: "Actualizar el call site de discoverOpenShift en pos-terminal.store.ts"
Task: "Actualizar los stubs de mock restoreShift → discoverOpenShift"
```

---

## Implementation Strategy

### MVP First

El MVP es **Foundational + US1** (Phases 1-3): sin el disparador reactivo, el defecto reportado
—"llega el primer pedido y no me deja cobrar"— sigue exactamente igual. US2 y US3 son
verificaciones de que ese fix no relaja el candado existente, no funcionalidad nueva.

1. Completar Phase 1 (Setup) + Phase 2 (Foundational).
2. Completar Phase 3 (US1) → **validar manualmente el escenario del bug reportado y desplegar**.
3. Completar Phase 4 (US2) y Phase 5 (US3) → validar que no hay regresión de seguridad de caja.
4. Phase 6 (Polish) al final.

### Incremental Delivery

Un solo repositorio (`pos-heladeria`), 0 endpoints nuevos, 0 migraciones (Principio VI): la
corrección completa (Foundational + las tres historias) es en la práctica un único incremento
pequeño y coherente — no tiene sentido desplegar Foundational sin US1, porque no corrige nada por
sí sola.
