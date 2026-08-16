# Feature Specification: Kardex de insumos, compras a proveedor y conversión de unidades

**Feature Branch**: `005-kardex-compras-conversion`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/inventory/stock.py`, `service.py`, `router.py`,
`app/core/inventory_reasons.py` y `app/core/units.py` (y el fragmento de `reports/service.py` que
calcula el reporte "insumos por reponer"), para que sirva de contrato formal de cara a la
modernización (Principio I y Principio III de la
[Constitución](../../.specify/memory/constitution.md)). Donde el resto de las specs de este
proyecto describen lo que el sistema **debe** hacer, esta describe lo que el sistema
**efectivamente hace** — incluidas dos anomalías con requisito de negocio ya confirmado (A-35,
ronda de entrevista P23; A-13, P9) que esta spec fija como contrato **corregido**, no como el
comportamiento actual, y una pieza de código muerto (A-31) cuya finalización se especifica con un
alcance deliberadamente acotado.

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE
del kardex de insumos, las compras a proveedor y la conversión de unidades del sistema POS
Heladería, tomado de `reglas-de-negocio.md` (RN-INV-01 a RN-INV-23, RN-CAT-40, RN-CAT-41) y de
`registro-de-anomalias.md` (A-13, A-31, A-35), para que sirva de contrato en la modernización."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `inventory/stock.py`, `service.py`,
  `router.py` y `core/units.py`, no uno deseado — salvo donde una anomalía tiene requisito de
  negocio ya confirmado (A-13/P9, A-35/RN-INV-10/RN-INV-11/RN-INV-17/P23, A-31/P19+P31), en cuyo
  caso el escenario fija el comportamiento CORREGIDO como contrato de modernización y lo marca
  explícitamente. Las anomalías que siguen sin decisión de negocio (A-35/RN-INV-05) se documentan
  sin especificar.
-->

### User Story 1 - `record_movement` es el único punto de mutación de stock (Priority: P1) — invariante protegido

Cualquier entrada o salida de stock de un insumo — venta, compra, daño, vencimiento, consumo
interno — pasa por una sola función. Esa función recibe siempre una cantidad positiva; quién suma
y quién resta lo decide el tipo de movimiento, nunca el signo del número que llega.

**Why this priority**: es el cimiento de todo el módulo — cualquier otra regla de este documento
(bloqueo de negativos, alertas de stock bajo, compras) asume que no existe ninguna otra vía para
tocar `current_stock`. Una regresión aquí invalida el resto de la spec.

**Independent Test**: se puede probar de forma aislada invocando `record_movement(db,
inventory_item_id, type=..., quantity=...)` contra un `InventoryItem` con `current_stock`
conocido, sin depender de ventas, compras ni caja.

**Acceptance Scenarios**:

1. **Given** un insumo con `current_stock=20`, **When** se registra un movimiento
   `type="out", quantity=5`, **Then** el nuevo stock es `15` — la cantidad recibida es siempre
   positiva y la dirección la decide `type`, nunca un signo en `quantity` (`RN-INV-01`).
2. **Given** el mismo insumo en `current_stock=20`, **When** se registra un movimiento
   `type="in", quantity=5`, **Then** el nuevo stock es `25` (`RN-INV-01`).
3. **Given** cualquier insumo, **When** se intenta registrar un movimiento con `quantity=0` o
   negativa, **Then** el sistema rechaza la operación con `ValueError("quantity must be > 0")`
   antes de tocar la fila del insumo — no se permite un movimiento "vacío" ni uno que use el
   signo de `quantity` para indicar dirección (`RN-INV-02`).
4. **Given** cualquier insumo, **When** se intenta registrar un movimiento con
   `type="adjustment"`, **Then** el sistema lo rechaza con `ValueError` indicando que los ajustes
   con signo deben pasar por `apply_adjustment` — `record_movement` solo acepta `'in'`/`'out'`
   (`RN-INV-03`). El propio código documenta que antes este caso "se colaba y se comportaba
   silenciosamente como una salida"; esta guarda existe para no repetir ese defecto.

---

### User Story 2 - Bloqueo pesimista de fila con orden canónico para evitar deadlocks (Priority: P1) — invariante protegido

Antes de leer o modificar el stock de un insumo, el sistema bloquea su fila en base de datos, de
forma que dos movimientos simultáneos sobre el mismo insumo se serialicen en vez de perderse uno
al otro (condición de carrera clásica de "leer-modificar-escribir"). Cuando una operación necesita
bloquear varios insumos a la vez (p. ej. confirmar un pedido con varios ingredientes), los bloquea
todos en una sola consulta, ordenados por id — nunca en el orden en que la operación los va
necesitando.

**Why this priority**: sin este orden canónico, dos confirmaciones concurrentes que consumen los
mismos insumos en distinto orden pueden bloquearse mutuamente (deadlock de base de datos) bajo
carga real de un local con varias cajas/meseros simultáneos.

**Independent Test**: se puede probar invocando `lock_items(db, item_ids)` con una lista de ids en
orden arbitrario y verificando que la consulta que bloquea las filas las ordena por id antes de
ejecutar `SELECT ... FOR UPDATE`.

**Acceptance Scenarios**:

1. **Given** cualquier movimiento individual (`record_movement`) o ajuste (`apply_adjustment`),
   **When** se ejecuta, **Then** la fila del `InventoryItem` afectado se bloquea con
   `SELECT ... FOR UPDATE` antes de leer `current_stock`, de modo que un segundo movimiento
   concurrente sobre el mismo insumo espera a que el primero termine (`RN-INV-06`).
2. **Given** una operación que necesita bloquear varios insumos a la vez (p. ej. una receta con
   tres ingredientes), **When** se invoca `lock_items(db, item_ids)`, **Then** el sistema ejecuta
   **una sola** consulta con todos los ids, ordenados ascendentemente por id, antes de aplicar
   `FOR UPDATE` — nunca una consulta por insumo en el orden en que la operación los procesa
   (`RN-INV-07`). Esto es lo que evita que dos confirmaciones concurrentes con recetas que
   comparten insumos, pero los recorren en distinto orden, se bloqueen mutuamente.

---

### User Story 3 - Una salida nunca deja el stock negativo (Priority: P1) — invariante protegido, con vía de escape sin uso confirmado

Un cajero o el sistema intentan registrar una salida de stock mayor a la disponible. El sistema
la rechaza; el stock puede llegar exactamente a cero, pero nunca queda por debajo.

**Why this priority**: es la garantía de integridad más básica del kardex — sin ella, el stock
reportado deja de reflejar existencias reales y cualquier reporte de inventario, alerta de stock
bajo o decisión de compra queda contaminada.

**Independent Test**: se puede probar invocando `record_movement(db, item_id, type="out",
quantity=...)` contra un insumo con `current_stock` conocido, variando la cantidad alrededor del
límite exacto.

**Acceptance Scenarios**:

1. **Given** un insumo con `current_stock=5`, **When** se registra una salida de exactamente `5`,
   **Then** el nuevo stock es `0` y la operación se acepta — quedar en cero no es un error
   (`RN-INV-04`).
2. **Given** el mismo insumo en `current_stock=5`, **When** se registra una salida de `5.001`,
   **Then** el sistema rechaza la operación con `InsufficientStockError` (HTTP 400), sin modificar
   el stock (`RN-INV-04`).

**Anomalía A-35 (porción `RN-INV-05`), clasificación `PENDIENTE` — documentada sin especificar
como contrato**: `record_movement` acepta un parámetro `allow_negative` que, si es `True`, permite
que una salida deje el stock negativo saltándose el bloqueo del Escenario 2. Ningún llamador
dentro del propio módulo `inventory` (`service.py`, `router.py`) lo invoca con `True` — existe la
vía, pero no hay evidencia de quién la usa. **Tratamiento acordado**
(`registro-de-anomalias.md`, A-35): documentar la existencia del parámetro sin fijar como
contrato bajo qué condición de negocio se permitiría vender o mover en negativo, hasta identificar
qué endpoint fuera de `inventory` (si alguno) lo invoca con `True`. Esta spec no prohíbe ni exige
su uso — solo constata que el mecanismo existe.

---

### User Story 4 - Ajuste manual de stock por delta con signo (Priority: P2) — dos anomalías, una a corregir de inmediato

Un encargado de inventario corrige el stock de un insumo tras un conteo físico, una merma o un
derrame, indicando cuánto sube o baja (un número con signo), no un tipo de movimiento separado. El
sistema registra ese ajuste en el kardex como un movimiento propio (`type='adjustment'`), con la
magnitud siempre positiva y el signo capturado solo en la dirección aplicada al stock.

**Why this priority**: es el mecanismo de corrección manual del módulo — su corrección directa
(RN-INV-10) y su requisito de negocio recién confirmado (RN-INV-11) hacen que esta historia deba
tratarse antes que las historias de compras, que dependen de que el kardex sea confiable.

**Independent Test**: se puede probar invocando `apply_adjustment(db, item_id,
signed_delta=..., reason=...)` contra un insumo con `current_stock` conocido, y
`POST /inventory/items/{id}/adjust` contra un `pos-backend` en ejecución para los casos HTTP.

**Acceptance Scenarios**:

1. **Given** un insumo con `current_stock=10`, **When** se aplica un ajuste con
   `signed_delta=-2.5` y `reason="Merma por derrame"`, **Then** el nuevo stock es `7.5` y el
   kardex registra `InventoryMovement(type="adjustment", quantity=2.5,
   reason="Merma por derrame")` — la magnitud almacenada es siempre positiva (`abs(signed_delta)`),
   el signo solo determinó la dirección aplicada (`RN-INV-08`).
2. **Given** un insumo con `current_stock=3`, **When** se aplica un ajuste con
   `signed_delta=-5`, **Then** el sistema rechaza la operación con `InsufficientStockError`
   ("El ajuste dejaría el stock... en negativo"), sin ninguna bandera equivalente a
   `allow_negative` para forzarlo — a diferencia de `record_movement` (User Story 3), un ajuste
   nunca puede forzar un resultado negativo (`RN-INV-09`).
3. **Given** cualquier insumo, **When** se envía `POST /inventory/items/{id}/adjust` con
   `{"signed_delta": 0}`, **Then** hoy el sistema lo rechaza con `ValueError("signed_delta must
   be != 0")` dentro de `apply_adjustment`, pero al no existir un handler dedicado para
   `ValueError` en la aplicación, la respuesta HTTP observada es un **500 genérico** en vez de un
   `400`/`422` con mensaje de negocio.

**Anomalía A-35 (porción `RN-INV-10`), clasificación `ACCIDENTAL` — corregir de inmediato**: la
regla de negocio (rechazar `signed_delta=0`) es intencional; el defecto es puramente de
transporte del error: `AdjustmentIn.signed_delta` no declara `ne=0` a nivel de schema, y no existe
un `exception_handler` para `ValueError` en `app/main.py` (verificado: el único manejo de errores
de negocio en este módulo pasa por `InsufficientStockError`, que sí es una `HTTPException`
propia). **Esta spec fija como contrato corregido**: un `signed_delta=0` DEBE responder `422` con
un mensaje de negocio equivalente a "el ajuste no puede ser cero", nunca un `500` sin detalle
accionable — ya sea agregando `ne=0` al schema Pydantic o un handler dedicado para
`ValueError` en las rutas de ajuste.

4. **Given** cualquier insumo, **When** se envía `POST /inventory/items/{id}/adjust` sin el campo
   `reason` (o con `reason=null`), **Then** el sistema **hoy** lo acepta y registra
   `InventoryMovement.reason=None` — `AdjustmentIn.reason` es opcional a nivel de schema
   (`RN-INV-11`, comportamiento actual).

**Anomalía A-35 (porción `RN-INV-11`), clasificación `DUDOSA` → **requisito de negocio
confirmado** (entrevista de negocio, P23) — **esta spec fija el motivo como obligatorio como
contrato de modernización**, no documenta el estado actual como comportamiento deseado**: un
ajuste manual puede ocultar mermas, fraude o errores de conteo; el negocio confirmó
explícitamente en P23 que el motivo **debe** ser obligatorio. El Escenario 4 documenta lo que el
backend acepta hoy; **el contrato para la modernización es que `POST
/inventory/items/{id}/adjust` sin `reason` (vacío o solo espacios) DEBE rechazarse con `422`**,
cerrando la brecha entre lo que el backend permite y lo que el negocio espera.

5. **Given** un ajuste ya registrado, **When** se inspecciona su `reason` en el kardex, **Then** es
   texto libre sin restricción de valores en base de datos (sin `CheckConstraint`) — el catálogo
   de motivos canónicos de `app/core/inventory_reasons.py` es una convención de la capa de
   aplicación, no una regla forzada por el esquema; decisión documentada explícitamente en el
   propio módulo como deliberada, "para no bloquear motivos nuevos" (`RN-INV-12`, INTENCIONAL —
   no cambia con el requisito de "obligatorio" del punto anterior: obligatorio en presencia, libre
   en contenido).
6. **Given** el catálogo de motivos de movimiento, **When** se enumeran los seis motivos
   canónicos, **Then** son: `venta` (solo salida), `compra` (solo entrada), `ajuste` (entrada o
   salida), `daño` (solo salida), `vencimiento` (solo salida), `consumo_interno` (solo salida) —
   `ajuste` es el único que aparece simultáneamente en el catálogo de motivos válidos de entrada y
   de salida (`RN-INV-13`).

---

### User Story 5 - Alertas de "stock bajo": umbral por insumo, y quién incluye insumos inactivos (Priority: P1) — anomalía a corregir, requisito de negocio confirmado

Un encargado consulta qué insumos están en o por debajo de su nivel mínimo configurado, para
decidir qué reponer. El umbral de cada insumo es su propio `min_stock`, no un valor global del
sistema; y un insumo que se marcó inactivo pero conserva existencias residuales bajo su mínimo
debe seguir apareciendo en esa alerta, no desaparecer silenciosamente.

**Why this priority**: el negocio confirmó explícitamente (P9) que la brecha actual esconde
insumos descontinuados con stock bajo, lo que puede llevar a decisiones de reposición
equivocadas — es la anomalía con requisito de negocio más directo de todo este documento después
del invariante de mutación de stock.

**Independent Test**: se puede probar comparando la respuesta de `GET /inventory/items?
low_stock=true`, `GET /inventory/items/low-stock` y el reporte "insumos por reponer" para el mismo
insumo marcado `active=false` con `current_stock<=min_stock`.

**Acceptance Scenarios**:

1. **Given** un insumo «Leche» con `min_stock=10`, **When** su `current_stock` es exactamente
   `10`, **Then** se considera en stock bajo — la comparación es `current_stock <= min_stock`,
   con `<=`, no `<` (`RN-INV-14`).
2. **Given** el mismo insumo con `current_stock=10.001`, **When** se evalúa el mismo umbral,
   **Then** ya no se considera en stock bajo (`RN-INV-14`).
3. **Given** dos insumos con `min_stock` distintos (p. ej. `Leche: min_stock=10`,
   `Fresa: min_stock=2`), **When** se evalúa cada uno, **Then** el umbral se calcula
   individualmente por insumo — no existe un valor mínimo global configurable a nivel de sistema
   (`RN-INV-14`).

**Anomalía A-13/`RN-INV-15`, clasificación `ACCIDENTAL` **confirmada en P9** — esta spec fija el
comportamiento **corregido** como contrato, no el actual**: hoy el filtro `low_stock=true` de
`GET /inventory/items` **no** excluye insumos inactivos por defecto (solo si el cliente pide
explícitamente `active=true` además), mientras que el endpoint dedicado `GET
/inventory/items/low-stock` y el reporte "insumos por reponer" (`reports/service.py`) sí exigen
`active=True` siempre, de forma hardcodeada. Un insumo desactivado con existencias residuales bajo
su mínimo aparece en una pantalla y no en las otras dos.

4. **Given** un insumo «Colorante descontinuado» con `active=false` y
   `current_stock=0 <= min_stock=5`, **When** se consulta cada una de las tres fuentes hoy,
   **Then** el comportamiento observado difiere: aparece en `GET /inventory/items?
   low_stock=true` (sin filtro adicional de `active`) pero **no** aparece en `GET
   /inventory/items/low-stock` ni en el reporte "insumos por reponer" — comportamiento actual,
   documentado para que no se reintroduzca por accidente.
5. **Given** el mismo insumo «Colorante descontinuado» inactivo con stock residual bajo mínimo,
   **When** se aplique el contrato corregido de modernización, **Then** las **tres** fuentes
   DEBEN incluirlo en sus resultados de "stock bajo" — un insumo inactivo con stock residual bajo
   su mínimo sigue siendo relevante para la alerta; "inactivo" no implica "fuera de las alertas de
   reposición" (requisito de negocio confirmado en P9). El criterio DEBE unificarse en las tres
   pantallas: `current_stock <= min_stock`, sin exigir `active=True` salvo que el cliente lo pida
   explícitamente como filtro adicional.

---

### User Story 6 - Compra directa: alta de stock inmediata y el costo unitario que sobrescribe (Priority: P2) — anomalía intencional confirmada

Un encargado registra una compra "de contado" a un proveedor: cada ítem se marca recibido en su
totalidad de inmediato y el stock sube en el mismo acto. El costo unitario del insumo queda fijado
al de esa compra específica, reemplazando cualquier costo anterior, sin promediar con compras
previas.

**Why this priority**: es el camino de compra más simple y frecuente; junto con User Story 7
(orden + recepción) cubre el 100% de cómo entra stock por compra al sistema.

**Independent Test**: se puede probar enviando `POST /inventory/purchases` con uno o más ítems
contra un insumo con `current_stock` y `unit_cost` conocidos.

**Acceptance Scenarios**:

1. **Given** un insumo con `current_stock=10`, **When** se registra una compra directa con
   `quantity=50, unit_cost=2.00`, **Then** el ítem de compra queda `received_quantity=50` de
   inmediato, se genera un movimiento `type="in", quantity=50, reason="compra"`, el
   `current_stock` resultante es `60`, y la compra completa queda en estado `"received"` con
   `total=100.00` — todo en el mismo acto de crear la compra (`RN-INV-16`).
2. **Given** un insumo «Fresa» con `unit_cost=3.00`, **When** se compran 20 unidades a
   `unit_cost=5.00`, **Then** `item.unit_cost` pasa a ser `5.00` — el costo unitario del insumo se
   **sobrescribe** con el de la compra más reciente, sin importar que sea más caro que el
   anterior y sin calcular ningún costo promedio ponderado (`RN-INV-17`).

**Anomalía A-35 (porción `RN-INV-17`) — clasificación `DUDOSA` → **`INTENCIONAL` confirmado**
(entrevista de negocio, P23)**: el negocio confirmó explícitamente que el comportamiento deseado
es "último costo de compra", no un costo promedio ponderado. **Esta spec especifica el
Escenario 2 tal cual, como comportamiento intencional y definitivo** — no es una brecha a
corregir en la modernización.

---

### User Story 7 - Orden de compra y recepción parcial repetible (Priority: P2)

Un encargado crea una orden de compra a un proveedor sin que eso mueva stock todavía; luego, a
medida que la mercancía llega (posiblemente en varios envíos), va registrando recepciones
parciales que suman stock incrementalmente, hasta completar el pedido. No se puede recibir de más
sobre lo pendiente, ni seguir recibiendo sobre una orden ya completada.

**Why this priority**: es el segundo camino de entrada de stock del módulo, más elaborado que la
compra directa, y el que sostiene el flujo real de "pedir hoy, recibir en varios días" con un
proveedor.

**Independent Test**: se puede probar creando una orden con `POST /inventory/purchases/order` y
enviando dos llamadas sucesivas a `POST /inventory/purchases/{id}/receive` con cantidades
parciales que en conjunto no excedan lo pedido.

**Acceptance Scenarios**:

1. **Given** un insumo con `current_stock=10`, **When** se crea una orden de compra
   (`POST /inventory/purchases/order`) con `quantity=50, unit_cost=2.00`, **Then** el
   `current_stock` **no cambia** (sigue en `10`), la orden queda en estado `"draft"` con
   `received_quantity=0` — el stock solo sube al recibirla, nunca al crearla (`RN-INV-18`,
   referenciado en el propio código como RF-022).
2. **Given** una orden con `quantity=30` y `received_quantity=20` acumulado, **When** se intenta
   recibir `15` más, **Then** el sistema rechaza la operación completa con `422` ("Recepción 15
   excede lo pendiente (10)") — no se puede recibir más de lo que falta por ítem, y el rechazo
   revierte cualquier otro ítem del mismo request de recepción que sí fuera válido (`RN-INV-20`).
3. **Given** una orden cuyo estado es `"received"` (recibida por completo), **When** se intenta
   una recepción adicional sobre ella, **Then** el sistema la rechaza con `409` ("La compra ya fue
   recibida por completo"), sin importar la cantidad solicitada (`RN-INV-19`).
4. **Given** la misma orden con `quantity=30, received_quantity=20`, **When** se recibe la
   cantidad pendiente exacta (`10`), **Then** el sistema suma stock por esos `10` (movimiento
   `type="in", reason="compra"`), acumula `received_quantity=30`, y el estado de la orden pasa a
   `"received"`; si en cambio se hubiera recibido solo una parte de los `10` pendientes, el estado
   habría quedado en `"partial"` — el estado se recalcula tras cada recepción según cuánto queda
   pendiente en el conjunto de ítems (`RN-INV-21`).
5. **Given** una orden con `unit_cost=2.00` pactado por ítem al crearla, **When** se recibe ese
   ítem, **Then** el costo aplicado al insumo es el `unit_cost` pactado en la orden — el payload
   de recepción no admite (ni permite sobrescribir con) un costo unitario distinto (`RN-INV-22`).
6. **Given** cualquier ítem de una compra directa, una orden, o una línea de recepción, **When**
   se valida el payload, **Then** la cantidad DEBE ser estrictamente mayor que cero y el costo
   unitario DEBE ser mayor o igual a cero, de forma consistente en los tres flujos (`RN-INV-23`).

---

### User Story 8 - Conversión de unidades entre presentaciones de compra y venta (Priority: P3) — código muerto a completar, alcance acotado

Un insumo se compra en una unidad (p. ej. litros) y se vende o se descuenta del inventario en
otra unidad de la misma magnitud física (p. ej. onzas) — el caso real es el "granizado": materia
prima comprada en litros, servida en vasos medidos en onzas. El sistema necesita convertir la
cantidad entre esas dos unidades dentro de la misma dimensión (volumen); convertir entre
dimensiones distintas (por ejemplo, de masa a volumen) no tiene ningún caso de uso real
confirmado y queda fuera de alcance.

**Why this priority**: a diferencia del resto de esta spec, esta historia documenta una pieza que
**no funciona hoy** — es código muerto de una migración de esquema "Fase 1" nunca completada, sin
un solo import o uso en el repositorio. Se prioriza P3 (no bloqueante para el resto del kardex),
pero se especifica con el alcance ya confirmado para que la modernización la complete sin
reabrir la pregunta de si vale la pena.

**Independent Test**: hoy no es ejecutable contra datos reales — cualquier intento de invocar
`convert()` con instancias reales de `UnitMeasure` lanza `AttributeError`, no el `422` que el
propio módulo documenta. Una vez completada la migración de esquema, se podrá probar invocando
`convert(quantity, from_unit, to_unit)` con dos unidades de volumen reales (litro, onza) y
verificando el factor resultante.

**Acceptance Scenarios (estado actual — código muerto, no ejecutable)**:

1. **Given** el módulo `app/core/units.py`, **When** se inspecciona su función `convert()`,
   **Then** accede a `from_unit.dimension`, `to_unit.dimension` y `factor_to_base` — columnas que
   el modelo real `UnitMeasure` **no** define (solo `name`, `abbreviation`, `active`); invocarla
   con datos reales de la base de datos actual lanza `AttributeError`, no el `422` documentado en
   su propio docstring (`RN-CAT-40`, `RN-CAT-41`).
2. **Given** todo el repositorio, **When** se busca cualquier import o llamada a
   `app.core.units.convert`, **Then** no se encuentra ninguna — el módulo no se invoca desde
   ningún endpoint, servicio ni script hoy (`RN-CAT-41`).

**Anomalía A-31, clasificación `ACCIDENTAL` — alcance confirmado (P19 + ronda 3 simulada, P31),
esta spec especifica el contrato de la migración pendiente**: el propio docstring de `units.py`
llama a este mecanismo "Fase 1", indicando una migración de esquema planeada pero nunca
completada. La entrevista de negocio confirmó (P19) que **sí** hace falta completarla, con el
caso real de litros↔onzas del producto "granizado". El alcance se acotó en la ronda 3 (simulada,
P31):

3. **Given** la migración de esquema pendiente (agregar `dimension` y `factor_to_base` reales a
   `UnitMeasure`), **When** se complete en la modernización, **Then** DEBE soportar convertir
   entre dos unidades de la **misma** dimensión (p. ej. litro↔onza, ambas de volumen) usando
   `cantidad × factor_to_base(origen) / factor_to_base(destino)`, tal como documenta el docstring
   actual del módulo muerto.
4. **Given** la misma migración, **When** se evalúe su alcance, **Then** la conversión entre
   dimensiones **distintas** (p. ej. masa↔volumen, gramos↔mililitros) **queda fuera de alcance**
   de esta spec — no hay caso de uso real confirmado hoy, a diferencia de litros↔onzas. Esta spec
   no prohíbe agregarla en el futuro; solo constata que no es parte del contrato de modernización
   actual.

---

### Edge Cases

- **Movimiento con `type` distinto de `"in"`/`"out"`/`"adjustment"`** (valor arbitrario no
  reconocido): `record_movement` lo rechaza con el mismo `ValueError` de tipo inválido que usa
  para `"adjustment"` — el chequeo es "está en `('in', 'out')`", no una lista de exclusión
  (`RN-INV-03`).
- **Bloqueo de un solo insumo vs. varios**: `lock_items` con una lista vacía devuelve un
  diccionario vacío sin ejecutar ninguna consulta; con un solo id, sigue pasando por el mismo
  camino de "una sola consulta ordenada", sin atajo especial.
- **Ajuste con `signed_delta` positivo que deja el stock en un valor no entero** (p. ej.
  `current_stock=7.5`, ajuste `+2.25`): se acepta igual — el modelo de datos permite decimales
  (`max_digits=12, decimal_places=3`), sin redondeo implícito en `apply_adjustment`.
  `record_movement` usa la misma precisión decimal.
- **Recepción de una orden que ya está en estado `"partial"`**: se trata igual que recibir sobre
  una orden `"draft"` — el único estado que bloquea toda recepción adicional es `"received"`
  (`RN-INV-19`); `"partial"` sigue aceptando recepciones hasta completarse.
- **Compra directa o recepción con varios ítems en el mismo request, uno de los cuales falla la
  validación**: la excepción hace `rollback()` de toda la transacción — ningún ítem previo del
  mismo request queda aplicado, incluida la actualización de `unit_cost` (`app/api/v1/inventory/
  service.py`, manejo `except HTTPException` / `except Exception` con `db.rollback()` explícito
  en los tres flujos de compra).
- **Insumo con `min_stock=0`**: cualquier `current_stock<=0` lo marca en stock bajo (incluido
  `current_stock=0` exactamente); no hay un valor especial que desactive la alerta para un insumo
  individual salvo desactivarlo (`active=false`), y esa desactivación ya no debe excluirlo de las
  tres fuentes de alerta bajo el contrato corregido de User Story 5.
- **Motivo (`reason`) de más de 255 caracteres**: el schema `AdjustmentIn.reason` limita a
  `max_length=255`; un valor más largo se rechaza a nivel de validación de payload, antes de
  llegar a `apply_adjustment` — no forma parte de las reglas `RN-INV-*` documentadas, es una
  restricción de transporte del schema.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Todo movimiento de entrada o salida de stock DEBE pasar por `record_movement`, que
  recibe siempre una cantidad positiva; la dirección (sumar o restar) la determina exclusivamente
  el tipo de movimiento (`'in'` suma, `'out'` resta), nunca el signo de la cantidad (`RN-INV-01`).
- **FR-002**: Un movimiento con cantidad `<=0` DEBE rechazarse antes de tocar la fila del insumo
  (`RN-INV-02`).
- **FR-003**: `record_movement` NO DEBE aceptar `type="adjustment"`; los ajustes con signo DEBEN
  pasar exclusivamente por `apply_adjustment` (`RN-INV-03`).
- **FR-004**: Antes de leer o modificar el stock de un insumo, su fila DEBE bloquearse con
  `SELECT ... FOR UPDATE`, de forma que movimientos concurrentes sobre el mismo insumo se
  serialicen (`RN-INV-06`).
- **FR-005**: Cuando una operación necesita bloquear varios insumos a la vez, DEBE hacerlo en una
  sola consulta ordenada ascendentemente por id — nunca en el orden de procesamiento de la
  operación — para prevenir deadlocks entre transacciones concurrentes (`RN-INV-07`).
- **FR-006**: Una salida DEBE rechazarse si dejaría el stock del insumo por debajo de cero; dejarlo
  exactamente en cero SÍ debe permitirse (`RN-INV-04`).
- **FR-007 [Anomalía A-35/`RN-INV-05`, `PENDIENTE` — documentada sin especificar como contrato]**:
  existe un parámetro `allow_negative` en `record_movement` que, si se invoca en `True`, permite
  que una salida deje el stock negativo. Esta spec documenta su existencia sin fijar como
  contrato bajo qué condición de negocio debe usarse, a la espera de identificar qué llamador (si
  alguno, fuera de `inventory`) lo invoca así.
- **FR-008**: Un ajuste manual DEBE registrarse mediante un delta con signo (`apply_adjustment`),
  no mediante `record_movement`; el kardex DEBE guardar el movimiento resultante como
  `type='adjustment'` con `quantity=abs(signed_delta)`, siempre positiva (`RN-INV-08`).
- **FR-009**: Un ajuste manual DEBE rechazarse si dejaría el stock del insumo en negativo, sin
  ningún parámetro equivalente a `allow_negative` para forzarlo (`RN-INV-09`).
- **FR-010 [Anomalía A-35/`RN-INV-10`, `ACCIDENTAL` — corregir de inmediato]**: un ajuste con
  `signed_delta=0` DEBE rechazarse con un código HTTP de error de negocio (`422`) y un mensaje
  accionable, nunca con un `500` genérico. Hoy el `ValueError` que lanza `apply_adjustment` no
  tiene handler dedicado y propaga como `500`; esta spec fija como contrato que debe corregirse
  agregando la restricción a nivel de schema (`ne=0`) o un handler dedicado de `ValueError`.
- **FR-011 [Anomalía A-35/`RN-INV-11`, requisito de negocio confirmado en P23 — contrato
  corregido, no el estado actual]**: `POST /inventory/items/{id}/adjust` DEBE exigir `reason` no
  vacío (rechazar con `422` si está ausente, `null`, o compuesto solo de espacios). El estado
  actual (donde `reason` es opcional) se documenta en el Escenario 4 de User Story 4 como
  comportamiento observado, no como el contrato para la modernización.
- **FR-012**: El `reason` del kardex (de cualquier tipo de movimiento) DEBE seguir siendo texto
  libre a nivel de base de datos, sin `CheckConstraint` que lo restrinja a una lista cerrada — el
  catálogo de `app/core/inventory_reasons.py` es convención de aplicación, no restricción de
  esquema, y esta spec no cambia esa decisión (`RN-INV-12`).
- **FR-013**: El catálogo de motivos de movimiento DEBE incluir exactamente seis motivos
  canónicos: `venta`, `compra`, `ajuste`, `daño`, `vencimiento`, `consumo_interno`; `ajuste` DEBE
  ser el único válido tanto como entrada como salida (`RN-INV-13`).
- **FR-014**: Un insumo se considera en "stock bajo" cuando `current_stock <= min_stock`
  (comparación con `<=`, no `<`); `min_stock` es un atributo propio de cada insumo, no un valor
  global del sistema (`RN-INV-14`).
- **FR-015 [Anomalía A-13/`RN-INV-15`, `ACCIDENTAL` confirmado en P9 — contrato corregido]**: las
  tres fuentes que reportan "stock bajo" (`GET /inventory/items?low_stock=true`, `GET
  /inventory/items/low-stock`, y el reporte "insumos por reponer") DEBEN usar el mismo criterio:
  `current_stock <= min_stock`, sin excluir insumos inactivos por defecto. Un insumo desactivado
  con existencias residuales bajo su mínimo DEBE seguir apareciendo en las tres. El estado actual
  (donde el filtro de `/items` no excluye inactivos pero el endpoint dedicado y el reporte sí)
  se documenta como el comportamiento observado a corregir, no como contrato.
- **FR-016**: Una compra directa (`POST /inventory/purchases`) DEBE marcar cada ítem como recibido
  en su totalidad de inmediato y dar entrada de stock completa en el mismo acto, quedando la
  compra en estado `"received"` (`RN-INV-16`).
- **FR-017 [Anomalía A-35/`RN-INV-17`, `INTENCIONAL` confirmado en P23]**: toda entrada de stock
  por compra directa o por recepción de orden DEBE sobrescribir el `unit_cost` del insumo con el
  costo de esa compra específica, sin promediar con el costo anterior — "último costo de compra"
  es el comportamiento deseado, especificado tal cual, no un defecto a corregir.
- **FR-018**: Crear una orden de compra (`POST /inventory/purchases/order`) DEBE dejar el stock
  intacto; la orden queda en estado `"draft"` con `received_quantity=0` por ítem. El stock solo
  DEBE subir al recibirla (`RN-INV-18`).
- **FR-019**: Una orden en estado `"received"` (recibida por completo) NO DEBE aceptar ninguna
  recepción adicional; el intento DEBE rechazarse con `409` (`RN-INV-19`).
- **FR-020**: Ninguna recepción DEBE exceder, por ítem, la cantidad pendiente
  (`quantity - received_quantity` acumulado); si ocurre, la operación completa DEBE rechazarse con
  `422` y revertir cualquier otro ítem válido del mismo request (`RN-INV-20`).
- **FR-021**: Cada recepción parcial DEBE sumar stock únicamente por lo recibido en ese request,
  acumular `received_quantity`, y recalcular el estado de la orden: `"received"` si todo está
  completo, `"partial"` si algo se recibió pero no todo, `"draft"` si nada se ha recibido — la
  recepción DEBE poder repetirse hasta completar el pedido (`RN-INV-21`).
- **FR-022**: El costo unitario aplicado en una recepción DEBE ser el pactado en la orden al
  crearla (`PurchaseItem.unit_cost`); el payload de recepción NO DEBE admitir un costo unitario
  distinto (`RN-INV-22`).
- **FR-023**: En compra directa, en la creación de una orden, y en cada línea de recepción, la
  cantidad DEBE ser estrictamente mayor que cero y el costo unitario DEBE ser mayor o igual a
  cero, de forma consistente en los tres flujos (`RN-INV-23`).
- **FR-024 [Anomalía A-31, `ACCIDENTAL` — alcance confirmado en P19 + ronda 3 simulada P31]**:
  `app/core/units.py` DEBE completarse en la modernización para soportar conversión de cantidades
  entre dos unidades de **la misma dimensión** (caso real confirmado: litros↔onzas para el
  producto "granizado", materia prima comprada en litros, vendida en vasos de onzas). Hoy es
  código muerto: `convert()` referencia columnas (`dimension`, `factor_to_base`) que no existen en
  el modelo real `UnitMeasure`, y ningún import o llamada lo invoca en todo el repositorio
  (`RN-CAT-40`, `RN-CAT-41`).
- **FR-025 [Anomalía A-31, alcance acotado]**: la conversión entre unidades de **dimensiones
  distintas** (p. ej. masa↔volumen) queda explícitamente **fuera de alcance** de la migración
  especificada en FR-024 — no hay caso de uso real confirmado hoy.

### Key Entities *(include if feature involves data)*

- **InventoryItem**: insumo del inventario. Atributos relevantes a esta spec: `current_stock`,
  `min_stock` (umbral de stock bajo, por insumo), `unit_cost` (sobrescrito en cada compra,
  `RN-INV-17`), `active` (determina inclusión en las alertas de stock bajo bajo el contrato
  corregido de `RN-INV-15`), `unit_measure_id` (unidad en la que se expresa el stock).
- **InventoryMovement**: fila del kardex, la única forma en que se registra un cambio de stock.
  Atributos relevantes: `type` (`'in' | 'out' | 'adjustment'`), `quantity` (siempre positiva,
  `RN-INV-01`/`RN-INV-08`), `reason` (texto libre, `RN-INV-12`; obligatorio para ajustes bajo el
  contrato corregido de `RN-INV-11`), `reference_type`/`reference_id` (de dónde viene el
  movimiento — compra, venta, orden — ortogonal al motivo), `moved_at`.
- **Purchase**: compra a proveedor, directa o por orden. Atributos relevantes: `status`
  (`"draft" | "partial" | "received"`), `total` (suma de `quantity × unit_cost` de sus ítems),
  `supplier_id` (opcional).
- **PurchaseItem**: línea de una compra u orden. Atributos relevantes: `quantity` (pedida),
  `received_quantity` (acumulada tras cada recepción, `RN-INV-21`), `unit_cost` (pactado al crear
  la orden, aplicado sin cambios en la recepción, `RN-INV-22`).
- **UnitMeasure**: unidad de medida de un insumo. Atributos reales hoy: `name`, `abbreviation`,
  `active`. **No** define `dimension` ni `factor_to_base` — esas columnas son las que la migración
  pendiente de `RN-CAT-40`/`RN-CAT-41`/A-31 debe agregar para soportar conversión dentro de la
  misma dimensión (FR-024).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-INV-01` a `RN-INV-23`, `RN-CAT-40` y `RN-CAT-41` puede
  verificarse ejecutando los pasos descritos en esta spec contra un `pos-backend` en ejecución
  (salvo `RN-CAT-40`/`RN-CAT-41`, no ejecutables hoy por ser código muerto — verificables por
  inspección de código hasta que se complete la migración de FR-024), sin necesitar leer
  `stock.py`/`service.py`/`units.py` para entender el comportamiento esperado.
- **SC-002**: Ninguna salida de stock ni ajuste manual puede dejar `current_stock` en negativo
  salvo por la vía documentada y no especificada de `allow_negative` (`RN-INV-05`) — el 100% de
  los intentos de dejarlo negativo por las vías estándar (`record_movement` sin
  `allow_negative`, `apply_adjustment`) se rechaza con un error de negocio, nunca con una
  excepción no controlada.
- **SC-003**: Tras la corrección de FR-010 (A-35/RN-INV-10), el 0% de los ajustes con
  `signed_delta=0` produce una respuesta `500` — el 100% produce `422` con mensaje accionable.
- **SC-004**: Tras la corrección de FR-011 (A-35/RN-INV-11) y FR-015 (A-13/RN-INV-15), un
  encargado de inventario puede confiar en que (a) todo ajuste manual queda con un motivo
  registrado, sin excepción, y (b) las tres pantallas de alerta de stock bajo (listado filtrado,
  endpoint dedicado, reporte) muestran el mismo conjunto de insumos para el mismo criterio,
  incluidos los inactivos con stock residual bajo mínimo.
- **SC-005**: Esta spec documenta un **gap de caracterización notable**: ninguno de los 12
  scripts `test_*.py` existentes en `pos-backend/app/scripts/` cubre `inventory`/`stock.py` de
  forma dedicada (el más cercano por nombre, `test_cancel_inventory.py`, prueba la reversa de
  pedidos — spec 008, fuera de esta spec). No existe golden master ni characterization test para
  el módulo de mayor criticidad económica directa del sistema (mueve el costo de inventario y el
  stock real). Se registra como hallazgo de esta spec, no como requisito que ella misma resuelva.
- **SC-006**: El caso real de conversión litros→onzas para "granizado" (FR-024) puede expresarse
  y verificarse con un factor de conversión configurado en `UnitMeasure`, sin necesitar código
  específico por producto — la migración de esquema, una vez completa, DEBE bastar para cualquier
  par de unidades de la misma dimensión, no solo para ese caso concreto.

## Out of Scope

- **Qué insumo y cuánto consume cada línea de venta** (receta fija, opciones elegidas, chequeo
  preventivo de disponibilidad) — cubierto por la spec 003
  (`003-consumo-de-inventario-por-receta-y-opcion`). Esta spec asume que el `record_movement`/
  `apply_adjustment` ya reciben la cantidad correcta a mover; no especifica cómo se calculó.
- **El precio de venta de la variante** — cubierto por la spec 002
  (`002-catalogo-productos-variantes-y-precios`).
- **La validación de la selección de opciones** (`min_select`/`max_select`,
  `STRICT_OPTION_SELECTION`) — cubierto por la spec 004.
- **El descuento real de stock en el momento de confirmar/cobrar un pedido** (el llamador que
  invoca `record_movement` desde `orders/checkout.py` o `sales/`) — pertenece a las specs de
  cobro (008, 010, 011); esta spec solo documenta la función que esos módulos invocan, no quién ni
  cuándo la invoca.
- **La UI de captura de compras, ajustes y conversión de unidades** en `pos-heladeria` — esta
  spec documenta el contrato del backend; el frontend puede o no exigir hoy lo que el backend no
  exige (p. ej. `reason` en un ajuste), pregunta que esta spec no resuelve.
- **Conversión de unidades entre dimensiones distintas** (masa↔volumen, p. ej. gramos↔mililitros)
  — explícitamente fuera de alcance de FR-024/FR-025, sin caso de uso real confirmado.

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia de la guía
  general de este template ("evitar detalles de implementación"), aquí los endpoints, códigos de
  estado HTTP, nombres de campo y valores literales **son** el contrato observable que se está
  documentando — se citan explícitamente porque los criterios de aceptación deben ser verificables
  directamente contra `pos-backend` en ejecución.
- **Tres anomalías de este documento se especifican como contrato *corregido*, no como el estado
  actual**: RN-INV-10 (500 → 422, A-35), RN-INV-11 (motivo obligatorio, A-35/P23) y RN-INV-15
  (unificar criterio de stock bajo incluyendo inactivos, A-13/P9). Los tres tienen requisito de
  negocio ya confirmado en la entrevista, a diferencia del resto del documento, que documenta lo
  que el sistema ya hace tal cual.
- **RN-INV-05 (`allow_negative`) se documenta sin especificar como contrato**: sigue clasificada
  `PENDIENTE` porque no se identificó el llamador real (si alguno) que la invoca en `True`; esta
  spec no asume ni prohíbe su uso.
- **RN-INV-17 (costo unitario sobrescrito, no promediado) se especifica como contrato
  `INTENCIONAL` definitivo**: a diferencia de otras reglas `DUDOSA` de este proyecto que quedan
  pendientes, esta alcanzó confirmación de negocio explícita en P23 y no debe reabrirse como
  pregunta en la modernización.
- **El alcance de la migración de conversión de unidades (RN-CAT-40/RN-CAT-41/A-31) es
  deliberadamente acotado a la misma dimensión**: aunque el docstring original del código muerto
  sugiere un mecanismo general de conversión entre cualquier dimensión, el negocio confirmó (P19,
  ronda 3 simulada P31) que solo hay caso de uso real para litros↔onzas (volumen). Ampliar a
  masa↔volumen requeriría una nueva decisión de negocio, no cubierta aquí.
- **La ronda 3 de entrevista que cerró el alcance de A-31 (P31) fue simulada, no una entrevista
  real con el negocio** (`registro-de-anomalias.md`, aviso de método): antes de tratar el alcance
  litros↔onzas como definitivo para producción, el equipo de modernización debería ratificarlo
  con el negocio real, tal como señala el propio registro de anomalías.
- **Los valores numéricos citados en los escenarios** (20, 5, 2.5, 50, 2.00, 5.00, 30, 10, etc.)
  son ilustrativos, tomados de `reglas-de-negocio.md` y de la lectura directa de
  `stock.py`/`service.py`/`schemas.py` — no representan necesariamente el catálogo real vigente
  hoy en producción.
- **Ninguno de los 12 scripts de characterization existentes cubre este módulo** (SC-005): a
  diferencia de las specs 002/003/004, esta spec no puede citar un script existente como golden
  master parcial — es, en sí misma, el primer artefacto de contrato formal para este módulo.
