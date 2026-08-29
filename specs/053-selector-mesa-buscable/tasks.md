---

description: "Task list template for feature implementation"
---

# Tasks: Selector de mesa buscable en la creación de orden manual

**Input**: Design documents from `/specs/053-selector-mesa-buscable/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [quickstart.md](./quickstart.md)

**Tests**: Se incluyen tareas de test. La Constitución (Principio X, Verificación Obligatoria) exige
verificar toda funcionalidad nueva; el campo `disabled` de `SearchableSelectOption` y la interacción
completa del select de mesas no tienen hoy cobertura, así que esta feature agrega esa cobertura en
vez de asumir el comportamiento.

**Organization**: Dos historias de usuario (spec.md: US1 P1, US2 P1 — ambas P1, se implementan en
conjunto porque comparten el mismo cambio de archivo). Todas las rutas de archivo son relativas al
repositorio de la aplicación `../pos-heladeria` (el código no vive en este repositorio de specs).
No hay ninguna tarea sobre `pos-backend` — esta spec no cambia backend (plan.md, Storage: N/A).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de tareas incompletas)
- **[Story]**: Historia de usuario a la que pertenece la tarea (US1, US2)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

Todas las rutas usan como raíz `pos-heladeria/` (repositorio hermano de este `pos-specs`), según la
`Project Structure` de [plan.md](./plan.md).

---

## Phase 1: Setup

**Purpose**: Confirmar el estado base del entorno antes de tocar cualquier archivo

- [X] T001 Ejecutar la suite de tests existente en `pos-heladeria` (`ng test`) y registrar el estado real como línea base de regresión (Principio X) — confirmado **499/510** (54/59 archivos), igual a la línea base ya conocida tras spec 052
- [X] T002 [P] Confirmar que `ng build` compila sin errores en `pos-heladeria`, como referencia antes del cambio — confirmado, solo warnings preexistentes de budget/CommonJS

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Extender el componente compartido antes de consumirlo desde la pantalla de mesas —
ambas historias de usuario dependen de que `SearchableSelectOption` soporte `disabled` (research.md
D2)

**⚠️ CRITICAL**: Ninguna tarea de las historias de usuario puede empezar hasta que esta fase esté
completa

### Tests for Foundational ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T003 [P] Agregar en `pos-heladeria/src/app/shared/searchable-select/searchable-select.component.spec.ts` un caso que confirme que `selectOption()` NO cambia `value()` ni llama a `onChange` cuando la opción tiene `disabled: true` (research.md D2) — confirmado en rojo (TS2353, el campo no existía) antes de T005-T007
- [X] T004 [P] Agregar en el mismo archivo un caso que confirme que una opción con `disabled: true` sigue apareciendo en `filteredOptions()` (sigue visible en el listado, solo no es seleccionable) (research.md D2, FR-004) — confirmado en rojo antes de T005-T007

### Implementation for Foundational

- [X] T005 En `pos-heladeria/src/app/shared/searchable-select/searchable-select.component.ts`, agregar el campo opcional `disabled?: boolean` a la interfaz `SearchableSelectOption` (línea 14-17 actual) (research.md D2, data-model.md)
- [X] T006 En el mismo archivo, hacer que `selectOption(opt)` (línea 125-129 actual) retorne sin efecto si `opt.disabled` es `true`, antes de tocar `value`/`onChange`/`open` (research.md D2) — T003 en verde tras este cambio
- [X] T007 En el mismo archivo, en el template del `<li>` de cada opción (línea 56-60 actual), agregar una clase condicional para verse no seleccionable cuando `o.disabled` es `true` (p. ej. `text-gray-300 cursor-not-allowed` en vez del resaltado/hover normal), sin ocultar la opción (research.md D2, FR-004) — T004 en verde tras este cambio; 501/512 tests pasan (54/59 archivos), mismos 11 preexistentes

**Checkpoint**: `SearchableSelectComponent` soporta opciones no seleccionables — las historias de
usuario pueden comenzar.

---

## Phase 3: User Story 1 - Elegir una mesa desde un select buscable (Priority: P1) 🎯 MVP

**Goal**: El bloque "Mesas" del panel derecho usa `app-searchable-select` en vez de una rejilla de
botones; buscar por nombre y seleccionar produce el mismo efecto que antes.

**Independent Test**: Abrir la creación de orden manual, abrir el select de mesas, escribir parte
del nombre de una mesa libre distinta a la actual, seleccionarla desde el listado filtrado, y
confirmar que la sesión de armado de pedido pasa a esa mesa.

### Tests for User Story 1 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T008 [P] [US1] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts`, reescribir el caso "el selector de mesas permite cambiar a otra mesa libre, pero no a una ocupada" para: abrir el select de mesas (clic en el botón `app-searchable-select`), escribir en su buscador el número de la mesa libre destino, hacer clic en la opción filtrada, y confirmar `store.selectedTableId()` cambia; luego repetir el intento con la mesa ocupada (buscarla, hacer clic en su opción) y confirmar que `store.selectedTableId()` NO cambia (research.md, impacto en tests) — confirmado en rojo (`Cannot read properties of null`) antes de T010-T012
- [X] T009 [P] [US1] Agregar en el mismo archivo un caso que confirme que el bloque "Mesas" ya no contiene un `<button>` con texto `M{n}` por cada mesa (la rejilla se retiró), sino un único control `app-searchable-select` (research.md, FR-001) — confirmado en rojo antes de T010-T012; se retiró además el caso obsoleto de spec 052 que verificaba la rejilla (research.md, impacto en tests)

### Implementation for User Story 1

- [X] T010 [US1] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`, agregar `FormsModule` (`@angular/forms`) y `SearchableSelectComponent` (`../../../shared/searchable-select/searchable-select.component`) al arreglo `imports` del `@Component` (línea 30 actual) (research.md D1)
- [X] T011 [US1] En el mismo archivo, agregar un `computed` nuevo `mesaOptions` en la clase del componente que mapee `store.tablesView()` a `SearchableSelectOption[]`: `id: t.id`, `label` combinando nombre + estado (research.md D2/D3: `` `${t.name ?? 'Mesa ' + t.number} — ${t.statusLabel}` ``), `disabled: t.statusLabel !== 'Libre' && t.id !== store.selectedTableId()` — importar `computed` de `@angular/core` y `SearchableSelectOption` del componente compartido (research.md D2)
- [X] T012 [US1] En el mismo archivo, reemplazar el bloque `<div class="grid grid-cols-4 gap-2">...</div>` (listado de botones de mesa, post-052) por `<app-searchable-select placeholder="Buscar mesa…" [options]="mesaOptions()" [ngModel]="store.selectedTableId()" (ngModelChange)="selectTable($event)" />` (research.md D1) — T008/T009 en verde tras este cambio

**Checkpoint**: En este punto, la Historia de Usuario 1 debe funcionar y poder probarse de forma
completa e independiente.

---

## Phase 4: User Story 2 - Ver el nombre y el estado de cada mesa en el listado (Priority: P1)

**Goal**: Cada fila del listado del select muestra el nombre de la mesa y su estado; las mesas
ocupadas siguen visibles pero no seleccionables.

**Independent Test**: Abrir el select de mesas y confirmar que cada fila muestra nombre + estado, y
que hacer clic sobre una mesa ocupada (no la ya seleccionada) no cambia la selección.

### Tests for User Story 2 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T013 [P] [US2] Agregar en `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts` un caso que, tras abrir el select de mesas, confirme que el listado (`<li>`) de una mesa libre incluye su nombre y "Libre", y el de una mesa ocupada incluye su nombre y "Ocupada" (research.md D2/D3, FR-003) — confirmado en rojo (la mesa ocupada de prueba resolvía a "En preparación" con el fixture de orden inicial; ajustado para no adjuntar pedido y así aislar el estado base "Ocupada", ver research.md) antes de T010-T012
- [X] T014 [P] [US2] Agregar en el mismo archivo un caso que, tras abrir el select y hacer clic sobre la opción de una mesa ocupada (no la ya seleccionada), confirme que `store.selectedTableId()` no cambia y el select sigue abierto o la opción sigue visible (no se cierra como si hubiera seleccionado) (research.md D2, FR-004) — confirmado en rojo antes de T010-T012

### Implementation for User Story 2

- [X] T015 [US2] Ejecutar T013-T014 contra el resultado de la Fase 3 (T010-T012) y confirmar que pasan sin cambios adicionales de código — `mesaOptions` (T011) ya calcula `label` con nombre+estado y `disabled` con la misma condición que bloqueaba mesas ocupadas en la rejilla, y `SearchableSelectComponent` (T005-T007, Fase 2) ya bloquea la selección de opciones `disabled` — confirmado: pasaron sin cambios adicionales de código (research.md D2)

**Checkpoint**: Ambas historias de usuario deben ser funcionales de forma independiente.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final

- [X] T018 Corrección post-implementación (FR-003, research.md D3): el dueño/desarrollador reportó, tras ver T012 en ejecución, que la etiqueta de mesas con nombre personalizado perdía el número ("Terraza - Libre" en vez de "Mesa 1 - Terraza - Libre"). Se corrigió `mesaOptions` en `manual-order-page.component.ts` para que el número de mesa encabece siempre la etiqueta; se agregó el caso "cuando la mesa tiene un nombre personalizado, el número sigue apareciendo en el listado" en `manual-order-page.component.spec.ts`; se actualizaron spec.md (FR-003, Assumptions) y research.md (D3) para reflejar la decisión corregida — 504/515 tests pasan tras el fix, mismos 11 preexistentes
- [X] T016 [P] Ejecutar manualmente los 2 escenarios de [quickstart.md](./quickstart.md) contra un entorno local con `pos-heladeria` y `pos-backend` corriendo (Principio X) — **parcial**: sin navegador disponible en este entorno de implementación no se pudo ejecutar la QA visual completa. Se verificó en su lugar: `ng build` sin errores y `ng serve` sirviendo `HTTP 200` sin errores de consola en el arranque, más la cobertura automatizada de T003-T004/T008-T009/T013-T014 que ejercita exactamente el mismo comportamiento a nivel de componente (buscar, seleccionar, nombre+estado visibles, mesa ocupada no seleccionable). **Queda pendiente que el usuario/QA ejecute el recorrido visual real** antes de dar la spec por verificada en producción (Principio X)
- [X] T017 Ejecutar la suite completa de tests de `pos-heladeria` (`ng test`) y confirmar que no hay regresiones más allá de los tests nuevos/reescritos de T003-T004, T008-T009 y T013-T014; confirmar que los demás casos de `manual-order-page.component.spec.ts` y `searchable-select.component.spec.ts` siguen en verde sin cambios de intención (research.md, "Resumen de impacto en tests existentes") — **504/515 tests pasan** (54/59 archivos, tras el fix de T018); los 11 que fallan son exactamente los mismos preexistentes ya documentados en specs 046/047/051/052; cero regresiones nuevas

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: Depende de Setup — BLOQUEA ambas historias de usuario (research.md D2: el select debe soportar `disabled` antes de poder usarse para mesas).
- **User Story 1 (Phase 3)**: Puede empezar tras la Fase 2.
- **User Story 2 (Phase 4)**: Depende de que T010-T012 (Historia 1) ya hayan reemplazado la rejilla por el select — T013-T015 verifican comportamiento que T011 ya implementa.
- **Polish (Phase 5)**: Depende de que las Fases 3-4 estén completas.

### Within Each User Story

- Los tests de cada fase se escriben y deben fallar antes de su implementación.
- T010 → T011 → T012 son secuenciales (mismo archivo, cada uno depende del anterior: imports antes que el computed, el computed antes de usarlo en el template).

### Parallel Opportunities

- T001/T002 (Setup) en paralelo.
- T003/T004 (tests Foundational) en paralelo entre sí — mismo archivo, casos independientes.
- T008/T009 (tests US1) en paralelo entre sí.
- T013/T014 (tests US2) en paralelo entre sí.

---

## Parallel Example: Foundational

```bash
# Lanzar juntos los dos tests de la Fase 2 (mismo archivo, casos independientes):
Task: "Confirmar que selectOption() no hace nada con una opción disabled"
Task: "Confirmar que una opción disabled sigue en filteredOptions()"
```

---

## Implementation Strategy

### MVP First (Foundational + User Story 1)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (T003-T007) — el select ya soporta opciones no seleccionables.
3. Completar Fase 3: User Story 1 (T008-T012).
4. **DETENERSE Y VALIDAR**: probar la Historia 1 de forma independiente (buscar y seleccionar una
   mesa desde el select).
5. Desplegar/demostrar si está listo.

### Incremental Delivery

1. Completar Setup + Foundational → el componente compartido ya soporta `disabled`.
2. Agregar Historia 1 (select reemplaza la rejilla) → probar de forma independiente →
   Desplegar/Demo (MVP).
3. Agregar Historia 2 (nombre+estado, mesas ocupadas no seleccionables) → probar de forma
   independiente → Desplegar/Demo.
4. Completar Fase 5: Polish (T016-T017).

---

## Notes

- [P] = archivos distintos o casos de test independientes sin dependencias entre sí.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- No hay ninguna tarea de backend — esta spec no cambia `pos-backend` (plan.md, Storage: N/A).
- Verificar que los tests fallan antes de implementar.
- Commit tras cada tarea o grupo lógico.
