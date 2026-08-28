---

description: "Task list template for feature implementation"
---

# Tasks: Eliminación de Dividir Cuenta y de Combinar Método de Pago en Toda la Aplicación

**Input**: Design documents from `/specs/046-dividir-cuenta-pago-pendiente/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/ui-store-contract.md](./contracts/ui-store-contract.md), [quickstart.md](./quickstart.md)

**Tests**: Se incluyen tareas de test. La Constitución (Principio X, Verificación Obligatoria) exige
verificar toda funcionalidad nueva y toda eliminación de funcionalidad ya cubierta por tests
existentes; `research.md` §5 ya decidió qué specs ajustar y qué caso nuevo agregar (regresión de
FR-003). Los specs no mencionados abajo (`payment-validation-block.component.spec.ts`,
`pos-terminal.store.spec.ts`) deben permanecer en verde sin ningún cambio.

**Organization**: Las tareas están agrupadas por historia de usuario (spec.md) para que cada una se
pueda implementar y probar de forma independiente. Todas las rutas de archivo son relativas al
repositorio de la aplicación `../pos-heladeria` (el código no vive en este repositorio de specs). No
hay ninguna tarea sobre `pos-backend` — esta spec no cambia backend (research.md §4).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de tareas incompletas)
- **[Story]**: Historia de usuario a la que pertenece la tarea (US1, US2, US3)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

Todas las rutas usan como raíz `pos-heladeria/` (repositorio hermano de este `pos-specs`), según la
`Project Structure` de [plan.md](./plan.md).

---

## Phase 1: Setup

**Purpose**: Confirmar el estado base del entorno antes de tocar cualquier archivo

- [X] T001 Ejecutar la suite de tests existente en `pos-heladeria` (`ng test` o el script configurado en `package.json`) y registrar el estado real (verde, o fallas preexistentes ajenas a esta spec) como línea base de regresión (Principio X) — **450/462 tests pasan** (54/59 archivos), 12 fallas preexistentes y ajenas a esta spec: `auth.service.spec.ts` (mustChangePassword), `sidebar.component.spec.ts` (nav super-admin), y una regresión ya documentada de `MoneyInputComponent` (el `<input>` interno pasó de `type="number"` a `type="text"`, dejando sin match a los selectores `input[type="number"]` de varios tests) en `pos-checkout-panel.component.spec.ts` (T025, T026, T032, "pedido ya en cocina" x2) y `session-bill-panel.component.spec.ts` (3 casos de un único método + el de combinar, este último se retira en T012)
- [X] T002 [P] Levantar el entorno de desarrollo local (`pnpm start`/`ng serve` en `pos-heladeria`) y confirmar que compila y sirve sin errores — `ng build` verificado sin errores (solo warnings preexistentes de budget/CommonJS)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Prerrequisitos bloqueantes compartidos por todas las historias

**Nota**: esta feature no tiene prerrequisitos fundacionales bloqueantes ni cambios de backend
(research.md §4) — las tres historias pueden implementarse en cualquier orden. Hay tres puntos de
posible colisión de archivo entre historias, a resolver secuenciando esas tareas puntuales (no
bloquean el resto de cada historia):

- `pos-checkout-panel.component.ts` — tocado por US1 (T005) y US2 (T008).
- `pos-checkout-panel.component.spec.ts` — tocado por US1 (T003, T004) y US2 (T006).
- `session-bill-panel.component.spec.ts` — tocado por US2 (T007) y US3 (T012).

**Checkpoint**: Fase 1 completa — cualquier historia de usuario puede comenzar.

---

## Phase 3: User Story 1 - Ocultar "Liberar Mesa" / "Cerrar Mesa" mientras el pago está pendiente (Priority: P1) 🎯 MVP

**Goal**: El botón "🔓 Liberar Mesa" del panel "Pedido de mostrador" deja de mostrarse mientras la
mesa seleccionada tenga al menos un pago pendiente de confirmar, y reaparece de inmediato en cuanto
se confirma/aprueba ese pago.

**Independent Test**: Seleccionar una mesa con un pago en efectivo o transferencia pendiente de
confirmar y verificar que el panel de mostrador no muestra "Liberar Mesa"; confirmar/aprobar ese
pago y verificar que el botón aparece de inmediato, sin reseleccionar la mesa.

### Tests for User Story 1 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T003 [P] [US1] Agregar caso en `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.spec.ts`: con `store.centralState()` en `'validar-pago'` (mockear `pendingOfSelectedTable`/`orders` de forma que la mesa seleccionada tenga un pedido QR pendiente), el botón "🔓 Liberar Mesa" no debe estar en el DOM aunque `store.sessionBill()` tenga valor
- [X] T004 [P] [US1] Agregar caso en el mismo archivo: tras pasar `store.centralState()` de `'validar-pago'` a `'pedido'` (simulando la confirmación de un pago pendiente sin volver a seleccionar la mesa), el botón "🔓 Liberar Mesa" debe aparecer en el siguiente ciclo de detección de cambios, sin ninguna llamada adicional del test

### Implementation for User Story 1

- [X] T005 [US1] En `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.ts`, envolver el botón "🔓 Liberar Mesa" (líneas 227-233) en la condición adicional `@if (store.centralState() !== 'validar-pago')`, dentro del bloque ya existente `@if (store.sessionBill(); as bill)` (línea 208) — sin tocar `store.releaseTable()` ni ningún otro botón del bloque (research.md §1)

**Checkpoint**: User Story 1 debe funcionar y probarse de forma completamente independiente en este punto.

---

## Phase 4: User Story 2 - "Dividir la cuenta entre varias personas" eliminada por completo de la aplicación (Priority: P2)

**Goal**: El botón y el flujo de "Dividir la cuenta entre varias personas" (spec 010) dejan de
existir en cualquier panel — ni en "Pedido de mostrador" ni en "Cuenta de la mesa" — y el código
que los implementaba, sin ningún punto de entrada restante, se elimina.

**Independent Test**: Recorrer el panel "Pedido de mostrador" y el panel "Cuenta de la mesa" (con
una mesa de más de un comensal) y verificar que "Dividir la cuenta entre varias personas" y el modo
de cobro "Dividir por comensal" ya no existen en ninguno de los dos.

### Tests for User Story 2 ⚠️

- [X] T006 [P] [US2] En `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.spec.ts`, retirar las aserciones que buscan el texto "Dividir la cuenta entre varias personas" (incluida la referencia en la línea 394) y agregar un caso que confirme que ese botón nunca se renderiza, con o sin `store.sessionBill()` — no había aserciones positivas que retirar (solo la negativa ya existente para el caso pagado); se agregaron 2 casos nuevos (mostrador y cobro por sesión de mesa)
- [X] T007 [P] [US2] En `pos-heladeria/src/app/modules/tables/components/session-bill-panel.component.spec.ts`, retirar todos los casos ligados al modo `split` (toggle "Dividir por comensal", `canSplit()`, `buildPayload()` con `billing_mode: 'split'`) y agregar un caso que confirme que el toggle de modo ya no se renderiza — el cobro siempre construye `billing_mode: 'unified'`

### Implementation for User Story 2

- [X] T008 [US2] En `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.ts`: retirar el import de `SplitBillPanelComponent` (línea 13) y de `imports: [...]`; retirar los dos botones "Dividir la cuenta entre varias personas" (líneas 98-104 y 137-143); retirar el bloque `@if (splitOpen() && store.sessionBill(); as bill) { <app-split-bill-panel …> }` (líneas 238-247); retirar el signal `splitOpen` (línea 253); retirar los métodos ahora huérfanos `sessionOrders()` (321-324), `tableLabel()` (327-330) y `onSplitSaved()` (333-336) (research.md §2)
- [X] T009 [US2] En `pos-heladeria/src/app/modules/tables/components/session-bill-panel.component.ts`: retirar el toggle "Cuenta única"/"Dividir por comensal" de la plantilla (líneas 116-140); retirar el signal `mode` (línea 222) — el componente se comporta siempre como `'unified'`; retirar `canSplit()` (línea 246); simplificar `ready()` (253-261) a solo la rama `unified`; retirar la rama `mode() === 'split'` de la plantilla de pago (156-173); retirar el llenado de `splits` en `ngOnChanges()` (283-290) y el signal `splits`/interfaz `SplitDraft` (líneas 34-40, 225); retirar `setSplitPayment()` (299-303); simplificar `buildPayload()` (334-352) para que siempre construya `{ cash_shift_id, billing_mode: 'unified', payments, customer_name? }` — el desglose de solo lectura por comensal (líneas 70-100) NO se toca (research.md §2)
- [X] T010 [US2] Eliminar por completo `pos-heladeria/src/app/modules/tables/components/split-bill-panel.component.ts` y `pos-heladeria/src/app/modules/tables/components/split-bill-panel.component.spec.ts` (research.md §2 — sin punto de entrada tras T008) — **hallazgo adicional durante la implementación**: `addParticipant()`/`removeParticipant()`/`setAssignments()` y `splitIncomplete()` en `pos-heladeria/src/app/modules/tables/services/table-session.service.ts` quedaron también sin ningún llamador (eran exclusivos de `split-bill-panel.component.ts`) — se retiraron por el mismo criterio de FR-006 (no permanece código sin punto de entrada); los endpoints de backend (`participants`/`assignments`) no se tocan (research.md §4)

**Checkpoint**: User Story 1 y 2 deben funcionar de forma independiente en este punto.

---

## Phase 5: User Story 3 - "Combinar método de pago" eliminado por completo de la aplicación (Priority: P2)

**Goal**: Ningún cobro (mostrador o "Cuenta de la mesa") permite registrar más de un método de
pago; cada cobro se hace con un único método por el total exacto o más, y un pago insuficiente se
rechaza en vez de completarse con un segundo método.

**Independent Test**: Abrir el cobro de mostrador y el cobro de "Cuenta de la mesa" y verificar que
ninguno ofrece "Combinar con otro método"; escribir un monto menor al total en cualquiera de los dos
y verificar que el cobro queda bloqueado con el mensaje de falta de saldo, sin ninguna forma de
completarlo con otro método.

### Tests for User Story 3 ⚠️

- [X] T011 [P] [US3] En `pos-heladeria/src/app/modules/tables/services/payment-draft.util.spec.ts`, retirar los casos que ejercitan `combined`/`secondMethodId`/`secondAmount` (p. ej. "manda las dos líneas al combinar métodos") y conservar/ajustar los casos de un único método: falta de monto, "faltan $X para cubrir la cuenta", y "lo no-efectivo no puede superar el total"
- [X] T012 [P] [US3] En `pos-heladeria/src/app/modules/tables/components/session-bill-panel.component.spec.ts`, retirar el caso "cobra con dos métodos y reparte el resto en el segundo" (línea 242) y cualquier otro caso que ejercite un segundo método de pago
- [X] T013 [P] [US3] En `pos-heladeria/src/app/modules/tables/components/payment-attempt-review-panel.component.spec.ts`, agregar un caso nuevo que confirme que `confirmCash()` con un `amountReceived` menor al total de la orden no puede confirmarse — fija en un test el comportamiento que ya provee el backend (`confirm_cash_payment_attempt`, `pos-backend/app/api/v1/orders/checkout.py:942-970`, research.md §3), hoy sin cobertura explícita en el frontend

### Implementation for User Story 3

- [X] T014 [US3] En `pos-heladeria/src/app/modules/tables/services/payment-draft.util.ts`: retirar los campos `combined`/`secondMethodId`/`secondAmount` de `PaymentDraft` (líneas 12-19) y de `emptyPaymentDraft()` (21-23); simplificar `paymentLines()` (26-34) para devolver siempre una única línea; simplificar `paidAmount()` (37-39) a `draft.amount`; simplificar `nonCashAmount()` (42-50) a la rama de un único método; retirar la validación de segundo método de `paymentIssue()` (líneas 77-80) — `changeDue()`/`missingAmount()`/la regla "no-efectivo no puede superar el total" quedan sin cambios de fórmula (research.md §3)
- [X] T015 [US3] En `pos-heladeria/src/app/modules/tables/components/payment-input.component.ts`: retirar el checkbox "Combinar con otro método" (líneas 58-66) y el bloque de segundo método (69-89) de la plantilla; retirar `toggleCombined()`, `setSecondMethod()`, `setSecondAmount()`, `remainder()` (146-164) y el computed `otherMethods()` (114-116, sin uso tras retirar el bloque de segundo método); `setMethod()`/`setAmount()` quedan sin la lógica de `combined`/`secondAmount` (research.md §3)

**Checkpoint**: Las tres historias deben funcionar de forma independiente en este punto.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final sobre las tres historias juntas

- [X] T016 [P] Ejecutar manualmente los 11 escenarios de [quickstart.md](./quickstart.md) contra un entorno local con `pos-heladeria` y `pos-backend` corriendo — **parcial**: sin navegador disponible en este entorno de implementación no se pudo ejecutar la QA manual completa de los 11 escenarios (mesas/turno de caja/pagos QR reales). Se verificó en su lugar: `ng build` sin errores, `ng serve` sirviendo `HTTP 200` sin errores de consola en el arranque, y la cobertura automatizada de T003-T004/T006-T007/T011-T013 que ejercita el mismo comportamiento a nivel de componente. **Queda pendiente que el usuario/QA ejecute el recorrido manual real** antes de dar la spec por verificada en producción (Principio X)
- [X] T017 Ejecutar la suite completa de tests de `pos-heladeria` (`ng test`) y confirmar que no hay regresiones más allá de los ajustes documentados en T003-T004, T006-T007 y T011-T013 — **449/460 tests pasan** (53/58 archivos); los 11 restantes son exactamente los mismos fallos preexistentes de la línea base de T001 (`app.spec.ts`, `auth.service.spec.ts`, `sidebar.component.spec.ts`, y la regresión ya documentada de `MoneyInputComponent`) menos uno, reemplazado intencionalmente por el nuevo test de T012; cero regresiones nuevas. No se tocó ningún archivo de `pos-backend`, así que sus characterization tests no aplican verificarse en esta spec (research.md §4)
- [X] T018 [P] Buscar en `pos-heladeria/src/app/modules/tables/` cualquier referencia residual a `SplitBillPanelComponent`, `split-bill-panel`, `combined`, `secondMethodId`, `secondAmount`, `canSplit`, `SplitDraft`, `SplitPayment` fuera de lo ya cubierto por T008-T010/T014-T015, para confirmar que no queda código huérfano (FR-006/FR-007, "no permanece como superficie viva alternativa") — **hallazgo adicional**: `TableSession.get()` (en `table-session.service.ts`) quedó sin ningún llamador tras retirar `split-bill-panel.component.ts`, pero se conserva por ser un getter genérico de la sesión (no exclusivo de "dividir cuenta") — decisión documentada aquí para trazabilidad. Se retiraron los tipos `ParticipantCreatePayload`/`ItemAssignment`/`AssignmentsPayload` de `dining.interface.ts` (sin ningún uso tras T010); `SplitPayment`/`BillingMode` se conservan porque describen un contrato de backend todavía vigente y no exclusivo de esta funcionalidad (research.md §4)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: Sin tareas bloqueantes; solo documenta colisiones de archivo entre historias (ver arriba).
- **User Stories (Phase 3-5)**: Todas pueden empezar tras la Fase 1. Pueden implementarse en paralelo por distintas personas, o en orden P1 → P2 (US2) → P2 (US3), respetando las colisiones de archivo señaladas en la Fase 2.
- **Polish (Phase 6)**: Depende de que las tres historias estén completas.

### User Story Dependencies

- **User Story 1 (P1)**: Sin dependencia de otras historias.
- **User Story 2 (P2)**: Sin dependencia funcional de US1/US3; comparte archivo con US1 (`pos-checkout-panel.component.ts`/`.spec.ts`) — secuenciar esas tareas puntuales si se implementa en paralelo.
- **User Story 3 (P2)**: Sin dependencia funcional de US1/US2; comparte archivo de test con US2 (`session-bill-panel.component.spec.ts`) — secuenciar T007/T012 si se implementa en paralelo.

### Within Each User Story

- Los tests (T003-T004, T006-T007, T011-T013) se escriben y deben fallar antes de la implementación correspondiente.
- Dentro de US2: T008 y T009 son independientes entre sí (archivos distintos); T010 depende de que T008 ya haya retirado el único invocador de `split-bill-panel.component.ts`.
- Dentro de US3: T014 (util) y T015 (componente) pueden hacerse en cualquier orden, pero ambos deben completarse antes de que T012/T013 puedan pasar en verde (los tests describen el comportamiento final).

### Parallel Opportunities

- T001/T002 (Setup) en paralelo.
- T003/T004 (tests US1) en paralelo entre sí.
- T006/T007 (tests US2) en paralelo entre sí, y en paralelo con T003/T004 si se coordina el orden de edición de `pos-checkout-panel.component.spec.ts`.
- T011/T012/T013 (tests US3) en paralelo entre sí.
- T008/T009 (implementación US2) en paralelo — archivos distintos.
- T016/T018 (Polish) en paralelo.

---

## Parallel Example: User Story 1

```bash
# Lanzar juntos los dos tests de la Historia 1 (mismo archivo, casos independientes):
Task: "Agregar caso: Liberar Mesa oculto en 'validar-pago' en pos-checkout-panel.component.spec.ts"
Task: "Agregar caso: Liberar Mesa reaparece tras confirmar el pago en pos-checkout-panel.component.spec.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (solo notas, sin tareas bloqueantes).
3. Completar Fase 3: User Story 1 (T003-T005).
4. **DETENERSE Y VALIDAR**: probar la Historia 1 de forma independiente (ocultar/reaparecer "Liberar Mesa").
5. Desplegar/demostrar si está listo — es el riesgo operativo más directo del reporte original.

### Incremental Delivery

1. Setup + Foundational → base lista.
2. Agregar Historia 1 → probar de forma independiente → demo (MVP).
3. Agregar Historia 2 → probar de forma independiente → demo.
4. Agregar Historia 3 → probar de forma independiente → demo.
5. Cada historia agrega valor sin romper las anteriores — las tres tocan superficies mayormente independientes (research.md, plan.md Constitution Check).

---

## Notes

- [P] = archivos distintos, sin dependencias entre sí.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- No hay ninguna tarea de backend — esta spec no cambia `pos-backend` (research.md §4, Constitution Check Principio VI/VII).
- Verificar que los tests fallan antes de implementar.
- Commit tras cada tarea o grupo lógico.
- Evitar: tareas vagas, conflictos de mismo archivo sin secuenciar, dependencias cruzadas entre historias que rompan su independencia.
