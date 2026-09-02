---

description: "Task list for feature implementation"
---

# Tasks: Mejoras de usabilidad en el formulario de administración de promociones

**Input**: Design documents from `/specs/071-mejoras-formulario-promociones/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: incluidos — el plan y los contratos ya comprometen ficheros de test concretos
(Principio X, Verificación Obligatoria), así que cada historia lleva su tarea de test antes de
la implementación.

**Organización**: cada fase es una historia de usuario de spec.md, en el mismo orden de
[plan.md §Fases de entrega](./plan.md#fases-de-entrega-principio-vi). US1 y US4 son ambas P1.

**Repositorios**: las rutas de archivo son relativas a cada repositorio hermano de
`pos-specs` — `pos-backend/...` y `pos-heladeria/...` (ver plan.md §Project Structure).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: se puede hacer en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: US1, US2, US3 o US4

---

## Phase 1: Setup

**Purpose**: confirmar que el entorno de los dos repositorios corre antes de tocar código.

- [X] T001 Levantar `pos-backend` (`uvicorn app.main:app --reload`) y `pos-heladeria` (`npm start`) siguiendo [quickstart.md §1](./quickstart.md#1-preparar-el-entorno)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: línea base de regresión antes de modificar nada — no hay infraestructura nueva que
crear (cero endpoints nuevos, cero migraciones, ver [data-model.md](./data-model.md)).

**⚠️ CRITICAL**: no empezar ninguna historia sin esta línea base en verde.

- [X] T002 Correr la batería completa actual y confirmar 0 fallos: `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` en `pos-backend/`, y `npm test` en `pos-heladeria/` ([quickstart.md §2](./quickstart.md#2-batería-automatizada))

**Checkpoint**: línea base en verde — se puede empezar cualquier historia.

---

## Phase 3: User Story 1 - El resumen de una regla dice qué producto lleva (Priority: P1) 🎯 MVP

**Goal**: la tarjeta colapsada de una regla nombra el/los producto(s) de su conjunto, con el
formato exacto de [contracts/resumen-de-regla.md](./contracts/resumen-de-regla.md).

**Independent Test**: crear una regla de precio de paquete (cantidad mínima 2, valor $12.000)
sobre "Gaseosa - Única", colapsarla y comprobar el texto; repetir con "Banana Split Especial -
Pequeña" y comprobar que cambia.

### Tests for User Story 1

- [X] T003 [P] [US1] Escribir tests en `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.spec.ts` para `ruleSummaryText(ruleIndex)` cubriendo la tabla de casos de [contracts/resumen-de-regla.md §3](./contracts/resumen-de-regla.md#3-tabla-de-casos-para-el-test-de-aceptación) (paquete con un nombre, porcentaje `min_qty=1`, porcentaje `min_qty>1` con varios nombres, conjunto vacío) — deben fallar antes de implementar

### Implementation for User Story 1

- [X] T004 [US1] Reescribir `ruleSummaryText` en `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts` (líneas ~1043-1050) para que reciba `ruleIndex`, resuelva nombres vía `selectedVariantsForRule(ruleIndex)`, reutilice `setDescriptor()` de `promotion-condition.util.ts` para el orden/tope de nombres, arme la oración según [contracts/resumen-de-regla.md §2](./contracts/resumen-de-regla.md#2-plantilla-de-salida), y actualizar el único call site en la plantilla inline (línea ~491, de `ruleSummaryText(rule)` a `ruleSummaryText(ruleIndex)`) — depende de T003

**Checkpoint**: US1 funciona y es verificable de forma independiente.

---

## Phase 4: User Story 2 - Elegir productos por búsqueda, no por catálogo completo (Priority: P2)

**Goal**: el listado bajo una regla solo muestra los productos ya seleccionados; el buscador
(con filtro de categoría) es el único camino para encontrar y agregar otros, según
[contracts/busqueda-y-seleccion.md](./contracts/busqueda-y-seleccion.md).

**Independent Test**: con un catálogo de más de 6 variantes en dos categorías, abrir una regla
sin selección, filtrar por categoría, buscar texto, marcar dos variantes y comprobar que solo
esas dos aparecen en el listado del conjunto; desmarcar una y comprobar que desaparece.

### Tests for User Story 2

- [X] T005 [P] [US2] Escribir tests en `promotions-page.component.spec.ts` para `searchResultsForRule` cubriendo la tabla de [contracts/busqueda-y-seleccion.md §2](./contracts/busqueda-y-seleccion.md#2-searchresultsforrule--tabla-de-casos-fr-007-fr-008) (sin categoría y sin texto → `[]`; sin categoría con texto → coincidencias de todo el catálogo; categoría sin texto → toda la categoría; categoría con texto → intersección), y para que `selectedVariantsForRule` no dependa del filtro activo — deben fallar antes de implementar

### Implementation for User Story 2

- [X] T006 [US2] Renombrar `visibleVariantsForRule` a `searchResultsForRule` en `promotions-page.component.ts` (línea ~867), agregando la guarda de FR-008 ("Todas las categorías" + texto vacío → `[]`) — depende de T005
- [X] T007 [US2] Actualizar `selectAllFilteredForRule` ("Agregar visibles", línea ~877) para unionar contra `searchResultsForRule` en vez de `visibleVariantsForRule` ([contracts/busqueda-y-seleccion.md §4](./contracts/busqueda-y-seleccion.md#4-agregar-visibles-fr-011)) — depende de T006
- [X] T008 [US2] Reestructurar el bloque "CONJUNTO" de la plantilla inline (líneas ~441-476) en dos listados: `selectedVariantsForRule(ruleIndex)` (conjunto actual, siempre visible, FR-006) y `searchResultsForRule(ruleIndex)` (resultados de búsqueda, con filas solo cuando no está vacío), ambos con casilla ligada a `toggleVariantForRule` (FR-009, FR-010) — depende de T007

**Checkpoint**: US1 y US2 funcionan juntas y por separado.

---

## Phase 5: User Story 3 - Una regla nueva aparece primero (Priority: P3)

**Goal**: "+ Agregar regla" inserta la regla nueva en la posición 1, no al final (FR-012).

**Independent Test**: con una promoción de dos reglas, presionar "+ Agregar regla" y comprobar
que la nueva ocupa la posición 1 y las anteriores se corren a la 2 y la 3 sin cambiar su orden
relativo.

### Tests for User Story 3

- [X] T009 [P] [US3] Extender/actualizar el test "FR-001: creación por lote — agregar y quitar reglas" en `promotions-page.component.spec.ts` para afirmar que `addRule()` inserta en la posición 0 y que `ruleFilters` se mueve junto con `form.rules` por índice — debe fallar antes de implementar

### Implementation for User Story 3

- [X] T010 [US3] Cambiar `addRule()` en `promotions-page.component.ts` (línea ~814) de `this.form.rules.push(...)` / `this.ruleFilters.push(...)` a `unshift`, y `expandedRuleIndex.set(this.form.rules.length - 1)` a `expandedRuleIndex.set(0)` — depende de T009

**Checkpoint**: US1, US2 y US3 funcionan juntas y por separado.

---

## Phase 6: User Story 4 - Corregir las reglas de una promoción vigente pausándola (Priority: P1)

**Goal**: en estado `Pausada`, agregar/quitar reglas completas y editar el conjunto de variantes
de las existentes, sin duplicar la promoción; `Activa` sigue bloqueando todo (FR-013 a FR-018,
[contracts/edicion-en-pausada.md](./contracts/edicion-en-pausada.md)). Único cambio de
comportamiento en producción de esta spec — registrado como **A-69**.

**Independent Test**: activar una promoción con una regla y un producto, pausarla, agregar y
quitar productos del conjunto de esa regla, agregar una regla nueva, quitar una regla existente,
reactivarla, y comprobar que el cobro usa las reglas y el conjunto actualizados.

- [X] T011 [US4] Confirmar la precondición del Principio II: `grep -n "^### A-69 " specs/000-reconocimiento/registro-de-anomalias.md` debe devolver una línea ([quickstart.md §0](./quickstart.md#0-precondición-principio-ii--ya-satisfecha)) — ya satisfecha en este repositorio

### Tests for User Story 4

- [X] T012 [P] [US4] Escribir tests nuevos en `pos-backend/app/characterization_tests/test_promotions_rules_admin.py`: `update_shape` permitido en `paused` (agregar producto a una regla existente; agregar una regla nueva; quitar una regla existente conservando al menos una), y confirmar sin tocarlo que `test_ca2_cambiar_reglas_de_una_activa_bloquea` (caso `Activa`) sigue en rojo — deben fallar antes de implementar
- [X] T013 [P] [US4] Escribir tests nuevos en `pos-heladeria/.../promotions-page.component.spec.ts` para `isPaused`, `canEditRuleSet()`, `canEditRuleTypeValue(rule)` (con `rule.isExisting` en `true` y en `false`) y para que `save()` invoque `updateShape` también cuando `isPaused()` — deben fallar antes de implementar

### Implementation for User Story 4

- [X] T014 [US4] Cambiar la condición de estado en `update_shape` (`pos-backend/app/api/v1/promotions/service.py`, línea ~760) de `if promo.status != "draft":` a `if promo.status not in ("draft", "paused"):`, actualizando el mensaje de error ([contracts/edicion-en-pausada.md §1](./contracts/edicion-en-pausada.md#1-backend--patch-promotionsidshape)) — depende de T012
- [X] T015 [P] [US4] Actualizar el `summary` del endpoint en `pos-backend/app/api/v1/promotions/router.py` (líneas ~99-101) de "solo en borrador" a "borrador o pausada"
- [X] T016 [P] [US4] Agregar el campo `isExisting: boolean` a `PromotionRuleForm` en `pos-heladeria/src/app/modules/promotions/interfaces/promotion.interface.ts` ([data-model.md](./data-model.md#único-tipo-nuevo-promotionruleformisexisting))
- [X] T017 [US4] En `promotions-page.component.ts`: agregar el computed `isPaused`; reemplazar `canEditShape()` por `canEditRuleSet()` y `canEditRuleTypeValue(rule)` ([contracts/edicion-en-pausada.md §2](./contracts/edicion-en-pausada.md#2-frontend--dos-permisos-no-uno)); poner `isExisting: false` en `emptyRule()` e `isExisting: true` al mapear las reglas en `openEdit()` — depende de T013, T016
- [X] T018 [US4] Actualizar la plantilla inline: los controles de tipo/valor/cantidad mínima usan `[disabled]="!canEditRuleTypeValue(rule)"`; los de agregar/quitar regla y las casillas del conjunto usan `canEditRuleSet()`; el párrafo explicativo (líneas ~301-309) distingue `Activa` (todo bloqueado), `Pausada` (reglas y conjunto editables, tipo/valor/cantidad mínima de las ya existentes bloqueado) y `Finalizada` (solo lectura) — depende de T017
- [X] T019 [US4] Actualizar `save()` (líneas ~921-944) para llamar `updateShape` también cuando `isPaused()`, no solo cuando `isDraft()` ([contracts/edicion-en-pausada.md §2](./contracts/edicion-en-pausada.md#2-frontend--dos-permisos-no-uno)) — depende de T018

**Checkpoint**: las cuatro historias funcionan juntas y por separado.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [X] T020 [P] Correr la batería completa (backend + frontend) y el recorrido manual completo de [quickstart.md §3-4](./quickstart.md#3-validación-manual-por-historia); confirmar 0 regresiones y que `test_ca2_cambiar_reglas_de_una_activa_bloquea` sigue en verde
- [X] T021 Revisar que todos los textos de UI y mensajes de error nuevos están en español de Colombia (Principio XIII)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: depende de Phase 1 — bloquea todas las historias.
- **US1 (Phase 3)**: depende de Phase 2. Sin dependencia de otras historias.
- **US2 (Phase 4)**: depende de Phase 2. Independiente de US1 (mismo archivo, distinto método —
  ver nota de paralelismo abajo).
- **US3 (Phase 5)**: depende de Phase 2. Independiente de US1/US2.
- **US4 (Phase 6)**: depende de Phase 2. Independiente de US1/US2/US3; toca `promotion.interface.ts`
  y partes de `promotions-page.component.ts` que ninguna otra historia modifica.
- **Polish (Phase 7)**: depende de que las historias que se vayan a entregar estén completas.

### Nota de paralelismo real

Las cuatro historias, dentro de `pos-heladeria`, tocan el **mismo archivo**
(`promotions-page.component.ts`), pero **funciones y bloques de plantilla distintos y no
solapados** (US1: `ruleSummaryText`; US2: `visibleVariantsForRule`→`searchResultsForRule` y el
bloque "CONJUNTO"; US3: `addRule`; US4: `canEditShape`→dos funciones, `save`, `emptyRule`,
`openEdit`). Son **independientes para revisar y probar**, como exige spec.md, pero **no se
pueden implementar literalmente a la vez por dos personas sin coordinar el merge de ese archivo**.
El backend (US4) sí es 100% independiente del frontend. Los marcados `[P]` dentro de cada fase son
paralelizables entre sí; entre fases, tratarlas como secuenciales si las hace una sola persona, o
coordinar el merge si las reparte un equipo.

### Within Each User Story

- Tests antes que implementación (TDD): deben fallar antes de escribir el código.
- US4: tests de backend y de frontend son independientes entre sí ([P]); la implementación de
  backend (T014-T015) y la de frontend (T016-T019) también lo son entre sí, aunque ambas dependen
  de sus propios tests.

---

## Parallel Example: User Story 4

```bash
# Los dos tests de US4 tocan repos distintos — en paralelo:
Task: "Escribir tests de update_shape en paused en pos-backend/app/characterization_tests/test_promotions_rules_admin.py"
Task: "Escribir tests de isPaused/canEditRuleSet/canEditRuleTypeValue en pos-heladeria/.../promotions-page.component.spec.ts"

# T015 (docstring del router) y T016 (campo isExisting) no dependen una de la otra ni de T014:
Task: "Actualizar el summary de PATCH /promotions/{id}/shape en pos-backend/app/api/v1/promotions/router.py"
Task: "Agregar isExisting: boolean a PromotionRuleForm en pos-heladeria/.../promotion.interface.ts"
```

---

## Implementation Strategy

### MVP First

US1 y US4 son ambas P1. El MVP más pequeño y autocontenido es **US1 sola** (Phase 3): un cambio
de una función, sin tocar el backend, con el mayor impacto de legibilidad reportado. Si el
objetivo es cerrar también el punto de fricción operativa (no poder corregir una promoción
vigente sin duplicarla), el MVP debe incluir **US1 + US4**.

1. Completar Phase 1 (Setup) + Phase 2 (Foundational).
2. Completar Phase 3 (US1) → **validar y, si aplica, desplegar (MVP mínimo)**.
3. Completar Phase 6 (US4) → validar y desplegar (cierra el segundo P1).
4. Completar Phase 4 (US2) → validar y desplegar.
5. Completar Phase 5 (US3) → validar y desplegar.
6. Phase 7 (Polish) al final, o después de cada historia si se prefiere validar de a una.

### Incremental Delivery

Cada historia es un incremento entregable por separado (Principio VI): ninguna mezcla
refactorización, arquitectura ni migración de datos con el cambio de comportamiento de US4, y
US4 es la única que toca `pos-backend`.
