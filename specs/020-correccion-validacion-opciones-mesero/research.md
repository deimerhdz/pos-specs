# Phase 0 Research: Corrección de la validación de opciones en el alta directa del mesero (A-04)

Esta spec no tiene incógnitas de "qué tecnología usar" — el código, el lenguaje y el framework de
test ya existen, y `spec.md` ya fija el criterio a replicar (`create_order`) con cita de línea
exacta. Lo que queda para esta fase es confirmar, con evidencia directa de `git show` (no
suposiciones), que el fix de una línea restaura exactamente `03469ca` sin reintroducir la
regresión de `ee94f30`, y decidir cómo tratar el test `CONGELA` existente que documenta el
defecto a propósito.

## Decisión 1 — El fix es agregar `variant=variant` en `consolidation.py:199`, nada más

**Decision**: en `add_item_to_table`, dentro de la rama `else` (variante normal, no combo), la
línea `options = load_valid_options(db, data.option_ids)` pasa a ser
`options = load_valid_options(db, data.option_ids, variant=variant)`.

**Rationale — verificado con `git show` sobre `pos-backend`, no supuesto**:

1. **`03469ca` (2026-08-03, el fix original)** tocó exactamente una línea de este fichero: cambió
   `load_valid_options(db, data.option_ids)` por
   `load_valid_options(db, data.option_ids, variant=variant)`, sobre la versión de
   `add_item_to_table` vigente en ese momento (`5550cc5`, previa a la expansión de combos).
2. **`ee94f30` (2026-08-04, un día después, autor distinto — `LeonardoGomezz`)** reestructuró la
   misma función para soportar combos (`data.combo_id`, expansión vía `promotions.expand_combo`,
   lista `lines` con `(product_variant_id, quantity, options, unit_price, combo_id)`), pero su
   diff parte del mismo commit base `5550cc5` que `03469ca` — **antes** del fix, no después. El
   resultado es que la línea reestructurada vuelve a ser
   `options = load_valid_options(db, data.option_ids)`, sin `variant=`: la regresión no fue un
   cambio deliberado, fue que la rama de combos nunca tuvo el fix de `03469ca` en su historia al
   momento de escribirse, y el merge posterior no lo recuperó.
3. `variant` sigue estando en el ámbito local exacto donde se necesita: la rama `else` ya hace
   `variant = get_or_404(db, ProductVariant, data.product_variant_id, "Variant not found")` dos
   líneas antes (línea 196 en el código actual) — el mismo nombre de variable que usaba `03469ca`
   sobre la versión pre-combos. Agregar `variant=variant` es sintácticamente el mismo cambio que
   hizo `03469ca`, aplicado sobre la estructura post-`ee94f30` vigente hoy.

Con esto, el fix de una línea es **exactamente** la reaplicación de `03469ca` sobre el código
actual: no reintroduce nada de lo que `ee94f30` cambió (la expansión de combos, el manejo de
`lines`, `deduct_order_items` en lote) porque esa reestructuración es ortogonal a la línea que
cambia — vive fuera de la rama `else` en el caso de combos, y dentro de ella no toca nada más que
la llamada a `load_valid_options`.

**Alternatives considered**:
- Revertir `ee94f30` sobre `consolidation.py` y reaplicar `03469ca` limpio: rechazado — perdería la
  expansión de combos (`data.combo_id`), que es funcionalidad vigente y en uso, no parte del
  defecto. La anomalía A-04 es específica al `else` de variante normal; los combos ya construyen
  su lista de opciones vacía (`[]`) por diseño (no seleccionan opciones propias, FR-004 de
  `spec.md`) y no la ejercitan.
- Envolver la llamada en una función auxiliar nueva (p. ej. `_load_options_for_add_item`) para
  "encapsular" el fix: rechazado — no hay ninguna razón para la abstracción; es una línea dentro de
  una función ya pequeña, y crear un helper de una sola línea de uso único no simplifica nada
  (regla de no introducir abstracciones no solicitadas).

## Decisión 2 — Modificación del test `CONGELA` existente (Principio II)

**Decision**: se modifica `test_add_item_to_table_a04_omite_validacion_de_seleccion_de_opciones`
(`test_orders_consolidation.py:65-83`) para verificar el comportamiento corregido — que la llamada
ahora lanza `HTTPException` con `422` en vez de aceptar la selección vacía en silencio — citando en
el mismo commit la entrada A-04 de `registro-de-anomalias.md`, tal como exige el Principio II. El
nombre del test se actualiza para dejar de describir "omite validación" y pasar a describir el
comportamiento corregido, conservando la referencia a A-04.

**Rationale**: el propio docstring del módulo ya anuncia esta spec ("el fix de una línea...
queda para una spec delta posterior", `test_orders_consolidation.py:11-12`) — el test se escribió
sabiendo que algún día se modificaría con esta cita exacta. Dejarlo intacto tras aplicar el fix lo
dejaría en rojo permanentemente sin decisión que lo ampare, el escenario que el Principio II
prohíbe. La cita ya existe (A-04, "Tratamiento acordado" en `registro-de-anomalias.md`, reforzada
por P4 de `entrevista-negocio.md`) y `spec.md` ya la referencia extensamente.

El test de contraste `test_create_order_contraste_a04_si_valida_seleccion_de_opciones`
(`test_orders_consolidation.py:87-106`) **no cambia** — ya verifica el comportamiento correcto de
`create_order`, que esta delta no toca; sigue siendo válido como referencia de paridad.

**Alternatives considered**: dejar el test T012 intacto (congelando el defecto) y añadir uno nuevo
separado que verifique el comportamiento corregido: rechazado — dejaría dos tests contradictorios
sobre el mismo camino de código, uno de los cuales quedaría en rojo tras el fix (viola el propio
Principio II). Modificar el existente, citando la decisión, es el mecanismo que el Principio II
define para este caso — mismo patrón ya usado en la spec 019 (Decisión 4 de su `research.md`).

## Decisión 3 — Caso de paridad nuevo (FR-003/FR-006)

**Decision**: se añade un caso de prueba que ejecuta el mismo escenario (variante con
`min_select=1` que descuenta inventario, selección vacía) por los dos caminos —
`add_item_to_table` y `create_order` — y verifica que ambos rechazan con el mismo código `422`,
documentando la paridad que FR-003 exige y que antes de esta corrección no existía.

**Rationale**: `test_create_order_contraste_a04_si_valida_seleccion_de_opciones` ya prueba
`create_order` de forma aislada; lo que falta tras el fix es un test que capture explícitamente que
ya **no hay contraste** — ambos caminos convergen. Vive en el mismo fichero
(`test_orders_consolidation.py`), reutilizando los helpers `_seed_variant_con_receta`/
`_seed_grupo_obligatorio_que_descuenta` ya existentes, sin fixtures nuevas.

**Alternatives considered**: extender el test modificado de la Decisión 2 para que además llame a
`create_order` y compare — rechazado, mezclaría dos aserciones de propósito distinto (comportamiento
de `add_item_to_table` en sí vs. paridad entre caminos) en un solo test, dificultando saber cuál
falló si algo se rompe.

## Decisión 4 — Convención de ejecución de tests

**Decision**: `python3 -m unittest`, igual que el resto de `app/characterization_tests/`.

**Rationale**: verificado — no hay `pytest.ini` ni configuración de `pytest` en `pos-backend`;
`test_orders_consolidation.py` ya usa `unittest.TestCase` y documenta su propio comando de
ejecución (`python -m unittest app.characterization_tests.test_orders_consolidation -v`, línea
16 de su docstring). Introducir `pytest` solo para esta spec rompería esa uniformidad sin ningún
beneficio pedido y contaría como dependencia nueva a justificar (Principio IV) sin necesidad real.

**Alternatives considered**: `pytest` — rechazado por la razón de arriba, sin necesidad real que lo
justifique.

## Resumen — incógnitas resueltas

No queda ningún `NEEDS CLARIFICATION` pendiente en el Technical Context de `plan.md`: lenguaje,
dependencias, storage, testing, plataforma y alcance ya estaban determinados por el código
existente. Esta fase añadió las cuatro decisiones de arriba — en particular la Decisión 1, que
prueba con `git show` real (no supuesto) que el fix de una línea restaura `03469ca` sin
reintroducir nada de lo que `ee94f30` cambió legítimamente.
