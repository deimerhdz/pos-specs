---

description: "Task list template for feature implementation"
---

# Tasks: Campo "Cliente" con valor por defecto "Consumidor final" en la creación de orden manual

**Input**: Design documents from `/specs/054-campo-cliente-orden-manual/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [quickstart.md](./quickstart.md)

**Tests**: Se incluyen tareas de test. La Constitución (Principio X, Verificación Obligatoria) exige
verificar toda funcionalidad nueva; el campo "Cliente", su valor por defecto y su persistencia no
tienen hoy ninguna cobertura, así que esta feature agrega esa cobertura en vez de asumir el
comportamiento.

**Organization**: Tres historias de usuario (spec.md: US1 P1, US2 P2, US3 P1). Todas las rutas de
archivo son relativas al repositorio de la aplicación `../pos-heladeria` (el código no vive en este
repositorio de specs). No hay ninguna tarea sobre `pos-backend` — `customer_name` ya existe y ya se
acepta sin cambios (plan.md, Storage: N/A).

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

- [X] T001 Ejecutar la suite de tests existente en `pos-heladeria` (`ng test`) y registrar el estado real como línea base de regresión (Principio X) — confirmado **504/515** (54/59 archivos), igual a la línea base ya conocida tras spec 053
- [X] T002 [P] Confirmar que `ng build` compila sin errores en `pos-heladeria`, como referencia antes del cambio — confirmado, solo warnings preexistentes de budget/CommonJS

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Prerrequisitos bloqueantes compartidos por todas las historias

**Nota**: esta feature no tiene prerrequisitos fundacionales bloqueantes ni cambios de backend
(plan.md, Storage: N/A) — `store.customerName` y el envío de `customer_name` ya existen
(data-model.md); todo el trabajo es agregar el campo de UI que los use.

**Checkpoint**: Fase 1 completa — las historias de usuario pueden comenzar.

---

## Phase 3: User Story 1 - Ver el campo "Cliente" con "Consumidor final" por defecto (Priority: P1) 🎯 MVP

**Goal**: El campo "Cliente" aparece con "Consumidor final" ya diligenciado, en modo de solo
lectura, sin que el mesero tenga que hacer nada.

**Independent Test**: Abrir la creación de orden manual y verificar que el campo "Cliente" ya
muestra "Consumidor final" sin ninguna interacción previa.

### Tests for User Story 1 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T003 [P] [US1] Agregar en `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts` un caso que confirme que, tras seleccionar una mesa (por parámetro de ruta o por el selector), existe un campo "Cliente" (un `<input>` ubicado justo después del `<h3>` "Cliente") con `value === 'Consumidor final'` y `readOnly === true` (research.md D1/D2, FR-001/FR-002) — confirmado en rojo (`Cannot read properties of undefined`) antes de T004-T005

### Implementation for User Story 1

- [X] T004 [US1] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`, agregar un método privado `applyDefaultCustomerName()` que aplique `'Consumidor final'` a `store.customerName` cuando esté vacío (research.md D1), y llamarlo al final de `ngOnInit()` (línea ~230-234 actual) y dentro de `selectTable(id)` (línea ~240-242 actual), después de `this.store.selectTable(...)` en ambos casos
- [X] T005 [US1] En el mismo archivo, agregar el bloque de template "Cliente" (`<h3>Cliente</h3>` + `<div class="relative"><input [value]="store.customerName()" [readOnly]="!editandoCliente()" ... /></div>`) entre el bloque "Mesas" (línea ~139-145 actual) y el bloque "Nueva orden" (línea ~148-153 actual); agregar el signal `editandoCliente = signal(false)` a la clase (research.md D2) — T003 en verde tras este cambio

**Checkpoint**: En este punto, la Historia de Usuario 1 debe funcionar y poder probarse de forma
completa e independiente.

---

## Phase 4: User Story 2 - Editar el nombre del cliente (Priority: P2)

**Goal**: Un botón ✏️ activa la edición del campo "Cliente"; al perder el foco, el campo vuelve a
solo lectura, con el nombre escrito o con "Consumidor final" si quedó vacío.

**Independent Test**: Activar la edición del campo "Cliente", escribir un nombre distinto, y
confirmar que el campo lo refleja tras perder el foco.

### Tests for User Story 2 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T006 [P] [US2] Agregar en `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts` un caso que haga clic en el botón ✏️ junto al campo "Cliente" y confirme que el `<input>` pasa a `readOnly === false` (research.md D2, FR-003) — confirmado en rojo antes de T009-T010
- [X] T007 [P] [US2] Agregar en el mismo archivo un caso que active la edición, escriba un nombre (`(input)` + valor), dispare `(blur)`, y confirme que el campo muestra ese nombre y vuelve a `readOnly === true` (research.md D2, FR-004) — confirmado en rojo antes de T009-T010
- [X] T008 [P] [US2] Agregar en el mismo archivo un caso que active la edición, deje el campo vacío (`value = ''` + `(input)`), dispare `(blur)`, y confirme que el campo vuelve a mostrar `'Consumidor final'` (research.md D3, FR-005) — confirmado en rojo antes de T009-T010

### Implementation for User Story 2

- [X] T009 [US2] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`, agregar al botón ✏️ del bloque "Cliente" (dentro del `<div class="relative">` de T005) `(click)="toggleEditarCliente()"`, y a la clase el método `toggleEditarCliente()` (pone `editandoCliente` en `true`) — mismo patrón `relative`/`absolute` que `password-input.component.ts:26-53` (research.md D2) — T006 en verde tras este cambio
- [X] T010 [US2] En el mismo archivo, agregar al `<input>` de T005 `(input)="store.customerName.set($any($event.target).value)"` y `(blur)="onClienteBlur()"`; agregar el método `onClienteBlur()` que pone `editandoCliente` en `false` y llama a `applyDefaultCustomerName()` (T004) (research.md D2/D3) — T007/T008 en verde tras este cambio

**Checkpoint**: En este punto, las Historias de Usuario 1 y 2 deben funcionar ambas de forma
independiente.

---

## Phase 5: User Story 3 - El nombre del cliente se guarda al crear la orden (Priority: P1)

**Goal**: Al confirmar y enviar la orden, el nombre que se ve en el campo "Cliente" (por defecto o
editado) queda guardado como `customer_name` de la orden creada.

**Independent Test**: Crear una orden manual (con el valor por defecto o con uno editado) y
confirmar, inspeccionando la llamada de creación, que la orden resultante tiene ese mismo nombre de
cliente.

### Tests for User Story 3 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T011 [P] [US3] Agregar en `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts` un caso que: seleccione una mesa, agregue un producto al draft, espíe `DiningSessionService.createManualOrder` (`vi.spyOn(...).mockResolvedValue(...)`) y `store.reload` (`mockResolvedValue(undefined)`, para no encadenar las peticiones de recarga), haga clic en "Confirmar y Enviar" sin tocar el campo "Cliente", y confirme que `createManualOrder` fue llamado con `customer_name: 'Consumidor final'` (research.md, "Resumen de impacto en tests existentes", FR-006) — pasó desde T004/T010 sin necesitar T014 (customerName ya tenía el default antes de confirmar)
- [X] T012 [P] [US3] Agregar en el mismo archivo un caso equivalente que edite el campo "Cliente" a un nombre específico antes de confirmar, y confirme que `createManualOrder` fue llamado con ese mismo nombre en `customer_name` (FR-006) — pasó desde T004/T010
- [X] T013 [P] [US3] Agregar en el mismo archivo un caso que active la edición del campo "Cliente", lo deje vacío, y confirme la orden **sin perder el foco primero** (llamando directamente al método `confirm()`/haciendo clic en "Confirmar y Enviar" mientras el input sigue en modo edición) — confirmar que `createManualOrder` de todas formas fue llamado con `customer_name: 'Consumidor final'`, nunca vacío (research.md D3, Edge Cases, FR-005) — confirmado en rojo (recibía `customer_name: null`, vía `this.customerName().trim() || null` en `createManualOrderFromDraft()`) antes de T014

### Implementation for User Story 3

- [X] T014 [US3] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`, en el método `confirm()` (línea ~252-257 actual), llamar a `applyDefaultCustomerName()` (T004) antes de `await this.store.createManualOrderFromDraft()` (research.md D3) — T013 en verde tras este cambio; se agregó además `vi.spyOn(router, 'navigate').mockResolvedValue(true)` a T011-T013 (no estaba en el plan original) para evitar un "Unhandled Rejection" de navegación sin rutas configuradas en el módulo de test, mismo patrón que ya usaban los casos existentes de "Confirmar y Enviar"

**Checkpoint**: Las tres historias de usuario deben ser funcionales de forma independiente.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final

- [X] T015 [P] Ejecutar manualmente los 3 escenarios de [quickstart.md](./quickstart.md) contra un entorno local con `pos-heladeria` y `pos-backend` corriendo (Principio X) — **parcial**: sin navegador disponible en este entorno de implementación no se pudo ejecutar la QA visual completa. Se verificó en su lugar: `ng build` sin errores y `ng serve` sirviendo `HTTP 200` sin errores de consola en el arranque, más la cobertura automatizada de T003/T006-T008/T011-T013 que ejercita exactamente el mismo comportamiento a nivel de componente (valor por defecto, edición, blur, persistencia con y sin edición, campo vacío sin blur). **Queda pendiente que el usuario/QA ejecute el recorrido visual real** (incluyendo confirmar en el backend real que la orden creada queda con el `customer_name` correcto) antes de dar la spec por verificada en producción (Principio X)
- [X] T016 Ejecutar la suite completa de tests de `pos-heladeria` (`ng test`) y confirmar que no hay regresiones más allá de los tests nuevos de T003, T006-T008 y T011-T013; confirmar que los 21 casos existentes de `manual-order-page.component.spec.ts` siguen en verde sin cambios de intención (research.md, "Resumen de impacto en tests existentes") — **511/522 tests pasan** (54/59 archivos); los 11 que fallan son exactamente los mismos preexistentes ya documentados en specs 046/047/051/052/053; cero regresiones nuevas

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: Sin tareas bloqueantes.
- **User Story 1 (Phase 3)**: Puede empezar tras la Fase 1.
- **User Story 2 (Phase 4)**: Depende de que T005 (Historia 1) ya haya agregado el campo — T009/T010 extienden el mismo bloque de template.
- **User Story 3 (Phase 5)**: Depende de que T004 (Historia 1) ya exista (`applyDefaultCustomerName()`) — T014 solo agrega una llamada a un método ya creado.
- **Polish (Phase 6)**: Depende de que las Fases 3-5 estén completas.

### Within Each User Story

- Los tests de cada historia se escriben y deben fallar antes de su implementación.
- T004 → T005 son secuenciales (el template de T005 usa el signal/método que T004 no crea — en
  realidad T004 crea `applyDefaultCustomerName()`, y T005 agrega `editandoCliente` — ambos pueden
  hacerse en cualquier orden dentro de la misma tarea de archivo, pero T005 depende de que el campo
  exista para que T003 tenga algo que verificar).
- T009 → T010 son secuenciales (mismo bloque de template, mismo `<input>`).

### Parallel Opportunities

- T001/T002 (Setup) en paralelo.
- T006/T007/T008 (tests US2) en paralelo entre sí.
- T011/T012/T013 (tests US3) en paralelo entre sí.

---

## Parallel Example: User Story 3

```bash
# Lanzar juntos los tres tests de la Historia 3 (mismo archivo, casos independientes):
Task: "Confirmar sin editar envía 'Consumidor final' como customer_name"
Task: "Confirmar tras editar envía el nombre editado como customer_name"
Task: "Confirmar con el campo vacío en modo edición (sin blur) igual envía 'Consumidor final'"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (sin tareas).
3. Completar Fase 3: User Story 1 (T003-T005).
4. **DETENERSE Y VALIDAR**: probar la Historia 1 de forma independiente (campo "Cliente" visible
   con "Consumidor final").
5. Desplegar/demostrar si está listo.

### Incremental Delivery

1. Completar Setup + Foundational → base lista.
2. Agregar Historia 1 (campo visible con valor por defecto) → probar de forma independiente →
   Desplegar/Demo (MVP).
3. Agregar Historia 3 (persistencia al confirmar) → probar de forma independiente →
   Desplegar/Demo — priorizada antes que la Historia 2 por ser P1 igual que la 1.
4. Agregar Historia 2 (edición) → probar de forma independiente → Desplegar/Demo.
5. Completar Fase 6: Polish (T015-T016).

---

## Notes

- [P] = archivos distintos o casos de test independientes sin dependencias entre sí.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- No hay ninguna tarea de backend — `customer_name` ya existe y ya se acepta sin cambios (plan.md,
  Storage: N/A).
- Verificar que los tests fallan antes de implementar.
- Commit tras cada tarea o grupo lógico.
