---

description: "Task list for: Corrección de la cuenta de mesas fusionadas (group_bill)"
---

# Tasks: Corrección de la cuenta de mesas fusionadas (`group_bill`)

**Input**: Design documents from `/specs/019-correccion-cuenta-mesas-fusionadas/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/group-bill-endpoint.md](./contracts/group-bill-endpoint.md),
[quickstart.md](./quickstart.md)

**Tests**: FR-006 exige explícitamente al menos un test de characterization dedicado — los tests
están incluidos abajo, no son opcionales en esta spec.

**Alcance**: todo el trabajo de código vive en el repositorio sibling `../pos-backend` (Constitución
§Alcance). Rutas de fichero relativas a la raíz de `pos-backend`.

**Nota sobre paralelismo**: esta spec toca exactamente **dos ficheros** —
`app/api/v1/orders/tables_advanced.py` (implementación) y
`app/characterization_tests/test_orders_tables_advanced.py` (tests) — un patrón esperado en una
corrección puntual de ~15 líneas (Constitución, Principio III: un módulo, sin extracción). No hay
oportunidades reales de `[P]` entre tareas que editan el mismo fichero; se marca `[P]` solo la
tarea de verificación de FR-004, que no edita ningún fichero.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3)

## Phase 1: Setup

**Purpose**: confirmar el estado defectuoso actual antes de tocar código (línea base de
regresión).

- [X] T001 Ejecutar `python3 -m unittest app.characterization_tests.test_orders_tables_advanced -v`
      y confirmar que el test T046 (`test_group_bill_a01_camino_c_incluye_todos_los_status_sin_descuentos`)
      pasa en verde **congelando el defecto** de A-01 camino C (quickstart.md Paso 1) —
      `app/characterization_tests/test_orders_tables_advanced.py`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: infraestructura mínima que bloquea tanto US1 como US2

**⚠️ CRITICAL**: ninguna historia puede implementarse hasta completar esta fase

- [X] T002 Añadir los imports `from app.api.v1.orders import checkout` y
      `from app.api.v1.promotions import service as promotions` en
      `app/api/v1/orders/tables_advanced.py` (research.md Decisión 1; `checkout` aporta `TERMINAL`,
      `order_sale_lines`, `promo_lines_for` — necesarios tanto para excluir terminales (US1) como
      para aplicar descuentos (US2))

**Checkpoint**: imports listos — puede empezar la implementación de US1 y US2.

---

## Phase 3: User Story 1 - No se vuelve a cobrar una orden ya pagada (Priority: P1) 🎯 MVP

**Goal**: `group_bill` excluye del cálculo cualquier orden `pagada`/`cancelada` del grupo (FR-001),
igual que `_billable_orders` ya excluye en `table_sessions.compute_bill`.

**Independent Test**: `GET /orders/group/{group_id}/bill` sobre un grupo con una orden `pagada` y
otra `abierta` (sin ninguna promoción vigente) → el total no incluye la orden pagada.

### Implementation for User Story 1

- [X] T003 [US1] En `group_bill` (`app/api/v1/orders/tables_advanced.py:92-114`), separar la carga
      de órdenes en dos conjuntos: todas las del `merged_group_id` (para el chequeo de existencia
      404, sin cambios) y las billables (`CustomerOrder.status.notin_(checkout.TERMINAL)`, filtrado
      en SQL) — research.md Decisión 2

      **Nota de implementación**: se optó por una sola consulta (todas las órdenes del grupo,
      igual que antes) con un `if o.status in checkout.TERMINAL` dentro del bucle en vez de dos
      queries separadas — porque `orders[]` de todas formas debe seguir listando las órdenes
      terminales (Decisión 3), así que había que cargarlas de cualquier manera; la rama en Python
      evita pagar el costo de `order_sale_lines`/`promotions.evaluate` para ellas sin necesitar una
      segunda consulta SQL. Resultado equivalente al descrito, sin violar el criterio de eficiencia
      de la Decisión 2.
- [X] T004 [US1] Ajustar el bucle de `group_bill` para que el `total` sea la suma de los
      `subtotal` **solo** de las órdenes billables; las órdenes terminales permanecen listadas en
      `orders[]` con `subtotal=Decimal("0")`, sin contribuir al `total` (research.md Decisión 3;
      Edge Case "todas pagada/cancelada" → `total=0`, no error) — `app/api/v1/orders/tables_advanced.py`
      (depende de T003)
- [X] T005 [US1] Añadir test: grupo con orden A `pagada` ($20.000) + orden B `abierta` ($15.000,
      sin promoción) → `total == Decimal("15000")`, orden A excluida del cálculo pero presente en
      `orders[]` (Historia 1, escenario 1) en
      `app/characterization_tests/test_orders_tables_advanced.py` (depende de T004) —
      `test_group_bill_excluye_orden_pagada_del_total_historia_1_escenario_1`
- [X] T006 [US1] Añadir test: orden `cancelada` excluida del total igual que `pagada` (Historia 1,
      escenario 2) en `app/characterization_tests/test_orders_tables_advanced.py` (depende de T004) —
      `test_group_bill_excluye_orden_cancelada_del_total_historia_1_escenario_2`
- [X] T007 [US1] Añadir test: grupo con **todas** sus órdenes en `pagada`/`cancelada` →
      `total == Decimal("0")`, sin `HTTPException` (Edge Case de `spec.md`, research.md Decisión 3)
      en `app/characterization_tests/test_orders_tables_advanced.py` (depende de T004) —
      `test_group_bill_todas_terminales_da_total_cero_sin_error`

**Checkpoint**: `group_bill` ya no vuelve a cobrar órdenes cerradas — verificable de forma aislada
con T005-T007, sin que exista todavía ninguna promoción en juego. El test legacy T046 (que sí
combina status **y** promoción en un mismo escenario) queda pendiente de actualizar hasta
completar US2 — ver nota en Phase 4.

---

## Phase 4: User Story 2 - La cuenta refleja promociones y combos vigentes (Priority: P1)

**Goal**: `group_bill` aplica `promotions.evaluate`/`combo_discount_for_lines` sobre las líneas
cobrables del grupo (FR-002), preservando la exclusión de ítems `anulado` (FR-003, ya heredada de
`checkout.order_sale_lines`).

**Independent Test**: `GET /orders/group/{group_id}/bill` sobre un grupo con una sola orden
`abierta` con una promoción vigente aplicable (sin ninguna orden `pagada`/`cancelada` en el grupo)
→ el total ya descuenta esa promoción.

### Implementation for User Story 2

- [X] T008 [US2] Reemplazar el cálculo por-ítem de `group_bill` (`unit_price * quantity` bruto) por,
      para cada orden billable: `lines = checkout.order_sale_lines(db, o.id)`,
      `raw = sum(l.line_total for l in lines)`,
      `promo, _ = promotions.evaluate(db, checkout.promo_lines_for(db, lines), now)`,
      `combo = promotions.combo_discount_for_lines(db, lines, now)`, `sub = raw - promo - combo`
      (`now = datetime.now(timezone.utc)`) — FR-002/FR-003, research.md Decisión 1 — en
      `app/api/v1/orders/tables_advanced.py` (depende de T002, T004)
- [X] T009 [US2] Añadir test: orden `abierta` sola con una promoción `percent` del 10% vigente
      sobre su categoría, $15.000 brutos, sin ninguna orden terminal en el grupo →
      `total == Decimal("13500")` (Historia 2, escenario 1) en
      `app/characterization_tests/test_orders_tables_advanced.py` (depende de T008) —
      `test_group_bill_aplica_promocion_percent_vigente_sin_terminales_historia_2_escenario_1`
- [X] T009b [US2] **Añadida durante `/speckit-implement` para cerrar el hallazgo G1 de
      `/speckit-analyze`** (FR-002/SC-002 cubrían combos en el código pero ningún test ejercitaba
      `combo_discount_for_lines`): test con un combo vigente (2 unidades de una variante a $10.000
      c/u, bundle de $18.000) → `total == Decimal("18000")`, sin ninguna orden terminal en el
      grupo, en `app/characterization_tests/test_orders_tables_advanced.py` (depende de T008) —
      `test_group_bill_aplica_combo_vigente_sin_terminales_fr_002`
- [X] T009c [US2] **Añadida durante `/speckit-implement` para cerrar el hallazgo G2 de
      `/speckit-analyze`** (FR-006 exige verificar FR-001/FR-002/**FR-003** pero ningún test
      re-verificaba la exclusión de ítems `anulado` a nivel de `group_bill`): test con un ítem
      `anulado` y uno no-anulado en la misma orden `abierta` → el `subtotal` de esa orden solo
      cuenta el no-anulado, en `app/characterization_tests/test_orders_tables_advanced.py`
      (depende de T008) — `test_group_bill_excluye_items_anulados_de_orden_billable_fr_003`
- [X] T010 [US2] Modificar el test `CONGELA` existente
      `test_group_bill_a01_camino_c_incluye_todos_los_status_sin_descuentos`
      (`app/characterization_tests/test_orders_tables_advanced.py:124-160`) para verificar el
      comportamiento corregido: orden A `pagada` $20.000 + orden B `abierta` $15.000 brutos con 10%
      vigente → `total == Decimal("13500")`, no `Decimal("35000")` (Historia 2, escenario 2 /
      SC-005, el ejemplo cuantitativo de A-01). Renombrado a
      `test_group_bill_a01_camino_c_excluye_pagadas_y_aplica_promocion_vigente`, con la entrada
      A-01 de `registro-de-anomalias.md` citada en el docstring — exigido por el Principio II de la
      Constitución y por FR-006 (depende de T008).
      **Recordatorio (hallazgo C1 de `/speckit-analyze`): el Principio II exige además citar A-01
      en el propio mensaje de commit que incluya este cambio ("sin esa cita, el cambio no se
      hace") — no basta con la cita en el docstring.**

**Checkpoint**: US1 + US2 juntas entregan el `group_bill` corregido completo; T005-T007, T009,
T009b, T009c y el T046 modificado (T010) pasan en verde simultáneamente — ningún test `CONGELA`
queda rojo sin cita.

---

## Phase 5: User Story 3 - Mismo resultado que una mesa individual (Priority: P2)

**Goal**: para un mismo conjunto de órdenes, `group_bill` produce el mismo total que
`table_sessions.compute_bill` si esas órdenes pertenecieran a una única mesa sin fusionar (FR-005).
Historia de verificación/consistencia — no introduce comportamiento propio nuevo.

**Independent Test**: sembrar una mesa individual con una combinación de órdenes
`abierta`/`pagada`/`cancelada` y una promoción vigente; calcular su cuenta por
`table_sessions.compute_bill`, fusionarla sola en un grupo, calcular `group_bill`, comparar
totales.

### Implementation for User Story 3

- [X] T011 [US3] Añadir test que siembra una `table_session` con `fx.make_table_session`/
      `fx.make_customer_order`/`fx.make_order_item` (mezcla de status y una promoción vigente),
      calcula `table_sessions.service.compute_bill(db, ts.id).total`, fusiona esa misma mesa sola
      con `tables_advanced.merge_orders`, calcula `tables_advanced.group_bill(db, group_id)["total"]`,
      y verifica `assertEqual` centavo a centavo (SC-003, quickstart.md Paso 5) en
      `app/characterization_tests/test_orders_tables_advanced.py` (depende de T008) —
      `test_group_bill_igual_a_compute_bill_para_mesa_fusionada_sola_historia_3`

**Checkpoint**: las tres historias quedan cubiertas — `group_bill` corregido y verificado contra la
referencia correcta y vigente (`table_sessions.compute_bill`).

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T012 Ejecutar `python3 -m unittest app.characterization_tests.test_orders_tables_advanced -v`
      completo y confirmar que los 4 tests preexistentes no tocados (T042-T045), el T046 modificado
      (T010), y los tests nuevos (T005-T007, T009, T009b, T009c, T011) pasan todos en verde, sin
      ninguna regresión (Principio II) — `app/characterization_tests/test_orders_tables_advanced.py`.
      **Resultado**: 12/12 tests en verde. También se corrió
      `python3 -m unittest discover -s app/characterization_tests -p "test_*.py"` (los 187 tests de
      `pos-backend`, no solo este módulo) para confirmar cero regresión fuera de este fichero —
      **187/187 en verde**.
- [X] T013 [P] Revisar `app/api/v1/orders/tables_advanced.py` y confirmar que `group_bill` sigue
      sin escribir en la base de datos (sin `db.commit()`/`db.add()`/`db.flush()` nuevos) — cálculo
      puramente de lectura en tiempo de consulta, sin recalcular ni alterar cuentas ya cobradas
      (FR-004). **Resultado**: `grep -n "db\.\(commit\|add\|flush\)"` sobre el fichero solo devuelve
      líneas dentro de `set_table_status`/`move_order`/`merge_orders` (líneas 42, 62, 72, 90) —
      ninguna dentro de `group_bill`.
- [X] T014 Recorrer [quickstart.md](./quickstart.md) de punta a punta (Pasos 1-5) contra el
      `pos-backend` con el fix aplicado y confirmar SC-001 a SC-005. **Resultado**: SC-001
      (`test_group_bill_excluye_orden_pagada_del_total_historia_1_escenario_1`), SC-002
      (`test_group_bill_aplica_promocion_percent_vigente_sin_terminales_historia_2_escenario_1` +
      `test_group_bill_aplica_combo_vigente_sin_terminales_fr_002`), SC-003
      (`test_group_bill_igual_a_compute_bill_para_mesa_fusionada_sola_historia_3`), SC-004 (T013),
      y SC-005 (`test_group_bill_a01_camino_c_excluye_pagadas_y_aplica_promocion_vigente`, total
      `$13.500` en vez de `$35.000`) — todos verificados en verde.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — arranca de inmediato
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA todas las historias
- **US1 (Phase 3)**: depende de Foundational
- **US2 (Phase 4)**: depende de Foundational; T008 depende además de T004 (US1) porque itera sobre
  el conjunto de órdenes billables que T003/T004 ya dejaron filtrado — **no es independiente de
  US1 a nivel de código**, aunque sí lo es a nivel de test (T009 no necesita ninguna orden
  terminal)
- **US3 (Phase 5)**: depende de T008 (US2) — la consistencia con `compute_bill` solo tiene sentido
  una vez `group_bill` aplica también los descuentos
- **Polish (Phase 6)**: depende de que US1+US2+US3 estén completas

### Notas de secuencia (a diferencia del patrón general de historias independientes)

A diferencia de features más grandes donde cada historia se puede entregar por separado, aquí
**US1 y US2 modifican la misma función de ~15 líneas** y comparten el mismo test legacy (T046).
Sus *tests* sí son independientes entre sí (T005-T007 no requieren ninguna promoción; T009 no
requiere ninguna orden terminal) — así que cada historia sigue siendo *verificable* de forma
aislada, tal como exige `spec.md` en su sección "Independent Test" — pero su *implementación* se
entrega en la práctica en un solo cambio de código (T003-T004 y T008 tocan la misma función), y el
test T046 solo puede quedar en verde una vez ambas están aplicadas (T010 depende de T008). La
recomendación de "Implementation Strategy" abajo refleja esto.

### Parallel Opportunities

Ninguna entre tareas de implementación o de test (mismo fichero cada una). Única tarea paralela
real: T013 (revisión de solo-lectura, no edita ningún fichero que las demás tareas estén editando).

---

## Implementation Strategy

### Orden recomendado (no hay "MVP parcial" real en esta spec)

Dado que US1 y US2 comparten función y test legacy (ver nota de arriba), la entrega work real es:

1. Phase 1 (Setup) → Phase 2 (Foundational).
2. Phase 3 (US1): filtro de status + sus 3 tests nuevos — queda verificable de forma aislada
   (T005-T007 en verde) sin tocar aún el test legacy T046.
3. Phase 4 (US2): lógica de descuentos + su test nuevo + la actualización de T046 (que exige que
   T008 de US2 ya esté aplicado) — al terminar esta fase, `group_bill` queda completamente
   corregido y **todo** el módulo de test pasa en verde de nuevo, incluido T046.
4. Phase 5 (US3): test de consistencia contra `compute_bill` — puramente de verificación, sin
   tocar `group_bill` de nuevo.
5. Phase 6 (Polish): corrida completa + revisión FR-004 + validación de quickstart.md.

### Incremental Delivery

No aplica despliegue por historia (endpoint único, sin flag de feature) — el PR natural de esta
spec entrega US1+US2+US3 juntas, dado su tamaño (~15 líneas de producción). El desglose por
historia existe para trazabilidad de requisito → tarea → test (FR-001→US1, FR-002/FR-003→US2,
FR-005→US3), no para sugerir despliegues separados.
