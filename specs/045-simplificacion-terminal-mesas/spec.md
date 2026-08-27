# Feature Specification: Simplificación de la Terminal de Mesas (placeholder único, botón fijo de mostrador, tarjetas solo-selección)

**Feature Branch**: `045-simplificacion-terminal-mesas`

**Created**: 2026-08-27

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección**, no una funcionalidad nueva desde cero. Igual
que las specs [019](../019-correccion-cuenta-mesas-fusionadas/spec.md),
[020](../020-correccion-validacion-opciones-mesero/spec.md),
[021](../021-correccion-orden-borrado-imagen-r2/spec.md),
[041](../041-correccion-bugs-menu-qr/spec.md) y
[044](../044-rechazo-pedido-pago-pendiente/spec.md), cita nombres de archivo y componente del
código actual (`pos-heladeria`) porque son el contrato observable que se está corrigiendo, no una
fuga de detalles de implementación. El usuario pidió documentar este cambio "en la spec 036"
(la que introdujo el layout actual de la Terminal de Mesas); siguiendo el mismo criterio que la
sesión anterior (spec 044 sobre spec 036 también), se documenta como spec nueva que cita y
**amend explícitamente** dos requisitos funcionales de la spec
[036](../036-terminal-mesas-rediseno-layout/spec.md) — nunca editando in-place una spec ya
implementada:

- **FR-004** de spec 036 (sección global "Pagos por confirmar" cuando no hay mesa seleccionada) —
  se retira; ver Historia 1.
- **FR-005** de spec 036 (seleccionar una tarjeta de mesa libre abre el "constructor de orden
  manual" embebido) — deja de aplicar a mesas libres; ver Historia 3. (Esa parte de FR-005 ya
  había quedado obsoleta por un ajuste posterior no documentado que migró la creación de pedido
  manual a una página dedicada — esta spec formaliza ese punto de entrada y lo corrige.)

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-27, junto con dos decisiones de alcance resueltas en
la sección de Clarifications. No reabre ninguna regla de negocio de precio, inventario o
facturación — es reordenamiento de navegación/UI sobre una pantalla ya existente; no aplica una
nueva entrada en `registro-de-anomalias.md`.

**Input**: User description (verbatim, traducido del mensaje original con una captura del estado
inicial de la Terminal de Mesas): "quiero hacer varias modificaciones: primero, no debería
mostrarse el pedido de la mesa ni pagos por confirmar porque no hay nada seleccionado — esa
sección puede reemplazarse por un placeholder informativo; segundo, en el panel de pedido del
mostrador quiero quitar esos mensajes y dejar solamente un botón fijo que permita crear una nueva
orden; y tercero, las tarjetas de las mesas — su única responsabilidad será mostrar el pedido
cuando se seleccione; las tarjetas con mesas disponibles no permitirán navegar al formulario de
nueva orden como se venía haciendo."

## Clarifications

### Session 2026-08-27

- Q: ¿Qué debe pasar en el panel central al hacer clic en una mesa libre, si ya no navega a
  ningún formulario? → A: Solo informativo — sin volver a armar ningún pedido embebido ahí.
- Q: ¿A dónde navega el nuevo botón fijo "crear nueva orden" del panel de mostrador? → A: Al
  formulario dedicado ya existente para crear un pedido manual (no a un pedido de mostrador sin
  mesa asociada — eso exigiría un cambio de modelo de datos mayor, fuera de alcance).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Un solo placeholder cuando no hay nada seleccionado (Priority: P1)

El cajero abre la Terminal de Mesas sin haber seleccionado ninguna mesa todavía. Hoy el panel
central muestra a la vez un placeholder ("Selecciona una mesa...") y, debajo o junto a él, la
sección global "Pagos por confirmar" — ambas cosas visibles aunque no hay nada seleccionado,
duplicando la explicación de qué hacer.

**Why this priority**: es lo primero que ve el cajero al abrir la pantalla; dos mensajes
compitiendo por el mismo espacio, sin que ninguno esté vinculado a una selección real, genera
confusión sobre cuál seguir.

**Independent Test**: se puede probar completamente abriendo la Terminal de Mesas sin seleccionar
ninguna mesa y verificando que se ve un único mensaje informativo, sin ninguna sección de "Pagos
por confirmar" en paralelo.

**Acceptance Scenarios**:

1. **Given** la Terminal de Mesas recién abierta, sin ninguna mesa seleccionada, **When** el
   cajero mira el panel central, **Then** ve un único placeholder informativo — no coexisten un
   mensaje genérico y la sección "Pagos por confirmar".
2. **Given** ese mismo estado, **When** el cajero necesita encontrar un pago pendiente de
   confirmar, **Then** el placeholder lo orienta hacia el filtro "Pendientes" ya existente sobre
   la grilla de mesas (spec 036), que sigue funcionando sin cambios.

---

### User Story 2 - Un solo botón fijo en el panel de mostrador (Priority: P1)

El cajero mira el panel derecho ("Pedido de mostrador") sin tener ningún pedido seleccionado.
Hoy ese panel muestra varios mensajes explicativos en lugar de una acción directa. El cajero
necesita una única acción clara: crear un pedido nuevo.

**Why this priority**: mismo problema de ruido informativo que la Historia 1, sobre el panel que
sí necesita ofrecer una acción (crear pedido), no solo texto.

**Independent Test**: se puede probar completamente abriendo la Terminal de Mesas sin ningún
pedido seleccionado y verificando que el panel de mostrador muestra un único botón fijo, que al
pulsarlo navega al formulario de creación de pedido manual con una mesa preseleccionada.

**Acceptance Scenarios**:

1. **Given** ningún pedido seleccionado, **When** el cajero mira el panel de mostrador, **Then**
   ve un único botón fijo para crear una orden nueva — no los mensajes previos.
2. **Given** una mesa libre ya seleccionada (Historia 3), **When** el cajero pulsa ese botón,
   **Then** llega al formulario dedicado de pedido manual con esa misma mesa.
3. **Given** ninguna mesa seleccionada, **When** el cajero pulsa ese botón, **Then** llega al
   mismo formulario con la primera mesa libre disponible.

---

### User Story 3 - Las tarjetas de mesa solo seleccionan, nunca navegan (Priority: P1)

El cajero hace clic en una tarjeta de mesa libre. Hoy eso navega de inmediato a un formulario de
nueva orden, saltándose la Terminal de Mesas — inconsistente con hacer clic en una mesa ocupada,
que sí se queda en la pantalla y solo cambia lo que se ve en el panel central.

**Why this priority**: es el cambio de comportamiento más visible de los tres — unifica la
responsabilidad de toda tarjeta de mesa (mostrar/seleccionar, nunca navegar), evitando que el
cajero pierda su lugar en la Terminal sin quererlo.

**Independent Test**: se puede probar completamente haciendo clic en una tarjeta de mesa libre y
verificando que la pantalla no navega a ningún otro lugar — solo selecciona esa mesa y el panel
central pasa a mostrar su estado informativo (Historia 1 aplicada a "mesa libre seleccionada").

**Acceptance Scenarios**:

1. **Given** la grilla de mesas con al menos una mesa libre, **When** el cajero hace clic en su
   tarjeta, **Then** la pantalla no navega — la tarjeta queda marcada como seleccionada y el panel
   central muestra un estado informativo de "mesa libre" (sin abrir ningún armado de pedido ahí).
2. **Given** una mesa ocupada o con pedidos, **When** el cajero hace clic en su tarjeta,
   **Then** el comportamiento no cambia respecto a hoy — el panel central muestra su pedido.
3. **Given** una mesa libre seleccionada (estado informativo), **When** el cajero necesita crear
   un pedido para esa mesa, **Then** lo hace desde el botón fijo del panel de mostrador (Historia
   2) o con la tecla F3 — ambos ya vinculados a la mesa seleccionada.

---

### Edge Cases

- **Ninguna mesa libre disponible en absoluto**: el botón fijo del panel de mostrador (Historia
  2) no tiene ninguna mesa a la cual navegar — queda deshabilitado en ese caso.
- **Cambiar de una mesa libre seleccionada a otra**: sigue siendo una simple reselección (como
  hoy con cualquier tarjeta) — no dispara ninguna navegación.
- **Filtro "Pendientes" con resultados**: sigue funcionando exactamente igual que hoy (spec 036);
  esta spec no le cambia ningún comportamiento, solo dejó de haber una sección global duplicada
  fuera de él.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Sin ninguna mesa ni pedido seleccionado, el panel central de la Terminal de Mesas
  DEBE mostrar un único mensaje informativo — nunca, a la vez, ese mensaje y la sección "Pagos por
  confirmar" (amend spec 036 FR-004: esa sección global se retira; el pago pendiente se sigue
  encontrando por mesa, vía el filtro "Pendientes" ya existente).
- **FR-002**: Sin ningún pedido seleccionado, el panel "Pedido de mostrador" DEBE mostrar un único
  botón fijo para crear una orden nueva, en lugar de los mensajes informativos previos.
- **FR-003**: El botón de FR-002 DEBE navegar al formulario dedicado de creación de pedido manual
  ya existente, usando la mesa ya seleccionada si la hay, o la primera mesa libre disponible si
  no la hay.
- **FR-004**: Si ninguna mesa libre existe (ni seleccionada ni disponible), el botón de FR-002
  DEBE quedar deshabilitado en vez de navegar a un destino inválido.
- **FR-005**: Hacer clic en la tarjeta de una mesa libre DEBE únicamente seleccionarla — NO DEBE
  navegar a ningún formulario ni pantalla distinta de la Terminal de Mesas (amend spec 036
  FR-005: deja de abrir el "constructor de orden manual" embebido para este caso).
  Para una mesa ocupada o con pedidos, el comportamiento de FR-005 de spec 036 no cambia.
- **FR-006**: Con una mesa libre seleccionada (FR-005) y sin ningún pedido en curso, el panel
  central DEBE mostrar un estado informativo de "mesa libre" — sin ofrecer ningún armado de
  pedido embebido en esa misma pantalla.
- **FR-007**: La tecla de atajo ya existente para abrir el formulario de pedido manual (F3) DEBE
  seguir funcionando sin cambios sobre la mesa ya seleccionada, como vía equivalente al botón de
  FR-002/FR-003.

### Key Entities *(include if feature involves data)*

Esta spec no agrega ni modifica entidades de datos — reorganiza exclusivamente la navegación y el
contenido informativo de una pantalla ya existente (Terminal de Mesas, spec 036). El formulario
dedicado de creación de pedido manual, al que ahora se llega solo por el botón fijo o F3, no
cambia en sí mismo.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Sin nada seleccionado, el panel central muestra exactamente un mensaje informativo
  — cero casos donde se ven dos secciones explicativas a la vez.
- **SC-002**: Sin ningún pedido seleccionado, el panel de mostrador ofrece exactamente una acción
  (el botón fijo) — cero mensajes informativos adicionales en ese estado.
- **SC-003**: El 100% de los clics en una tarjeta de mesa libre resultan en selección, no en
  navegación — cero cambios de URL/pantalla disparados desde esa tarjeta.
- **SC-004**: El cajero puede llegar al formulario de pedido manual, con la mesa correcta
  preseleccionada, en un máximo de un clic adicional (botón fijo o F3) desde cualquier estado de
  la Terminal de Mesas.

## Out of Scope

- **Cualquier cambio al formulario dedicado de creación de pedido manual en sí** (sus campos, su
  propio selector de mesa, su flujo de guardado) — sigue exactamente igual; esta spec solo cambia
  desde dónde se llega a él.
- **Un pedido de mostrador sin mesa asociada** (compra sin sentarse) — descartado explícitamente
  en Clarifications; exigiría un cambio de modelo de datos mayor, fuera de alcance.
- **El flujo por-mesa de "Pagos por confirmar"** (spec 044, cuando sí hay una mesa con pago
  pendiente seleccionada) — sin cambios; esta spec retira únicamente la sección **global** que se
  veía sin nada seleccionado.
- **Las pestañas "Domicilios"/"Para llevar"** y su filtro de ocupación (spec 036) — sin cambios.

## Assumptions

- **La "primera mesa libre disponible" (FR-003) usa el mismo orden ya visible en la grilla de
  mesas** — no se define un criterio de orden nuevo; es una conveniencia para no bloquear el
  botón cuando no hay ninguna mesa preseleccionada, no una decisión de negocio sobre qué mesa
  "corresponde" asignar.
- **El código embebido que quedó sin ningún punto de entrada tras este cambio (el "constructor de
  orden manual" en línea, y la sección global "Pagos por confirmar") se elimina del código fuente**
  — no se documenta como una superficie viva alternativa; su reemplazo es el formulario dedicado
  y el filtro "Pendientes" citados en FR-001/FR-003, ambos ya existentes y sin cambios.
- **Implementación ya completada y verificada** (frontend: suite completa sin regresiones sobre
  la línea base ya conocida) al momento de escribir esta spec — se documenta después de
  implementar, siguiendo lo pedido explícitamente por el usuario.
