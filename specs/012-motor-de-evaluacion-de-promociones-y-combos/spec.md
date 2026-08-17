# Feature Specification: Motor de evaluación de promociones y combos — vigencia, mejor promoción por línea, y expansión de combo

**Feature Branch**: `012-motor-de-evaluacion-de-promociones-y-combos`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/promotions/service.py` (RN-PROMO-01 a RN-PROMO-45), para que sirva de
contrato formal de cara a la modernización (Principio I y Principio III de la
[Constitución](../../.specify/memory/constitution.md)). Es el dominio de "cuánto descuento le
corresponde a esta línea del carrito, cuál promoción gana si dos aplican a la vez, y cuándo un
combo se convierte en descuento real": la evaluación de vigencia en hora local del tenant (regla
`[PROTEGIDA]` A-07, con tres cambios estructurales de una reescritura reciente), el cálculo del
descuento por tipo (`percent`/`fixed`/`qty_price`), la selección de la mejor promoción por línea
cuando varias aplican, y la expansión/descuento de combos por bundle completo. Incluye una regla
protegida (A-07), tres anomalías con clasificación cerrada (A-08 y A-46 `ACCIDENTAL`, A-10
`ACCIDENTAL`), una anomalía `PENDIENTE` mitigada operativamente (A-09) y dos clusters `PENDIENTE`
sin decisión de negocio (A-36 porción promo, A-37 completo).

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE
del motor de cálculo de descuentos y combos del sistema POS Heladería — vigencia, mejor promoción
por línea, y expansión de combo, tomado de `reglas-de-negocio.md` (RN-PROMO-01 a RN-PROMO-45) y de
`registro-de-anomalias.md` (A-07, A-08, A-09, A-10, A-36, A-37, A-46), para que sirva de contrato
en la modernización."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `promotions/service.py`, no uno
  deseado. Las anomalías conocidas se marcan inline con su tratamiento acordado
  (registro-de-anomalias.md). A-07 es la regla protegida más importante de esta spec: una
  reescritura completa del motor con tres cambios estructurales, respaldada por
  `memoria-historica.md` #15 y por el único script de test que corre en CI.
-->

### User Story 1 - A-07 [PROTEGIDA]: la vigencia se evalúa en hora local del tenant, nunca en UTC (Priority: P1)

`local_now()`/`_tz()` convierten `now` a la zona horaria del tenant (`ZoneInfo(TENANT_TIMEZONE)`,
por defecto `America/Bogota`) antes de comparar estado, fechas, día de la semana y ventana
horaria. Antes de la reescritura del 2026-08-07 (commit `2e94a3ad`), la vigencia se evaluaba en
UTC crudo, lo que en UTC-5 corría no solo la ventana horaria sino el día de la semana, el día del
mes y el corte de `ends_at`.

**Why this priority**: es el defecto que la propia reescritura documenta como motivo de su
existencia — "un 20% los martes empezaba el lunes a las 19:00 locales, que es justo cuando una
heladería vende" (`memoria-historica.md` #15). Un error aquí cobra el precio equivocado en la hora
de mayor venta.

**Independent Test**: se puede probar invocando `_valid_now(promo, now)` con un `now` UTC
construido a mano y verificando que el día de la semana y la ventana horaria evaluados son los
correspondientes a la conversión a `America/Bogota`, sin necesitar un cobro real.

**Acceptance Scenarios**:

1. **Given** `TENANT_TIMEZONE=America/Bogota` (UTC-5) y una promoción con `days_of_week="1"`
   (martes), **When** se evalúa vigencia con `now` UTC = 2026-08-19 01:00 (miércoles en UTC),
   **Then** el sistema la considera vigente — la conversión a local da 2026-08-18 20:00 (martes),
   el día correcto (`RN-PROMO-01`, `RN-PROMO-47`).
2. **Given** la misma promoción, **When** se evalúa con `now` UTC = 2026-08-15 00:30, **Then** el
   día local evaluado es viernes 14 (19:30 local), no sábado 15 — el corte de día ocurre en hora
   local, no en UTC (`RN-PROMO-01`, ejemplo de `reglas-de-negocio.md`).
3. **Given** cualquier evaluación de vigencia, **When** se inspecciona `evaluate_detailed`,
   **Then** devuelve un desglose por línea (`LineDiscount` por cada línea del carrito), no un
   escalar total — el segundo de los tres cambios estructurales de la reescritura
   (`service.py:1-20`).
4. **Given** dos promociones aplicables a la misma línea con distinto `priority`, **When** se
   elige la mejor, **Then** gana la de mayor `priority` explícita, incluso si su descuento
   monetario es menor que el de la otra — el tercer cambio estructural (`RN-PROMO-13`,
   `RN-PROMO-14`; ver User Story 4 para el criterio completo de desempate).

**Nota — gobernanza y pregunta de negocio sin cerrar**: el commit que introdujo esta reescritura
(`2e94a3ad`, 2026-08-07) queda registrado bajo el autor genérico `refactor <dev@local>`, no una
persona identificable en el historial de `pos-backend`. El negocio podría querer indagar quién lo
escribió y lo revisó. Además, `memoria-historica.md` #15 deja abierta la pregunta de si el bug de
zona horaria llegó a afectar una promoción real en producción antes de esta fecha (reclamos de
clientes o cajeros). Ninguna de las dos preguntas bloquea la especificación de esta spec — el
comportamiento corregido es el que se documenta a continuación tal cual.

**Especificación de esta spec**: los tres cambios estructurales (hora local, desglose por línea,
prioridad explícita en el desempate) se fijan como invariantes de la reimplementación —
**especificar tal cual, no tocar**.

---

### User Story 2 - Ventana horaria con cruce de medianoche, límites inclusivos, y evaluación como AND estricto con cortocircuito (Priority: P1)

`_in_time_window()` admite que `start_time > end_time` (la ventana cruza medianoche): en ese caso
aplica si la hora actual es `>= start` **o** `<= end`. Sin `end_time`, la ventana va desde `start`
hasta el fin del día; sin `start_time`, desde medianoche hasta `end_time` inclusive; sin ninguno de
los dos, todo el día. `_valid_now` evalúa estado, `starts_at`, `ends_at`, día de la semana y hora
en ese orden exacto, con cortocircuito: si cualquier paso falla, no evalúa los siguientes.

**Why this priority**: la vigencia horaria es la condición de entrada a todo el resto del motor —
una promoción que no debería estar vigente en este instante no debe llegar nunca a la fase de
cálculo de descuento.

**Independent Test**: se puede probar invocando `_in_time_window(hora, start, end)` con los seis
casos de frontera (con/sin cruce de medianoche, límites exactos, sin `start`, sin `end`, sin
ninguno), sin depender de una promoción real ni de una fecha completa.

**Acceptance Scenarios**:

1. **Given** una ventana `22:00-02:00` (cruza medianoche), **When** la hora local es exactamente
   `22:00:00`, **Then** aplica — límite inclusivo (`RN-PROMO-02`).
2. **Given** la misma ventana, **When** la hora local es exactamente `02:00:00`, **Then** aplica
   — también inclusivo en el segundo límite (`RN-PROMO-02`).
3. **Given** la misma ventana, **When** la hora local es `02:00:01`, **Then** NO aplica — un
   segundo después del límite ya está fuera (`RN-PROMO-02`).
4. **Given** una promoción con `days_of_week="1"` (martes) evaluada un martes local a las 20:00
   Bogotá (01:00 miércoles UTC), **When** se filtra por día, **Then** califica como martes — el
   día se calcula sobre la hora ya convertida a local, nunca sobre UTC (`RN-PROMO-04`).
5. **Given** `days_of_week=" 1, ,3 "` (con espacios y un vacío), **When** se parsea el CSV,
   **Then** el resultado es `{1, 3}` — espacios y entradas vacías se descartan sin error
   (`RN-PROMO-05`).
6. **Given** una promoción con `status="paused"` que además cumple fecha, día y hora, **When** se
   evalúa `_valid_now`, **Then** el chequeo de estado falla primero y el resto de las condiciones
   ni se evalúa — el orden es estado→`starts_at`→`ends_at`→día→hora, con cortocircuito
   (`RN-PROMO-52`).

---

### User Story 3 - El descuento por línea se calcula según el target que coincide y el tipo de promoción (Priority: P1)

`_matching_target()` decide qué configuración de la promoción aplica a una línea del carrito: si
hay un target de producto exacto y uno de categoría que también coincide, gana el de producto.
Sin targets configurados, la promoción aplica global; con targets pero sin coincidencia, no aplica
a esa línea. A partir del target elegido, `_line_discount()` calcula el monto según el tipo:
`percent` es el porcentaje exacto del total de línea sin redondeo intermedio; `fixed` se topa al
total de línea (nunca queda negativo); `qty_price` solo descuenta paquetes completos, dejando el
remanente a precio normal.

**Why this priority**: es el cálculo económico central del motor — cualquier error aquí cobra un
monto distinto al que la promoción configurada debería producir.

**Independent Test**: se puede probar invocando `_matching_target` y `_line_discount` con líneas y
targets construidos a mano, sin pasar por un carrito ni un cobro real.

**Acceptance Scenarios**:

1. **Given** Target A (categoría "Granizados") y Target B (producto "Granizado de mora", que
   pertenece a esa categoría), **When** se evalúa una línea de ese producto, **Then** gana el
   Target B (producto) sobre el Target A (categoría) (`RN-PROMO-06`).
2. **Given** una línea con `line_total=15000` y una promoción `percent, value=20`, **When** se
   calcula el descuento, **Then** resulta `3000.00` exacto, sin redondeo intermedio
   (`RN-PROMO-08`).
3. **Given** una línea con `line_total=8000` y una promoción `fixed, value=50000`, **When** se
   calcula el descuento, **Then** resulta `8000` (la línea queda en $0, nunca negativo)
   (`RN-PROMO-09`).
4. **Given** un target `qty_price` con `pack=3`, `price_paquete=20000`, `unit_price=8000`, y una
   línea con `quantity=7`, **When** se calcula el descuento, **Then** `packs=floor(7/3)=2`,
   `normal=8000*3*2=48000`, descuento=`8000` — la 7ª unidad se cobra a $8.000 precio normal, sin
   entrar en ningún paquete (`RN-PROMO-10`).
5. **Given** una promoción `percent, value=100`, **When** se aplica a cualquier línea, **Then** la
   línea queda en `$0` exacto (`RN-PROMO-20`).
6. **Given** una línea sin `unit_price` explícito pero con `quantity=0` y `line_total=0`, **When**
   se deriva `unit_price = line_total/quantity`, **Then** el resultado es `0`, sin
   `ZeroDivisionError` (`RN-PROMO-18`).
7. **Given** una promoción `qty_price` con `min_qty` propio del target, **When** la cantidad de la
   línea es exactamente igual al mínimo, **Then** SÍ califica — frontera inclusiva (`>=`)
   (`RN-PROMO-12`).
8. **Given** una promoción sin ningún `Target` configurado (ni producto ni categoría), **When** se
   evalúa `qty_price`, **Then** nunca hay descuento — el paquete (tamaño+precio) vive solo en el
   target, `min_qty`/`value` de la `Promotion` nunca se usan como fallback ("sin precio no hay
   descuento: el fallo seguro en vez del caro") (`RN-PROMO-07`).

---

### User Story 4 - Selección de la mejor promoción por línea: prioridad, luego monto, luego antigüedad; exclusiones fijas del motor automático (Priority: P1)

Cuando varias promociones automáticas aplican a la misma línea, el motor elige exactamente una —
nunca acumula dos. El criterio de comparación es `(priority, amount, -created_at.timestamp())`:
gana la mayor prioridad; en empate, el mayor descuento monetario; en empate persistente, la
promoción más antigua (`created_at` menor). Una promoción cuyo descuento calculado es `<=0` no se
considera candidata. El motor automático excluye siempre el tipo `combo` y el tipo `buy_x_get_y`
(hardcodeado como no implementado). El filtro SQL de vigencia solo cubre `status` y la fecha de
corte; el resto (hora, día) se valida en Python sobre ese subconjunto ya reducido.

**Why this priority**: es el mecanismo que hace determinista y reproducible el resultado cuando
dos promociones compiten por la misma línea — sin él, el orden de evaluación sería arbitrario.

**Independent Test**: se puede probar invocando `best_line_discount`/`_best_line_match` con dos o
más promociones candidatas construidas a mano con distintos `priority`, `value` y `created_at`,
verificando cuál gana en cada combinación, sin pasar por un cobro real.

**Acceptance Scenarios**:

1. **Given** Promo A (`priority=1`, descuento resultante $2.000) y Promo B (`priority=2`,
   descuento resultante $1.000), **When** ambas aplican a la misma línea, **Then** gana B pese a
   descontar menos (`RN-PROMO-13`).
2. **Given** dos promociones con el mismo `priority` y el mismo monto calculado, **When** se
   desempata, **Then** gana la de `created_at` menor (más antigua) (`RN-PROMO-14`).
3. **Given** una promoción `percent`/`fixed` con `value=0` (permitido por el `CHECK` de base de
   datos), **When** se evalúa como candidata, **Then** nunca gana ninguna línea — un descuento
   `<=0` la descalifica como candidata aunque cumpla vigencia y cantidad mínima
   (`RN-PROMO-15 [DUDOSA]`, ver User Story 10, A-37).
4. **Given** el conjunto de promociones vigentes para una tenant, **When** el motor automático
   evalúa una línea, **Then** nunca considera promociones `type=combo` ni `type=buy_x_get_y` —
   ambas excluidas por diseño de `AUTO_TYPES` (`RN-PROMO-16`, `RN-PROMO-45`).
5. **Given** la consulta SQL que trae candidatas vigentes, **When** se inspecciona su `WHERE`,
   **Then** solo filtra por `status` y por la fecha de corte (`starts_at`/`ends_at`); el resto de
   las condiciones (hora, día de la semana) se evalúa después en Python sobre ese subconjunto —
   optimización de índice, no el filtro completo (`RN-PROMO-17`).
6. **Given** un cobro con `excluded_promotion_ids` conteniendo el ID de la promoción que ganaría
   una línea, **When** se reevalúa esa línea, **Then** el sistema recalcula considerando solo las
   demás promociones vigentes, permitiendo que otra de menor prioridad gane esa línea
   (`RN-PROMO-22`).
7. **Given** un cobro cuyas líneas descontadas fueron ganadas por dos promociones DIFERENTES,
   **When** se rellena el campo legado `Sale.promotion_id`, **Then** queda `NULL` — solo se
   rellena si TODAS las líneas descontadas comparten una única promoción, aunque el desglose
   completo (`evaluate_detailed`) sí registre cada una (`RN-PROMO-23`).
8. **Given** el desglose final de un cobro, **When** se suman los descuentos individuales de cada
   línea (sin redondear cada uno) y se calcula el total, **Then** el redondeo (`ROUND_HALF_UP` a 2
   decimales) ocurre una única vez, sobre la suma total — el desglose por línea puede no sumar
   exactamente al total redondeado (`RN-PROMO-21`).

---

### User Story 5 - El combo agrupa líneas por `combo_id` y descuenta solo bundles completos (Priority: P1)

`combo_discount_for_lines` agrupa las líneas del carrito que comparten un mismo `combo_id` y
calcula cuántos "bundles" completos puede armar con lo que hay: para cada ítem de la receta del
combo, divide la cantidad disponible en el carrito entre la cantidad requerida (división entera) y
toma el mínimo de todos los ítems como número de bundles completos. Si la misma variante aparece en
el carrito con precios distintos (por ejemplo, dos líneas separadas), se usa el precio **mínimo**
entre ellas para calcular lo que se cubre. `expand_combo` (usado al agregar el combo al carrito, no
al cobrar) exige que TODAS las variantes componentes estén activas, y solo permite expandir una
`Promotion` con `type='combo'` vigente.

**Why this priority**: es el segundo mecanismo de descuento automático del sistema, con su propia
lógica de agrupación independiente del resto del motor (el motor automático de User Story 4
excluye combos explícitamente).

**Independent Test**: se puede probar invocando `combo_discount_for_lines` con un carrito
construido a mano cuyas líneas comparten `combo_id`, en distintas proporciones respecto a la
receta del combo, sin pasar por un cobro real. `expand_combo` se puede probar por separado con
variantes activas/inactivas.

**Acceptance Scenarios**:

1. **Given** una receta de combo `{Variante A: 2, Variante B: 1}` y un carrito con A=5 unidades y
   B=3 unidades (mismo `combo_id`), **When** se calculan los bundles completos, **Then**
   `bundle_units = min(floor(5/2)=2, floor(3/1)=3) = 2` — solo 2 bundles completos, el resto queda
   a precio normal (`RN-PROMO-24`).
2. **Given** la Variante A del combo apareciendo en dos líneas del carrito con precios distintos
   (por ejemplo $8.000 y $8.500), **When** se calcula cuánto cubre el combo, **Then** usa el
   precio **mínimo** ($8.000) — "el cliente no paga el alto" (`RN-PROMO-25`).
3. **Given** `bundle_units=2` y `promo.value=15000` (precio del combo), **When** se calcula el
   descuento, **Then** precio del combo = `15000*2=30000`; si el costo normal cubierto
   (`covered_normal`) es $40.000, el descuento es `10000` (`RN-PROMO-26`, ver User Story 10, A-37
   para el caso límite sin validación contra combos mal configurados).
4. **Given** un combo cuyas variantes componentes incluyen una inactiva (`is_active=False`),
   **When** se intenta `expand_combo` (agregarlo al carrito), **Then** el sistema rechaza — TODAS
   las variantes deben estar activas (`RN-PROMO-28`).
5. **Given** una `Promotion` con `type != 'combo'` o con `status` distinto de vigente, **When** se
   intenta `expand_combo`, **Then** el sistema rechaza — solo un combo `type='combo'` vigente
   puede expandirse (`RN-PROMO-29`).

---

### User Story 6 - El solapamiento entre promociones es solo advertencia informativa, nunca bloquea (Priority: P2)

`find_overlaps()` y sus auxiliares (`_ranges_overlap`, `_csv_overlap`, `_times_overlap`,
`_scope_overlap`) detectan si dos promociones podrían competir por la misma línea en algún
instante, pero el resultado nunca bloquea la creación, edición ni el cálculo — es información para
quien administra promociones, resuelta en el momento del cobro por el criterio de prioridad de
User Story 4. El chequeo de rango de fechas y de días es conservador (asume solapamiento si falta
información en cualquiera de las dos promociones comparadas).

**Why this priority**: es un mecanismo de apoyo administrativo, no parte del cálculo de cobro en
sí — de ahí P2, un escalón por debajo de las User Stories 1-5, que sí determinan cuánto se cobra.

**Independent Test**: se puede probar invocando `find_overlaps` con pares de promociones
construidas a mano (con y sin `start_time`, con y sin `days_of_week`, con targets que se cruzan o
no), sin depender de datos reales de catálogo.

**Acceptance Scenarios**:

1. **Given** dos promociones vigentes con rangos de fecha y horario que se cruzan, **When** se
   listan o editan, **Then** el sistema las marca como solapadas en la respuesta, pero ninguna
   operación de creación/edición se bloquea por ello (`RN-PROMO-30`).
2. **Given** una promoción sin `ends_at` (indefinida) y otra con rango de fechas acotado, **When**
   se evalúa el solapamiento de fechas, **Then** se asume solapamiento (comportamiento
   conservador, prioriza avisar de más sobre avisar de menos) (`RN-PROMO-31`).
3. **Given** una promoción con `days_of_week=null` (todos los días) y otra con
   `days_of_week="1,3"`, **When** se evalúa el solapamiento de días, **Then** se asume
   solapamiento total — un CSV nulo en cualquiera de las dos implica solapamiento (`RN-PROMO-32`).
4. **Given** un target de producto de una promoción y un target de categoría de otra, donde el
   producto pertenece a esa categoría, **When** se evalúa el solapamiento de alcance, **Then** se
   consideran en conflicto (`RN-PROMO-34`).

---

### User Story 7 - A-08: la convención de hora local del motor de promociones no llegó al carrito del comensal ni al menú público (Priority: P2) — regla compartida, referenciada, no respecificada aquí

`cart/service.py` y `menu/router.py` siguen evaluando qué promociones mostrar con
`datetime.now(timezone.utc).replace(tzinfo=None)` — un `datetime` naive que en realidad está en
UTC. Como `local_now()` (User Story 1) asume que cualquier `datetime` naive que recibe **ya está**
en hora local y no lo convierte, estos dos puntos reproducen exactamente el bug de zona horaria que
la reescritura de A-07 corrigió en el resto del sistema.

**Why this priority**: el monto que se cobra realmente (los cuatro caminos de cobro real) no se ve
afectado — solo lo que el comensal ve antes de pagar, vía QR. P2, no P1, porque el dinero está a
salvo; el riesgo es de confianza y reclamos, no de pérdida económica directa.

**Independent Test**: se puede verificar por inspección de código comparando la construcción de
`now` en `cart/service.py:52-53,205-206` y `menu/router.py:82-83` contra `local_now()` en
`promotions/service.py:57-67`, sin necesitar reproducir el bug con datos reales.

**Acceptance Scenarios**:

1. **Given** `TENANT_TIMEZONE=America/Bogota` (UTC-5), **When** un comensal ve el menú público o
   su carrito por QR cerca de un límite de vigencia de promoción, **Then** el sistema puede
   mostrar una promoción como vigente o no vigente de forma distinta a como realmente se cobraría
   al confirmar, porque el `datetime` naive construido en UTC se trata como si ya fuera local
   (`RN-PROMO-01` no aplicada aquí; evidencia: `cart/service.py:52-53,205-206`,
   `menu/router.py:82-83`).
2. **Given** cualquiera de los cuatro caminos de cobro real (mostrador, cierre unificado, cierre
   dividido, `pay_order` legado), **When** se evalúa la promoción al momento de cobrar, **Then**
   todos usan `datetime` aware o `local_now()` correctamente — el monto cobrado nunca se ve
   afectado por este defecto, solo la vista previa del comensal.

**Tratamiento acordado** (`registro-de-anomalias.md`, A-08): **ACCIDENTAL**, en contraste directo
con A-07 (que sí se corrigió en los cuatro caminos de cobro real). La corrección natural (aplicar
el mismo patrón que ya funciona en `checkout.py`/`table_sessions/service.py`/`sales/service.py`)
pertenece a la **spec 007** (menú, carrito y QR del comensal), que es donde vive el código
afectado. Esta spec solo deja constancia de que el motor de promociones en sí (alcance de esta
spec) está corregido, y que la corrección pendiente vive fuera de su alcance.

---

### User Story 8 - A-09: el POS de staff previsualiza promociones con el reloj del dispositivo, sin conversión a la zona del tenant (Priority: P2) — `PENDIENTE`, mitigado operativamente

El frontend (`promotion-pricing.util.ts`, un "port" deliberado y documentado línea por línea del
motor Python) evalúa vigencia contra `new Date()` del navegador/sistema operativo del terminal —
sin conversión a `TENANT_TIMEZONE`, porque el cliente no tiene ese dato en ningún endpoint que
consuma. El cálculo real del descuento lo sigue haciendo siempre el backend (User Stories 1-5); el
frontend solo decide qué insignia y qué precio mostrar en pantalla antes de cobrar.

**Why this priority**: la divergencia solo se manifiesta si el reloj/zona del terminal físico
difiere de `America/Bogota` — un detalle de configuración de hardware, no del motor de cálculo en
sí (que es correcto). P2 porque el efecto es de experiencia y confianza del cajero, no de dinero
mal cobrado (el backend nunca delega el monto real al cliente).

**Independent Test**: se puede verificar por inspección de código, comparando
`isPromoActiveNow(promo, now)` en `promotion-pricing.util.ts:35-48` contra `_valid_now` del
backend, y confirmando que el comentario propio del port ("el cálculo real... lo sigue haciendo el
backend") reconoce la limitación.

**Acceptance Scenarios**:

1. **Given** un granizado de $8.000 con una promoción `percent, value=20` vigente 17:00-19:00 hora
   Bogotá, y un terminal cuyo reloj/SO marca UTC sin corregir, **When** el instante real es
   2026-08-15 17:30 Bogotá (22:30 UTC, dentro de la ventana), **Then** el backend lo considera
   vigente y cobra $6.400 al confirmar, pero el terminal (evaluando `22:30` contra la ventana
   `17:00-19:00`) lo considera fuera de ventana y muestra $8.000 sin descuento durante toda la
   construcción del pedido — diferencia de $1.600 por unidad que aparece recién al confirmar el
   pago (`contradiccion-01-motor-promociones-frontend-backend.md §3.1`).
2. **Given** un terminal con el reloj correctamente configurado a `America/Bogota`, **When** se
   compara lo que muestra el POS de staff contra lo que cobra el backend, **Then** ambos coinciden
   siempre — la divergencia depende exclusivamente de la configuración de hardware del
   dispositivo, no de un defecto del cálculo.

**Tratamiento acordado** (`registro-de-anomalias.md`, A-09): **PENDIENTE**, pero **mitigado
operativamente** — respuesta P6 de la entrevista de negocio (cajero jefe/soporte técnico): los
relojes de los terminales del local están verificados y fijados a `America/Bogota`. El defecto de
diseño sigue existiendo en el código (riesgo latente sin corregir), pero no se materializa hoy
mientras se mantenga esa disciplina de configuración; no hay incidente activo reportado. Esta spec
documenta el comportamiento observado sin fijarlo como contrato deseado ni obligatorio para la
modernización.

---

### User Story 9 - A-10: el desempate de "mejor promoción" del frontend no replica el criterio real del backend (Priority: P3) — `ACCIDENTAL`, sin efecto visible hoy

El backend desempata promociones con el mismo `priority` y el mismo monto calculado por
`created_at` (gana la más antigua, User Story 4). El frontend (`bestProductDiscount`, comparación
estricta `>`, nunca `>=`) no tiene acceso a `created_at` — su comportamiento real, no documentado
como tal, es "conservar la primera promoción del array", que llega ordenado por `GET /promotions`
con `priority.desc(), name` (orden alfabético en el empate).

**Why this priority**: divergencia de código verificable, pero sin efecto visible hoy — ninguna
pantalla del staff expone el *nombre* de la promoción ganadora en un empate, solo el monto. P3, la
más baja de esta spec.

**Independent Test**: se puede verificar por inspección de código, comparando la clave de
comparación `(priority, amount, -created_at.timestamp())` de `promotions/service.py:268` contra la
comparación estricta `>` sin tercer criterio de `promotion-pricing.util.ts:118-121,160`.

**Acceptance Scenarios**:

1. **Given** dos promociones `percent, value=10, priority=5` (empatadas), ambas vigentes, mismo
   destino: "Zapatos Gratis" (`created_at=2026-01-10`, más antigua) y "Aniversario"
   (`created_at=2026-07-01`, más reciente); para una línea de $10.000 ambas producen el mismo
   `amount` ($1.000), **When** el backend elige la mejor, **Then** gana "Zapatos Gratis" (la más
   antigua) — queda registrada en `Sale.promotion_id` y en `LineDiscount.promotion_name`
   (`RN-PROMO-14`).
2. **Given** el mismo escenario, **When** el frontend calcula `bestProductDiscount` sobre la lista
   ordenada por `priority.desc(), name`, **Then** conserva "Aniversario" (alfabéticamente antes
   que "Zapatos Gratis") — un resultado distinto al del backend.
3. **Given** que el monto numérico coincide en ambos casos ($1.000, porque el `value` es igual),
   **When** se inspecciona lo que ve el cajero (insignia `-10%`, precio con descuento), **Then**
   nada en pantalla delata la divergencia hoy — ninguna pantalla del staff muestra el *nombre* de
   la promoción ganadora en un empate.

**Tratamiento acordado** (`registro-de-anomalias.md`, A-10): **ACCIDENTAL**, documentado sin
especificar. Se registra como riesgo latente en el código, no como divergencia numérica confirmada
con datos reales de catálogo. Se corrige en fase de modernización solo si se agrega una pantalla
que muestre el nombre de la promoción ganadora antes de cobrar — el momento en que este defecto se
volvería visible.

---

### User Story 10 - A-37: cluster de casos límite del cálculo de descuento y combo (Priority: P3) — `PENDIENTE`, sin especificar

Cuatro comportamientos del cálculo (de los siete del cluster completo de A-37; los otros tres
pertenecen a la máquina de estados de administración, fuera de alcance — spec 013): (1) `qty_price`
nunca genera un descuento negativo si el paquete configurado resulta más caro que lo normal,
ocultando en silencio una mala configuración de precios; (2) lo mismo para combos —
precio=`value×bundles` sin validar contra el costo real de los componentes; (3) un descuento
calculado en `$0` descalifica la promoción como candidata (User Story 4), sin distinguir "promo
deshabilitada a propósito" de "error de captura"; (4) la cantidad de línea se trunca a entero, no
se redondea, al comparar contra `min_qty`; (5) un combo que deja de ser vigente entre agregarlo al
carrito y cobrar no avisa al cajero — el cobro simplemente no descuenta, a diferencia de
`expand_combo` (al agregarlo), que sí lanza una excepción explícita en los mismos casos.

**Why this priority**: son hallazgos de configuración y de casos límite, sin impacto económico
demostrado — de ahí P3, la más baja junto con User Story 9.

**Independent Test**: cada uno se puede verificar por inspección de código y con datos de prueba
puntuales — por ejemplo, un target `qty_price` con `price_paquete` mayor al normal por paquete
(`service.py:159`), o una `Promotion` con `value=0` evaluada como candidata (`service.py:265-267`)
— sin necesitar reproducir el escenario con datos reales de catálogo.

**Acceptance Scenarios**:

1. **Given** un target `qty_price` con `pack=2`, `price_paquete=25000`, `unit_price=8000` (normal
   por 2 = $16.000), **When** se calcula el descuento, **Then** `16000-25000=-9000` se topa a `0`
   — la mala configuración (paquete más caro que lo normal) queda enmascarada en silencio, sin
   aviso (`RN-PROMO-11 [DUDOSA]`).
2. **Given** `bundle_units=2` y `promo.value=45000` (precio del combo más caro que sus componentes
   normales), **When** se calcula el descuento del combo, **Then** el resultado es `0` — mismo
   patrón de enmascaramiento silencioso (`RN-PROMO-26 [DUDOSA]`).
3. **Given** una promoción `percent`/`fixed` con `value=0`, **When** se evalúa como candidata,
   **Then** queda descalificada de "mejor promoción" para cualquier línea — no hay forma de
   distinguir, desde el resultado, si `value=0` es una promoción deshabilitada a propósito o un
   error de captura (`RN-PROMO-15 [DUDOSA]`, User Story 4).
4. **Given** una línea con `quantity=3.9` y `min_qty=4`, **When** se compara contra el mínimo,
   **Then** `int(3.9)=3` no califica, pese a estar más cerca de 4 que de 3 — la cantidad se trunca,
   no se redondea (`RN-PROMO-19 [DUDOSA]`).
5. **Given** un combo agregado al carrito mientras estaba vigente, que pasa a `finished` antes de
   confirmarse el cobro, **When** se calcula el cobro final, **Then** el sistema no falla ni avisa
   — simplemente no aplica el descuento del combo, cobrando esas líneas a precio normal
   (`RN-PROMO-27 [DUDOSA]`, contraste: `expand_combo` al agregarlo sí lanza 409/422 en el mismo
   caso).

**Tratamiento acordado** (`registro-de-anomalias.md`, A-37): **PENDIENTE** las cinco pertenecientes
al cálculo (las otras dos del cluster completo pertenecen a la máquina de estados de
administración, spec 013). Documentar sin especificar hasta respuesta de negocio; ninguna de las
cinco se fija como comportamiento deseado ni obligatorio para la modernización.

---

### User Story 11 - A-46: la zona horaria de evaluación es un único valor global de la instancia, no por tenant (Priority: P3) — `ACCIDENTAL`, cerrado sin urgencia

`_tz()` lee un único `TENANT_TIMEZONE` de la configuración de la instancia — no una columna por
tenant. El propio comentario del código reconoce esta limitación explícitamente ("cuando `Tenant`
tenga su columna `timezone`, este es el único punto que cambia").

**Why this priority**: sin impacto hoy, porque hasta donde el código revela solo hay una zona
configurada para toda la instancia. P3, la más baja junto con User Stories 9 y 10.

**Independent Test**: se puede verificar por inspección de código —`core/config.py:17-21` define
`TENANT_TIMEZONE` como un único valor global, y `promotions/service.py:51-54` lo consume sin
parámetro de tenant— sin necesitar una instancia multi-tenant real para confirmarlo.

**Acceptance Scenarios**:

1. **Given** una instancia con múltiples tenants, **When** se evalúa la vigencia de cualquier
   promoción de cualquier tenant, **Then** todos usan la misma zona horaria configurada a nivel de
   instancia — no existe ningún mecanismo para que un tenant en otra región tenga su propia zona
   (`RN-PROMO-01`, evidencia: `core/config.py:17-21`, `promotions/service.py:51-54`).

**Tratamiento acordado** (`registro-de-anomalias.md`, A-46): **ACCIDENTAL**, limitación de diseño
reconocida explícitamente en el código, sin impacto hoy. **Cerrado sin urgencia** — respuesta P26
de la entrevista de negocio (dueño/gerente): sin planes de expansión a otra zona horaria. Esta spec
documenta el comportamiento actual sin exigir su corrección; se corrige solo si el negocio confirma
planes de multi-tenant multi-zona horaria.

---

### User Story 12 - A-36 (porción promociones): asimetría de precisión entre `starts_at`/`ends_at`, y casos límite sin cobertura de test (Priority: P3) — `PENDIENTE`, sin especificar

Tres casos límite de precisión, sin confirmación de negocio ni cobertura de test completa: (1)
`starts_at` se compara con datetime completo (hora incluida) mientras `ends_at` se compara solo por
fecha, ignorando la hora — así "Hasta 04/08" cubre el 04/08 completo hasta medianoche local del
05/08; (2) el solapamiento de horario entre dos promociones (User Story 6) solo se detecta con
precisión si **ambas** definen `start_time` — si a cualquiera le falta, se asume solapamiento
total; (3) la ventana horaria con cruce de medianoche (User Story 2) no está cubierta por test en
su segundo límite exacto.

**Why this priority**: son casos límite de precisión sin impacto económico demostrado — P3, la más
baja de esta spec, compartida con User Stories 9, 10 y 11.

**Independent Test**: (1) y (3) se pueden verificar comparando el código de `_valid_now`
(`service.py:94-99`) contra `test_promotions_rules.py:92-96` (no cubre el instante exacto del
segundo límite); (2) se puede verificar por inspección de `_times_overlap`
(`service.py:462-466`).

**Acceptance Scenarios**:

1. **Given** `starts_at=2026-08-15 09:00:00`, `ends_at=2026-08-20 00:00:00`, **When** se evalúa a
   las 2026-08-15 08:59 local, **Then** NO es válida (falta un minuto para `starts_at`, comparado
   con precisión de hora) (`RN-PROMO-03 [DUDOSA]`).
2. **Given** la misma promoción, **When** se evalúa a las 2026-08-20 23:59 local, **Then** SÍ es
   válida — `ends_at` se compara solo por fecha, cubriendo el día 20 completo hasta medianoche del
   21 (`RN-PROMO-03`, `RN-PROMO-49`).
3. **Given** la misma promoción, **When** se evalúa a las 2026-08-21 00:00 local, **Then** ya NO es
   válida — el corte real es medianoche del día siguiente a `ends_at.date()`.
4. **Given** dos promociones donde solo una define `start_time`, **When** se evalúa si sus
   horarios se solapan (User Story 6), **Then** se asume solapamiento total — el chequeo preciso
   exige que **ambas** definan `start_time` (`RN-PROMO-33 [DUDOSA]`).
5. **Given** la ventana con cruce de medianoche de User Story 2, **When** se revisa la suite
   `test_promotions_rules.py`, **Then** el segundo límite exacto (`02:00:00` en el ejemplo
   `22:00-02:00`) no tiene un caso de test dedicado — el comportamiento inclusivo se infiere solo
   de los operadores del código, sin evidencia de ejecución (`RN-PROMO-51 [DUDOSA]`).

**Tratamiento acordado** (`registro-de-anomalias.md`, A-36, porción promociones): **PENDIENTE**
las tres, sin impacto económico demostrado. Documentar sin especificar; candidatas naturales a
convertirse en casos de test explícitos si se decide reforzar `test_promotions_rules.py`.

---

### Edge Cases

- **Promoción vigente por fecha/hora pero `status="paused"`**: nunca se considera vigente — el
  chequeo de estado es el primer paso del AND estricto con cortocircuito (`RN-PROMO-52`, User
  Story 2).
- **Dos promociones automáticas empatadas en `priority` y en monto**: gana la más antigua por
  `created_at`; el frontend diverge en este punto sin efecto visible hoy (`RN-PROMO-14`, User
  Stories 4 y 9).
- **Target `qty_price` configurado con un precio de paquete más caro que el precio normal**: el
  descuento resultante nunca es negativo, se topa a `0` silenciosamente (`RN-PROMO-11`, User
  Story 10).
- **Combo cuya vigencia expira entre que se agrega al carrito y se cobra**: el cobro no falla ni
  avisa, simplemente no descuenta esas líneas (`RN-PROMO-27`, User Story 10).
- **Comensal viendo el menú/carrito por QR cerca de un límite de vigencia**: puede ver una
  promoción vigente/no vigente distinta de lo que el sistema cobraría — el motor en sí (esta spec)
  está corregido, el defecto vive en `cart/service.py`/`menu/router.py` (`RN-PROMO-01` no
  aplicada, A-08, fuera de alcance — spec 007).
- **Terminal de staff con reloj/zona horaria mal configurada**: puede mostrar un precio con o sin
  descuento distinto al que el backend cobrará realmente al confirmar — mitigado operativamente
  hoy (A-09, User Story 8).
- **`ends_at` sin hora explícita comparado contra `starts_at` con hora explícita**: asimetría de
  precisión documentada, sin impacto económico confirmado (`RN-PROMO-03`, User Story 12).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Regla protegida A-07, User Story 1]**: La vigencia de cualquier promoción DEBE
  evaluarse siempre convirtiendo `now` a la hora local del tenant (`ZoneInfo(TENANT_TIMEZONE)`),
  nunca en UTC crudo — el día de la semana, el corte de fecha y la ventana horaria se calculan
  todos sobre esa conversión (`RN-PROMO-01`, `RN-PROMO-47`).
- **FR-002 [Regla protegida A-07]**: La evaluación detallada de un cobro (`evaluate_detailed`)
  DEBE devolver un desglose por línea (una entrada por cada línea con su promoción y monto), nunca
  un total escalar (`service.py:1-20`).
- **FR-003 [Regla protegida A-07]**: Ante dos o más promociones automáticas aplicables a la misma
  línea, DEBE ganar siempre la de mayor `priority` explícita, sin importar si su descuento
  monetario es menor que el de otra candidata (`RN-PROMO-13`).
- **FR-004**: El desempate entre promociones con la misma `priority` DEBE resolverse primero por
  mayor monto de descuento calculado, y si persiste el empate, por la promoción de `created_at`
  menor (más antigua) (`RN-PROMO-14`).
- **FR-005**: La ventana horaria de una promoción DEBE admitir cruce de medianoche
  (`start_time > end_time`): aplica si la hora local es `>= start` o `<= end`; ambos límites DEBEN
  ser inclusivos (`RN-PROMO-02`).
- **FR-006**: Sin `end_time` configurado, la ventana DEBE cubrir desde `start_time` hasta el fin
  del día; sin `start_time`, desde medianoche hasta `end_time` inclusive; sin ninguno de los dos,
  el día completo (`RN-PROMO-02`).
- **FR-007 [`[DUDOSA]`, anomalía A-36, `PENDIENTE` — documentada sin especificar como contrato]**:
  `starts_at` DEBE compararse con precisión de datetime completo (fecha y hora); `ends_at` DEBE
  compararse solo por fecha, cubriendo el día completo hasta medianoche local del día siguiente
  (`RN-PROMO-03`, `RN-PROMO-48`, `RN-PROMO-49`).
- **FR-008**: El filtro por día de la semana DEBE leerse de un CSV de enteros donde `0=lunes` según
  `datetime.weekday()` (`RN-PROMO-04`, `RN-PROMO-50`).
- **FR-009**: El parsing del CSV de días DEBE ser tolerante: espacios y entradas vacías se
  descartan sin generar error (`RN-PROMO-05`).
- **FR-010 [Regla crítica]**: La evaluación de vigencia DEBE seguir un AND estricto con
  cortocircuito, en este orden exacto: estado → `starts_at` → `ends_at` → días de la semana → hora
  (`RN-PROMO-52`).
- **FR-011**: Cuando un target de producto y uno de categoría coinciden ambos con la misma línea
  (el producto pertenece a la categoría del target), DEBE ganar el target de producto
  (`RN-PROMO-06`).
- **FR-012**: Sin ningún target configurado, la promoción DEBE aplicar de forma global; con
  targets configurados pero sin ninguna coincidencia, la promoción NO DEBE aplicar a esa línea
  (`RN-PROMO-06`).
- **FR-013**: En promociones `qty_price`, el paquete (tamaño y precio) DEBE vivir exclusivamente en
  el target elegido — `min_qty`/`value` propios de la `Promotion` NUNCA DEBEN usarse como fallback
  (`RN-PROMO-07`).
- **FR-014**: El descuento `percent` DEBE calcularse como el porcentaje exacto del total de línea,
  sin ningún redondeo intermedio (`RN-PROMO-08`).
- **FR-015**: El descuento `fixed` DEBE toparse al total de línea — nunca puede dejar la línea en
  un monto negativo (`RN-PROMO-09`).
- **FR-016**: El descuento `qty_price` DEBE aplicarse solo a paquetes completos (división entera de
  la cantidad de línea entre el tamaño del paquete); el remanente DEBE cobrarse a precio normal
  (`RN-PROMO-10`).
- **FR-017 [`[DUDOSA]`, anomalía A-37, `PENDIENTE` — documentada sin especificar como contrato]**:
  Si el precio de paquete configurado en `qty_price` resulta más caro que el precio normal
  equivalente, el descuento calculado DEBE toparse a `0` (nunca negativo), sin ninguna validación
  que impida esa configuración (`RN-PROMO-11`).
- **FR-018**: La cantidad mínima de una promoción (o del target, en `qty_price`) DEBE evaluarse con
  frontera inclusiva: `quantity == minimo` SÍ califica (`RN-PROMO-12`).
- **FR-019**: El motor de evaluación automática DEBE excluir siempre las promociones `type=combo`
  del cálculo por línea (`RN-PROMO-16`); combos se calculan por un mecanismo separado (User
  Story 5).
- **FR-020**: El filtro SQL de vigencia DEBE ser parcial (solo `status` y fecha de corte); el resto
  de las condiciones de vigencia (hora, día) DEBEN validarse en Python sobre ese subconjunto ya
  reducido (`RN-PROMO-17`).
- **FR-021**: Si `unit_price` no viene explícito, DEBE derivarse de `line_total/quantity`, con
  protección explícita contra división por cero (resultado `0` si `quantity=0`) (`RN-PROMO-18`).
- **FR-022 [`[DUDOSA]`, anomalía A-37, `PENDIENTE` — documentada sin especificar como contrato]**:
  La cantidad de línea DEBE truncarse a entero (no redondearse) antes de compararla contra la
  cantidad mínima de una promoción (`RN-PROMO-19`).
- **FR-023**: Un descuento `percent` con `value=100` DEBE dejar la línea en `$0` exacto
  (`RN-PROMO-20`).
- **FR-024 [Regla crítica]**: El redondeo del descuento total DEBE ocurrir una única vez, al final,
  con `ROUND_HALF_UP` a 2 decimales — los montos individuales por línea NO DEBEN redondearse antes
  de sumarse; el desglose por línea puede no sumar exactamente al total ya redondeado
  (`RN-PROMO-21`).
- **FR-025**: El sistema DEBE aceptar un conjunto `excluded_promotion_ids` al evaluar un cobro,
  recalculando las líneas afectadas considerando solo las promociones vigentes restantes
  (`RN-PROMO-22`).
- **FR-026**: El campo legado `Sale.promotion_id` DEBE rellenarse únicamente si TODAS las líneas
  descontadas de ese cobro comparten una única promoción; en cualquier otro caso (cero promociones,
  o dos o más distintas) DEBE quedar `NULL`, aunque el desglose completo sí registre cada línea
  individualmente (`RN-PROMO-23`).
- **FR-027**: El motor automático DEBE excluir por diseño el tipo `buy_x_get_y` — hardcodeado como
  no implementado en el cálculo, aunque el dominio conceptual lo contemple (`RN-PROMO-45`).
- **FR-028**: El descuento de combo DEBE agrupar las líneas del carrito por `combo_id` y calcular
  el número de bundles completos como el mínimo, entre todos los ítems de la receta, de la
  división entera entre cantidad disponible y cantidad requerida (`RN-PROMO-24`).
- **FR-029**: Cuando la misma variante componente de un combo aparece en el carrito con precios
  distintos en líneas separadas, el cálculo DEBE usar el precio **mínimo** entre ellas
  (`RN-PROMO-25`).
- **FR-030 [`[DUDOSA]`, anomalía A-37, `PENDIENTE` — documentada sin especificar como contrato]**:
  El precio del combo DEBE calcularse como `value × bundle_units`; el descuento resultante NUNCA
  DEBE ser negativo, sin ninguna validación que impida configurar un combo más caro que sus
  componentes normales (`RN-PROMO-26`).
- **FR-031 [`[DUDOSA]`, anomalía A-37, `PENDIENTE` — documentada sin especificar como contrato]**:
  Un combo que ya no existe, no está vigente, o no tiene receta configurada, al momento del cobro
  final NO DEBE generar ningún error — simplemente no aplica descuento a esas líneas, a diferencia
  de `expand_combo` (al agregarlo al carrito), que sí DEBE rechazar esos mismos casos con una
  excepción explícita (`RN-PROMO-27`).
- **FR-032**: `expand_combo` DEBE exigir que TODAS las variantes componentes del combo estén
  activas; si alguna no lo está, DEBE rechazar la operación (`RN-PROMO-28`).
- **FR-033**: Solo una `Promotion` con `type='combo'` y vigente DEBE poder expandirse/activarse
  como combo (`RN-PROMO-29`).
- **FR-034**: El solapamiento detectado entre dos promociones DEBE ser exclusivamente informativo
  — NUNCA DEBE bloquear la creación, edición ni el cálculo de un cobro (`RN-PROMO-30`).
- **FR-035**: El chequeo de solapamiento de rango de fechas DEBE ser conservador: si a cualquiera
  de las dos promociones comparadas le falta información de fecha, DEBE asumirse solapamiento
  (`RN-PROMO-31`).
- **FR-036**: El chequeo de solapamiento de días DEBE asumir solapamiento total si el CSV de
  `days_of_week` de cualquiera de las dos promociones es nulo (`RN-PROMO-32`).
- **FR-037 [`[DUDOSA]`, anomalía A-36, `PENDIENTE` — documentada sin especificar como contrato]**:
  El chequeo de solapamiento de horario SOLO DEBE detectarse con precisión si AMBAS promociones
  comparadas definen `start_time`; si a cualquiera le falta, DEBE asumirse solapamiento total
  (`RN-PROMO-33`).
- **FR-038**: El chequeo de solapamiento de alcance DEBE considerar en conflicto un target de
  producto y un target de categoría cuando el producto pertenece a esa categoría (`RN-PROMO-34`).
- **FR-039 [Anomalía A-08, ACCIDENTAL — corrección fuera del alcance de esta spec, ver spec 007]**:
  Esta spec documenta que la convención de hora local (`FR-001`) NO está aplicada en
  `cart/service.py` ni en `menu/router.py` — ambos siguen construyendo `now` en UTC naive,
  reproduciendo el bug que `FR-001` corrigió en el resto del sistema. El monto cobrado en los
  cuatro caminos de cobro real no se ve afectado; solo la vista previa del comensal por QR. La
  corrección de esta discrepancia pertenece a la spec 007, no a esta.
- **FR-040 [Anomalía A-09, `PENDIENTE`, mitigado operativamente — documentada sin especificar como
  contrato]**: El POS de staff (`promotion-pricing.util.ts`) evalúa vigencia con el reloj del
  dispositivo terminal, sin conversión a `TENANT_TIMEZONE` — el cliente no recibe ese dato en
  ningún endpoint. El backend nunca delega el monto real al cliente: el cálculo de `FR-001` a
  `FR-038` sigue ejecutándose siempre en el servidor al confirmar el cobro. Riesgo mitigado
  operativamente (relojes de terminal verificados y fijados a `America/Bogota`); sin incidente
  activo reportado.
- **FR-041 [Anomalía A-10, ACCIDENTAL — documentada sin especificar como contrato]**: El desempate
  de "mejor promoción" del frontend (`bestProductDiscount`) NO replica el tercer criterio del
  backend (`created_at`, `FR-004`) — al no tener ese campo, conserva la primera promoción del
  array recibido (ordenado por `priority.desc(), name`). Sin efecto visible hoy porque ninguna
  pantalla del staff muestra el nombre de la promoción ganadora en un empate.
- **FR-042 [Anomalía A-46, ACCIDENTAL, cerrado sin urgencia — documentada sin especificar como
  contrato]**: La zona horaria usada por `FR-001` DEBE leerse hoy de un único `TENANT_TIMEZONE`
  global de la instancia (`core/config.py`), no de una columna por tenant — limitación de diseño
  reconocida explícitamente en el propio comentario del código, sin impacto mientras la
  plataforma aloje una sola zona horaria.

### Key Entities *(include if feature involves data)*

- **Promotion**: la promoción o combo configurado. Atributos relevantes a esta spec: `type`
  (`percent`/`fixed`/`qty_price`/`combo`/`buy_x_get_y`, solo los tres primeros y `combo` participan
  del cálculo automático, `RN-PROMO-16`, `RN-PROMO-45`), `status` (solo `active` habilita el
  descuento, primer paso del AND de `FR-010`), `priority` (criterio principal de desempate,
  `FR-003`), `created_at` (desempate final, `FR-004`), `starts_at`/`ends_at`
  (`FR-007`), `start_time`/`end_time`/`days_of_week` (ventana horaria y filtro de día, `FR-005` a
  `FR-009`), `value` (monto o porcentaje del descuento).
- **PromotionTarget**: el alcance de una promoción sobre un producto o categoría específicos.
  Atributos relevantes: `product_id` XOR `category_id` (target gana por especificidad,
  `FR-011`/`FR-012`), `value`/`min_qty` propios (solo usados en `qty_price`, `FR-013`).
- **ComboItem**: componente de la receta de un combo (`combo_id`, variante requerida, cantidad
  requerida). Gobierna el cálculo de bundles completos (`FR-028`) y la exigencia de variantes
  activas al expandir (`FR-032`).
- **LineDiscount**: entrada del desglose por línea que devuelve `evaluate_detailed` (`FR-002`) —
  línea, promoción ganadora (si hay), monto de descuento. Es la unidad atómica que consumen los
  cuatro caminos de cobro real y el reporte de promociones aplicadas.
- **Sale**: venta resultante de un cobro. Atributo relevante a esta spec: `promotion_id` (legado,
  solo se rellena si una única promoción explica todas las líneas descontadas, `FR-026`).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-PROMO-01` a `RN-PROMO-45` puede verificarse ejecutando los
  pasos descritos en esta spec contra un `pos-backend` en ejecución, sin necesitar leer
  `promotions/service.py` para entender el comportamiento esperado.
- **SC-002**: `test_promotions_rules.py` — el único de los 12 scripts de test que corre en CI
  (`.github/workflows/deploy.yml:14-22`) — cubre explícitamente la vigencia en hora local (no UTC,
  `FR-001`) y la ventana con cruce de medianoche (`FR-005`); es la base directa y verificable de
  User Stories 1 y 2, y el candidato más maduro de todo el reconocimiento a convertirse en golden
  master formal de esta spec.
- **SC-003**: La regla protegida A-07 (hora local del tenant, desglose por línea, prioridad
  explícita en el desempate) queda fijada como invariante de test obligatorio en cualquier
  reimplementación — ningún cambio futuro puede revertir a evaluación en UTC ni a un criterio de
  desempate distinto de `(priority, amount, -created_at)` sin que `test_promotions_rules.py` lo
  detecte.
- **SC-004**: El motor produce siempre exactamente un descuento por línea (nunca acumula dos
  promociones automáticas en la misma línea) — verificable inspeccionando que `evaluate_detailed`
  nunca devuelve más de una entrada de `LineDiscount` por línea de carrito.
- **SC-005**: El redondeo del descuento total ocurre en un único punto del cálculo, verificable
  comparando la suma sin redondear del desglose por línea contra el total final ya redondeado
  (`FR-024`).
- **SC-006**: Las anomalías con clasificación cerrada (A-08 `ACCIDENTAL`, A-10 `ACCIDENTAL`, A-46
  `ACCIDENTAL` cerrada sin urgencia en P26, A-09 `PENDIENTE` mitigada operativamente en P6) quedan
  documentadas con su tratamiento acordado, de forma que la modernización no las reintroduzca por
  accidente ni las trate como bloqueantes cuando no lo son.
- **SC-007**: Las anomalías sin decisión de negocio (A-36 porción promociones, A-37 completo)
  quedan documentadas con su comportamiento observado y su evidencia de código, sin fijarse como
  contrato deseado, para que la próxima ronda de entrevista de negocio las resuelva con contexto
  completo sin necesidad de releer el código fuente.

## Out of Scope

- **La administración de una promoción** (creación, edición, validación de forma, máquina de
  estados `draft`/`active`/`paused`/`finished`, duplicado, `PATCH /shape`) — RN-PROMO-46 a
  RN-PROMO-78, cubierto por la spec 013, aún no escrita en este reconocimiento.
- **El consumo de este motor en el menú público y el carrito del comensal por QR** — cubierto por
  la spec 007; esta spec solo documenta que la convención de hora local (`FR-001`) no llegó a esos
  dos puntos (A-08, User Story 7), sin especificar su corrección.
- **El consumo de este motor en cada camino de cobro real** (mostrador, cierre unificado, cierre
  dividido de mesa, `pay_order` legado) — cubierto por las specs 008, 010 y 011; esta spec
  documenta únicamente cómo el motor calcula el descuento en sí, no cuándo ni cómo cada camino lo
  invoca ni registra su resultado.
- **La expiración automática de promociones vencidas** (`expire_promotions`, job periódico de
  medianoche) — comparte patrón de lock distribuido con el barrido de sesiones de mesa (spec 010),
  pero es un mecanismo de mantenimiento de estado, no de cálculo de descuento; fuera del alcance
  declarado de esta spec.
- **El panel de administración de promociones** (`promotions-page.component.ts`, etiquetas
  `live`/`out_of_window`, uso de `findOverlaps` para mostrar advertencias al administrador) — el
  mecanismo de solapamiento en sí se documenta aquí (User Story 6) porque vive en
  `promotions/service.py`, pero su presentación en el panel administrativo no es objeto de esta
  spec.

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia del resto de
  las guías de este template ("evitar detalles de implementación"), aquí los nombres de función,
  constantes internas (`TENANT_TIMEZONE`, `AUTO_TYPES`), tipos de promoción y valores literales
  **son** el contrato observable que se está documentando — se citan explícitamente porque los
  criterios de aceptación deben ser verificables directamente contra `pos-backend` en ejecución o
  contra `test_promotions_rules.py`.
- **A-07 (`RN-PROMO-01`, `RN-PROMO-13`, `RN-PROMO-14`) se especifica tal cual, sin tocar**: es la
  regla `[PROTEGIDA]` de esta spec, con dos testigos (CÓDIGO + `memoria-historica.md` #15) y
  respaldo de test en CI. La pregunta de gobernanza sobre el autor no identificable del commit
  `2e94a3ad`, y la pregunta de negocio sobre si el bug de zona horaria afectó una promoción real
  antes del 2026-08-07, quedan registradas como abiertas pero no bloquean esta spec.
- **A-08 se referencia, no se respecifica**: esta spec confirma que el motor de cálculo en sí
  (alcance de esta spec) aplica correctamente la hora local; la corrección de `cart/service.py` y
  `menu/router.py` pertenece a la spec 007, para no duplicar el mecanismo de conversión ya
  documentado aquí.
- **A-09 se documenta pero NO se especifica como contrato**: clasificada `PENDIENTE` en
  `registro-de-anomalias.md`, mitigada operativamente por P6 (relojes de terminal verificados).
  Esta spec no asume que la mitigación operativa sea permanente ni la convierte en parte del
  contrato de software — solo dejar constancia de que, hoy, el riesgo de código no se ha
  materializado.
- **A-10 se documenta pero NO se especifica como contrato**: `ACCIDENTAL`, sin efecto visible hoy.
  Se corrige en fase de modernización solo si se agrega una pantalla que exponga el nombre de la
  promoción ganadora en un empate — el evento que la volvería visible.
- **A-37 (porción evaluación: `RN-PROMO-11`, `RN-PROMO-15`, `RN-PROMO-19`, `RN-PROMO-26`,
  `RN-PROMO-27`) y A-36 (porción promociones: `RN-PROMO-03`, `RN-PROMO-33`, `RN-PROMO-51`) se
  documentan pero NO se especifican como contrato**: siguiendo instrucción explícita de alcance,
  ambas anomalías quedan con clasificación `PENDIENTE` en `registro-de-anomalias.md` — se describe
  el comportamiento observado hoy (porque esta spec documenta "lo que el sistema YA hace"), pero
  no se fija como comportamiento correcto ni obligatorio para la modernización.
- **A-46 se documenta como limitación de diseño reconocida, no se corrige en esta spec**: cerrada
  sin urgencia en P26 (sin planes de expansión a otra zona horaria); esta spec fija el
  comportamiento actual (una sola zona horaria por instancia) como el contrato vigente, sin
  implicar que deba seguir siendo así indefinidamente.
