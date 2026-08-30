# Feature Specification: Carga diferida de datos y tarjetas de pedido de Domicilio/Para Llevar en la Terminal de Mesas

**Feature Branch**: `059-terminal-mesas-carga-y-pedidos`

**Created**: 2026-08-29

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección**, doble. Igual que las specs
[045](../045-simplificacion-terminal-mesas/spec.md),
[048](../048-pestanas-pago-pendiente-pedido/spec.md),
[049](../049-rediseno-panel-pedido-mesa/spec.md),
[055](../055-canal-tipo-orden/spec.md) y
[056](../056-domicilio-orden-manual/spec.md), cita nombres de archivo, función y línea del código
actual (`pos-heladeria`) porque son el contrato observable que se está corrigiendo, no una fuga de
detalles de implementación. Corrige dos cosas distintas sobre la misma pantalla ya implementada
(`table-sessions.component.ts`, spec 028/036):

1. **Rendimiento de carga inicial**: hoy `TableSessionsComponent.ngOnInit()` llama
   `store.init()` (`pos-terminal.store.ts:836`), que dispara en un solo `Promise.all` **todas**
   estas peticiones sin importar qué vaya a hacer el cajero: `tableService.loadTables()`,
   `reloadOrders()`, `paymentMethodService.load()` + `.loadAvailableForCheckout()`,
   `menuService.loadMenu()`, `promotionService.loadActive()` y `cash.restoreShift()`. De esas,
   `paymentMethodsAvailable()` y `cashShiftId` solo se consumen dentro de
   `pos-checkout-panel.component.ts` (panel de cobro, líneas 81/103/175 y 82/104
   respectivamente) — nunca antes de que haya algo seleccionado para cobrar. En cambio,
   `menuService.categories()` y `promotionService.activePromotions()` sí son necesarios para el
   primer render correcto: `orderSubtotal()` (`pos-terminal.store.ts:1788`) los usa para calcular
   el total con descuento que ya se muestra en cada tarjeta de la grilla (`tablesView()`,
   `pos-terminal.store.ts:624-666`) — diferirlos mostraría totales incorrectos en el primer
   render. Esta spec por lo tanto solo difiere las dos peticiones que hoy no tienen ningún
   consumidor antes del cobro (ver Assumptions).
2. **Pedidos "Para Llevar"/"Domicilio" invisibles tras crearlos**: la pestaña
   "Domicilios"/"Para llevar" (`pos-tables-panel.component.ts:110-120`) sigue siendo hoy el
   mensaje estático fijo que definió spec 036 ("Todavía no hay una forma de crear órdenes de...").
   Spec 055 y 056 ya habilitaron crear pedidos reales `TAKEAWAY`/`DELIVERY` desde
   `manual-order-page.component.ts`, con `hold_for_payment: true` (pendientes de cobro) — al
   confirmarlos, `createManualOrderFromDraft()` muestra el toast "Pedido creado — cóbralo desde el
   panel de la derecha" y navega de vuelta a `/dashboard/mesas-sesiones`. Pero esa ruta monta una
   instancia **nueva** de `PosTerminalStore` (`providers: [PosTerminalStore]` en ambos
   componentes), así que el pedido recién creado queda sin ninguna forma de encontrarlo ni
   cobrarlo desde la Terminal de Mesas: **la promesa del toast no se cumple hoy**. `store.orders()`
   ya trae esos pedidos completos (`DiningSessionService.listOrders()`, sin filtrar por tipo), pero
   `tablesView()` solo itera mesas físicas — nunca esos pedidos. No existe tampoco ningún
   componente de "tarjeta de mesa" reutilizable: el markup vive inline dentro de
   `pos-tables-panel.component.ts:74-95`.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-29, con dos capturas de pantalla — una del panel de
red (DevTools) mostrando las peticiones HTTP disparadas al abrir la Terminal de Mesas sin haber
seleccionado nada todavía, y una tarjeta de referencia de diseño ("Llevar #403 · María G. ·
Preparando · 1 Item · $8.500"). Es reordenamiento de cuándo se piden datos y de cómo se presentan
pedidos ya existentes — no reabre ninguna regla de negocio de precio, inventario o facturación; no
aplica una nueva entrada en `registro-de-anomalias.md`.

**Input**: User description (verbatim, traducido del mensaje original con dos capturas): "quiero
mejorar la terminal de mesas, como puedes observar en la imagen adjunta, en el estado inicial se
cargan muchas peticiones http de la cuales el usuario no ve informacion en su mayoria, quiero
mejorar ese rendimiento, por ejemplo en esa primera parte todavia no estoy seleccionando un metodo
de pago entonces no se deberia de hacer esa peticion, en ese orden de ideas, creo que el patron de
lazy load resolveria ese problema si no es asi sugiere uno, en principio tu funcion es analizar los
endpoint que se deberian de llamar a medida que el usuario navega por la pagina web y no hacer
todas las peticiones de golpe, el otro ajuste es que las pestañas de domicilio y para llevar,
debera hacer un filtro de los pedidos y mostrarlos en el mismo formato card que se muestran las
mesas y cada tarjeta debera verse como en la segunda imagen y debera reutilizarse para los pedidos
a llevar y pedidos a domicilio y seguir el mismo comportamiento de las mesas que cuando se
selecciona una mesa, se muestra el detalle del pedido".

## Clarifications

### Session 2026-08-29

- Q: ¿Cuándo exactamente debe dispararse la carga diferida de métodos de pago y turno de caja: al
  seleccionar cualquier mesa (aunque esté libre, sin pedido todavía), o solo al seleccionar un
  pedido real (de mesa, o de Domicilio/Para llevar)? → A: Solo al seleccionar un pedido real —
  confirmado por código: el selector de método de pago (`pos-checkout-panel.component.ts:173-177`)
  y el turno de caja (`pos-terminal.store.ts:1636`, usado en `checkout()`) solo se necesitan cuando
  `store.selectedOrder()` es un pedido real; una mesa libre seleccionada solo muestra el botón
  "+ Crear pedido nuevo" (`pos-checkout-panel.component.ts:131-144`), sin selector de pago ni turno
  de caja.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - No pedir datos de cobro que el cajero todavía no necesita (Priority: P1)

El cajero abre la Terminal de Mesas y todavía no ha seleccionado ninguna mesa ni pedido. Hoy, en
ese mismo instante, el sistema ya pidió el catálogo completo de métodos de pago y el estado del
turno de caja — datos que solo se usan dentro del panel de cobro, cuando hay algo que cobrar.

**Why this priority**: es el ejemplo explícito del usuario ("todavia no estoy seleccionando un
metodo de pago entonces no se deberia de hacer esa peticion") y el cambio de mayor impacto en
tiempo de carga percibido — reduce peticiones simultáneas al abrir la pantalla sin afectar ninguna
información que el cajero ya está viendo.

**Independent Test**: se puede probar completamente abriendo la Terminal de Mesas con las
herramientas de red del navegador visibles, sin seleccionar nada, y verificando que no aparece
ninguna petición de métodos de pago ni de turno de caja hasta seleccionar un pedido real.

**Acceptance Scenarios**:

1. **Given** la Terminal de Mesas recién abierta sin ninguna mesa ni pedido seleccionado, **When**
   se inspeccionan las peticiones de red disparadas, **Then** no aparece ninguna petición al
   catálogo de métodos de pago ni al estado del turno de caja.
2. **Given** ese mismo estado, **When** el cajero selecciona una mesa libre (sin ningún pedido
   todavía), **Then** tampoco se disparan esas peticiones — seleccionar una mesa libre solo
   habilita el botón "+ Crear pedido nuevo", que no necesita ninguno de esos datos.
3. **Given** ese mismo estado, **When** el cajero selecciona, por primera vez en esa sesión de
   pantalla, un pedido real (de una mesa con pedido activo, o un pedido de Domicilio/Para llevar),
   **Then** en ese momento (y no antes) se disparan las peticiones de métodos de pago y turno de
   caja, exactamente una vez cada una.
4. **Given** que esas peticiones ya se cargaron una vez en la sesión de pantalla, **When** el
   cajero selecciona otro pedido distinto, **Then** no se repiten — se reutiliza lo ya cargado,
   igual que hoy ya ocurre con la carga inicial (`methods().length === 0 ? load() : null`).
5. **Given** los datos ya necesarios para el primer render correcto de la grilla (mesas, pedidos
   activos, menú/catálogo y promociones activas — de los que dependen el conteo de productos y el
   total con descuento que ya muestra cada tarjeta), **When** se abre la terminal, **Then** esos
   siguen cargándose de inmediato, sin cambios respecto a hoy.

---

### User Story 2 - Ver los pedidos de Domicilio y Para Llevar como tarjetas (Priority: P1)

El cajero abre la pestaña "Para llevar" (o "Domicilios") de la Terminal de Mesas. Hoy ve siempre el
mismo mensaje vacío fijo, incluso si ya existen pedidos reales de ese tipo, creados y pendientes de
cobro desde la pantalla dedicada de pedido manual (spec 055/056). Con esta mejora, cada pestaña
filtra y muestra los pedidos de su propio tipo como tarjetas, en el mismo formato visual que ya
usan las tarjetas de mesa (identificador, insignia de estado, cantidad de productos y total).

**Why this priority**: cierra una brecha funcional real, no solo visual — esos pedidos ya existen
en el sistema (spec 055/056) pero hoy son invisibles e inalcanzables desde la Terminal de Mesas.

**Independent Test**: se puede probar completamente creando un pedido "Para Llevar" desde la
creación manual, abriendo la pestaña "Para llevar" de la Terminal de Mesas, y verificando que
aparece una tarjeta con su información — sin necesitar todavía poder seleccionarla (Historia 3).

**Acceptance Scenarios**:

1. **Given** un pedido "Para Llevar" ya creado y pendiente de cobro, **When** el cajero abre la
   pestaña "Para llevar", **Then** ve una tarjeta con una referencia identificable del pedido, el
   nombre del cliente asociado, un indicador de estado, la cantidad de productos y el total.
2. **Given** un pedido "Domicilio" ya creado y pendiente de cobro, **When** el cajero abre la
   pestaña "Domicilios", **Then** ve su propia tarjeta con la misma información, solo en esa
   pestaña — nunca mezclada con "Para llevar" ni con mesas.
3. **Given** que no existe ningún pedido pendiente de cobro de un tipo determinado, **When** el
   cajero abre esa pestaña, **Then** sigue viendo el mismo mensaje informativo de listado vacío ya
   existente (spec 036, FR-003) — sin regresión.
4. **Given** un pedido "Para Llevar"/"Domicilio" ya cobrado/facturado, **When** el cajero mira esa
   pestaña, **Then** ya no aparece en el listado — mismo criterio ya definido en spec 036 para
   estos tipos de orden (desaparece automáticamente al cobrarse, sin ningún paso de "Cerrar").

---

### User Story 3 - Seleccionar una tarjeta de pedido muestra su detalle y permite cobrarlo (Priority: P1)

El cajero selecciona una tarjeta de la pestaña "Para llevar"/"Domicilios". Hoy no hay ninguna
tarjeta que seleccionar (Historia 2 resuelve eso primero); esta historia asegura que, una vez
visible, seleccionarla se comporta igual que seleccionar una mesa: el panel central muestra el
detalle del pedido y el panel derecho permite cobrarlo — cerrando la brecha donde hoy el mensaje
"cóbralo desde el panel de la derecha" no tenía ningún pedido real al cual llevar al cajero.

**Why this priority**: sin esto, ver las tarjetas (Historia 2) no tiene ningún valor de negocio —
el cajero seguiría sin poder cobrar el pedido que ya armó.

**Independent Test**: se puede probar completamente, a partir de las tarjetas de la Historia 2,
seleccionando una y verificando que el panel central muestra sus productos y que el panel derecho
ofrece cobrarlo con el mismo flujo ya existente.

**Acceptance Scenarios**:

1. **Given** una tarjeta de pedido de Domicilio/Para llevar visible, **When** el cajero la
   selecciona, **Then** el panel central muestra el detalle de ese pedido (productos, cantidades,
   precio y estado), con el mismo comportamiento ya existente al seleccionar el pedido de una mesa.
2. **Given** ese pedido seleccionado, **When** el cajero mira el panel derecho, **Then** puede
   cobrarlo con el mismo flujo de cobro ya implementado (selección de método de pago, cálculo de
   cambio, envío a cocina si aplica).
3. **Given** un pedido de Domicilio seleccionado, **When** el cajero mira su detalle, **Then**
   también ve los datos propios ya capturados al crearlo (dirección, teléfono si tiene, valor del
   domicilio — spec 056).
4. **Given** el cajero tenía una mesa seleccionada, **When** selecciona en cambio una tarjeta de
   pedido de Domicilio/Para llevar, **Then** la mesa queda deseleccionada — la pantalla mantiene
   una única selección activa a la vez, igual que ya ocurre hoy entre mesas distintas.

---

### Edge Cases

- **Pedido "Para Llevar" sin nombre de cliente editado** (queda con "Consumidor final" por
  defecto, spec 055): la tarjeta se muestra igual, usando ese mismo valor por defecto.
- **Dirección de un pedido Domicilio muy larga**: se trunca visualmente en la tarjeta y en el
  detalle, igual que ya trunca hoy el nombre en una tarjeta de mesa.
- **El cajero cambia de "Mesas" a "Para llevar" mientras tenía una mesa seleccionada con pedido en
  curso**: la selección de mesa se conserva en memoria (no se pierde el trabajo en curso), pero
  deja de estar resaltada mientras la pestaña activa es otra; volver a "Mesas" la muestra tal como
  la dejó — mismo comportamiento ya existente de las pestañas de tipo de orden.
- **Dos pedidos "Para Llevar" nuevos casi simultáneos**: ambos aparecen como tarjetas
  independientes; seleccionar cualquiera no afecta al otro — mismo aislamiento que ya existe entre
  pedidos de distintas mesas.
- **Falla la petición diferida de métodos de pago o turno de caja** justo cuando el cajero
  selecciona algo para cobrar (red caída): el panel de cobro muestra el mismo tipo de error ya
  manejado hoy para esas peticiones, sin bloquear el resto de la pantalla.
- **El cajero hace clic en una tarjeta antes de que termine la carga inicial de mesas/pedidos**: el
  comportamiento no cambia respecto a hoy (esos datos siguen sin diferirse, Historia 1).
- **El cajero selecciona una mesa libre y luego crea un pedido nuevo desde ella** (vía
  "+ Crear pedido nuevo"/F3, que navega a `manual-order-page.component.ts`): mientras la mesa
  estuvo seleccionada sin pedido, no se dispararon las peticiones de métodos de pago ni turno de
  caja (FR-001); solo se disparan cuando el cajero regresa a la Terminal de Mesas y selecciona el
  pedido real ya creado.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema NO DEBE solicitar el catálogo de métodos de pago ni el estado del turno de
  caja al cargar la Terminal de Mesas, ni mientras el cajero solo tenga seleccionada una mesa libre
  sin ningún pedido todavía — mismo umbral que ya usa hoy el panel de cobro para decidir si muestra
  el selector de método de pago (`store.selectedOrder()` con valor real, ver Clarifications).
- **FR-002**: El sistema DEBE solicitar el catálogo de métodos de pago y el estado del turno de
  caja en cuanto el cajero seleccione, por primera vez en esa sesión de pantalla, un pedido real
  (de una mesa, o de Domicilio/Para llevar) — nunca por solo seleccionar una mesa libre sin pedido.
- **FR-003**: Una vez cargados el catálogo de métodos de pago y el turno de caja en la sesión de
  pantalla, el sistema NO DEBE volver a solicitarlos al seleccionar otro pedido distinto — misma
  regla de caché ya aplicada hoy a la carga inicial.
- **FR-004**: El sistema DEBE seguir solicitando de inmediato, sin diferir, los datos ya necesarios
  para el primer render correcto de la grilla (mesas, pedidos activos, menú/catálogo y promociones
  activas), porque el total con descuento y el conteo de productos que ya muestra cada tarjeta
  dependen de ellos.
- **FR-005**: Las pestañas "Domicilios" y "Para llevar" DEBEN mostrar, cuando existan pedidos
  pendientes de cobro de su propio tipo, una tarjeta por cada pedido, en el mismo formato visual
  ya usado por las tarjetas de mesa (identificador/referencia, insignia de estado, cantidad de
  productos, total).
- **FR-006**: Cada pestaña DEBE mostrar únicamente los pedidos de su propio tipo (`DELIVERY` para
  "Domicilios", `TAKEAWAY` para "Para llevar") — nunca mezclados entre sí ni con pedidos de mesa
  (`DINE_IN`).
- **FR-007**: Cada tarjeta de pedido de Domicilio/Para llevar DEBE mostrar: el tipo de pedido con
  una referencia identificable, el nombre del cliente asociado, un indicador de estado (reutilizando
  el mismo vocabulario de estado ya usado por las tarjetas de mesa), la cantidad de productos y el
  total a cobrar.
- **FR-008**: Un pedido de Domicilio/Para llevar ya cobrado/facturado NO DEBE seguir apareciendo en
  el listado de su pestaña — mismo criterio ya definido en spec 036 para estos tipos de orden
  (desaparece automáticamente al cobrarse, sin ningún paso de "Cerrar").
- **FR-009**: Cuando no exista ningún pedido pendiente de cobro de un tipo determinado, esa pestaña
  DEBE seguir mostrando el mismo mensaje informativo de listado vacío ya existente (spec 036,
  FR-003), sin regresión.
- **FR-010**: Seleccionar una tarjeta de pedido de Domicilio/Para llevar DEBE mostrar el detalle de
  ese pedido en el panel central, con el mismo comportamiento ya implementado al seleccionar una
  mesa con pedido activo (lista de ítems, cantidad, precio y estado).
- **FR-011**: Con un pedido de Domicilio/Para llevar seleccionado, el panel derecho de cobro DEBE
  ofrecer el mismo flujo de cobro ya implementado (selección de método de pago, cálculo de cambio,
  envío a cocina si aplica) para ese pedido específico.
- **FR-012**: Seleccionar un pedido de Domicilio DEBE mostrar además, dentro del mismo panel de
  detalle, sus datos propios ya capturados al crearlo (dirección, teléfono si tiene, valor del
  domicilio — spec 056).
- **FR-013**: Seleccionar una tarjeta de pedido de Domicilio/Para llevar DEBE deseleccionar
  cualquier mesa que estuviera seleccionada previamente — la pantalla mantiene una única selección
  activa a la vez, igual que ya ocurre hoy entre mesas distintas.

### Key Entities *(include if feature involves data)*

- **Pedido (`DiningOrder`)**: entidad ya existente; ya tiene tipo de orden (`DINE_IN`/`TAKEAWAY`/
  `DELIVERY`, spec 055) y los campos de domicilio (spec 056). Esta spec no agrega ningún atributo
  nuevo — solo cambia cuándo se piden ciertos datos relacionados y cómo/cuándo se presentan
  visualmente los pedidos `TAKEAWAY`/`DELIVERY` que ya existían pero no tenían ningún punto de
  entrada visible en esta pantalla.
- **Métodos de pago / Turno de caja**: entidades ya existentes (spec 032/006); esta spec no cambia
  su modelo de datos, solo el momento en que la pantalla los solicita.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Al abrir la Terminal de Mesas sin seleccionar nada, la cantidad de peticiones HTTP
  disparadas de inmediato se reduce (métodos de pago y turno de caja dejan de dispararse) sin
  perder ninguna información ya visible en la grilla inicial.
- **SC-002**: El cajero puede encontrar y abrir cualquier pedido "Para Llevar" o "Domicilio"
  pendiente de cobro sin salir de la Terminal de Mesas, en un máximo de dos interacciones (elegir
  pestaña y seleccionar tarjeta) — hoy no existe ninguna forma de encontrarlo desde esta pantalla.
- **SC-003**: El 100% de los pedidos "Para Llevar"/"Domicilio" pendientes de cobro creados desde la
  creación manual (spec 055/056) quedan localizables y cobrables desde su pestaña correspondiente
  en la Terminal de Mesas.
- **SC-004**: El tiempo hasta que el cajero puede empezar a operar la grilla de mesas no aumenta
  respecto a hoy — la carga diferida solo reduce peticiones que no eran necesarias, sin retrasar
  las que sí lo eran.
- **SC-005**: Cero pedidos "Para Llevar"/"Domicilio" ya cobrados permanecen visibles en su pestaña.

## Out of Scope

- **Cualquier cambio de backend**: ningún endpoint, campo o migración nueva — spec 055/056 ya
  expone todo el dato necesario (`order_type`, `customer_name`, campos `delivery_*`); esta spec es
  una corrección de cuándo el frontend pide ciertos datos y de cómo presenta datos que ya existen.
- **Un número de orden/consecutivo corto nuevo** para pedidos aún no facturados — fuera de alcance
  (ver Assumptions); se reutiliza información ya disponible del pedido.
- **Crear pedidos de Domicilio/Para llevar desde la propia Terminal de Mesas**: esta spec solo hace
  visibles y cobrables los que ya se crean desde `manual-order-page.component.ts` (spec 055/056) —
  la creación sigue viviendo en esa pantalla dedicada, sin cambios.
- **Diferir la carga del menú/catálogo o de las promociones activas**: se investigó y se determinó
  que ambos son necesarios para el primer render correcto de los totales ya visibles en la grilla
  (ver Assumptions) — diferirlos queda fuera de esta spec.
- **Logística de domicilios, asignación de repartidor o notificaciones al cliente**: sin cambios,
  igual que ya definieron specs 036/056.

## Assumptions

- **La referencia identificable de la tarjeta (FR-007) no introduce ningún número de orden nuevo**:
  `DiningOrder` no tiene hoy ningún consecutivo corto antes de facturar (solo un `id` tipo UUID; el
  consecutivo fiscal solo existe después de cobrar, al generarse la venta). Esta spec reutiliza
  datos ya disponibles del pedido (por ejemplo, la hora de creación, con el mismo patrón "🕐" que ya
  usa la tarjeta de mesa) en vez de agregar un campo de numeración nuevo — más fiel además al pedido
  explícito de "mismo formato card que se muestran las mesas" que inventar un dato nuevo solo para
  imitar el número de la imagen de referencia.
- **El indicador de estado de la tarjeta (FR-007) reutiliza el mismo vocabulario ya usado por las
  tarjetas de mesa** (`STATUS_META`/`deriveTableStatus`, spec 036) en vez de introducir una palabra
  nueva como "Preparando" — ese texto no existe hoy en ningún estado del sistema (los estados
  existentes son "Por confirmar"/"Abierta"/"Bloqueada"/"Pagada"/"Cancelada" a nivel de pedido, y
  "Pendiente"/"En preparación"/"Listo"/"Anulado" a nivel de ítem); la imagen de referencia se toma
  como guía de formato visual, no como texto literal a introducir.
- **El disparador exacto de FR-001/FR-002 es "seleccionar un pedido real"** (`store.selectedOrder()`
  con valor, ver Clarifications) — no cualquier selección de mesa. Una mesa libre seleccionada solo
  habilita el botón "+ Crear pedido nuevo" del panel de cobro (`pos-checkout-panel.component.ts:
  131-144`), sin necesitar el selector de método de pago ni el turno de caja; ambos solo se
  necesitan una vez que existe un pedido real que cobrar (de mesa, o de Domicilio/Para llevar). No
  exige que el cajero llegue a tocar el selector de método de pago en sí — basta con que la
  selección deje visible el flujo de cobro editable, para que la petición ya esté resuelta cuando
  la necesite.
- **El flujo de cobro reutilizado en la Historia 3 (FR-011) es exactamente el mismo ya implementado
  para pedidos de mostrador sin mesa** (el mismo que hoy ya cobra pedidos `TAKEAWAY`/`DELIVERY`
  creados desde `manual-order-page.component.ts`, solo que hoy no hay forma de volver a encontrar
  ese pedido después de crearlo) — esta spec no cambia ninguna regla de cálculo de precio,
  descuento o inventario, solo cómo se llega a ese pedido desde la Terminal de Mesas.
- **Implementación pendiente al momento de escribir esta spec** — a diferencia de spec 045 (que se
  documentó después de implementar), esta spec se escribe antes de construir el cambio.
