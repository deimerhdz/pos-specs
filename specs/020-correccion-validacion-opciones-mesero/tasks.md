---

description: "Task list for: Corrección de la validación de opciones en el alta directa del mesero (A-04)"
---

# Tasks: Corrección de la validación de opciones en el alta directa del mesero (A-04)

**Input**: Design documents from `/specs/020-correccion-validacion-opciones-mesero/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md),
[contracts/add-item-to-table-endpoint.md](./contracts/add-item-to-table-endpoint.md),
[quickstart.md](./quickstart.md)

**Tests**: FR-006 exige explícitamente al menos un test de characterization dedicado — los tests
están incluidos abajo, no son opcionales en esta spec.

**Alcance**: todo el trabajo de código vive en el repositorio sibling `../pos-backend` (Constitución
§Alcance). Rutas de fichero relativas a la raíz de `pos-backend`.

**Nota sobre paralelismo**: esta spec toca exactamente **dos ficheros** —
`app/api/v1/orders/consolidation.py` (una sola línea) y
`app/characterization_tests/test_orders_consolidation.py` (tests) — el cambio de producción más
pequeño de todo `pos-specs` hasta ahora (Constitución, Principio III: un módulo, sin extracción).
No hay oportunidades reales de `[P]` entre tareas que editan el mismo fichero de test; se marca
`[P]` solo entre corridas de suites de test independientes en la fase de Polish.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3)

## Phase 1: Setup

**Purpose**: confirmar el estado defectuoso actual antes de tocar código (línea base de
regresión).

- [X] T001 Ejecutar `python3 -m unittest app.characterization_tests.test_orders_consolidation -v`
      y confirmar que `test_add_item_to_table_a04_omite_validacion_de_seleccion_de_opciones`
      (T012) pasa en verde **congelando el defecto** de A-04, y que
      `test_create_order_contraste_a04_si_valida_seleccion_de_opciones` (T013) ya pasa en verde
      documentando el comportamiento correcto de referencia (quickstart.md Paso 1) —
      `app/characterization_tests/test_orders_consolidation.py`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: infraestructura compartida que bloquearía el resto de historias.

Sin tareas — `variant` ya está en el ámbito local de `add_item_to_table`
(`consolidation.py:196`, dos líneas antes de la llamada que cambia) y no se agrega ningún import
ni dependencia nueva (research.md Decisión 1, plan.md Constitution Check, Principio IV). El único
cambio de producción vive enteramente dentro de US1 (Phase 3); no hay nada que otra historia
necesite preparar antes de empezar.

**Checkpoint**: no aplica — se pasa directo a Phase 3.

---

## Phase 3: User Story 1 - El mesero no puede agregar un ítem con sabores/opciones obligatorios incompletos (Priority: P1) 🎯 MVP

**Goal**: `add_item_to_table` rechaza con `422` una selección de opciones que incumple
`min_select`/`max_select` del grupo obligatorio de la variante (FR-001/FR-002), sin crear el
`OrderItem` ni descontar inventario.

**Independent Test**: agregar, vía `add_item_to_table`, una variante con `min_select=3` (que
descuenta inventario) seleccionando solo 2 opciones válidas, y verificar que la operación se
rechaza con `422` sin crear ningún `OrderItem`.

### Implementation for User Story 1

- [X] T002 [US1] En `add_item_to_table`, rama de variante normal (`app/api/v1/orders/consolidation.py:199`),
      cambiar `options = load_valid_options(db, data.option_ids)` por
      `options = load_valid_options(db, data.option_ids, variant=variant)` — research.md Decisión 1,
      data-model.md §Entidad `add_item_to_table`
- [X] T003 [US1] Modificar el test `CONGELA` existente
      `test_add_item_to_table_a04_omite_validacion_de_seleccion_de_opciones`
      (`app/characterization_tests/test_orders_consolidation.py:65-83`) para verificar que la
      llamada lanza `HTTPException` con `status_code == 422` en vez de aceptar la selección vacía
      en silencio (Historia 1, escenario 1). Renombrar a
      `test_add_item_to_table_a04_valida_seleccion_de_opciones_tras_la_correccion`, citando la
      entrada A-04 de `registro-de-anomalias.md` en el docstring — exigido por el Principio II de
      la Constitución y por FR-006 (depende de T002).
      **Recordatorio: el Principio II exige además citar A-04 en el propio mensaje de commit que
      incluya este cambio — no basta con la cita en el docstring.**
- [X] T004 [US1] Añadir test: la misma variante y grupo, seleccionando los 3 sabores correctos
      (`min_select=3`, `max_select=3`), se agrega normalmente — `OrderItem` creado, precio completo
      cobrado, inventario descontado de los 3 sabores, sin cambio frente al comportamiento de hoy
      (Historia 1, escenario 2) en `app/characterization_tests/test_orders_consolidation.py`
      (depende de T002) — `test_add_item_to_table_seleccion_completa_se_acepta_historia_1_escenario_2`
- [X] T005 [US1] Añadir test: la misma variante y grupo (`max_select=3`), seleccionando 4 sabores
      (excede el máximo del grupo obligatorio), se rechaza con `422` — el mismo mecanismo de
      `min_select`/`max_select` cubre tanto elegir de menos como de más (Historia 1, escenario 3)
      en `app/characterization_tests/test_orders_consolidation.py` (depende de T002) —
      `test_add_item_to_table_excede_maximo_del_grupo_rechaza_historia_1_escenario_3`

**Checkpoint**: `add_item_to_table` ya valida `min_select`/`max_select` igual que `create_order` —
verificable de forma aislada con T003-T005, sin tocar todavía el camino de `create_order` ni el de
combos.

---

## Phase 4: User Story 2 - La validación es idéntica sin importar el camino de entrada (Priority: P2)

**Goal**: confirmar que `add_item_to_table` y `create_order` convergen ante la misma selección
(FR-003), y que la corrección no afecta el camino de combos, que no selecciona opciones propias
(FR-004).

**Independent Test**: invocar ambos caminos con el mismo conjunto de opciones fuera de rango y
comparar que ambos rechazan con el mismo código de error.

### Implementation for User Story 2

- [X] T006 [US2] Añadir test de paridad: el mismo escenario de selección vacía en un grupo
      `min_select=1` que descuenta inventario, ejecutado por separado vía `add_item_to_table` y vía
      `create_order`, produce el mismo `status_code` en ambos — cierra la divergencia que motivaba
      A-04 (Historia 2, escenario 1; research.md Decisión 3) en
      `app/characterization_tests/test_orders_consolidation.py` (depende de T002) —
      `test_add_item_to_table_y_create_order_convergen_tras_la_correccion_historia_2_escenario_1`
- [X] T007 [US2] Verificar (sin cambio de código) que
      `test_add_item_to_table_combo_expande_componentes_a_precio_normal`
      (`app/characterization_tests/test_orders_consolidation.py:234`, ya existente y sin
      modificar) sigue en verde tras T002 — confirma que la corrección no introduce ninguna
      exigencia nueva sobre el camino de combos, que ya construye `options=[]` por diseño
      (Historia 2, escenario 2; FR-004) — `app/characterization_tests/test_orders_consolidation.py`
      (depende de T002)

**Checkpoint**: US1 + US2 juntas entregan la corrección completa — `add_item_to_table` y
`create_order` ya no divergen ante ninguna selección, y el camino de combos queda intacto.

---

## Phase 5: User Story 3 - La corrección no altera ningún pedido ni factura ya generados (Priority: P1)

**Goal**: confirmar que el fix no introduce ningún mecanismo de recálculo retroactivo sobre
`OrderItem`s, órdenes o facturas ya existentes (FR-005, Principio V de la Constitución). Historia
de garantía estructural, no de comportamiento nuevo a probar en tiempo de ejecución.

**Independent Test**: revisar que ningún código nuevo introducido por T002 lee ni escribe sobre
`OrderItem`s previamente creados.

### Implementation for User Story 3

- [X] T008 [US3] Revisar `app/api/v1/orders/consolidation.py` completo y confirmar que
      `add_item_to_table` sigue sin ningún recorrido, consulta de actualización en lote ni
      migración sobre `OrderItem`s existentes — el cambio de T002 es local a la creación de un
      ítem nuevo, sin ningún `UPDATE`/backfill que pueda alcanzar filas ya escritas antes del fix
      (FR-005, quickstart.md nota de Paso 2). **Verificación**: `grep -n "db\.execute\|update("`
      sobre el fichero no debe mostrar ninguna consulta nueva fuera de las ya existentes en el
      resto del módulo.
      **Resultado**: las 5 llamadas a `db.execute` en el fichero (líneas 38, 78, 109, 148, 173) son
      todas `select()` de lectura; ninguna `.update()`/bulk mutation. Ninguna toca `OrderItem`s
      preexistentes.

**Checkpoint**: las tres historias quedan cubiertas — la corrección es completa, verificada por
paridad (US2) y confirmada como no retroactiva (US3).

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T009 Ejecutar `python3 -m unittest app.characterization_tests.test_orders_consolidation -v`
      completo y confirmar que los tests preexistentes no tocados (T014-T020: sesión/orden sobre
      la marcha, `consolidate_table`, combo, variante sin receta), el T012 modificado (T003), y los
      tests nuevos (T004, T005, T006) pasan todos en verde, sin ninguna regresión (Principio II) —
      `app/characterization_tests/test_orders_consolidation.py`.
      **Resultado**: 12/12 en verde.
- [X] T010 [P] Ejecutar `python3 -m unittest app.characterization_tests.test_catalog_line_pricing -v`
      y confirmar que `test_rn_cat_33_a04_sin_pasar_variant_load_valid_options_no_valida_nada`
      sigue en verde sin modificarse — confirma que esta corrección no cambió `load_valid_options`
      en sí (spec 004, fuera de alcance) — `app/characterization_tests/test_catalog_line_pricing.py`.
      **Resultado**: 25/25 en verde.
- [X] T011 [P] Ejecutar `python3 -m unittest app.characterization_tests.test_orders_service -v` y
      confirmar que los tests de `create_order` (incluido el de contraste A-04 del lado de
      `service.py`) siguen en verde sin cambios — `app/characterization_tests/test_orders_service.py`.
      **Resultado**: 3/3 en verde.
- [X] T012 Ejecutar `python3 -m unittest discover -s app/characterization_tests -p "test_*.py"`
      (la suite completa de `pos-backend`, no solo los módulos tocados) y confirmar cero
      regresiones fuera del alcance de esta spec (Principio II).
      **Resultado**: 190/190 en verde.
- [X] T013 Recorrer [quickstart.md](./quickstart.md) de punta a punta (Pasos 1-5) contra el
      `pos-backend` con el fix aplicado y confirmar SC-001 a SC-004: SC-001 (T005), SC-002 (T006),
      SC-003 (T009-T012 en verde), SC-004 (T004 + T007, sin cambio en combos ni en selección
      completa).
      **Resultado**: los 5 pasos de quickstart.md se ejecutaron contra `pos-backend` con el fix
      aplicado — SC-001 a SC-004 confirmados, todos en verde.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — arranca de inmediato
- **Foundational (Phase 2)**: vacía — no bloquea nada
- **US1 (Phase 3)**: depende de Setup; es la única historia que toca producción (T002)
- **US2 (Phase 4)**: depende de T002 (US1) — sus tests de paridad y de no-regresión de combos solo
  tienen sentido una vez aplicado el fix; a nivel de *test* T006 y T007 son independientes entre sí
- **US3 (Phase 5)**: depende de T002 (US1) — revisa el código ya cambiado, no introduce cambio
  propio
- **Polish (Phase 6)**: depende de que US1+US2+US3 estén completas

### Notas de secuencia

A diferencia de features más grandes donde cada historia se entrega con su propio cambio de
código, aquí **solo US1 modifica producción** (una línea); US2 y US3 son verificación (paridad y
garantía de no-retroactividad) sobre ese mismo cambio. Sus *tests* sí son independientes entre sí
(T004/T005/T006/T007 no dependen unos de otros, solo de T002) — así que cada historia sigue siendo
*verificable* de forma aislada, tal como exige `spec.md` en su sección "Independent Test" — pero la
*implementación* real de esta spec es un único commit de una línea (T002) más su batería de tests.

### Parallel Opportunities

Ninguna entre tareas que editan `test_orders_consolidation.py` (mismo fichero). En Polish, T010 y
T011 corren suites sobre ficheros distintos entre sí y respecto a T009 — pueden lanzarse en
paralelo.

---

## Parallel Example: Polish

```bash
# Estas tres corridas de suite son independientes entre sí (ficheros de test distintos):
python3 -m unittest app.characterization_tests.test_orders_consolidation -v   # T009
python3 -m unittest app.characterization_tests.test_catalog_line_pricing -v   # T010
python3 -m unittest app.characterization_tests.test_orders_service -v        # T011
```

---

## Implementation Strategy

### Orden recomendado (no hay "MVP parcial" real en esta spec)

Dado que toda la implementación es una línea de producción (T002), la entrega real es:

1. Phase 1 (Setup): confirmar el defecto congelado (T001).
2. Phase 3 (US1): aplicar T002, luego T003-T005 — al terminar esta fase, `add_item_to_table` ya
   queda corregido y verificable de forma aislada.
3. Phase 4 (US2): T006-T007 — confirma paridad con `create_order` y que combos no se ven afectados.
4. Phase 5 (US3): T008 — confirma por revisión que no hay recálculo retroactivo.
5. Phase 6 (Polish): corridas completas + validación de quickstart.md.

### Incremental Delivery

No aplica despliegue por historia (un solo endpoint, sin flag de feature, un solo commit de
producción) — el PR natural de esta spec entrega US1+US2+US3 juntas, dado su tamaño (1 línea de
producción + 3 tests nuevos + 1 test modificado). El desglose por historia existe para
trazabilidad de requisito → tarea → test (FR-001/FR-002→US1, FR-003/FR-004→US2, FR-005→US3), no
para sugerir despliegues separados.
