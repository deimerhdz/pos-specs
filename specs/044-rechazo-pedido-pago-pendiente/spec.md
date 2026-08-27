# Feature Specification: Rechazo de Pedido con Pago Pendiente y Corrección de Selección Obsoleta

**Feature Branch**: `044-rechazo-pedido-pago-pendiente`

**Created**: 2026-08-27

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección**, no una funcionalidad nueva desde cero.
Igual que las specs [019](../019-correccion-cuenta-mesas-fusionadas/spec.md),
[020](../020-correccion-validacion-opciones-mesero/spec.md),
[021](../021-correccion-orden-borrado-imagen-r2/spec.md) y
[041](../041-correccion-bugs-menu-qr/spec.md), cita nombres de archivo y línea del código actual
(`pos-heladeria` y `pos-backend`) porque son el contrato observable que se está corrigiendo, no
una fuga de detalles de implementación. Agrupa dos correcciones independientes de la pantalla
"Pagos por confirmar" de la Terminal de Mesas (spec [036](../036-terminal-mesas-rediseno-layout/spec.md)),
reportadas juntas por el mismo motivo (la misma pantalla) y verificables por separado:

1. **Decisión de negocio**: revierte, para pago en efectivo y para transferencia sin comprobante
   subido aún, la Decisión D5 de la spec [028](../028-terminal-mesas-modo-hibrido/spec.md)
   (`research.md`), que había retirado a propósito la posibilidad de rechazar un pedido completo
   desde esta pantalla, dejando "Rechazar" con un único significado (el del intento de pago, con
   reintento). Para transferencia con comprobante ya subido, esa decisión sigue vigente sin
   ningún cambio — conserva su "Rechazar" de siempre.
2. **Corrección de bug**: sin relación con ninguna decisión de negocio previa — un pedido recién
   confirmado/aprobado no se veía de inmediato en el panel (quedaba en el estado vacío "Pedido
   nuevo sin guardar") hasta que el cajero volvía a tocar la tarjeta de la mesa.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-27, junto con las dos decisiones de alcance resueltas
en la sección de Clarifications.

**Input**: User description: "cuando tengo un pago pendiente por confirmar deberia tener la opcion
de rechazar la orden y agregar un motivo, actualmente, no sale esa opcion y lo otro es que cuando
confirmo el pago, se queda asi como en la imagen y el pedido aceptado no se muestra de immediato".
Adjunta una captura de la Terminal de Mesas mostrando una mesa "En preparación" cuyo panel de
pedido aparece vacío ("Pedido nuevo sin guardar", "Aún no hay productos en este pedido").

## Clarifications

### Session 2026-08-27

- Q: ¿Qué debe pasar cuando el cajero "rechaza" un pedido con pago en efectivo pendiente? → A:
  Debe rechazar el pedido completo **y** el intento de pago, con motivo obligatorio — no un
  rechazo que permita reintentar, como sí ocurre hoy con la transferencia con comprobante.
- Q: ¿Esta nueva capacidad de rechazo debe llegar solo a los pagos en efectivo, o también a las
  transferencias que aún no subieron comprobante? → A: A ambos casos — efectivo y transferencia
  sin comprobante todavía. La transferencia con comprobante ya subido conserva su "Rechazar"
  actual (rechaza el intento, permite reintentar), sin ningún cambio.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Rechazar un pedido con pago en efectivo pendiente (Priority: P1)

Un comensal envía su pedido por QR indicando que pagará en efectivo. El cajero, antes de
confirmar el efectivo recibido, determina que el pedido no debe seguir adelante (el comensal se
fue sin pagar, faltó un insumo, o cualquier otro motivo operativo). Hoy el panel de "Pagos por
confirmar" solo ofrece "Confirmar efectivo" — ninguna forma de rechazarlo; el backend además
rechaza explícitamente cualquier intento de rechazar un intento de pago en efectivo. El cajero
necesita poder rechazar el pedido completo, dejando constancia del motivo.

**Why this priority**: sin esta opción, un pedido en efectivo que no debe seguir adelante queda
indefinidamente "pendiente de revisión" en la pantalla, sin ninguna salida operativa desde ahí.

**Independent Test**: se puede probar completamente abriendo el panel de un pedido con un intento
de pago en efectivo pendiente, pulsando "Rechazar pedido", escribiendo un motivo y confirmando —
verificando que el pedido pasa a `cancelada`, el intento de pago queda `rechazado` con ese motivo,
y el pedido desaparece de "Pagos por confirmar".

**Acceptance Scenarios**:

1. **Given** un pedido con un intento de pago en efectivo `pendiente`, **When** el cajero pulsa
   "Rechazar pedido", escribe un motivo y confirma, **Then** el pedido queda `cancelada`, el
   intento de pago queda `rechazado` con ese motivo, y ambos registran quién y cuándo lo rechazó.
2. **Given** el mismo panel, **When** el cajero intenta confirmar el rechazo sin escribir ningún
   motivo, **Then** el sistema lo impide — el motivo es obligatorio.
3. **Given** un pedido ya rechazado, **When** se consulta la Terminal de Mesas, **Then** ese
   pedido ya no aparece en "Pagos por confirmar" (ni en la tarjeta de la mesa, ni en el listado
   global).

---

### User Story 2 - Rechazar un pedido con transferencia pendiente sin comprobante aún (Priority: P1)

Un comensal indica que pagará por transferencia, pero todavía no sube ningún comprobante. Hoy el
panel solo muestra "Esperando que el comensal suba el comprobante…", sin ninguna acción
disponible. El cajero necesita la misma salida que para efectivo: rechazar el pedido completo con
motivo, sin tener que esperar indefinidamente a que el comensal suba algo.

**Why this priority**: mismo problema operativo que la Historia 1, sobre el otro método de pago
que hoy tampoco ofrece ninguna acción mientras no hay comprobante.

**Independent Test**: se puede probar completamente abriendo el panel de un pedido con un intento
de pago por transferencia `pendiente` sin `receipt_file_url`, pulsando "Rechazar pedido",
escribiendo un motivo y confirmando — mismas verificaciones que la Historia 1.

**Acceptance Scenarios**:

1. **Given** un pedido con un intento de pago por transferencia `pendiente` sin comprobante
   subido, **When** el cajero rechaza el pedido con motivo, **Then** el pedido queda `cancelada`
   y el intento de pago queda `rechazado` con ese motivo — mismo resultado que la Historia 1.
2. **Given** un pedido con un intento de pago por transferencia `pendiente` **con** comprobante ya
   subido, **When** el cajero abre el panel, **Then** **no** ve la opción "Rechazar pedido" — solo
   "Aprobar" y "Rechazar" a nivel de intento (comportamiento actual, sin cambios): ese rechazo
   deja el pedido pendiente para que el comensal reintente con un comprobante distinto.

---

### User Story 3 - El pedido recién confirmado se ve de inmediato (Priority: P1)

Un cajero aprueba un comprobante de transferencia o confirma un pago en efectivo. El pedido debe
pasar a verse de inmediato en el panel de la mesa (con sus productos, listo para enviarse a
cocina si corresponde), sin que el cajero tenga que volver a tocar la tarjeta de la mesa para que
aparezca.

**Why this priority**: es el mismo flujo de todos los días — confirmar un pago es una acción
constante en la Terminal de Mesas; que el panel se vea vacío justo después rompe la confianza en
que la acción funcionó, y le agrega un paso manual innecesario a cada cobro.

**Independent Test**: se puede probar completamente abriendo una mesa cuyo único pedido está
pendiente de confirmación, aprobando/confirmando su pago, y verificando que el panel de la mesa
muestra ese pedido (con sus productos) de inmediato, sin volver a hacer clic en la tarjeta.

**Acceptance Scenarios**:

1. **Given** una mesa con un único pedido `recibida` por QR (pago pendiente) ya seleccionada,
   **When** el cajero aprueba/confirma su pago, **Then** el panel de la mesa muestra ese pedido de
   inmediato — con sus productos, sin el estado "Pedido nuevo sin guardar".
2. **Given** una mesa con varios pedidos activos donde el cajero ya eligió uno manualmente entre
   varias pestañas, **When** se confirma el pago de **otro** pedido de esa misma mesa, **Then** la
   selección manual del cajero no cambia — solo se reajusta automáticamente cuando la que estaba
   seleccionada ya no es válida.

---

### Edge Cases

- **Pedido de mostrador/mesero sin ningún intento de pago QR**: rechazarlo (`cancelar pedido`,
  flujo ya existente en el panel de cobro) sigue funcionando exactamente igual que hoy — no hay
  ningún intento de pago que resolver.
- **Un intento de pago ya `confirmado`/`rechazado` en el momento del rechazo del pedido**: no se
  toca — solo se resuelve el intento `pendiente` (a lo sumo uno por pedido).
- **Liberación de la mesa**: rechazar un pedido no libera la mesa por sí solo — puede haber otros
  pedidos activos en la misma sesión; la liberación sigue siendo una acción explícita aparte
  ("Liberar Mesa"), sin cambios.
- **Doble confirmación del rechazo**: el motivo escrito se limpia y el formulario de rechazo se
  cierra tras un rechazo exitoso, evitando un reenvío accidental del mismo pedido ya cancelado.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE permitir rechazar un pedido completo (no solo el intento de pago)
  cuando su intento de pago pendiente es en efectivo, con un motivo obligatorio.
- **FR-002**: El sistema DEBE permitir rechazar un pedido completo cuando su intento de pago
  pendiente es una transferencia **sin** comprobante subido todavía, con un motivo obligatorio.
- **FR-003**: Rechazar un pedido por FR-001/FR-002 DEBE, en la misma operación: marcar el pedido
  como cancelado, y marcar el intento de pago `pendiente` de ese pedido como rechazado con el
  mismo motivo — ninguno de los dos queda en un estado a medias.
- **FR-004**: El motivo del rechazo (FR-001/FR-002) DEBE ser obligatorio — el sistema no debe
  permitir confirmar el rechazo con un motivo vacío.
- **FR-005**: El "Rechazar" que ya existe hoy para una transferencia con comprobante ya subido
  (rechaza el intento de pago, deja el pedido pendiente para reintentar) NO DEBE cambiar de
  comportamiento — sigue siendo exclusivamente sobre el intento de pago, sin la opción de
  rechazar el pedido completo desde ese mismo caso.
- **FR-006**: Tras aprobar un comprobante de transferencia o confirmar un pago en efectivo, el
  sistema DEBE reflejar de inmediato, en el panel de la mesa correspondiente, el pedido recién
  confirmado (con sus productos) — sin requerir que el cajero vuelva a seleccionar la tarjeta de
  la mesa para verlo.
- **FR-007**: FR-006 NO DEBE alterar la selección de pedido de una mesa cuando el pedido ya
  seleccionado por el cajero sigue siendo válido tras la confirmación (p. ej. una mesa con varios
  pedidos activos donde ya se había elegido uno manualmente).

### Key Entities *(include if feature involves data)*

- **Pedido (`CustomerOrder`)**: entidad ya existente. Esta spec no le agrega atributos; usa su
  transición ya existente a `cancelada` (`POST /orders/{id}/cancel`, spec 029) también para el
  caso de un pago QR todavía pendiente.
- **Intento de pago (`OrderPaymentAttempt`, spec 024)**: entidad ya existente, con sus estados
  `pendiente`/`confirmado`/`rechazado` ya definidos. Esta spec extiende **cuándo** un intento
  queda `rechazado` (también al rechazar el pedido completo que lo contiene), no su forma.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un cajero puede rechazar un pedido con pago en efectivo pendiente, con motivo, en
  menos de 15 segundos, sin salir de la pantalla de "Pagos por confirmar".
- **SC-002**: Lo mismo para un pedido con transferencia pendiente sin comprobante.
- **SC-003**: El 100% de los pedidos rechazados por esta vía quedan con su intento de pago
  también resuelto (`rechazado`, mismo motivo) — nunca un intento `pendiente` huérfano sobre un
  pedido ya `cancelada`.
- **SC-004**: El 100% de las confirmaciones/aprobaciones de pago reflejan el pedido en el panel
  de la mesa de inmediato, sin ninguna interacción manual adicional del cajero.
- **SC-005**: El "Rechazar" de transferencia-con-comprobante sigue funcionando exactamente igual
  que antes de esta spec — cero regresión observable en ese caso.

## Out of Scope

- **Cualquier cambio al "Rechazar" de transferencia con comprobante ya subido** (rechaza el
  intento, permite reintentar) — se mantiene intacto, spec 024/028.
- **Liberar la mesa automáticamente al rechazar un pedido** — sigue siendo una acción explícita
  aparte, sin cambios.
- **El flujo de "Rechazar pedido" que ya existe en el panel de cobro** (mostrador/sesión, motivo
  fijo) — pantalla distinta, sin cambios.
- **Cualquier otro contenido de la spec 036** (layout de la Terminal de Mesas) fuera de la acción
  de rechazo agregada aquí — esta spec no reabre esa spec.

## Assumptions

- **El pedido rechazado no se reintenta**: a diferencia del rechazo de intento de transferencia
  (que sí permite reintentar con otro comprobante), rechazar el pedido completo (FR-001/FR-002)
  lo termina — si el comensal quiere pedir de nuevo, envía un pedido nuevo.
- **El botón "Rechazar pedido" vive dentro del mismo panel embebido** (`payment-attempt-review-panel`)
  que ya usan tanto la tarjeta por mesa como el listado global de "Pagos por confirmar" (spec
  036) — un solo cambio de componente cubre ambas superficies, sin wiring adicional en ninguna
  de las dos.
- **FR-006/FR-007 no dependen de ningún cambio de contrato del backend** — es una corrección de
  sincronización de estado en el frontend (`PosTerminalStore`); el backend ya devuelve el pedido
  con su estado actualizado tras confirmar/aprobar.
