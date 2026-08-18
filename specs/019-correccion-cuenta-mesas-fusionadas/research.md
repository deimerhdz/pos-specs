# Phase 0 Research: Corrección de la cuenta de mesas fusionadas (`group_bill`)

Esta spec no tiene incógnitas de "qué tecnología usar" — el código, el lenguaje y el framework de
test ya existen y `spec.md` ya fija el criterio a replicar (`table_sessions.compute_bill`) con cita
de línea exacta. Lo que queda para esta fase es resolver el **cómo** exacto de la corrección: la
única decisión no trivial es si `group_bill` debe agrupar líneas por comensal (como hace
literalmente `compute_bill`) o puede seguir agrupando por orden (su propia estructura actual) sin
perder equivalencia. Cada afirmación de abajo se verificó leyendo el código real de `pos-backend`
(no se asume nada no confirmado en el repositorio).

## Decisión 1 — Evaluar promociones/combos por orden, no por comensal

**Decision**: `group_bill` evalúa `promotions.evaluate`/`combo_discount_for_lines` una vez **por
orden billable** (usando `checkout.order_sale_lines(db, order.id)`, sin filtrar por
`participant_id`), no una vez por comensal a través de todas las órdenes del grupo como hace
`compute_bill` (`table_sessions/service.py:160-173`).

**Rationale — prueba de equivalencia**: se verificó leyendo `promotions/service.py` completo que
`evaluate_detailed` (líneas 290-333) y `combo_discount_for_lines` (líneas 394-441) tienen una
propiedad clave para esta decisión:

1. **`evaluate_detailed` es puramente por-línea.** El bucle `for index, line in enumerate(lines)`
   llama a `_best_line_match` usando *solo* los campos de esa línea (`quantity`, `product_id`,
   `category_id`, `line_total`/`unit_price` propios) — nunca agrega `quantity` entre líneas del
   lote recibido. El umbral `min_qty`/`_pack_terms` de una promoción se compara contra la cantidad
   de **esa** línea, no contra la suma del lote. Consecuencia directa: agrupar N líneas en un solo
   lote o repartirlas en varios lotes más pequeños (por orden, por comensal, o cualquier otra
   partición) produce **exactamente el mismo descuento por línea**, y por tanto la misma suma total
   — el agrupamiento es irrelevante para este tipo de promoción.
2. **`combo_discount_for_lines` agrupa internamente por `combo_id`**, no por lo que el caller le
   pase agrupado — el `by_combo: dict[UUID, list]` se reconstruye desde `line.combo_id` dentro de
   la propia función. Solo importa que **todas** las líneas de un mismo `combo_id` estén presentes
   en la misma llamada; en qué unidad más grande (orden, comensal, grupo completo) se las
   incluyó no cambia el resultado si esa condición se cumple.
3. Los ítems de un mismo combo se crean **atómicamente dentro de una sola orden**: el combo se
   arma al consolidar un carrito en una orden (`cart_item.combo_id` → `order_item.combo_id`,
   verificado en el flujo de consolidación de la spec 009), no hay operación en el sistema que
   reparta componentes de un mismo combo entre dos órdenes distintas. Agrupar por orden, entonces,
   siempre incluye el combo completo en una sola llamada — la misma garantía que agrupar por
   comensal le da a `compute_bill` hoy, salvo el caso extremo de que los ítems de un mismo combo se
   reasignen a comensales distintos *dentro* de la misma orden (ver Edge case abajo).

Con (1) y (2)+(3), agrupar por orden en `group_bill` da el mismo total, línea por línea, que
agrupar por comensal en `compute_bill` para el mismo conjunto de órdenes — que es exactamente lo
que exige FR-005/SC-003 (coincidencia de **totales**, no de la estrategia interna de agrupamiento).

**Por qué esta opción y no replicar literalmente el bucle por comensal de `compute_bill`**:

- `GroupBillResponse`/`GroupBillOrderLine` (`orders/schemas.py:77-88`) no tiene desglose por
  comensal — solo `orders[].subtotal` por orden. Replicar el bucle por comensal habría exigido
  además decidir cómo repartir el descuento de cada comensal de vuelta a sus órdenes de origen
  (prorrateo con redondeo) solo para rellenar un campo que la propia spec no pide cambiar de forma
  (Out of Scope: "esta delta... no cambia el contrato de respuesta"). Agrupar por orden evita ese
  problema por construcción: cada orden ya sabe su propio `subtotal` neto, sin prorrateo.
- Agrupar por orden es el cambio de menor superficie sobre el `group_bill` legado: conserva su
  estructura de iteración (`for o in orders: ...`) y su invariante ya existente
  `total == sum(per_order["subtotal"])` — la última línea del `group_bill` actual ya construye
  `total` sumando cada `sub` de orden. Replicar el bucle por comensal habría requerido reestructurar
  esa función alrededor de una entidad (comensal) que no existe en su forma de retorno.
- Ninguna historia ni FR de `spec.md` pide igualar la *estrategia interna* de `compute_bill`, solo
  su *resultado* ("mismo criterio de `_billable_orders`", "mismo criterio de evaluación... que ya
  aplica `table_sessions.compute_bill`", "resultado... idéntico"): son requisitos sobre el
  comportamiento observable (qué se cobra), no sobre la forma del código que lo calcula.

**Edge case documentado (no cubierto por ningún escenario de `spec.md`)**: si algún día los ítems
de un mismo combo llegaran a tener `participant_id` distintos *dentro de la misma orden* (ninguna
operación actual del sistema lo produce, verificado), agrupar por orden seguiría detectando el
combo completo correctamente — a diferencia de `compute_bill`, que al filtrar por comensal
*fragmentaría* ese combo y podría no alcanzar el mínimo de bundle. Este caso no está entre los
Edge Cases de `spec.md` (que solo define el criterio de desempate entre promociones, ya cubierto
por `evaluate_detailed`) y no le compete a esta spec inventar un criterio nuevo — se documenta aquí
como propiedad conocida de la implementación elegida, no como una discrepancia con FR-005 (no hay
ninguna combinación real de datos, dado cómo se crean los combos, que la ejercite).

**Alternatives considered**:
- Replicar el bucle por comensal de `compute_bill` verbatim, agregando líneas de todas las órdenes
  del grupo por `participant_id` y luego prorrateando el descuento resultante de vuelta a cada
  orden para rellenar `subtotal`: rechazado — introduce lógica de prorrateo/redondeo (repartir un
  descuento agregado entre N órdenes garantizando que la suma cierre exacta al centavo) que ningún
  requisito pide, es una abstracción no solicitada (regla de "no diseñar para lo hipotético"), y no
  cambia el resultado observable respecto a la opción elegida (ver prueba arriba).
- Evaluar promociones sobre **todas** las líneas del grupo en un solo lote (sin partición alguna) y
  luego repartir el descuento a cada orden: mismo problema de prorrateo que la alternativa
  anterior, sin ningún beneficio adicional — la partición por orden ya da el resultado correcto sin
  prorrateo.

## Decisión 2 — Dónde vive la exclusión de órdenes terminales (FR-001)

**Decision**: filtrar `CustomerOrder.status.notin_(("pagada", "cancelada"))` en la misma consulta
que hoy carga las órdenes del grupo (`select(CustomerOrder)...where(CustomerOrder.merged_group_id
== group_id)`), agregando la condición de status ahí — igual que `_billable_orders`
(`table_sessions/service.py:126-136`) filtra en SQL, no en Python después de cargar.

**Rationale**: es el mismo patrón que ya usa el camino A (`_billable_orders`) y que ya importa
`checkout.py` como constante pública `TERMINAL = ("pagada", "cancelada")` (línea 42) — reutilizada
tal cual en vez de declarar una tupla nueva y arriesgar que diverja de la de `checkout`/
`tables_advanced` (que ya define su propia `TERMINAL` idéntica en `tables_advanced.py:17`, usada
por `_active_orders_on_table`/`move_order`). Filtrar en SQL, no en Python, evita traer de la base
de datos órdenes que de todas formas se van a descartar — mismo criterio de eficiencia que ya aplica
`_billable_orders`.

**Alternatives considered**: cargar todas las órdenes del grupo (como hoy) y filtrar el `status` en
el bucle Python: rechazado — funciona igual de correcto pero diverge del patrón ya establecido por
`_billable_orders` sin ninguna ventaja; además obligaría a decidir si el `group_id` sigue
respondiendo 404 cuando el grupo existe pero **todas** sus órdenes son terminales (ver Decisión 3).

## Decisión 3 — 404 del grupo vs. grupo con total $0 (Edge Case de `spec.md`)

**Decision**: el `404 Grupo de mesas no encontrado` se sigue evaluando sobre **todas** las órdenes
del grupo (terminales incluidas, consulta sin el filtro de status), no solo sobre las billables. Si
el grupo existe pero ninguna de sus órdenes queda tras excluir `pagada`/`cancelada`, la respuesta es
`200` con `total=0` y `orders=[]` (o, si se decide listar igualmente las órdenes terminales con su
`subtotal` en $0 para trazabilidad — ver nota abajo).

**Rationale**: el Edge Case de `spec.md` ("¿Qué pasa si todas las órdenes del grupo están pagada o
cancelada? El total debe ser $0... no un error") fija el comportamiento del **total**, no dice
explícitamente si `orders[]` debe seguir listando las órdenes terminales (ya excluidas del cálculo)
o quedar vacía. Se decide **mantenerlas listadas con `subtotal=0`** — no se filtran de `orders[]`,
solo de la suma — porque: (a) es el comportamiento actual de la función (`orders[]` siempre refleja
"todas las órdenes del grupo", nunca se filtró); (b) cambiar la forma de `orders[]` (quitar
entradas) es un cambio de contrato que ninguna historia pide y que además le quitaría al cajero la
visibilidad de que esa orden fue pagada aparte — justo el caso que motiva la Historia 1. Solo se
excluyen del **cálculo** del `total`, tal como pide FR-001 literalmente ("excluir del cálculo"),
no de la lista.

**Alternatives considered**: devolver 404 si no queda ninguna orden billable (aunque el grupo
exista con órdenes terminales): rechazado — contradice explícitamente el Edge Case citado arriba
("El total debe ser $0... no un error").

## Decisión 4 — Modificación del test `CONGELA` existente (Principio II)

**Decision**: se modifica `test_group_bill_a01_camino_c_incluye_todos_los_status_sin_descuentos`
(`test_orders_tables_advanced.py:124-160`) para verificar el comportamiento corregido, citando en
el mismo commit la entrada A-01 de `registro-de-anomalias.md` — tal como exige el Principio II
("Modificar cualquier test cuyo nombre lleve el prefijo `CONGELA comportamiento actual:`... exige
citar en el propio commit la decisión del registro de anomalías que lo autoriza"). El nombre del
test se actualiza para dejar de describir el comportamiento defectuoso (`sin_descuentos`) y pasar a
describir el corregido, conservando la referencia a A-01 y camino C.

**Rationale**: es el propio Principio II el que exige esto — dejar el test tal cual perpetuaría un
test rojo (una vez corregido `group_bill`) sin decisión que lo ampare, que es justo el escenario
que la Constitución prohíbe en sentido inverso ("sin esa cita, el cambio no se hace"). La cita ya
existe (A-01, "Tratamiento acordado" 2026-08-15/16) y `spec.md` ya la referencia extensamente, así
que el commit que toque este test la cita explícitamente. FR-006 exige además que el nombre o
comentario del test cite A-01 y la decisión — se satisface actualizando el nombre y conservando (y
ampliando) el docstring existente.

**Alternatives considered**: dejar el test T046 intacto (congelando el defecto) y añadir un test
nuevo separado que verifique el comportamiento corregido: rechazado — dejaría dos tests
contradictorios verificando el mismo camino de código con expectativas opuestas, uno de los cuales
quedaría en rojo permanentemente tras el fix (viola el propio Principio II: un test `CONGELA` en
rojo sin decisión que lo ampare es una regresión). Modificar el existente, citando la decisión, es
exactamente el mecanismo que el Principio II define para este caso.

## Decisión 5 — Convención de ejecución de tests

**Decision**: `python3 -m unittest`, igual que el resto de `app/characterization_tests/`.

**Rationale**: verificado — no hay `pytest.ini` ni configuración de `pytest` en `pos-backend`;
`test_orders_tables_advanced.py` ya usa `unittest.TestCase` y se ejecuta con
`python -m unittest app.characterization_tests.test_orders_tables_advanced -v` (citado en su propio
docstring, línea 21). Introducir `pytest` solo para esta spec rompería esa uniformidad sin ningún
beneficio pedido y contaría como dependencia nueva a justificar (Principio IV) sin necesidad real.

**Alternatives considered**: `pytest` — rechazado por la razón de arriba, sin necesidad real que lo
justifique.

## Resumen — incógnitas resueltas

No queda ningún `NEEDS CLARIFICATION` pendiente en el Technical Context de `plan.md`: lenguaje,
dependencias, storage, testing, plataforma y alcance ya estaban determinados por el código
existente. Esta fase añadió las cinco decisiones de arriba — en particular la Decisión 1, que
resuelve con una prueba basada en el código real por qué agrupar por orden (en vez de por comensal)
sigue satisfaciendo FR-002/FR-005 sin necesitar investigación externa adicional.
