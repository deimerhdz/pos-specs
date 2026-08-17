# Feature Specification: Cocina, consolidación de carritos y mesas físicas

**Feature Branch**: `009-cocina-consolidacion-y-mesas-fisicas`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/orders/kitchen.py`, `consolidation.py`, `tables_advanced.py` (y, para
las dos reglas de la comanda de staff, `service.py`), para que sirva de contrato formal de cara
a la modernización (Principio I y Principio III de la
[Constitución](../../.specify/memory/constitution.md)). Es el dominio de "cómo se prepara y se
sirve un pedido una vez que ya existe": el ciclo de vida de `estado_cocina` en la terminal
compartida, los dos caminos por los que el mesero mete líneas en la cuenta de una mesa
(consolidar el carrito del comensal, o agregar un ítem suelto), y las operaciones sobre mesas
físicas (estado, mover una orden, fusionar mesas). Incluye la anomalía con la evidencia más
fuerte de todo el reconocimiento (A-04: regresión de fusión reconstruida con `git log`, con
testimonio de negocio de merma real) y tres anomalías accidentales de menor impacto (A-16, A-26)
que la segunda ronda de entrevista de negocio cerró como riesgo latente sin incidente activo, más
una decisión operativa deliberada (A-48, deprecación del KDS) que se especifica tal cual.

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE
de la preparación en cocina, la toma de pedidos por el mesero desde la terminal, la consolidación
de carritos, y la administración de mesas físicas del sistema POS Heladería, tomado de
`reglas-de-negocio.md` (RN-ORD-36 a RN-ORD-64, RN-ORD-66) y de `registro-de-anomalias.md` (A-04,
A-16, A-26, A-48), para que sirva de contrato en la modernización. Prioriza A-04 —el único
hallazgo del reconocimiento con prueba directa de `git log` de cómo y cuándo se rompió, reforzado
por testimonio de negocio de merma real hace ~15 días— como el hallazgo de mayor criticidad de
esta spec."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `kitchen.py`, `consolidation.py`,
  `tables_advanced.py` y `service.py`, no uno deseado. Las anomalías conocidas se marcan inline
  con su tratamiento acordado (registro-de-anomalias.md), incluyendo el cierre de sus porciones
  PENDIENTE en la segunda ronda de entrevista de negocio (P4, P16-bis, P20-bis, P28).
-->

### User Story 1 - A-04: el único camino con botón real en la terminal del mesero no valida los sabores/opciones obligatorios que promete cobrar (Priority: P1)

Un mesero agrega, desde la terminal, un ítem directo a la cuenta de una mesa (sin pasar por el
carrito del comensal). Si esa variante exige elegir un número mínimo y máximo de opciones de un
grupo que descuenta inventario (p. ej. "elige 3 sabores" en una copa grande), el sistema **no**
comprueba que la selección enviada cumpla esa regla — acepta cualquier número de opciones válidas
individualmente, cobra el precio completo de la variante, y descuenta inventario solo de las
opciones efectivamente enviadas. El mismo chequeo, invocado con un argumento adicional, sí se
ejecuta en la función hermana que crea una comanda de staff — la validación existe en el sistema,
pero el camino que el mesero usa de verdad no la activa.

**Why this priority**: es el hallazgo de mayor prioridad de todo el reconocimiento, no solo de
esta spec — el único con prueba directa de `git log`/`git show` de cómo y cuándo se rompió (no
una suposición), y el único con testimonio de negocio (P4) que confirma merma real observada en
conteo físico, coincidiendo exactamente con el patrón que la reconstrucción de código predijo.
Cualquier reimplementación de este módulo que no restaure el argumento faltante reproduce el bug
en la modernización tal como sigue activo hoy.

**Independent Test**: se puede probar comparando dos invocaciones sobre la misma variante con un
grupo `min_select=2, max_select=3` que descuenta inventario: `add_item_to_table` con una sola
opción enviada (se acepta, sin rechazo) contra `create_order` con la misma selección incompleta
(se rechaza con `422`) — la divergencia es observable sin necesitar leer el código fuente.

**Acceptance Scenarios**:

1. **Given** una variante "Copa Grande — 3 bolas" con el grupo "Sabores" (`min_select=2`,
   `max_select=3`, `quantity_per_option=120g`, un grupo que descuenta inventario y es
   obligatorio), **When** un mesero la agrega desde `POST /orders/tables/{table_id}/items` con
   un solo sabor elegido, **Then** el sistema la acepta sin error: crea el `OrderItem` al precio
   completo de "Copa Grande" y descuenta únicamente 120g de un insumo — no los 360g que
   corresponden a una copa de 3 bolas servida de verdad (`RN-CAT-33`, camino
   `add_item_to_table`).
2. **Given** la misma variante y la misma selección incompleta, **When** se envía en cambio por
   `create_order` (comanda de staff, `POST /orders`), **Then** el sistema la rechaza con `422`:
   "exige exactamente 3 opción(es), se enviaron 1" — la única diferencia entre ambos caminos es
   que este sí pasa `variant=variant` a `load_valid_options` (`app/api/v1/orders/service.py:102`)
   y el otro no.
3. **Given** el historial de commits de `consolidation.py`, **When** se reconstruye con `git
   log`/`git show`, **Then** se confirma una regresión de fusión, no un descuido original: el
   commit `03469ca` (2026-08-03) añadió el parámetro `variant` a `load_valid_options` y
   **corrigió** `add_item_to_table` para pasarlo; el commit `ee94f30` (2026-08-04, un día
   después, autor distinto — LeonardoGomezz, funcionalidad de combos) partió de una copia del
   fichero anterior a esa corrección, y al fusionarse reintrodujo la línea sin `variant` en el
   mismo lugar exacto donde la corrección la había puesto, sin que git marcara conflicto.
4. **Given** la entrevista de negocio con el jefe de cocina (P4), **When** se le pregunta si se
   ha notado en un conteo físico una merma de algún sabor/insumo sin explicación, **Then**
   confirma que sí, hace aproximadamente 15 días (≈2026-08-01), específicamente en
   sabores/toppings elegibles — no en insumos de receta fija — coincidiendo exactamente con el
   patrón que predice esta anomalía.
5. **Given** el estado actual del código, **When** se compara `consolidate_table` (el tercer
   camino de alta de línea del mesero, que consolida el carrito del comensal), **Then** ese
   camino no está afectado: no llama a `load_valid_options` en absoluto, porque copia
   directamente las opciones de los `CartItem`, que ya pasaron su propia validación cuando el
   comensal las eligió desde el QR — la anomalía es exclusiva de `add_item_to_table`.

**Nota — corrección y su límite (A-04, tratamiento acordado)**: la corrección en fase de
modernización es pasar `variant=variant` en `consolidation.py:199` — una línea, ya aplicada una
vez en `03469ca` y perdida en el merge posterior. **No es retroactiva**: no existe forma de
recalcular el inventario que ya se consumió incorrectamente desde que se perdió la corrección;
solo detiene el sangrado hacia adelante. Antes de corregir, conviene auditar el catálogo con
`min_select>0` y `quantity_per_option>0` para dimensionar el desfase acumulado. Queda pendiente
de negocio quién asume la responsabilidad de esa auditoría — no bloquea la corrección de la
línea en sí.

---

### User Story 2 - La preparación de un ítem avanza siempre hacia adelante, y anularlo solo revierte inventario si cocina aún no lo había tocado (Priority: P1)

Un ítem de pedido nace `pendiente`. Cocina lo avanza manualmente a `en_preparacion` y luego a
`listo`, o directamente de `pendiente` a `listo` de un solo toque — ambos caminos son legales a
propósito. Un ítem `listo` o `anulado` no tiene ninguna transición legal hacia adelante por esta
vía. Anular un ítem revierte su inventario al stock solo si todavía estaba `pendiente`; si cocina
ya lo había empezado a preparar, el insumo se considera físicamente comprometido y no vuelve. Un
ítem anulado puede reemplazarse por uno nuevo, que nace `pendiente` con su propio descuento de
inventario y su propio precio recalculado — nunca copiado del original.

**Why this priority**: es el ciclo de vida operativo diario que cocina ejecuta decenas de veces
por turno; cualquier regresión en las transiciones permitidas o en qué reversa aplica al anular
rompe la operación de cocina de forma inmediata y visible, o descuadra el kardex en silencio si
la reversa se ejecuta de más o de menos.

**Independent Test**: se puede probar invocando `transition_kitchen`/`void_item` directamente
sobre un `OrderItem` de prueba en cada uno de sus cuatro estados posibles, sin depender de
confirmación de pedido ni de cobro.

**Acceptance Scenarios**:

1. **Given** un ítem `pendiente`, **When** se transiciona a `en_preparacion` o directamente a
   `listo`, **Then** ambas transiciones se aceptan — el salto directo es el botón de un toque
   que usa la terminal, porque "quien toma el pedido es quien lo prepara" y obligar a pasar por
   `en_preparacion` sería un clic de más (`RN-ORD-36`).
2. **Given** un ítem `en_preparacion`, **When** se intenta transicionarlo de vuelta a
   `pendiente`, o un ítem `listo` a cualquier otro estado por esta vía, **Then** el sistema
   rechaza con `409` "Transición de preparación inválida", indicando el estado de origen y el
   destino solicitado — no existen transiciones hacia atrás ni desde `listo`/`anulado`
   (`RN-ORD-36`).
3. **Given** un ítem `pendiente` que se anula, **When** se completa la anulación, **Then** su
   insumo vuelve al stock — cocina no lo había tocado físicamente (`RN-ORD-40`).
4. **Given** un ítem `en_preparacion` o `listo` que se anula, **When** se completa la anulación,
   **Then** su insumo **no** vuelve al stock — el descuento original queda firme porque ya se
   consumió físicamente (`RN-ORD-40`).
5. **Given** un ítem ya `anulado`, **When** se intenta anularlo de nuevo, **Then** el sistema
   rechaza con `409` "El ítem ya está anulado" — evita doble reversa de inventario y doble
   registro de auditoría (`RN-ORD-41`).
6. **Given** una anulación con reemplazo cuyo `replacement` especifica un `combo_id`, **When** se
   procesa, **Then** el sistema rechaza con `422` — un ítem anulado no puede reemplazarse por un
   combo; hay que anular todos sus componentes y agregarlo de nuevo (`RN-ORD-42`).
7. **Given** una anulación con reemplazo cuya variante de reemplazo está `active=false`, **When**
   se procesa, **Then** el sistema rechaza con `422` "Variante inactiva" (`RN-ORD-43`).
8. **Given** una anulación con reemplazo válido, **When** se completa, **Then** el nuevo ítem
   nace siempre `pendiente` (sin importar el estado del original), enlazado al original vía
   `void_de`, con su propio descuento de inventario ejecutado en la misma transacción —
   si ese descuento falla por falta de stock, toda la anulación (incluida la reversa del
   original, si aplicaba) se revierte (`RN-ORD-44`).
9. **Given** el precio del ítem original (`unit_price=8000`) y una variante de reemplazo con
   otro precio de catálogo, **When** se crea el ítem de reemplazo, **Then** su `unit_price` se
   recalcula desde cero (`compute_line_price` sobre la nueva variante y sus opciones) — nunca se
   copia del original (`RN-ORD-45`).

---

### User Story 3 - Consolidar los carritos de una mesa los agrupa en un solo lote en la orden del mesero, con el precio congelado al momento de agregarlo (Priority: P1)

El mesero consolida en un solo paso todos los carritos abiertos y con ítems de los comensales
activos de una mesa: cada línea de cada carrito se convierte en una línea de la orden del
mesero, trazada por comensal, con el precio que tenía en el carrito — no el precio de catálogo
vigente en el momento de consolidar. Todas las líneas se insertan primero y el inventario se
descuenta en un único lote al final, para que los bloqueos de fila se tomen en orden canónico y
dos consolidaciones concurrentes no puedan bloquearse mutuamente. Los carritos consolidados
quedan marcados y no vuelven a ofrecerse para una segunda consolidación.

**Why this priority**: es, junto con agregar un ítem directo (User Story 1), uno de los dos
caminos reales por los que el mesero cierra la vuelta de pedidos de una mesa; cualquier
regresión aquí bloquea el flujo de cobro completo de la mesa, no solo un ítem.

**Independent Test**: se puede probar invocando `consolidate_table(db, table_id, user)` sobre
una mesa con comensales que tienen carritos `abierto` con ítems, y verificando el estado
resultante de la orden, de los carritos y del stock, sin depender de cocina ni de cobro.

**Acceptance Scenarios**:

1. **Given** una mesa cuyos comensales activos no tienen ningún carrito `abierto` con ítems,
   **When** se intenta consolidar, **Then** el sistema rechaza con `409` "No hay carritos con
   ítems para consolidar" (`RN-ORD-46`).
2. **Given** un `CartItem` con `unit_price=5000` (el precio vigente cuando el comensal lo
   agregó), **When** se consolida y el precio de catálogo actual ya subió a 5500, **Then** el
   `OrderItem` resultante conserva `unit_price=5000` — snapshot copiado del carrito, no
   recalculado (`RN-ORD-47`).
3. **Given** varios carritos con líneas que tocan varios insumos distintos, **When** se
   consolidan, **Then** todas las líneas de todos los carritos se insertan primero, y el
   descuento de inventario se ejecuta en una sola llamada al final sobre el lote completo — evita
   que dos consolidaciones concurrentes en mesas distintas que comparten insumos se
   bloqueen entre sí (`RN-ORD-48`).
4. **Given** un carrito recién consolidado, **When** se inspecciona su estado, **Then** queda
   `confirmado` — no vuelve a aparecer como candidato en una consolidación posterior de la misma
   mesa (`RN-ORD-49`).
5. **Given** una mesa `libre` sin sesión activa, **When** el mesero agrega el primer ítem (por
   consolidación o directo), **Then** el sistema crea la sesión de mesa de forma perezosa en ese
   momento y la mesa pasa a `ocupada` — la sesión no nace solo cuando un comensal escanea el QR
   (`RN-ORD-51`).
6. **Given** una mesa que ya tiene una orden `abierta` de canal `waiter` (creada por un mesero
   anteriormente), **When** se consolida o se agrega un ítem directo de nuevo, **Then** las
   líneas nuevas se acumulan en esa misma orden existente en vez de crear una orden `waiter`
   distinta — la búsqueda de la orden abierta del mesero se ordena por `created_at` para ser
   determinista si excepcionalmente hubiera más de una (`RN-ORD-51`).

---

### User Story 4 - El mesero también agrega ítems sueltos directo a la orden de la mesa, sin pasar por el carrito de ningún comensal (Priority: P2)

Además de consolidar el carrito del comensal, la terminal ofrece un segundo camino: agregar
directamente una línea a la orden de la mesa, sin que ningún comensal la haya armado en su
propio carrito. Un combo agregado por esta vía se expande en sus componentes reales al precio
normal de catálogo — el ahorro del combo no se calcula aquí, se resuelve al cobrar. Las líneas
agregadas así no quedan atribuidas a ningún comensal específico.

**Why this priority**: es el mismo camino donde vive A-04 (User Story 1), pero esta historia
cubre su comportamiento normal y correcto — el diseño de expansión de combo y de
comensal-sin-asignar no está en cuestión, a diferencia de la validación de opciones. Prioridad
P2 porque, a diferencia de User Story 1 y 3, aquí no hay ninguna anomalía activa que corregir.

**Independent Test**: se puede probar invocando `add_item_to_table` con un `combo_id` y por
separado con una variante+opciones válida, verificando la expansión de precio y el
`participant_id` resultante.

**Acceptance Scenarios**:

1. **Given** un combo con dos componentes de $6000 cada uno, **When** el mesero lo agrega
   directo a la orden de la mesa, **Then** el sistema crea dos `OrderItem` de $6000 cada uno
   (total $12000) — el mismo patrón que usa el carrito del comensal (`RN-CART-11`); el
   descuento propio del combo solo se calcula al cobrar (`RN-ORD-52`).
2. **Given** cualquier ítem agregado por esta vía (con o sin combo), **When** se inspecciona su
   `participant_id`, **Then** siempre es `None` — a diferencia de `consolidate_table`, que sí
   asigna el comensal dueño del carrito (`RN-ORD-53`).

---

### User Story 5 - A-16: cocina puede avanzar y anular ítems sin comprobar el estado del pedido padre, mitigado por el ritmo operativo real (Priority: P2)

Avanzar el `estado_cocina` de un ítem, o anularlo, funciona exactamente igual sin importar el
`status` de la `CustomerOrder` a la que pertenece — incluida una orden ya `pagada`, `cancelada`,
o `bloqueada` (congelada para cobro). Esto contrasta con `mark_order_ready` (marcar todos los
ítems en curso como listos de una vez), que sí rechaza explícitamente si la orden ya es
terminal (`pagada`/`cancelada`), aunque tampoco bloquea sobre una orden `bloqueada`.

**Why this priority**: la inconsistencia de código (mark_order_ready sí valida, transition_kitchen
y void_item no) es un defecto verificable por sí solo — clasificado ACCIDENTAL. La pregunta de
si eso debería impedir avanzar cocina sobre una orden `bloqueada` se repreguntó directamente al
negocio en la segunda ronda de entrevista (P16-bis) y se cerró sin encontrar riesgo operativo
activo, así que esta historia queda en P2: recomendable de corregir en modernización, sin
urgencia.

**Independent Test**: se puede probar invocando `transition_kitchen` o `void_item` sobre un
ítem cuya orden padre ya está `pagada`, `cancelada` o `bloqueada`, y observando que ambas
operaciones se completan sin ningún rechazo relacionado con el estado de la orden.

**Acceptance Scenarios**:

1. **Given** un ítem `pendiente` de una orden ya `pagada`, `cancelada` o `bloqueada`, **When**
   se transiciona su `estado_cocina`, **Then** el sistema lo procesa igual que si la orden
   estuviera `abierta` — no existe ninguna comprobación del `status` de la orden en
   `transition_kitchen` (`RN-ORD-38`, ACCIDENTAL).
2. **Given** el mismo escenario, **When** se anula el ítem (`void_item`) en vez de
   transicionarlo, **Then** ocurre lo mismo: se procesa sin comprobar el `status` de la orden,
   mientras el propio ítem no esté ya `anulado` (`RN-ORD-39`, ACCIDENTAL).
3. **Given** una orden `bloqueada` (congelada para cobro) con ítems en curso, **When** se invoca
   `mark_order_ready`, **Then** sí se permite terminar de marcarlos "listo" — solo rechaza si la
   orden ya es terminal (`pagada`/`cancelada`), no si está `bloqueada` (`RN-ORD-37`).

**Nota — cierre de la porción `PENDIENTE` (A-16, ronda 2, P16-bis)**: la pregunta de si
`bloqueada` debería impedir el avance de cocina se repreguntó directamente al dueño/gerente
(la pregunta original iba dirigida al jefe de cocina, que no sabría decir), planteando el
escenario concreto de un cajero cobrando la mesa 4 mientras cocina marca "listo" un ítem de esa
misma mesa sin aviso del sistema. Respuesta: **"Es prácticamente imposible"** que esos dos
momentos coincidan, dado el ritmo de trabajo actual del local. Esto cierra `RN-ORD-37` como
riesgo mitigado por la operación real, no activo — mismo patrón que otros hallazgos de código sin
corregir pero sin incidente porque la operación diaria no lo dispara. La porción ya `ACCIDENTAL`
(`RN-ORD-38`/`RN-ORD-39`, la inconsistencia frente a `mark_order_ready`) no cambia con esta
respuesta: sigue siendo recomendable corregir en fase de modernización, replicando la misma
validación de estado que ya tiene `mark_order_ready`, pero sin urgencia operativa.

---

### User Story 6 - Una mesa admite varias órdenes activas a la vez; la comanda de staff descuenta inventario al crearse, no al cobrar (Priority: P2)

El modelo de datos no impone que una mesa tenga una única orden abierta: puede haber varios
pedidos activos simultáneos (uno por comensal, por ronda, o por canal). La agrupación para
cobrar se resuelve por `table_session_id`, no por una relación 1:1 mesa-orden. Cuando el staff
crea una comanda directamente (mostrador o mesero, sin pasar por el carrito de un comensal),
nace ya `abierta` — no `recibida` como el pedido QR — y descuenta inventario en la misma
transacción de creación; si falta stock en cualquier línea, toda la comanda se revierte, no se
llega a crear.

**Why this priority**: son reglas de modelo de datos y de punto único de descuento que, si se
malinterpretan durante la modernización (p. ej. asumiendo "una orden por mesa" o "el descuento
siempre ocurre al confirmar"), producirían un sistema que rechaza operaciones legítimas del
negocio o sobrestima el inventario en silencio para este camino específico.

**Independent Test**: se puede probar creando dos órdenes activas sobre la misma mesa por canales
distintos y verificando que ambas coexisten sin error; por separado, invocando `create_order`
con una línea que fuerza falta de stock y verificando que ninguna línea ni la orden quedan
creadas.

**Acceptance Scenarios**:

1. **Given** una mesa con una orden `waiter` `abierta` y un pedido `recibida` distinto del canal
   QR, **When** se inspecciona la mesa, **Then** ambas órdenes coexisten activas sin conflicto —
   no existe ningún índice único de "una orden abierta por mesa" (`RN-ORD-50`).
2. **Given** una comanda de staff para un comensal cuyo `status` ya no es `open` (se retiró de la
   mesa), **When** se intenta crear, **Then** el sistema rechaza con `409` "El comensal ya no
   está en la mesa" (`RN-ORD-54`).
3. **Given** una comanda de staff con dos líneas, donde la primera descuenta bien y la segunda
   falla por stock insuficiente, **When** se crea, **Then** toda la transacción se revierte:
   ni la orden ni ninguna de sus líneas quedan creadas — a diferencia del pedido QR, que nace
   `recibida` y descuenta solo al confirmar, la comanda de staff nace ya `abierta` y descuenta en
   el mismo paso de creación, porque si no descontara aquí no descontaría nunca (`RN-ORD-55`).

---

### User Story 7 - A-48: el KDS (pantalla de cocina separada) se retiró a propósito, no por descarte del concepto (Priority: P3)

El sistema ya no ofrece una pantalla de cocina independiente (KDS): el ciclo de vida completo de
un ítem (`pendiente→en_preparacion→listo`, anular) se opera desde la misma terminal que atiende
las mesas. Una migración retiró el estado intermedio `entregado`, que existía para una pantalla
separada y dejó de aportar una decisión distinta de `listo` una vez fusionados los dos
dispositivos en uno.

**Why this priority**: no hay ningún comportamiento activo que corregir — es documentación de
una decisión de producto ya tomada y confirmada, incluida aquí para que la modernización no
reintroduzca un concepto de KDS separado creyendo que se trata de una funcionalidad incompleta.

**Independent Test**: se puede verificar leyendo el docstring de `kitchen.py` ("el ciclo de vida
del ítem... lo mueve la terminal de mesas, antes había un KDS aparte, ya deprecado") y la
migración `c5d6e7f8a9b0_simplify_kitchen_status`, que retira `entregado` del `CHECK` de
`estado_cocina`.

**Acceptance Scenarios**:

1. **Given** el CHANGELOG de v1.0.0 (2026-07-17), **When** se lee, **Then** todavía describía
   "notificaciones push al KDS" como trabajo futuro vigente.
2. **Given** la migración `c5d6e7f8a9b0` (2026-08-07, tres semanas después del CHANGELOG),
   **When** se aplica, **Then** elimina `entregado` de los valores permitidos de
   `estado_cocina` y consolida cualquier fila existente en ese estado — el ciclo de vida queda
   reducido a `pendiente/en_preparacion/listo/anulado`, los cuatro estados que rige esta spec.
3. **Given** la repregunta directa al dueño/jefe de cocina en la segunda ronda de entrevista
   (P28) — "¿fue por decidir que un solo dispositivo funciona mejor, o porque ya no se
   necesitaba el concepto?" —, **When** responde, **Then** confirma la opción (a): decidieron
   que un solo dispositivo/terminal compartido funciona mejor — decisión operativa deliberada,
   no un descarte de concepto ni un plan abandonado sin más.

**Especificación de esta spec**: la ausencia de una pantalla de cocina separada, y el estado
`entregado` ya retirado del modelo, se fijan como comportamiento intencional a preservar en la
modernización — no como una funcionalidad pendiente de reintroducir.

---

### User Story 8 - Las operaciones sobre mesas físicas tienen reglas más estrictas de lo necesario en dos puntos, ambos sin uso real hoy (Priority: P3)

El staff puede cambiar el estado operativo de una mesa (`libre`, `ocupada`, `reservada`), mover
una orden de una mesa a otra, y fusionar varias órdenes de mesas distintas en un solo grupo de
cobro. Ninguna mesa puede crearse con un número ya usado por otra. Dos de estas operaciones
tienen comportamiento más estricto o menos predecible de lo que el resto del modelo permitiría,
pero la segunda ronda de entrevista de negocio confirmó que ninguna de las dos se usa en la
práctica hoy.

**Why this priority**: son hallazgos de bajo impacto operativo individual — casos límite de
funciones que el equipo confirmó no usar — de ahí la prioridad más baja de esta spec.

**Independent Test**: se puede probar invocando `move_order` hacia una mesa con una orden activa
preexistente y verificando el rechazo; por separado, invocando `merge_orders` sobre órdenes que
ya pertenecían a grupos distintos y observando que solo uno de los grupos sobrevive.

**Acceptance Scenarios**:

1. **Given** una mesa con una orden `bloqueada` (no terminal), **When** se intenta cambiar su
   estado a `libre` o `reservada`, **Then** el sistema rechaza con `409` "La mesa tiene órdenes
   sin cerrar" (`RN-ORD-56`).
2. **Given** una orden y su propia mesa actual como destino, **When** se invoca `move_order`,
   **Then** la operación es un no-op: devuelve la orden sin cambios (`RN-ORD-57`).
3. **Given** una mesa 7 con una orden `recibida` activa de otro comensal, **When** se intenta
   mover una orden distinta hacia esa mesa, **Then** el sistema rechaza con `409` "La mesa
   destino ya tiene una orden activa" — más estricto que el resto del modelo, que sí permite
   varias órdenes activas por mesa (`RN-ORD-50` vs. `RN-ORD-58`, `[DUDOSA]`).
4. **Given** una orden movida exitosamente a una mesa nueva, **When** se completa el movimiento,
   **Then** la mesa origen se libera (`status="libre"`) únicamente si, tras el movimiento, ya no
   le queda ninguna orden activa — puede quedar `ocupada` si tenía otras órdenes (`RN-ORD-59`).
5. **Given** el manejador de `IntegrityError` dentro de `move_order`, **When** se inspecciona el
   modelo de datos actual, **Then** no existe ya ningún índice único que pudiera disparar esa
   excepción — es código huérfano de una constraint que existió y fue removida (`RN-ORD-60`,
   ACCIDENTAL).
6. **Given** tres UUIDs de orden para fusionar, de los cuales solo dos existen, **When** se
   invoca `merge_orders`, **Then** el sistema rechaza la fusión completa con `404` "Alguna orden
   no existe" — ninguna orden se modifica (`RN-ORD-62`).
7. **Given** una orden A que ya pertenece al grupo G1 (junto con otra orden Z no listada en esta
   fusión) y una orden B que ya pertenece a un grupo distinto G2 (junto con otra orden Y no
   listada), **When** se invoca `merge_orders([A, B])`, **Then** el sistema conserva
   únicamente uno de los dos grupos preexistentes — el primero que devuelva un `SELECT` sin
   `ORDER BY`, por lo tanto no determinista — y las órdenes no listadas del grupo "perdedor"
   (Y en este ejemplo) quedan huérfanas, separadas de B aunque antes compartían grupo
   (`RN-ORD-63`, ACCIDENTAL).
8. **Given** dos mesas con el mismo número solicitado, **When** se intenta crear la segunda,
   **Then** el sistema rechaza con `409` "Ya existe una mesa con ese número" (`RN-ORD-66`).

**Nota — cierre de la porción `PENDIENTE` (A-26/RN-ORD-58, ronda 2, P20-bis)**: se repreguntó
directamente al dueño/gerente (la pregunta original iba a mesero/supervisor, que no sabría
decir o no usa la función) si el equipo alguna vez mueve un pedido de una mesa a otra.
Respuesta: **"No la usamos."** Esto cierra `RN-ORD-58` como riesgo latente, no activo — el mismo
patrón que otros hallazgos donde el código restringe algo que nadie ejerce en la práctica. El
resto de A-26 (`RN-ORD-60`, manejador huérfano; `RN-ORD-63`, no determinismo de `merge_orders`),
ya `ACCIDENTAL` confirmado por código sin necesitar testimonio de negocio, no cambia con esta
respuesta — se recomienda corregir ambos en fase de modernización: retirar o documentar el
manejador huérfano, y definir una regla explícita y determinista de qué grupo sobrevive en
`merge_orders` (p. ej., el de menor `created_at`).

---

### Edge Cases

- **Un ítem con selección de opciones incompleta agregado por el mesero, luego cocina lo
  prepara y se cobra**: el sistema nunca vuelve a validar la selección después del alta — la
  única puerta de control era `add_item_to_table` en el momento de crear el ítem, y ahí es
  exactamente donde A-04 la omite (User Story 1).
- **Anular un ítem con reemplazo cuya nueva variante también tiene un grupo de opciones
  obligatorio**: `void_item` sí valida ese reemplazo con `variant` (`repl_variant`) — la omisión
  de A-04 es exclusiva de `add_item_to_table`, no de todo el módulo de cocina.
- **Consolidar una mesa cuyos carritos ya se consolidaron antes**: los carritos `confirmado` no
  vuelven a aparecer como candidatos; una segunda consolidación sin carritos nuevos rechaza con
  el mismo `409` de `RN-ORD-46`.
- **Fusionar tres órdenes donde ninguna pertenecía antes a ningún grupo**: `merge_orders` crea
  un `merged_group_id` nuevo — el comportamiento no determinista de `RN-ORD-63` solo se
  manifiesta cuando hay grupos preexistentes en colisión.
- **Avanzar cocina y anular casi simultáneamente sobre el mismo ítem**: ninguna de las dos
  operaciones toma un bloqueo de fila (`with_for_update`) — un posible pisado de escrituras
  concurrentes sobre el mismo `OrderItem` es un riesgo de código conocido (registro de riesgos
  R1), no cubierto por las reglas RN-ORD de esta spec, que documentan el comportamiento
  secuencial observado.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Regla crítica, User Story 2]**: Un ítem de pedido DEBE transicionar su
  `estado_cocina` únicamente hacia adelante: `pendiente→{en_preparacion, listo}`,
  `en_preparacion→listo`. El salto directo `pendiente→listo` DEBE permitirse — es intencional,
  no un hueco de validación (`RN-ORD-36`).
- **FR-002**: Ninguna transición fuera de las listadas en FR-001 (incluidas las transiciones
  desde `listo` o `anulado`, o hacia atrás) DEBE aceptarse; el sistema DEBE rechazarlas con `409`
  indicando el estado de origen y el destino solicitado (`RN-ORD-36`).
- **FR-003 [`[DUDOSA]`, anomalía A-16, `PENDIENTE` cerrado en ronda 2 — mitigado por operación
  real, sin urgencia]**: `mark_order_ready` bloquea solo si la orden ya es terminal
  (`pagada`/`cancelada`); no bloquea si la orden está `bloqueada` (congelada para cobro).
  Confirmado en entrevista de negocio (P16-bis): "prácticamente imposible" que cocina y cobro
  coincidan en la práctica operativa actual (`RN-ORD-37`).
- **FR-004 [Anomalía A-16, ACCIDENTAL]**: `transition_kitchen` NO DEBE (hoy no lo hace)
  comprobar el `status` de la `CustomerOrder` padre — funciona igual con la orden `pagada`,
  `cancelada` o `bloqueada`. Inconsistente con el cuidado explícito de `mark_order_ready`
  (FR-003) en el mismo módulo; corregir en modernización replicando esa misma validación, sin
  urgencia operativa (`RN-ORD-38`).
- **FR-005 [Anomalía A-16, ACCIDENTAL]**: `void_item` NO DEBE (hoy no lo hace) comprobar el
  `status` de la orden padre — mismo patrón de omisión que FR-004; solo rechaza si el ítem ya
  está `anulado` (`RN-ORD-39`).
- **FR-006 [Regla crítica]**: Anular un ítem `pendiente` DEBE devolver su insumo al stock —
  cocina aún no lo había consumido físicamente (`RN-ORD-40`).
- **FR-007 [Regla crítica]**: Anular un ítem `en_preparacion` o `listo` NO DEBE devolver
  inventario al stock — el descuento original ya representa consumo físico (`RN-ORD-40`).
- **FR-008**: Un ítem ya `anulado` NO DEBE poder anularse de nuevo; el sistema DEBE rechazar con
  `409` (`RN-ORD-41`).
- **FR-009**: El reemplazo de un ítem anulado NO DEBE aceptar un `combo_id`; DEBE rechazarse con
  `422` (`RN-ORD-42`).
- **FR-010**: La variante de reemplazo DEBE estar activa; una variante inactiva DEBE rechazarse
  con `422` (`RN-ORD-43`).
- **FR-011**: El ítem de reemplazo DEBE nacer siempre `pendiente` (sin importar el estado del
  original), enlazado al original vía `void_de`, con su propio descuento de inventario ejecutado
  en la misma transacción de la anulación (`RN-ORD-44`).
- **FR-012**: El precio del ítem de reemplazo DEBE recalcularse desde la nueva variante y sus
  opciones; NO DEBE copiarse del ítem original (`RN-ORD-45`).
- **FR-013**: Consolidar una mesa DEBE exigir al menos un carrito `abierto` con ítems entre los
  comensales activos; sin ninguno, DEBE rechazarse con `409` (`RN-ORD-46`).
- **FR-014**: El precio de cada línea consolidada DEBE copiarse del `unit_price` del `CartItem`
  al momento de agregarlo, NO recalcularse contra el precio de catálogo vigente al consolidar
  (`RN-ORD-47`).
- **FR-015**: La consolidación DEBE insertar todas las líneas de todos los carritos primero, y
  ejecutar el descuento de inventario en un único lote al final, con los locks de insumo en
  orden canónico — para evitar deadlocks entre consolidaciones concurrentes (`RN-ORD-48`).
- **FR-016**: Un carrito consolidado DEBE quedar en estado `confirmado` y NO DEBE volver a
  ofrecerse para una consolidación posterior (`RN-ORD-49`).
- **FR-017 [Regla de modelo de datos]**: El sistema NO DEBE imponer una relación de "una única
  orden abierta por mesa" — una mesa DEBE poder tener varias órdenes activas simultáneas; la
  agrupación para cobrar se resuelve por `table_session_id` (`RN-ORD-50`).
- **FR-018**: La sesión de mesa activa DEBE crearse de forma perezosa al agregar el primer ítem
  (por consolidación o directo) si la mesa aún no tenía una, marcando la mesa `ocupada`
  (`RN-ORD-51`).
- **FR-019**: Un combo agregado directo por el mesero (`add_item_to_table`) DEBE expandirse en
  sus componentes reales a precio normal de catálogo; el descuento propio del combo NO DEBE
  calcularse en este paso, solo al cobrar (`RN-ORD-52`).
- **FR-020**: Un ítem agregado directo por el mesero DEBE guardar `participant_id=None` — no
  atribuido a ningún comensal, a diferencia de la consolidación, que sí asigna el comensal dueño
  del carrito (`RN-ORD-53`).
- **FR-021 [Regla crítica, anomalía A-04, `[DUDOSA]` en RN-CAT-33 — BUG HISTÓRICO CON
  DEPENDIENTES, prioridad de corrección alta]**: `add_item_to_table`, el único camino con botón
  real en la terminal del mesero en producción, invoca `load_valid_options` **sin** pasar el
  parámetro `variant` — a diferencia de `create_order` (`service.py:102`), que sí lo pasa. En
  consecuencia, no valida `min_select`/`max_select`/pertenencia de grupo de la selección de
  opciones en el camino de uso diario real, mientras que el camino sin caller de UI confirmado
  sí la valida y rechaza con `422` la misma selección incompleta. Reconstruido con `git
  log`/`git show`: regresión de fusión entre la rama de corrección `03469ca` (2026-08-03) y la
  rama de combos `ee94f30` (2026-08-04, autor distinto), que partió de una copia del fichero
  anterior a la corrección. Confirmado con testimonio de negocio (P4, jefe de cocina): merma
  real observada en conteo físico hace ~15 días, en sabores/toppings elegibles, coincidiendo con
  el patrón predicho. **Corrección**: pasar `variant=variant` en `consolidation.py:199` (una
  línea, ya aplicada una vez en `03469ca` y perdida en el merge). **No retroactivo**: no existe
  forma de recalcular el inventario ya consumido incorrectamente; la corrección solo detiene el
  sangrado hacia adelante (`RN-CAT-33`, cross-ref spec 004).
- **FR-022**: Solo se puede crear una comanda de staff para un comensal cuyo `status` sea
  `open`; si ya no está activo en la mesa, DEBE rechazarse con `409` (`RN-ORD-54`).
- **FR-023 [Regla crítica]**: La comanda de staff DEBE descontar inventario en la misma
  transacción de su creación (nace `abierta`, no `recibida`) — a diferencia del pedido QR, que
  descuenta solo al confirmar. Si cualquier línea falla por falta de stock, la transacción
  completa DEBE revertirse: ni la orden ni ninguna de sus líneas quedan creadas (`RN-ORD-55`).
- **FR-024**: Cambiar el estado operativo de una mesa a `libre` o `reservada` DEBE exigir que la
  mesa no tenga ninguna orden activa (no terminal); si la tiene, DEBE rechazarse con `409`
  (`RN-ORD-56`).
- **FR-025**: Mover una orden hacia su propia mesa actual DEBE ser un no-op — devuelve la orden
  sin cambios (`RN-ORD-57`).
- **FR-026 [`[DUDOSA]`, anomalía A-26, `PENDIENTE` cerrado en ronda 2 — riesgo latente, no
  activo]**: `move_order` exige que la mesa destino esté completamente sin órdenes activas antes
  de aceptar el traslado — más estricto que el modelo general de FR-017, que sí permite varias
  órdenes por mesa. Confirmado en entrevista de negocio (P20-bis): el equipo no usa la función de
  mover pedidos entre mesas hoy — riesgo de diseño latente, no un problema operativo activo
  (`RN-ORD-58`).
- **FR-027**: Mover una orden exitosamente DEBE liberar la mesa origen (`status="libre"`)
  únicamente si, tras el movimiento, ya no le queda ninguna orden activa (`RN-ORD-59`).
- **FR-028 [Anomalía A-26, ACCIDENTAL, código muerto]**: `move_order` captura `IntegrityError` y
  la traduce a `409`, pero el modelo de datos actual ya no define ningún índice único que pueda
  disparar esa excepción — es un manejador huérfano de una constraint removida. Corregir en
  modernización: retirarlo o documentar explícitamente por qué se conserva (`RN-ORD-60`).
- **FR-029**: `merge_orders` NO DEBE permitir fusionar ninguna orden que ya esté en estado
  terminal (`pagada`/`cancelada`); DEBE rechazarse con `409` la fusión completa (`RN-ORD-61`).
- **FR-030**: Si algún `order_id` solicitado en una fusión no existe, la fusión completa DEBE
  rechazarse con `404` — ninguna orden se modifica (`RN-ORD-62`).
- **FR-031 [Anomalía A-26, ACCIDENTAL — corregir en modernización]**: Cuando las órdenes a
  fusionar ya pertenecían a grupos (`merged_group_id`) distintos preexistentes, `merge_orders`
  DEBE (comportamiento observado hoy, a corregir) conservar solo uno de los grupos, determinado
  por un `SELECT` **sin `ORDER BY`** — no determinista —, dejando huérfanas las órdenes no
  listadas del grupo "perdedor". La modernización DEBE definir una regla explícita y
  determinista de qué grupo sobrevive (p. ej., el de menor `created_at`) (`RN-ORD-63`).
- **FR-032**: No puede existir más de una mesa con el mismo número; crear una mesa con un número
  ya usado DEBE rechazarse con `409` (`RN-ORD-66`).
- **FR-033 [Precisión de comportamiento intencional, anomalía A-48, cerrada en ronda 2 —
  INTENCIONAL confirmado]**: El sistema NO DEBE ofrecer ninguna pantalla de cocina (KDS)
  separada de la terminal de mesas; el ciclo de vida completo del ítem (transición, anulación)
  se opera desde la misma terminal. El estado intermedio `entregado` (retirado por la migración
  `c5d6e7f8a9b0`) NO DEBE reintroducirse — es una decisión operativa deliberada ("un solo
  dispositivo funciona mejor", confirmado P28), no una funcionalidad pendiente de completar
  (`memoria-historica.md` #12).

### Key Entities *(include if feature involves data)*

- **OrderItem**: línea de un pedido. Atributos relevantes a esta spec: `estado_cocina`
  (`pendiente`, `en_preparacion`, `listo`, `anulado` — gobierna las transiciones de FR-001/002 y
  la reversa de FR-006/007), `participant_id` (asignado en consolidación, `None` en alta
  directa del mesero — FR-020), `void_de` (traza el ítem de reemplazo al original anulado —
  FR-011), `unit_price` (copiado del carrito en consolidación, recalculado en reemplazo —
  FR-014/012).
- **OrderItemOption**: relación entre un `OrderItem` y las `Option` elegidas; su validez contra
  `min_select`/`max_select` del grupo depende de que el caller pase `variant` a
  `load_valid_options` (FR-021, la regla propia vive en RN-CAT-33/spec 004).
- **CustomerOrder**: pedido. Atributos relevantes: `status` (no comprobado por
  `transition_kitchen`/`void_item`, FR-004/005), `dining_table_id`, `table_session_id`,
  `merged_group_id` (grupo de fusión de mesas, FR-029 a FR-031), `channel` (`waiter` para las
  órdenes que acumula el mesero, FR-017).
- **Cart** / **CartItem**: carrito del comensal. Sus ítems, ya validados al agregarse desde el
  QR, se copian a `OrderItem` durante la consolidación sin revalidarse (FR-013 a FR-016); el
  carrito consolidado pasa a `confirmado` (FR-016).
- **TableSession**: sesión activa de una mesa; se crea perezosamente al agregar el primer ítem
  (FR-018) y es la unidad de agrupación de cobro, no la relación mesa-orden 1:1 (FR-017).
- **DiningTable**: mesa física. Atributos relevantes: `number` (único, FR-032), `status`
  (`libre`/`ocupada`/`reservada`, gobernado por FR-024 y por los efectos de mover una orden,
  FR-027).
- **OrderItemVoidLog**: registro de auditoría de cada anulación de ítem, con `motivo` y el actor
  que la ejecutó — escrito en la misma transacción que la reversa de inventario (FR-006/007).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-ORD-36` a `RN-ORD-64` y `RN-ORD-66` puede verificarse
  ejecutando los pasos descritos en esta spec contra un `pos-backend` en ejecución, sin necesitar
  leer `kitchen.py`, `consolidation.py`, `tables_advanced.py` ni `service.py` para entender el
  comportamiento esperado.
- **SC-002**: A-04 (`RN-CAT-33`/`FR-021`), el hallazgo de mayor evidencia de todo el
  reconocimiento, queda especificado con su reconstrucción completa de `git log` y su testimonio
  de negocio, de forma que la corrección de una línea (`variant=variant` en
  `consolidation.py:199`) pueda aplicarse en modernización con la certeza de que restaura
  exactamente el comportamiento de `03469ca`, sin reintroducir la regresión de `ee94f30`.
- **SC-003**: Existe al menos un script de characterization dedicado que cubre
  `kitchen.py`/`consolidation.py`/`tables_advanced.py` y, en particular, demuestra la divergencia
  de A-04 entre `add_item_to_table` y `create_order` sobre la misma selección de opciones
  incompleta — cerrando el gap de caracterización señalado como prioritario (ninguno de los 12
  scripts existentes los cubre hoy).
- **SC-004**: Las anomalías cerradas en la segunda ronda de entrevista de negocio (A-16/P16-bis,
  A-26/P20-bis, A-48/P28) quedan documentadas con su resolución y su cita de evidencia exacta,
  de forma que el equipo de modernización no las trate como decisiones aún pendientes ni vuelva
  a preguntarlas.
- **SC-005**: Las dos correcciones de código de bajo riesgo y sin urgencia operativa (A-16:
  replicar la validación de `mark_order_ready` en `transition_kitchen`/`void_item`; A-26:
  retirar/documentar el manejador huérfano y definir una regla determinista de fusión de grupos)
  quedan descritas con precisión suficiente para implementarse sin ambigüedad en la fase de
  modernización.

## Out of Scope

- **La confirmación de pedidos QR que descuenta inventario y su cancelación** (`confirm_order`,
  `cancel_order`, ciclo de cobro legado `block`/`pay`) — cubierto por la spec 008.
- **El cierre real de la cuenta de una mesa**: sesión de mesa, reparto entre comensales, cobro
  unificado o dividido, barrido de sesiones abandonadas, y la convención vigente y correcta de
  "cuánto se le debe cobrar a una mesa" (`table_sessions.compute_bill`) — cubierto por la spec
  010. La cuenta de un **grupo de mesas fusionadas** (`group_bill`, `RN-ORD-64`) sí se documenta
  aquí como comportamiento observado (User Story 8, escenario relacionado), porque es función
  propia de `tables_advanced.py`, pero su eventual unificación con la convención de la spec 010
  se decide en esa spec, no en esta.
- **La regla de validación de opciones que el caller de esta spec omite** (`min_select`/
  `max_select`, `validate_option_selection`, `STRICT_OPTION_SELECTION`) — la regla en sí
  (`RN-CAT-33` y las reglas relacionadas de tolerancia de migración) se especifica por completo
  en la spec 004; esta spec solo documenta **cuál caller la omite y cuáles no** (A-04).
- **El cálculo de qué consume cada línea** (receta fija, opciones, `plan_line_consumption`) —
  cubierto por la spec 003; esta spec asume que el cálculo de consumo es correcto y se limita a
  documentar cuándo se invoca (`deduct_order_items`/`reverse_order_items`) y con qué
  argumentos exactos (el punto exacto donde A-04 falla).
- **El motor de evaluación de promociones y combos** (`promotions.expand_combo`, prioridad entre
  promociones) — cubierto por las specs 012/013, aún no escritas en este reconocimiento; esta
  spec solo documenta cuándo `add_item_to_table` y `create_order` lo invocan para expandir un
  combo, no cómo decide el motor mismo.

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia del resto de
  las guías de este template ("evitar detalles de implementación"), aquí los endpoints, códigos
  de estado HTTP, nombres de función y campo, y valores literales **son** el contrato observable
  que se está documentando — se citan explícitamente porque los criterios de aceptación deben
  ser verificables directamente contra `pos-backend` en ejecución.
- **A-04/RN-CAT-33/FR-021 se especifica como el hallazgo de mayor prioridad de corrección de
  esta spec**: es el único de todo el reconocimiento con evidencia de `git log` de cómo y cuándo
  se rompió, reforzado por testimonio de negocio de merma real (P4). La corrección se especifica
  con precisión de una sola línea, pero explícitamente **no es retroactiva** — no existe
  mecanismo para recalcular el inventario ya consumido incorrectamente desde 2026-08-04.
- **A-16 (`RN-ORD-37`), A-26 (`RN-ORD-58`) y A-48 se documentan con su resolución de la segunda
  ronda de entrevista de negocio (P16-bis, P20-bis, P28), no como preguntas abiertas**: esas
  rondas ya cerraron sus porciones `PENDIENTE`, así que esta spec las fija como "riesgo latente
  mitigado por la operación real" (A-16, A-26) o "comportamiento intencional confirmado" (A-48),
  en vez de repetir la pregunta a la modernización.
- **Las porciones `ACCIDENTAL` de A-16 (`RN-ORD-38`/39) y A-26 (`RN-ORD-60`, `RN-ORD-63`) se
  especifican como defectos a corregir sin urgencia operativa**: el testimonio de negocio cerró
  el riesgo de que ocurran en la práctica diaria, pero no cambia su clasificación de código —
  siguen siendo inconsistencias verificables frente al resto del módulo, recomendadas para
  fase de modernización.
- **No se solicita ningún script de characterization nuevo como parte de esta spec**: SC-003
  señala el gap (ninguno de los 12 scripts existentes cubre estos tres ficheros) como criterio de
  aceptación pendiente de cerrar en la fase de implementación/modernización, no como algo que
  esta spec de documentación construya por sí misma.
- **El alcance de RN-ORD cubierto es RN-ORD-36 a RN-ORD-64 y RN-ORD-66** (RN-ORD-65, "no existe
  transición libre de status", es una regla protegida que se especifica por completo en la spec
  008, aunque su evidencia de código viva en el mismo router de órdenes).
