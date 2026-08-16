# Feature Specification: Consumo de inventario por receta y opción

**Feature Branch**: `003-consumo-de-inventario-por-receta-y-opcion`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/catalog/consumption_plan.py` (y el fragmento de `catalog/router.py` que
guarda la receta), para que sirva de contrato formal de cara a la modernización (Principio I y
Principio III de la [Constitución](../../.specify/memory/constitution.md)). Donde el resto de las
specs de este proyecto describen lo que el sistema **debe** hacer, esta describe lo que el
sistema **efectivamente hace** — incluida una regla protegida que corrige un bug histórico real
(A-02) y dos anomalías que se documentan **sin especificar como contrato** porque su
clasificación sigue `PENDIENTE` (A-33, A-34), con su tratamiento acordado citado de
`registro-de-anomalias.md`.

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE
del cálculo de consumo de inventario por línea de venta (receta fija + opciones elegidas) del
sistema POS Heladería, tomado de `reglas-de-negocio.md` (RN-CAT-12 a RN-CAT-26, RN-CAT-34,
RN-CAT-35) y de `registro-de-anomalias.md` (A-02, A-03, A-33, A-34, A-47), para que sirva de
contrato en la modernización."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `catalog/consumption_plan.py` y en el
  fragmento de `catalog/router.py` que guarda la receta, no uno deseado. Las anomalías conocidas
  se marcan inline con su tratamiento acordado (registro-de-anomalias.md). A-02 es la regla más
  importante de esta spec: corrige un bug real de doble descuento y se especifica como invariante
  de test obligatorio, no solo documental.
-->

### User Story 1 - Una sola cantidad manda: tamaño sobre opción, nunca se suman (Priority: P1) — regla protegida A-02

Un cajero vende una presentación (p. ej. «Copa Grande») con un sabor elegido que tiene, a la vez,
una cantidad propia de consumo y una cantidad definida por el tamaño de la presentación. El
sistema descuenta **una sola** cantidad — la que define el tamaño, si la define — y nunca suma
ambas.

**Why this priority**: esta regla corrige un bug de inventario real y silencioso: con los sabores
en 80 g (su valor histórico) y una ensalada pequeña configurada en 60 g, cada venta descontaba
140 g y "nadie se enteraba hasta el conteo físico" (`memoria-historica.md` entrada #8,
2026-08-03, commit `03469cad`, Deimer Hernandez). Es la regla con más evidencia de daño operativo
real de todo el módulo — cualquier regresión aquí reintroduce ese bug.

**Independent Test**: se puede probar de forma aislada invocando `plan_line_consumption(db,
variant_id, quantity, options)` con una variante cuyo grupo de opciones define
`quantity_per_option` y una opción con su propio `item_quantity`, sin depender de carrito, caja
ni disponibilidad de stock.

**Acceptance Scenarios**:

1. **Given** una variante «Copa Grande» cuyo grupo «Sabores» define `quantity_per_option=120`
   (gramos) y una opción «Fresa» con `item_quantity=80`, **When** se vende 1 copa con «Fresa»
   elegida, **Then** el consumo de fresa es exactamente **120 g** — la cantidad del tamaño manda,
   la de la opción (80 g) se ignora, y en ningún caso el resultado es `120 + 80 = 200`
   (`RN-CAT-18`).
2. **Given** el mismo grupo «Sabores», pero configurado como en el bug histórico
   (`quantity_per_option` del tamaño en 60 g para una presentación pequeña, `item_quantity` del
   sabor en 80 g), **When** se vende esa presentación con un sabor elegido, **Then** el consumo es
   **60 g**, nunca `60 + 80 = 140 g` — este es exactamente el escenario que motivó la corrección
   (`RN-CAT-18`, `memoria-historica.md` #8).
3. **Given** una variante cuyo grupo de opciones **no** define `quantity_per_option`
   (`quantity_per_option=0`), **When** se vende con una opción que sí tiene `item_quantity=50`
   propio, **Then** el consumo es **50** — a falta de cantidad del tamaño, la de la opción queda
   como respaldo (`RN-CAT-18`, rama "manda la opción").
4. **Given** «Copa Grande» (`quantity_per_option=3`, `max_select=2`) y «Copa Pequeña»
   (`quantity_per_option=1`, `max_select=1`) del mismo producto, ofreciendo el **mismo** grupo
   «Sabores» con el mismo insumo por opción, **When** se vende una Grande con un sabor elegido y,
   por separado, una Pequeña con el mismo sabor elegido, **Then** la Grande descuenta 3× la
   cantidad de ese insumo y la Pequeña 1× — misma opción, cantidades distintas según el tamaño que
   la vendió (`RN-CAT-18`). **Verificado por**
   `app/scripts/test_variant_option_groups.py` (fixture con `POR_OPCION_GRANDE=3`,
   `POR_OPCION_PEQUENA=1`; citado en el propio script como el caso que "el modelo anterior no
   podía expresar").
5. **Given** una opción elegida cuyo consumo por unidad resultante (tras aplicar la regla anterior)
   es `0` o negativo, **When** se arma el plan de consumo, **Then** no se genera ninguna línea de
   consumo para esa opción — ni con cantidad 0 ni con cantidad negativa (`RN-CAT-23`).
6. **Given** un grupo de opciones donde el cliente elige varias opciones distintas (p. ej. dos
   sabores en una presentación de 3 bolas), **When** se arma el plan de consumo, **Then** cada
   opción elegida descuenta la cantidad **completa** configurada, no un total repartido entre las
   opciones elegidas — elegir Fresa y Chocolate en un grupo de `quantity_per_option=120`
   descuenta 120 g de cada uno, 240 g en total, no 120 g repartidos (`RN-CAT-20`).

**Nota documental — anomalía A-03 (ACCIDENTAL)**: el docstring del campo `quantity_per_option` en
el modelo `VariantOptionGroup` (`app/models/variant_option_group.py:46-49`) todavía dice "Se suma
a `options.item_quantity`", contradiciendo directamente el comportamiento real desde la
corrección de A-02. Es una contradicción puramente textual, sin testigo de negocio necesario
porque es un hecho verificable en el propio repositorio. **Tratamiento acordado**: corregir el
comentario en fase de modernización para que describa el comportamiento real; no cambia
comportamiento, solo documentación. Se cita aquí para que nadie use ese docstring como referencia
al reimplementar esta regla.

---

### User Story 2 - Vender una variante sin descontar NADA de inventario está bloqueado (Priority: P1)

Un cajero intenta vender una presentación que no tiene receta fija ni ningún grupo de opciones
configurado para descontar inventario. El sistema rechaza la venta con `409` en vez de dejarla
pasar sin dejar rastro en el kardex.

**Why this priority**: es la guarda que evita el defecto más costoso del módulo — una venta que
no mueve stock no falla, no avisa, y el inventario queda sobrestimado en silencio hasta el conteo
físico. El propio código lo describe como "peor que un error, porque nadie se entera".

**Independent Test**: se puede probar completamente ejecutando
`python -m app.scripts.test_receta_obligatoria` contra un `pos-backend` en ejecución, sin
depender de ningún otro módulo (usa datos desechables propios y los borra al terminar).

**Acceptance Scenarios**:

1. **Given** una variante «Malteada Especial» sin receta fija y sin ningún grupo de opciones
   vinculado, **When** se intenta vender, **Then** el sistema responde `409` con detalle
   `{"error": "«Malteada Especial» no tiene receta configurada, así que venderlo no descontaría
   inventario. Cárgasela en Productos → Recetas para poder venderlo.", "variantes_sin_receta":
   [...]}` (`RN-CAT-34`).
2. **Given** una variante con receta fija **y además** un grupo de opciones que también descuenta
   inventario, **When** se vende sin elegir ninguna opción del grupo, **Then** la venta **se
   permite** — la receta fija ya garantiza que algo se descuenta; no elegir una opción de un grupo
   adicional no es, por sí solo, motivo de bloqueo (distinción explícita en el docstring de
   `ensure_lines_consume_inventory` entre "sin nada configurado" y "configurado, pero no
   elegido"; `RN-CAT-34`).
3. **Given** una variante sin receta fija cuyo **único** grupo de opciones configurado para
   descontar inventario no recibió ninguna selección, **When** se intenta vender, **Then** el
   sistema responde `409` con detalle `{"error": "... consume inventario según la opción que
   elija el cliente, pero no se eligió ninguna, así que venderlo no descontaría nada.",
   "variantes_sin_opcion": [...]}` — mensaje distinto al del escenario 1, porque el problema es
   "no se eligió", no "no está configurado" (`RN-CAT-34`). Este escenario es, a su vez, el
   disparador de la anomalía A-33/RN-CAT-35 (ver User Story 7).
4. **Given** un lote con varias líneas, donde una o más no descontarían nada, **When** se valida
   el lote completo, **Then** el rechazo agrupa **todas** las variantes problemáticas del lote en
   una sola respuesta `409` (sin duplicados, orden de aparición preservado), no solo la primera
   que encuentra (`app/api/v1/catalog/consumption_plan.py:185-226`).

**Verificado por**: `app/scripts/test_receta_obligatoria.py`, motivado por un caso real: "7 de 13
variantes activas sin receta en un tenant real, eso ocurría a diario" (comentario del propio
script).

---

### User Story 3 - Consumo por receta fija (Priority: P2)

Un cajero vende varias unidades de una presentación que tiene insumos fijos en su receta (p. ej.
un vasito, una cuchara). El sistema descuenta cada insumo fijo multiplicando su cantidad
configurada por las unidades vendidas de esa línea.

**Why this priority**: es la fórmula base del módulo — el consumo por opción (User Story 1) se
suma a este resultado, nunca lo reemplaza.

**Independent Test**: se puede probar invocando `plan_line_consumption` con una variante que solo
tiene receta fija (sin grupos de opciones), variando la cantidad vendida.

**Acceptance Scenarios**:

1. **Given** la receta de «Copa Grande» incluye `Vasito: 1 unidad`, **When** se venden 3 copas,
   **Then** el consumo de vasitos es `1 × 3 = 3` unidades (`RN-CAT-17`).
2. **Given** una receta con varios insumos fijos, **When** se vende cualquier cantidad, **Then**
   cada insumo se descuenta de forma independiente con su propia multiplicación — no hay
   interacción entre insumos distintos de la misma receta (`RN-CAT-17`).

---

### User Story 4 - Guardar la receta de una variante: reemplazo total e idempotente (Priority: P2) — anomalía A-34

Un administrador guarda o actualiza la receta de una presentación desde el panel. El sistema
borra la receta anterior completa y la reemplaza íntegramente por la lista enviada, sin dejar
insumos huérfanos de una versión previa.

**Why this priority**: es el punto de entrada que alimenta todo el cálculo de consumo (User
Story 3); un reemplazo parcial o inconsistente contaminaría cualquier venta futura de esa
variante.

**Independent Test**: se puede probar enviando `PUT /variants/{id}/recipe` dos veces seguidas con
listas de insumos distintas y verificando que solo la segunda lista queda persistida.

**Acceptance Scenarios**:

1. **Given** una variante cuya receta actual es `[Leche: 200g]`, **When** se envía
   `PUT /variants/{id}/recipe` con `[Fruta: 150g, Vasito: 1un]`, **Then** la receta anterior se
   borra por completo y quedan únicamente las dos líneas nuevas — «Leche» desaparece (`RN-CAT-13`).
2. **Given** un ítem de receta en el payload con `quantity=0` o negativa, **When** se intenta
   guardar, **Then** el sistema lo rechaza con `422` ("Input should be greater than 0") **antes**
   de tocar la receta existente — la cantidad de un insumo en receta debe ser estrictamente mayor
   que cero, con respaldo adicional de un `CheckConstraint` en base de datos (`RN-CAT-12`).
3. **Given** un payload de receta con el mismo `inventory_item_id` repetido dos veces, **When** se
   envía `PUT /variants/{id}/recipe`, **Then** el sistema rechaza la operación completa con `422`
   "Insumo repetido en la receta" (`RN-CAT-14`).

**Anomalía A-34 (clasificación `PENDIENTE`) — documentada sin especificar como contrato**: en el
escenario 3, el `DELETE` de la receta anterior ya se ejecutó (dentro de la misma sesión de base de
datos, sin `commit()` previo) **antes** de que el bucle detecte el insumo repetido y lance el
`422`. Si la variante queda efectivamente sin receta tras ese `422` depende del comportamiento de
rollback de `app/core/db.py`, **no verificado en este reconocimiento**. Por eso esta spec no fija
como contrato si la receta anterior sobrevive o no a un intento de guardado rechazado por
duplicado — solo documenta que el `DELETE` se ejecuta antes de la validación, tal como el código
lo hace hoy. **Tratamiento acordado** (`registro-de-anomalias.md`, A-34): documentar sin
especificar hasta verificar el manejo de rollback; no se prioriza corrección mientras no haya
evidencia de que deja variantes sin receta en producción.

---

### User Story 5 - Movimientos de inventario por opción elegida (Priority: P2)

Un cajero vende una presentación con varias opciones elegidas, algunas de las cuales comparten
insumo o no tienen insumo ligado en absoluto. El sistema genera un movimiento de inventario
independiente por cada opción que sí consume, sin fusionar ni omitir por parecido.

**Why this priority**: gobierna la trazabilidad del kardex cuando hay varias opciones en juego —
necesaria para auditar qué se vendió, no solo cuánto se descontó en total.

**Independent Test**: se puede probar invocando `plan_line_consumption` con una lista de opciones
que incluya dos opciones distintas apuntando al mismo insumo y una opción sin insumo ligado.

**Acceptance Scenarios**:

1. **Given** dos opciones distintas («Fresa» y «Fresa Premium») que apuntan al **mismo**
   `inventory_item_id`, **When** ambas se eligen en la misma línea, **Then** el sistema genera
   **dos** líneas de consumo separadas sobre ese insumo, no una sola cantidad fusionada — "dos
   apuntes separados son auditables ('2 de fresa, 1 de fresa premium'), uno fusionado no"
   (docstring del módulo; `RN-CAT-21`).
2. **Given** una opción elegida con `inventory_item_id=None` (sin insumo ligado), aunque tenga
   `item_quantity` configurado, **When** se arma el plan de consumo, **Then** esa opción se ignora
   por completo — no genera ninguna línea de consumo (`RN-CAT-22`).
3. **Given** las líneas de consumo resultantes de una venta, **When** se inspecciona cada una,
   **Then** ninguna tiene cantidad `<=0` — las cantidades no positivas se filtran antes de llegar
   al plan final (`RN-CAT-23`, mismo chequeo que User Story 1 escenario 5).

---

### User Story 6 - Chequeo preventivo de disponibilidad de stock (Priority: P2) — regla confirmada A-47

Un cajero arma una línea de venta con insumos que tienen poco stock disponible. El sistema
compara el stock actual contra el total requerido y rechaza con `409` solo si el stock es
estrictamente insuficiente. Este chequeo es una advertencia temprana de experiencia de usuario,
no una reserva real de stock.

**Why this priority**: es la única señal que recibe el cajero antes de intentar cobrar; que sea
estricta y que no reserve son dos decisiones de diseño distintas que conviene separar.

**Independent Test**: se puede probar invocando `check_availability(db, required)` con un
diccionario de insumo→cantidad requerida contra un `InventoryItem` con `current_stock` conocido.

**Acceptance Scenarios**:

1. **Given** un insumo con `current_stock=120` y un requerido de `120` (exactamente igual),
   **When** se chequea disponibilidad, **Then** la venta **se permite** — la comparación es
   `stock_actual < requerido`, estrictamente menor, no `<=`; quedar en 0 no es faltante
   (`RN-CAT-24`).
2. **Given** el mismo insumo con `current_stock=119` frente a un requerido de `120`, **When** se
   chequea disponibilidad, **Then** el sistema responde `409` con detalle `{"error": "Stock
   insuficiente", "insumo": ..., "disponible": "119", "requerido": "120"}` (`RN-CAT-24`).
3. **Given** un insumo cuyo requerido total agregado es `0` o negativo, **When** se chequea
   disponibilidad, **Then** ese insumo se omite del chequeo por completo — no se consulta su
   stock ni puede bloquear la venta (`RN-CAT-25`).
4. **Given** que la venta pasó el chequeo de disponibilidad en el momento de armar el carrito,
   **When** otra venta concurrente consume el mismo insumo antes de que esta se confirme o cobre,
   **Then** el chequeo previo **no** impide que la confirmación/cobro posterior falle por falta
   real de stock — este chequeo no reserva ni bloquea filas, es puramente preventivo (`RN-CAT-26`).
   El bloqueo real (con lock de fila) vive en el módulo de inventario, en el paso de
   confirmación/cobro, fuera del alcance de esta spec.

**Anomalía A-47 — `INTENCIONAL` confirmado**: el propio código distingue explícitamente este
chequeo best-effort del bloqueo real con `SELECT ... FOR UPDATE` (`registro-de-anomalias.md`,
A-47). En la primera ronda de reconocimiento esta clasificación se sostenía solo con testimonio de
CÓDIGO (1 de 2 testigos requeridos), por lo que quedó `PENDIENTE` en sentido estricto del método.
**En la segunda ronda de entrevista de negocio (pregunta P27-bis) el dueño confirmó
explícitamente que prefiere el diseño actual (best-effort, sin reservar stock) antes que invertir
en reservar stock preventivamente** — con ese segundo testimonio (CÓDIGO + NEGOCIO), la regla
queda **`INTENCIONAL` confirmada**, no una brecha a corregir. Esta spec la especifica como
comportamiento de contrato para la modernización, no como algo pendiente de decisión. La
consecuencia visible de esta regla para el comensal (pedido rechazado tarde en hora pico) se
documenta en la spec del carrito/QR (spec 007, aún no escrita en este reconocimiento).

---

### User Story 7 - Un grupo opcional que es la única fuente de consumo bloquea la venta si nadie elige (Priority: P3) — anomalía A-33/RN-CAT-35, documentada sin especificar

Un cajero vende una presentación que **no** tiene receta fija y cuyo único grupo de opciones que
descuenta inventario es opcional (`min_select=0` — "no elegir nada" es, en principio, una
decisión legítima del comensal). Si nadie elige ninguna opción de ese grupo, el sistema bloquea la
venta con `409`, aunque el propio código que valida la selección de opciones llama a "no elegir"
una decisión legítima.

**Why this priority**: es una tensión real entre dos piezas de código con distinto criterio, no
un bug obvio ni una regla claramente deliberada — se documenta el comportamiento observado sin
fijarlo como contrato mientras no haya una decisión de negocio.

**Independent Test**: se puede probar con una variante sin receta fija, con un único grupo
opcional (`min_select=0`) que define `quantity_per_option>0`, vendida sin elegir ninguna opción de
ese grupo.

**Acceptance Scenarios (comportamiento observado, no especificado como contrato)**:

1. **Given** «Cono Simple» sin receta fija, que solo ofrece el grupo «Sabores»
   (`min_select=0, max_select=1, quantity_per_option=80`), **When** se valida la **selección** de
   opciones (`validate_option_selection`, fuera de alcance de esta spec — ver spec 004) sin elegir
   ningún sabor, **Then** esa validación **lo permite**: el propio comentario del código llama a
   esto "una decisión legítima del comensal" (`RN-CAT-35`, comentario en
   `consumption_plan.py:174-179`).
2. **Given** la misma variante y la misma selección vacía, **When** se ejecuta
   `ensure_lines_consume_inventory` (la guarda de User Story 2) sobre esa línea, **Then** la venta
   se **bloquea** con `409` "consume inventario según la opción que elija el cliente, pero no se
   eligió ninguna" — el mismo caso que la primera función acaba de calificar como legítimo, la
   segunda lo rechaza (`RN-CAT-35`, lógica real en `consumption_plan.py:188-198,214-226`).

**Anomalía A-33/RN-CAT-35 — `[TENSIÓN/DISCREPANCIA]` con la convención de spec 004**: la spec de
grupos de opciones (spec 004, aún no escrita en este reconocimiento) establece la convención de
que `min_select=0` significa "no elegir nada es válido". Esta spec documenta que, cuando ese
grupo opcional es la **única** fuente de consumo de inventario de la variante, esa convención deja
de sostenerse en la práctica: el resultado observable es un bloqueo, no el paso silencioso que la
convención de spec 004 sugeriría. **Clasificación: `PENDIENTE`** — no reúne el testimonio de
negocio necesario para decidir si el bloqueo es el comportamiento correcto (evitar la venta
fantasma de User Story 2, a costa de invalidar la opcionalidad del grupo) o si debería, en este
caso específico, permitirse la venta sin consumo. **Tratamiento acordado**: documentar el
comportamiento tal cual, sin especificarlo como contrato obligatorio para la modernización, hasta
que exista una decisión de negocio explícita.

---

### Edge Cases

- **Grupo de opciones sin `quantity_per_option` y opción sin `item_quantity`**: ambas cantidades
  en `0` → `per_unit=0` → sin línea de consumo para esa opción, cubierto por `RN-CAT-23` (User
  Story 1, escenario 5).
- **Opción de un grupo que la variante no ofrece**: `plan_line_consumption` no valida pertenencia
  al grupo — solo usa `por_grupo.get(option.option_group_id)`, que devuelve `None`/`0` si la
  variante no vincula ese grupo, y la opción cae a su propio `item_quantity` como respaldo (rama
  "manda la opción" de `RN-CAT-18`). La validación de que la opción efectivamente pertenezca a un
  grupo ofrecido por la variante es responsabilidad de otra función (`validate_option_selection`,
  fuera de alcance — ver spec 004, RN-CAT-32).
- **Receta fija vacía y grupos de opciones inexistentes**: variante sin ninguna fuente de consumo
  configurada → `required_consumption` devuelve un diccionario vacío → bloqueo con el mensaje
  "no tiene receta configurada" de User Story 2, escenario 1.
- **Cantidad vendida (`quantity`) igual a 0**: no está cubierto por ninguna regla `RN-CAT` de este
  recorte; multiplicar cualquier cantidad de receta u opción por `0` produce `0`, que cae en la
  guarda de `RN-CAT-23`/`RN-CAT-34` como si no hubiera consumo configurado.
- **Guardar una receta con un único insumo repetido tres o más veces**: se rechaza en la primera
  repetición detectada (segunda aparición del mismo `inventory_item_id`), sin distinguir cuántas
  veces más aparece — mismo `422` de `RN-CAT-14`.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Guardar la receta de una variante (`PUT /variants/{id}/recipe`) DEBE ser un
  reemplazo total e idempotente: la receta anterior completa se borra y se reemplaza íntegramente
  por la lista enviada (`RN-CAT-13`).
- **FR-002**: La cantidad de un insumo dentro de un ítem de receta DEBE ser estrictamente mayor
  que cero; un valor `0` o negativo DEBE rechazarse con `422` antes de tocar la receta existente,
  con respaldo adicional de un `CheckConstraint` en base de datos (`RN-CAT-12`).
- **FR-003**: El mismo insumo NO DEBE aparecer dos veces en el payload de una sola receta; si
  ocurre, la operación completa DEBE rechazarse con `422` "Insumo repetido en la receta"
  (`RN-CAT-14`). **[Anomalía A-34, `PENDIENTE`]**: el `DELETE` de la receta anterior se ejecuta
  antes de esta detección, dentro de la misma transacción sin `commit()` previo; si la variante
  queda o no sin receta tras el rechazo depende del comportamiento de rollback de
  `app/core/db.py`, no verificado en este reconocimiento — no se especifica como contrato hasta
  verificarlo.
- **FR-004**: El consumo de cada insumo fijo de la receta DEBE calcularse como
  `cantidad_configurada × cantidad_vendida` de la línea (`RN-CAT-17`).
- **FR-005 [Regla protegida A-02]**: El consumo por cada opción elegida DEBE usar **una sola**
  cantidad: si el grupo de opciones de la variante define `quantity_per_option > 0`, esa cantidad
  es la que se descuenta por cada opción elegida y el `item_quantity` propio de la opción se
  ignora; solo si el grupo no define nada (`quantity_per_option = 0`) se usa el `item_quantity` de
  la opción como respaldo. Las dos cantidades NUNCA DEBEN sumarse (`RN-CAT-18`). Esta regla corrige
  un bug histórico de doble descuento (140 g en vez de 60 g u 80 g según el tamaño,
  `memoria-historica.md` #8) y DEBE tratarse como invariante de test obligatorio, no solo
  documental, en cualquier reimplementación.
- **FR-006**: El consumo por opción es "por cada opción elegida", no un total repartido entre las
  opciones del grupo: elegir varias opciones del mismo grupo descuenta la cantidad completa
  configurada por cada una (`RN-CAT-20`).
- **FR-007**: Dos opciones distintas que apuntan al mismo insumo DEBEN generar dos líneas de
  consumo (y por tanto dos movimientos de inventario) separadas, nunca una cantidad fusionada
  (`RN-CAT-21`).
- **FR-008**: Una opción elegida sin `inventory_item_id` ligado NO DEBE generar ninguna línea de
  consumo, aunque tenga `item_quantity` configurado (`RN-CAT-22`).
- **FR-009**: Una línea de consumo (fija o por opción) cuya cantidad resultante sea `0` o negativa
  NO DEBE incluirse en el plan de consumo final (`RN-CAT-23`).
- **FR-010**: El chequeo preventivo de disponibilidad DEBE considerar stock insuficiente
  únicamente cuando `stock_actual < requerido` (comparación estrictamente menor); un stock
  exactamente igual al requerido DEBE permitir la venta (`RN-CAT-24`).
- **FR-011**: Un requerimiento total de consumo `<=0` para un insumo DEBE omitirse del chequeo de
  disponibilidad — ese insumo no se consulta ni puede bloquear la venta (`RN-CAT-25`).
- **FR-012 [Regla confirmada A-47, `INTENCIONAL`]**: El chequeo preventivo de disponibilidad NO
  DEBE reservar ni bloquear stock de ninguna forma; es exclusivamente una advertencia temprana de
  experiencia de usuario. Confirmado por el negocio en la segunda ronda de entrevista (P27-bis):
  se prefiere el diseño actual antes que invertir en reservar stock preventivamente (`RN-CAT-26`).
- **FR-013**: Si el plan de consumo agregado de una línea de venta resulta vacío, la venta de esa
  línea DEBE rechazarse con `409`, distinguiendo en el mensaje "variante sin receta ni grupo
  configurado" de "grupo configurado que descuenta pero sin opción elegida" (`RN-CAT-34`).
- **FR-014 [Anomalía A-33/RN-CAT-35, `PENDIENTE` — documentada sin especificar como contrato]**:
  el sistema actualmente bloquea con `409` la venta de una variante cuya única fuente de consumo
  configurada es un grupo de opciones opcional (`min_select=0`) si el cliente no elige ninguna
  opción de ese grupo — aun cuando la validación de selección de opciones (spec 004) califica
  explícitamente esa misma situación como "una decisión legítima del comensal". Esta spec
  documenta el comportamiento observado tal cual, sin fijarlo como contrato obligatorio para la
  modernización, en tensión abierta con la convención de `min_select=0` de spec 004, hasta que
  exista una decisión de negocio explícita.

### Key Entities *(include if feature involves data)*

- **RecipeItem**: insumo fijo de la receta de una variante. Atributos relevantes: `product_variant_id`,
  `inventory_item_id`, `quantity` (`> 0`). Se reemplaza en bloque en cada `PUT` (`RN-CAT-13`).
- **VariantOptionGroup**: vínculo entre una variante y un grupo de opciones, con la cardinalidad
  (`min_select`/`max_select`, fuera de alcance de esta spec — ver spec 004) y la cantidad de
  consumo por opción propia de ese tamaño (`quantity_per_option`). Es la pieza que hace que la
  misma opción consuma distinto según qué variante la vendió (`RN-CAT-18`).
- **Option**: valor dentro de un grupo de opciones (p. ej. un sabor). Atributos relevantes a esta
  spec: `inventory_item_id` (opcional — `None` si no consume, `RN-CAT-22`), `item_quantity`
  (respaldo cuando el tamaño no define cantidad propia).
- **ConsumptionLine**: estructura interna (no persistida) que representa un movimiento de stock a
  aplicar — `inventory_item_id`, `quantity` (siempre positiva, ya multiplicada por la cantidad
  vendida) y `source` (`'receta' | 'variante' | 'opcion'`, para diagnóstico). Es la salida de
  `plan_line_consumption`, consumida tanto por el chequeo de disponibilidad como por el descuento
  real de stock (fuera de alcance de esta spec).
- **InventoryItem**: insumo del inventario. Atributo relevante a esta spec: `current_stock`,
  comparado contra el consumo requerido agregado (`RN-CAT-24`).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-CAT-12` a `RN-CAT-26`, `RN-CAT-34` y `RN-CAT-35` puede
  verificarse ejecutando los pasos descritos en esta spec contra un `pos-backend` en ejecución,
  sin necesitar leer `consumption_plan.py` para entender el comportamiento esperado.
- **SC-002**: `RN-CAT-18`/A-02 (la regla protegida más crítica de esta spec) queda cubierta por
  `app/scripts/test_variant_option_groups.py` como el characterization test más cercano a un
  golden master: "la grande descuenta 3× del sabor elegido y la pequeña 1× — misma opción,
  cantidades distintas por tamaño". El script no corre en CI hoy (ver spec 013 para el contexto
  de cobertura de CI, aún no escrita en este reconocimiento; A-27 documenta la brecha general de
  1 de 12 scripts ejecutados automáticamente).
- **SC-003**: `RN-CAT-34` queda cubierta por `app/scripts/test_receta_obligatoria.py`, motivado
  por un caso real documentado en el propio script ("7 de 13 variantes activas sin receta en un
  tenant real"). Tampoco corre en CI hoy.
- **SC-004**: Ningún cajero puede completar una venta de una línea que no descuente nada de
  inventario — el 100% de esos intentos recibe un `409` con un mensaje que distingue si el
  problema es "sin receta configurada" o "sin opción elegida", permitiendo actuar sin adivinar la
  causa.
- **SC-005**: Las dos anomalías `PENDIENTE` de esta spec (A-33/RN-CAT-35, A-34) quedan
  documentadas con su comportamiento observado, su evidencia de código y su tratamiento acordado
  (documentar sin especificar), de forma que el equipo de modernización no las reintroduzca por
  accidente ni las trate como si ya tuvieran una decisión de negocio cuando no la tienen.

## Out of Scope

- **El precio y el SKU de la variante** — cubierto por la spec 002
  (`002-catalogo-productos-variantes-y-precios`).
- **La validación de forma de la selección de opciones** (`min_select`/`max_select`,
  `STRICT_OPTION_SELECTION`, unicidad de opción dentro de su grupo, pertenencia de la opción al
  grupo ofrecido por la variante) — cubierto por la spec 004, aún no escrita en este
  reconocimiento (`RN-CAT-27` a `RN-CAT-33`, `RN-CAT-36` a `RN-CAT-39`). Esta spec asume que la
  lista de `options` que recibe `plan_line_consumption` ya pasó (o deliberadamente no pasó, ver
  siguiente punto) esa validación.
- **El descuento real de stock y su bloqueo transaccional** (`SELECT ... FOR UPDATE`,
  `record_movement`) — pertenece al módulo de inventario, consumido por esta spec solo a través
  del chequeo preventivo de `RN-CAT-24`/`RN-CAT-26`.
- **El caller real que omite pasar `variant` a la validación de selección de opciones**
  (`add_item_to_table`, anomalía A-04) — la regla de que la validación puede omitirse
  (`RN-CAT-33`) vive en spec 004; su manifestación real en el camino del mesero se documenta en
  spec 009, aún no escrita en este reconocimiento.

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia del resto de
  las guías de este template ("evitar detalles de implementación"), aquí los endpoints, códigos
  de estado HTTP, nombres de campo y valores literales **son** el contrato observable que se está
  documentando — se citan explícitamente porque los criterios de aceptación deben ser verificables
  directamente contra `pos-backend` en ejecución o contra los scripts de characterization
  citados.
- **A-02/RN-CAT-18 se especifica como invariante obligatorio, no como comportamiento opcional**:
  a diferencia de la mayoría de las reglas `INTENCIONAL` de este proyecto, esta corrige un bug
  real con daño operativo documentado; cualquier reimplementación de este módulo debe tratarla
  como test de regresión de máxima prioridad.
- **A-33/RN-CAT-35 y A-34 se documentan pero NO se especifican como contrato**: siguiendo
  instrucción explícita de alcance, estas dos anomalías quedan con clasificación `PENDIENTE` — se
  describe el comportamiento observado hoy (porque esta spec documenta "lo que el sistema YA
  hace"), pero esta spec no lo fija como el comportamiento correcto ni obligatorio para la
  modernización. Quedan como decisiones de negocio abiertas.
- **A-47/RN-CAT-26 sí se especifica como contrato `INTENCIONAL`**: a diferencia de A-33/A-34, esta
  anomalía alcanzó el estándar de dos testigos (CÓDIGO + NEGOCIO, confirmado en P27-bis de la
  segunda ronda de entrevista) y por tanto se documenta como comportamiento deliberado y
  definitivo, no como algo pendiente de decisión.
- **Los valores numéricos citados en los escenarios (120 g, 80 g, 60 g, 140 g, `3`/`1` de
  `test_variant_option_groups.py`) son ilustrativos**, tomados de `reglas-de-negocio.md`, de
  `memoria-historica.md` y de los scripts de characterization — no representan necesariamente el
  catálogo real vigente hoy en producción.
- **Ninguno de los dos scripts de characterization citados (`test_variant_option_groups.py`,
  `test_receta_obligatoria.py`) corre en CI actualmente**: ambos viven en `app/scripts/` como
  scripts autoejecutables contra una base de datos real, no como suite de `pytest` automatizada
  (brecha general documentada en A-27, fuera del alcance de esta spec).
