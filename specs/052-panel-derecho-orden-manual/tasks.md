---

description: "Task list template for feature implementation"
---

# Tasks: Panel derecho unificado — tipo de orden, mesas y pedido en la creación de orden manual

**Input**: Design documents from `/specs/052-panel-derecho-orden-manual/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [quickstart.md](./quickstart.md)

**Tests**: Se incluyen tareas de test. La Constitución (Principio X, Verificación Obligatoria) exige
verificar toda funcionalidad nueva; la nueva ubicación de los controles y el nuevo ancho del panel
no tienen hoy cobertura (es un layout que no existía antes de esta spec), así que esta feature
agrega esa cobertura en vez de asumir el comportamiento.

**Organization**: Dos historias de usuario (spec.md: US1 P1, US2 P2). Todas las rutas de archivo
son relativas al repositorio de la aplicación `../pos-heladeria` (el código no vive en este
repositorio de specs). No hay ninguna tarea sobre `pos-backend` — esta spec no cambia backend
(plan.md, Storage: N/A).

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

- [X] T001 Ejecutar la suite de tests existente en `pos-heladeria` (`ng test`) y registrar el estado real como línea base de regresión (Principio X) — confirmado **495/506** (54/59 archivos), igual a la línea base ya conocida tras spec 051
- [X] T002 [P] Confirmar que `ng build` compila sin errores en `pos-heladeria`, como referencia antes del cambio — confirmado, solo warnings preexistentes de budget/CommonJS

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Prerrequisitos bloqueantes compartidos por todas las historias

**Nota**: esta feature no tiene prerrequisitos fundacionales bloqueantes ni cambios de backend ni
de store (plan.md, Storage: N/A) — es un reordenamiento de bloques de template ya existentes dentro
de un único componente.

**Checkpoint**: Fase 1 completa — las historias de usuario pueden comenzar.

---

## Phase 3: User Story 1 - Configurar tipo de orden, mesa y pedido sin cruzar la pantalla (Priority: P1) 🎯 MVP

**Goal**: "Tipo de Orden" y "Mesas" viven en el panel derecho, junto con "Nueva orden"; la barra
superior solo conserva "← Volver a la Terminal".

**Independent Test**: Abrir la creación de orden manual y verificar que "Tipo de Orden", "Mesas",
"Nueva orden" (carrito), el resumen de totales y "Confirmar y Enviar" están todos dentro del mismo
panel derecho, sin necesidad de tocar el catálogo de la izquierda.

### Tests for User Story 1 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T003 [P] [US1] Agregar en `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts` un caso que ubique el botón "← Volver a la Terminal", suba a su contenedor inmediato (`closest('div')`) y confirme que ese contenedor NO incluye el texto "Tipo de Orden" (la barra superior queda reducida al botón de volver) (research.md D1, FR-005) — confirmado en rojo contra el código anterior antes de T006-T008
- [X] T004 [P] [US1] Agregar en el mismo archivo un caso que ubique el panel derecho (contenedor con clases `border-l` + `bg-white`, `fixture.nativeElement.querySelector('.border-l.bg-white')`) y confirme que su `textContent` incluye "Tipo de Orden", "Mesas", "Nueva orden" y "Confirmar y Enviar", pero NO incluye "Volver a la Terminal" (research.md D1, FR-001/FR-002/FR-006) — confirmado en rojo antes de T006-T008
- [X] T005 [P] [US1] Agregar en el mismo archivo un caso que confirme que el contenedor del listado de mesas ya no tiene la clase `overflow-x-auto` y sí tiene las clases `grid` y `grid-cols-4` (research.md D2, FR-002) — confirmado en rojo antes de T006-T008

### Implementation for User Story 1

- [X] T006 [US1] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`, simplificar la barra superior (líneas 34-96 actuales) para que solo contenga el botón "← Volver a la Terminal" (líneas 38-44 actuales), retirando de ahí el `<h2>Tipo de Orden</h2>`, las tres pestañas (líneas 47-70 actuales) y el bloque "Mesas" + listado (líneas 75-95 actuales) (research.md D1, FR-005)
- [X] T007 [US1] En el mismo archivo, insertar el bloque de pestañas de tipo de orden (retirado en T006) como un nuevo bloque `shrink-0` al inicio del panel derecho (línea ~155 actual, `<div class="w-full sm:w-[320px] ...">`), antes del bloque "Nueva orden" que ya empieza ahí (research.md D1, FR-001)
- [X] T008 [US1] En el mismo archivo, insertar el bloque "Mesas" + listado (retirado en T006) como un segundo bloque `shrink-0` en el panel derecho, inmediatamente después del bloque de T007 y antes de "Nueva orden"; cambiar el contenedor del listado de `flex gap-2 overflow-x-auto pb-1` a `grid grid-cols-4 gap-2`, y quitar `shrink-0 w-20` de cada botón de mesa (research.md D2, FR-002) — T003-T005 en verde tras T006-T008

**Checkpoint**: En este punto, la Historia de Usuario 1 debe funcionar y poder probarse de forma
completa e independiente.

---

## Phase 4: User Story 2 - Panel derecho con más espacio (Priority: P2)

**Goal**: El panel derecho es más ancho que antes de esta spec (320px → 400px), sin que ninguna de
sus tres secciones quede recortada.

**Independent Test**: Abrir la creación de orden manual y confirmar que el panel derecho mide más
que 320px, y que "Tipo de Orden", "Mesas" y "Nueva orden" se ven completas sin scroll horizontal
interno.

### Tests for User Story 2 ⚠️

> **NOTE: Escribir este test primero y confirmar que falla antes de implementar**

- [X] T009 [P] [US2] Agregar en `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts` un caso que confirme que el panel derecho (`querySelector('.border-l.bg-white')`) tiene la clase `sm:w-[400px]` y NO tiene la clase `sm:w-[320px]` (research.md D3, FR-007) — confirmado en rojo antes de T010

### Implementation for User Story 2

- [X] T010 [US2] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`, cambiar la clase del contenedor del panel derecho de `w-full sm:w-[320px]` a `w-full sm:w-[400px]` (research.md D3, FR-007) — depende de T006-T008 (mismo bloque ya reestructurado); T009 en verde tras este cambio

**Checkpoint**: Ambas historias de usuario deben ser funcionales de forma independiente.

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final

- [X] T011 [P] Ejecutar manualmente los 3 escenarios de [quickstart.md](./quickstart.md) contra un entorno local con `pos-heladeria` y `pos-backend` corriendo (Principio X) — **parcial**: sin navegador disponible en este entorno de implementación no se pudo ejecutar la QA visual completa. Se verificó en su lugar: `ng build` sin errores y `ng serve` sirviendo `HTTP 200` sin errores de consola en el arranque, más la cobertura automatizada de T003-T005/T009 que ejercita exactamente el mismo comportamiento a nivel de componente (ubicación de los controles dentro del panel derecho, rejilla de mesas, ancho del panel). **Queda pendiente que el usuario/QA ejecute el recorrido visual real** (incluyendo confirmar que las 8 mesas se ven completas en la rejilla y que ninguna sección del panel se ve recortada) antes de dar la spec por verificada en producción (Principio X)
- [X] T012 Ejecutar la suite completa de tests de `pos-heladeria` (`ng test`) y confirmar que no hay regresiones más allá de los tests nuevos de T003-T005 y T009; confirmar que los 13 casos existentes de `manual-order-page.component.spec.ts` (8 originales + 5 de spec 051) siguen en verde sin cambios de intención (research.md, "Resumen de impacto en tests existentes") — **499/510 tests pasan** (54/59 archivos); los 11 que fallan son exactamente los mismos preexistentes ya documentados en specs 046/047/051 (`app.spec.ts`, `auth.service.spec.ts`, `sidebar.component.spec.ts`, regresión de `MoneyInputComponent`); cero regresiones nuevas; los 13 casos previos de `manual-order-page.component.spec.ts` siguen en verde sin cambios de intención

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: Sin tareas bloqueantes.
- **User Story 1 (Phase 3)**: Puede empezar tras la Fase 1.
- **User Story 2 (Phase 4)**: Depende de que T006-T008 (Historia 1) ya hayan reestructurado el panel derecho — T010 solo cambia una clase de ancho sobre el mismo contenedor que T006-T008 ya movieron.
- **Polish (Phase 5)**: Depende de que las Fases 3-4 estén completas.

### Within Each User Story

- Los tests de cada historia se escriben y deben fallar antes de su implementación.
- T006 → T007 → T008 son secuenciales (mismo archivo, mismo bloque de template, cada uno depende del resultado del anterior).
- T010 depende de T006-T008 (ver Phase Dependencies).

### Parallel Opportunities

- T001/T002 (Setup) en paralelo.
- T003/T004/T005 (tests US1) en paralelo entre sí — mismo archivo pero casos independientes.
- T009 (test US2) puede escribirse en paralelo con T003-T005, aunque su implementación (T010) depende de que la Historia 1 ya haya movido el contenedor.

---

## Parallel Example: User Story 1

```bash
# Lanzar juntos los tres tests de la Historia 1 (mismo archivo, casos independientes):
Task: "Confirmar que la barra superior solo tiene 'Volver a la Terminal'"
Task: "Confirmar que 'Tipo de Orden'/'Mesas'/'Nueva orden' están en el mismo panel derecho"
Task: "Confirmar que el listado de mesas usa grid-cols-4, sin overflow-x-auto"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (sin tareas).
3. Completar Fase 3: User Story 1 (T003-T008).
4. **DETENERSE Y VALIDAR**: probar la Historia 1 de forma independiente (controles unificados en el
   panel derecho).
5. Desplegar/demostrar si está listo.

### Incremental Delivery

1. Completar Setup + Foundational → base lista.
2. Agregar Historia 1 (unificación en el panel derecho) → probar de forma independiente →
   Desplegar/Demo (MVP).
3. Agregar Historia 2 (panel más ancho) → probar de forma independiente → Desplegar/Demo.
4. Completar Fase 5: Polish (T011-T012).

---

## Notes

- [P] = archivos distintos o casos de test independientes sin dependencias entre sí.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- No hay ninguna tarea de backend — esta spec no cambia `pos-backend` (plan.md, Storage: N/A).
- Verificar que los tests de US1/US2 fallan antes de implementar.
- Commit tras cada tarea o grupo lógico.
