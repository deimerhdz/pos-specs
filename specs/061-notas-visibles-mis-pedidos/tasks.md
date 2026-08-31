# Tasks: Notas del ítem visibles en "Mis pedidos" del Menú QR

**Input**: Design documents from `/specs/061-notas-visibles-mis-pedidos/`

**Prerequisites**: plan.md, spec.md, research.md (D1), quickstart.md

**Tests**: incluidos — Principio X (Verificación obligatoria) exige un caso de test nuevo para esta
corrección (plan.md, Constitution Check).

**Organization**: una sola historia de usuario (US1, la única definida en spec.md); sin fase de
Setup ni Foundational — no hay infraestructura compartida que crear, el componente y su archivo de
test ya existen.

## Phase 1: User Story 1 - El comensal ve la nota que escribió en su propio pedido (Priority: P1) 🎯 MVP

**Goal**: en la sección "Mis pedidos" del Menú QR, cada línea de ítem con `notes` no vacío muestra
esa nota; los ítems sin nota no cambian.

**Independent Test**: `npx ng test --watch=false public-menu.component` (quickstart.md) — pedido
con dos ítems idénticos en producto/opciones, uno con nota y otro sin nota, verificando que la nota
aparece asociada solo a su línea.

### Tests for User Story 1

- [x] T001 [US1] Agregar caso de test en
  `../pos-heladeria/src/app/modules/tables/pages/public-menu.component.spec.ts`: seedear
  `component.myOrders.set([...])` con un pedido que tenga dos ítems del mismo
  `product_variant_id`/opciones, uno con `notes: 'sin banana'` y otro con `notes: null`, hacer
  `component.section.set('pedidos')` + `fixture.detectChanges()`, y verificar que el texto
  `'sin banana'` aparece en el DOM exactamente una vez (cubre FR-001 a FR-003, Acceptance Scenarios
  1, 2 y 4 de spec.md). Confirmar que el test FALLA contra el código actual (sin el cambio de T002)
  antes de implementar.

### Implementation for User Story 1

- [x] T002 [US1] En
  `../pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts`, dentro del `@for (item
  of order.items ?? []; ...)` de la sección "Mis pedidos" (líneas 241-255), agregar un bloque
  `@if (item.notes)` inmediatamente después del `@if (optionLabels(item))` existente, con la misma
  clase `text-xs text-gray-400 pl-5`, mostrando `item.notes` (research.md D1). No agregar ningún
  método nuevo a la clase del componente — `item.notes` se lee directo del binding.
- [x] T003 [US1] Ejecutar `npx ng test --watch=false public-menu.component` en `../pos-heladeria` y
  confirmar que el caso de T001 pasa y que los tests existentes del archivo (Bug 1, Bug 3 de la spec
  041) siguen pasando sin cambios (Principio II, comportamiento existente protegido).

**Checkpoint**: única historia de usuario completa — feature lista para validación manual
(quickstart.md, sección "Validación manual end-to-end").

---

## Dependencies & Execution Order

- T001 antes que T002 (test primero, debe fallar contra el código actual).
- T002 depende de T001 (implementación que hace pasar el test).
- T003 depende de T002.
- Sin dependencias con ninguna otra spec ni feature — cambio aislado a un único archivo de
  producción y su archivo de test ya existente.

## Implementation Strategy

MVP = única historia. Completar T001 → T002 → T003 y correr `quickstart.md` (validación manual)
antes de dar la spec por implementada.

## Notes

- Sin tareas de Setup/Foundational: no hay estructura de proyecto, dependencia ni infraestructura
  nueva que crear (research.md D1, plan.md Technical Context).
- Sin tareas `[P]`: T001-T003 tocan el mismo par de archivos en secuencia (test → implementación →
  verificación), no hay paralelismo real posible.
