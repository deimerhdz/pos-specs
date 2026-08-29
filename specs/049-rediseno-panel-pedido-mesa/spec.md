# Feature Specification: Rediseño del panel de pedido de mesa — cliente, pedidos y cuenta

**Feature Branch**: `049-rediseno-panel-pedido-mesa`

**Created**: 2026-08-28

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección**, no una funcionalidad nueva desde cero. Igual
que las specs [019](../019-correccion-cuenta-mesas-fusionadas/spec.md),
[020](../020-correccion-validacion-opciones-mesero/spec.md),
[021](../021-correccion-orden-borrado-imagen-r2/spec.md),
[041](../041-correccion-bugs-menu-qr/spec.md),
[044](../044-rechazo-pedido-pago-pendiente/spec.md),
[045](../045-simplificacion-terminal-mesas/spec.md) y
[048](../048-pestanas-pago-pendiente-pedido/spec.md), cita nombres de archivo y componente del
código actual (`pos-heladeria`) porque son el contrato observable que se está corrigiendo, no una
fuga de detalles de implementación.

**Amend explícito** de spec [036](../036-terminal-mesas-rediseno-layout/spec.md): esa spec definió
que `pos-order-panel.component.ts` (panel central de la Terminal de Mesas) muestra, para el pedido
seleccionado, el nombre de la mesa, el campo "Cliente" editable, las pestañas por pedido con
"+ Nuevo pedido", la lista de ítems y el resumen Subtotal/Descuento/Total. Esta spec retira de ese
panel el control "+ Nuevo pedido" y el resumen Subtotal/Descuento/Total, y rediseña por completo
cómo se muestran el cliente y sus pedidos dentro de ese mismo panel. El resumen de totales se
traslada a `session-bill-panel.component.ts` ("Cuenta de la mesa"), que hoy ya muestra el
desglose por comensal y el "Total" pero no una fila agregada de "Subtotal"/"Descuento".

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-28, junto con dos capturas de pantalla (estado actual
de `pos-order-panel.component.ts` para "Mesa 3", y una referencia visual de diseño objetivo para
"Mesa 2") y tres decisiones de alcance resueltas en la sección de Clarifications. Es
reordenamiento/simplificación de navegación y visualización sobre una pantalla ya existente — no
reabre ninguna regla de negocio de precio, inventario o facturación (el cálculo de
subtotal/descuento/total no cambia, solo dónde se muestra); no aplica una nueva entrada en
`registro-de-anomalias.md`.

**Input**: User description (verbatim): "quiero mejorar el diseño actual de la visualizacion de
pedidos, quita la accion de nuevo pedido, la informacion del subtotal descuento y total la vas a
migrar para el panel de cuenta de la mesa, y la forma en que muestras el cliente y sus pedidos
deberas implementarla tal cual aparece en la segunda imagen adjunta, si tienes dudas me
preguntas" — con una primera captura mostrando el diseño actual de `pos-order-panel.component.ts`
("Mesa 3", pedido en preparación, campo Cliente editable, pestañas de pedido con "+ Nuevo pedido",
ítems y resumen Subtotal/Descuento/Total/"Marcar pedido listo"), y una segunda captura de
referencia mostrando el diseño objetivo ("Mesa 2 [Ocupada] · 👤 Deimer Hernandez", pestañas
"Todos los pedidos (2)" / "Pedido 1" / "Pedido 2", cada pedido en una tarjeta con hora y estado
propio, e ítems con cantidad, nombre, variante/nota y precio).

## Clarifications

### Session 2026-08-28

- Q: El campo "Cliente" hoy es un input editable (para nombrar pedidos de mostrador a mano). En la
  imagen de referencia aparece solo como texto de lectura. ¿Cómo debe quedar en este panel? → A:
  Siempre solo lectura — se retira la capacidad de asignar/editar el nombre del cliente desde este
  panel en todos los casos.
- Q: Al quitar "+ Nuevo pedido", una mesa ya ocupada pierde la forma de iniciar una segunda ronda
  desde este panel. ¿Qué debe pasar? → A: Sin reemplazo — no se ofrece ninguna otra forma de
  iniciar una segunda ronda manual para una mesa ocupada desde este panel.
- Q: Hoy cada ítem del carrito tiene su propio estado ("Pendiente"/"✓ Listo") y su botón para
  marcarlo listo individualmente. La imagen de referencia solo muestra un estado por pedido
  completo. ¿Qué se mantiene? → A: Se mantiene por ítem — cada ítem conserva su pill de estado y su
  acción individual para marcarlo listo; solo cambia el estilo visual del contenedor.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - La cuenta de la mesa concentra toda la información de cobro (Priority: P1)

El cajero necesita saber cuánto debe cobrar por una mesa. Hoy esa información
(Subtotal/Descuento/Total) aparece duplicada: una vez al pie del panel de pedido
(`pos-order-panel.component.ts`, junto al carrito) y otra vez, con otro formato, dentro del panel
de cuenta (`session-bill-panel.component.ts`, "Cuenta de la mesa"), que hoy solo muestra el
desglose por comensal y un único "Total" agregado, sin fila de "Subtotal" ni "Descuento" agregados.

**Why this priority**: es el cambio con mayor impacto en el flujo de cobro — dos lugares
mostrando información de dinero, con formato distinto, es la fuente más probable de confusión o
de mirar el panel equivocado antes de cobrar.

**Independent Test**: se puede probar completamente seleccionando una mesa con un pedido activo
que tenga descuento aplicado, y verificando que el panel de pedido ya no muestra ninguna fila de
Subtotal/Descuento/Total, mientras que el panel de cuenta de esa misma mesa muestra las tres cifras
agregadas junto al desglose por comensal que ya existía.

**Acceptance Scenarios**:

1. **Given** una mesa con un pedido activo seleccionado, **When** el cajero mira el panel de
   pedido (`pos-order-panel.component.ts`), **Then** ya no ve ninguna fila "Subtotal", "Descuento"
   ni "Total".
2. **Given** esa misma mesa, **When** el cajero mira el panel de cuenta de la mesa
   (`session-bill-panel.component.ts`), **Then** ve, además del desglose por comensal ya existente,
   un resumen agregado con "Subtotal", "Descuento" (si aplica) y "Total" para el conjunto de la
   mesa.
3. **Given** un pedido sin ningún descuento aplicado, **When** el cajero mira el resumen agregado
   en el panel de cuenta, **Then** la fila "Descuento" se muestra en cero, igual que hacía hoy el
   panel de pedido.
4. **Given** el resumen agregado ya visible en el panel de cuenta, **When** el cajero revisa el
   resto del panel, **Then** el desglose por comensal, el selector de método de pago y el botón
   "Cobrar y cerrar mesa" siguen funcionando exactamente igual que hoy — el cambio es aditivo, no
   reemplaza nada existente en ese panel.

---

### User Story 2 - Ver el cliente y sus pedidos en el nuevo formato de pestañas (Priority: P1)

El cajero selecciona una mesa con más de un pedido activo (por ejemplo, dos rondas distintas del
mismo comensal). Hoy `pos-order-panel.component.ts` muestra el nombre de la mesa y el estado en
líneas separadas, un input de texto para "Cliente", y una fila de botones tipo pastilla — uno por
pedido, rotulados con el nombre del cliente — sin ninguna forma de ver todos los pedidos a la vez.

**Why this priority**: es el corazón del pedido explícito del usuario ("la forma en que muestras
el cliente y sus pedidos deberás implementarla tal cual aparece en la segunda imagen") — sin este
cambio, el resto de la spec es solo limpieza menor.

**Independent Test**: se puede probar completamente seleccionando una mesa con dos o más pedidos
activos del mismo cliente, y verificando que aparecen las pestañas "Todos los pedidos (N)" y una
pestaña "Pedido N" por cada pedido, que cada tarjeta de pedido muestra su propia hora y estado, y
que los ítems dentro de cada tarjeta muestran cantidad, nombre, variante/nota y precio en el nuevo
formato.

**Acceptance Scenarios**:

1. **Given** una mesa seleccionada con al menos un pedido activo, **When** el cajero mira la
   cabecera del panel, **Then** ve el número de mesa, un indicador de estado de la mesa (p. ej.
   "Ocupada") y el nombre del cliente en modo solo lectura, en una sola cabecera — no un input de
   texto editable ni etiquetas separadas "Cliente" arriba de un campo.
2. **Given** una mesa con más de un pedido activo, **When** el cajero mira las pestañas, **Then**
   ve una pestaña "Todos los pedidos (N)" (N = cantidad de pedidos activos) seguida de una pestaña
   "Pedido 1", "Pedido 2", etc., una por cada pedido, en el mismo orden en que se crearon.
3. **Given** la pestaña "Todos los pedidos (N)" activa, **When** el cajero la mira, **Then** ve,
   en una sola vista desplazable, la tarjeta de cada pedido de la mesa, cada una con su propia
   hora de creación y su propio estado.
4. **Given** una pestaña individual "Pedido N" activa, **When** el cajero la mira, **Then** ve
   únicamente la tarjeta de ese pedido específico.
5. **Given** una tarjeta de pedido visible (en cualquiera de las dos vistas), **When** el cajero
   mira sus ítems, **Then** cada ítem muestra la cantidad, el nombre del producto, su variante o
   nota si tiene (p. ej. sabor, "sin chispas de chocolate"), el precio unitario y el precio total
   de esa línea — conservando además el pill de estado por ítem y la acción para marcarlo listo
   individualmente (ver Clarifications).
6. **Given** una mesa con un único pedido activo, **When** el cajero la selecciona, **Then** no se
   muestra ningún selector de pestañas — se ve directamente ese pedido, igual que ya ocurre hoy
   con el patrón existente de pestañas por pedido (`orderTabs()`, que hoy tampoco se muestran
   cuando hay un solo pedido).

---

### User Story 3 - El panel de pedido ya no ofrece iniciar una segunda ronda (Priority: P2)

El cajero tiene una mesa ocupada seleccionada, con uno o más pedidos activos. Hoy, junto a las
pestañas de pedido, hay un enlace "+ Nuevo pedido" que limpia la selección y deja el carrito vacío
para armar una ronda adicional manualmente sobre esa misma mesa, desde este mismo panel.

**Why this priority**: es el pedido explícito más simple de los tres — retirar un control — pero
se separa como historia propia porque, a diferencia de las otras dos, no tiene ningún reemplazo
funcional (ver Clarifications), por lo que su criterio de aceptación es puramente negativo
(ausencia del control).

**Independent Test**: se puede probar completamente seleccionando una mesa ocupada y verificando
que en ningún estado de las pestañas de pedido aparece un control "+ Nuevo pedido" ni ninguna otra
acción equivalente para iniciar una ronda manual adicional desde este panel.

**Acceptance Scenarios**:

1. **Given** una mesa ocupada con uno o más pedidos activos seleccionada, **When** el cajero mira
   el panel de pedido, **Then** no ve ningún control "+ Nuevo pedido" ni ninguna acción equivalente
   para iniciar una ronda adicional desde ahí.
2. **Given** ese mismo estado, **When** el cajero necesita registrar una ronda adicional para esa
   mesa por fuera del canal QR del comensal, **Then** no existe ninguna vía para hacerlo desde este
   panel (limitación conocida y aceptada, ver Assumptions) — queda fuera de alcance de esta spec
   ofrecer una alternativa.

---

### Edge Cases

- **Mesa sin ningún pedido guardado todavía (borrador nuevo sin guardar)**: el campo de cliente en
  modo solo lectura muestra el mismo placeholder informativo que ya existe hoy (p. ej.
  "Cliente · Mesa 3") en vez de un nombre real, ya que todavía no hay ningún pedido con nombre
  asociado.
- **Pedido de canal QR ya pagado (solo lectura por otras reglas, spec 036 T004)**: sigue sin
  ofrecer "+ Agregar producto" dentro de su tarjeta; las pestañas y el nuevo formato de tarjeta
  aplican igual, sin relación con esta restricción existente.
- **Ítem anulado dentro de una tarjeta de pedido**: conserva el mismo criterio de visualización que
  ya existe hoy (spec 029) — este rediseño no cambia qué ítems se muestran u ocultan, solo cómo se
  presentan los que ya se mostraban.
- **Todos los ítems de un pedido ya están "✓ Listo"**: el pedido puede seguir marcándose como listo
  a nivel completo con el botón "Marcar pedido listo" que ya existe hoy (fuera del carrito, junto
  al resumen); esta spec no cambia esa mecánica, solo retira de su misma sección visual las filas
  de Subtotal/Descuento/Total (User Story 1).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema NO DEBE mostrar ningún control "+ Nuevo pedido" (ni equivalente) dentro
  del panel de pedido de una mesa seleccionada (`pos-order-panel.component.ts`), sin ofrecer
  ninguna alternativa dentro de ese mismo panel para iniciar una ronda de pedido adicional sobre
  una mesa ya ocupada (decisión confirmada en Clarifications).
- **FR-002**: El sistema DEBE retirar del panel de pedido (`pos-order-panel.component.ts`) las
  filas "Subtotal", "Descuento" y "Total" que hoy se muestran junto al carrito de la mesa
  seleccionada.
- **FR-003**: El sistema DEBE mostrar, dentro del panel de cuenta de la mesa
  (`session-bill-panel.component.ts`, "Cuenta de la mesa"), un resumen agregado con las filas
  "Subtotal", "Descuento" y "Total" para el conjunto de pedidos activos de esa mesa, adicional al
  desglose por comensal y al "Total" que ese panel ya muestra hoy.
- **FR-004**: El resumen agregado del FR-003 DEBE reflejar los mismos importes que hoy calcula el
  sistema para esos conceptos — esta spec no cambia ninguna regla de cálculo de subtotal,
  descuento o total, solo dónde se presenta esa información.
- **FR-005**: El sistema DEBE conservar sin cambios de comportamiento el resto del panel de cuenta
  de la mesa (desglose por comensal, selector de método de pago, botón "Cobrar y cerrar mesa",
  aviso de "La cuenta cambió", modo de solo lectura para pedidos ya pagados por QR) — el cambio de
  este FR es aditivo.
- **FR-006**: El campo que muestra el nombre del cliente en el panel de pedido
  (`pos-order-panel.component.ts`) DEBE presentarse siempre como texto de solo lectura, sin ningún
  input editable, independientemente del origen o estado del pedido (decisión confirmada en
  Clarifications).
- **FR-007**: Cuando el pedido seleccionado no tenga ningún nombre de cliente asociado, el sistema
  DEBE mostrar en su lugar el mismo texto de referencia (placeholder) que ya usa hoy para orientar
  al cajero sobre qué mesa es, en modo de solo lectura.
- **FR-008**: El sistema DEBE reorganizar la cabecera del panel de pedido para mostrar, en una sola
  fila, el número de mesa, un indicador del estado de la mesa y el nombre del cliente en modo
  lectura (junto con el control "← Volver" ya existente), en vez del formato actual apilado con una
  etiqueta "Cliente" separada.
- **FR-009**: Cuando la mesa seleccionada tenga más de un pedido activo, el sistema DEBE mostrar
  una pestaña "Todos los pedidos (N)" (N = cantidad de pedidos activos) seguida de una pestaña por
  cada pedido individual ("Pedido 1", "Pedido 2", …), numeradas en el mismo orden en que ya se
  devuelven hoy (`orderTabs()`/`ordersOfTable()`).
- **FR-010**: Cuando la mesa seleccionada tenga un único pedido activo, el sistema NO DEBE mostrar
  ningún selector de pestañas — se muestra directamente ese pedido, igual que ya ocurre hoy con el
  patrón existente de pestañas por pedido.
- **FR-011**: La pestaña "Todos los pedidos (N)" DEBE mostrar, en una sola vista desplazable, una
  tarjeta por cada pedido activo de la mesa, cada una con su propia hora de creación y su propio
  estado.
- **FR-012**: Cada pestaña individual "Pedido N" DEBE mostrar únicamente la tarjeta correspondiente
  a ese pedido.
- **FR-013**: Cada tarjeta de pedido DEBE mostrar, por cada ítem: cantidad, nombre del producto,
  su variante o nota si la tiene, precio unitario y precio total de esa línea.
- **FR-014**: Cada ítem dentro de una tarjeta de pedido DEBE conservar el pill de estado de
  preparación ("Pendiente"/"✓ Listo") y la acción para marcarlo listo individualmente, con el mismo
  comportamiento que tiene hoy — solo cambia la disposición visual de su contenedor (decisión
  confirmada en Clarifications).
- **FR-015**: El sistema DEBE conservar sin cambios de comportamiento las demás acciones ya
  existentes en el panel de pedido que no fueron mencionadas por el usuario: agregar producto al
  pedido (catálogo embebido), anular ítem, guardar pedido en borrador ("Guardar pedido") y marcar
  el pedido completo como listo ("Marcar pedido listo").

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un cajero puede identificar el subtotal, el descuento y el total a cobrar de una
  mesa consultando un único panel (el de cuenta), sin necesitar mirar el panel de pedido para
  encontrar esa información.
- **SC-002**: Un cajero puede ver todos los pedidos activos de una mesa a la vez, o enfocarse en
  uno solo, alternando pestañas en un único clic, sin perder de vista el nombre del cliente ni el
  estado de la mesa.
- **SC-003**: El panel de pedido de una mesa ocupada no ofrece ninguna acción para iniciar una
  ronda de pedido adicional (0 controles "+ Nuevo pedido" o equivalentes presentes).
- **SC-004**: El 100% de los ítems que hoy pueden marcarse individualmente como listos conservan
  esa misma capacidad tras el rediseño, sin pérdida de información de preparación por ítem.

## Assumptions

- El indicador de estado de la mesa mostrado en la nueva cabecera (p. ej. "Ocupada") reutiliza el
  mismo cálculo de estado que ya existe hoy para la grilla de mesas (`deriveTableStatus`, spec
  036), sin introducir una nueva regla de estado.
- La numeración "Pedido 1"/"Pedido 2" sigue el mismo orden cronológico de creación que ya devuelve
  hoy `ordersOfTable()`/`orderTabs()` — no se introduce un criterio de orden nuevo.
- El estado que se muestra a nivel de cada tarjeta de pedido (p. ej. equivalente a "en
  preparación"/"listo") es el mismo estado agregado por pedido que ya calcula hoy
  `headerStatusText()`; el detalle exacto de su presentación visual (etiqueta y color por estado)
  se resuelve en la fase de planeación técnica, no en esta spec.
- La fila "Descuento" del nuevo resumen agregado en el panel de cuenta sigue siendo
  exclusivamente el descuento automático por promoción ya vigente (spec 029, Historia 2) — esta
  spec no reintroduce ni habilita ningún control de descuento manual.
- Fuera de alcance de esta spec: cualquier cambio a la lógica de cálculo de subtotal, descuento o
  total; al mecanismo de cobro y cierre de mesa; a cómo se crean pedidos nuevos por canal QR o
  desde la vista dedicada de pedido manual (`manual-order-page.component.ts`); y a los estados de
  cocina por ítem o por pedido en sí mismos (solo cambia su presentación).
