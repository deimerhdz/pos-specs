# Feature Specification: Eliminación de Dividir Cuenta y de Combinar Método de Pago en Toda la Aplicación

**Feature Branch**: `046-dividir-cuenta-pago-pendiente`

**Created**: 2026-08-28

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección**, no una funcionalidad nueva desde cero. Igual
que las specs [044](../044-rechazo-pedido-pago-pendiente/spec.md) y
[045](../045-simplificacion-terminal-mesas/spec.md), reorganiza el comportamiento de pantallas ya
existentes (Terminal de Mesas, spec [036](../036-terminal-mesas-rediseno-layout/spec.md); panel
"Cuenta de la mesa", spec [026](../026-mejoras-ux-comanda/spec.md)), y **amend explícitamente**
requisitos funcionales de esas specs — nunca editando in-place una spec ya implementada:

- **FR-010** de spec 036 (el panel derecho "Pedido de mostrador" conserva siempre, sin condición,
  el botón "Dividir la cuenta entre varias personas") — ese botón se **elimina por completo** (no
  solo se oculta ni se condiciona); ver Historia 1 e Historia 2.
- **FR-007** de spec 026 (el panel "Cuenta de la mesa" permite dividir la cuenta asignando cada
  ítem/unidad a un comensal) — se elimina por el mismo motivo; ver Historia 2.
- **FR-008** de spec 026 (cobrar la cuenta, desde el panel "Cuenta de la mesa", combinando varios
  métodos de pago en un mismo cobro — spec 011) — **también se elimina** en esta ronda: cada cobro
  DEBE hacerse con un único método, por el total exacto o más; ver Historia 3. Esto revierte lo que
  la ronda de Clarifications anterior de esta misma spec 046 había decidido (conservar FR-008 sin
  cambios y exponer "Combinar método de pago" también en el bloque de confirmación de pago
  pendiente) — esa decisión queda sin efecto.
- **FR-016** de spec 028 (acción manual explícita "Cerrar Mesa" / "Liberar Mesa") — se condiciona a
  que el pedido seleccionado no tenga ningún pago pendiente de confirmar; ver Historia 1. Es el
  mismo botón ya documentado con ambos nombres en spec 028 — esta spec no introduce una acción
  nueva, solo normaliza que "Liberar Mesa" y "Cerrar Mesa" son un único control.
- La funcionalidad original de reparto por comensal (spec [010](../010-sesion-mesa-reparto-cierre-barrido/spec.md),
  FR-005) y la funcionalidad original de combinar métodos de pago en un mismo cobro
  (spec [011](../011-venta-mostrador-constructor-factura/spec.md)) quedan **eliminadas** (no solo
  deprecadas/ocultas) como capacidad de negocio en toda la aplicación — decisión explícita del
  dueño del producto, escalada en dos rondas sucesivas de Clarifications el 2026-08-28,
  documentada en Clarifications y en Assumptions, no cuestionada por esta spec.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-28, incluyendo la decisión explícita de eliminar tanto
"Dividir la cuenta entre varias personas" como "Combinar método de pago" en toda la aplicación
(sesión de Clarifications, escalada en tres rondas sucesivas el mismo día: primero reubicar,
luego eliminar dividir cuenta, luego eliminar también combinar método de pago). Es reordenamiento
de navegación/UI y retiro de dos funcionalidades ya implementadas sobre pantallas ya existentes,
sin reabrir ninguna regla de negocio de precio, inventario o facturación — no aplica una nueva
entrada en `registro-de-anomalias.md`.

**Input**: User description (dos mensajes sucesivos del dueño del producto). Primero (incluye una
captura de la Terminal de Mesas con la mesa 3 seleccionada en estado "Por confirmar"): "cuando
tengo un pedido por confirmar el pago en efectivo o en cualquier otro medio de pago, no deberían
mostrarse las opciones de 'Dividir la cuenta' ni 'Liberar Mesa' en el panel de mostrador, sino
hasta que el cajero acepte o valide el pago; la opción de dividir la cuenta entre varias personas
debería estar junto con las opciones de confirmación del pago porque así tendría más sentido."
Segundo, durante `/speckit-clarify`, corrigiendo el enfoque del primero: "la opción de dividir
cuenta queda deprecado, se agrega la opción de combinar método de pago que creo que ya existe para
que la persona pague una parte en efectivo y la combine con transferencia por ejemplo, y así mismo
aplica para todas las confirmaciones — ejemplo, si se recibe el comprobante, debe tener la opción
de combinar con otra opción de pago." Tercero, en una nueva sesión de `/speckit-clarify` el mismo
día, escalando el alcance del segundo mensaje: "cambio de planes: depreca y elimina todo lo
relacionado con la opción de dividir la cuenta; el botón de cerrar mesa solo debe aparecer después
de que el cajero haya confirmado o validado los pagos de la orden." Cuarto, en una tercera sesión
de `/speckit-clarify` el mismo día, escalando aún más el alcance: "depreca también la opción de
combinar con otro método de pago."

## Clarifications

### Session 2026-08-28

- Q: Cuando "dividir cuenta" queda deprecado, ¿aplica en toda la aplicación o solo dentro del
  alcance de esta spec (panel de mostrador durante pago pendiente)? → A: En toda la aplicación —
  "Dividir la cuenta entre varias personas" (spec 010) se retira también del panel de cobro normal
  (spec 036 FR-010), no solo del panel de confirmación de pago pendiente.
- Q: ¿Cuándo debe aparecer "combinar método de pago" durante la confirmación de un pago
  pendiente — siempre, o solo cuando el monto recibido no cubre el total? → A: Siempre disponible
  — aparece en todo pago pendiente de confirmar (efectivo o transferencia), cubra o no el total,
  para que el cajero pueda añadir otro método aunque el monto ya alcance.
- Q: ¿"Combinar método de pago" en la confirmación reutiliza el mismo mecanismo ya implementado
  (spec 011, múltiples métodos por venta, cambio solo desde el excedente en efectivo), o necesita
  un comportamiento adaptado a este contexto? → A: Reutiliza spec 011 tal cual — mismo
  componente/lógica, invocado también desde el bloque de confirmación de pago pendiente, sin
  ningún cambio de comportamiento.

### Session 2026-08-28 (continuación — cambio de planes)

- Q: ¿"Dividir la cuenta" debe quedar solo oculta/deprecada, o su código y todos sus puntos de
  entrada deben eliminarse por completo? → A: Eliminación completa — no solo se oculta el botón
  (FR-005 de la sesión anterior), se retira toda vía de acceso a esa función y el código que ya no
  tiene ningún punto de entrada válido (amplía FR-005 a un nuevo FR-006).
- Q: ¿El "botón de cerrar mesa" que solo debe aparecer tras confirmar/validar el pago es una acción
  nueva o el mismo "Liberar Mesa" ya cubierto por FR-001/FR-002? → A: Es el mismo botón — spec 028
  (FR-016) ya lo documenta con ambos nombres, "Cerrar Mesa" / "Liberar Mesa"; no hay ninguna acción
  nueva que agregar, solo se normaliza el nombre en esta spec (sin cambiar FR-001/FR-002, que ya
  implementan exactamente esa condición).

### Session 2026-08-28 (tercera ronda — "Combinar método de pago" también eliminado)

- Q: ¿"Depreca también la opción de combinar con otro método de pago" se refiere solo al punto de
  entrada agregado en la ronda anterior (bloque de confirmación de pago pendiente), o a "Combinar
  método de pago" (spec 011) en toda la aplicación, incluyendo el cobro normal desde "Cuenta de la
  mesa" (spec 026, FR-008)? → A: Toda la aplicación — se elimina también de "Cuenta de la mesa"
  (spec 026, FR-008), no solo del bloque de confirmación. **Esto sustituye y deja sin efecto** las
  dos respuestas de la primera ronda sobre "Combinar método de pago" (cuándo aparece en la
  confirmación, y que reutilizaba spec 011 tal cual) — ya no aplica en ningún lugar.
- Q: Con "combinar método de pago" también eliminado, ¿qué debe pasar cuando el monto pagado (en
  efectivo o por transferencia) no cubre el total del pedido? → A: Se rechaza hasta cubrir el
  total — el cajero rechaza el pedido (spec 044) o espera a que el cliente complete el monto exacto
  con ese mismo método antes de confirmar; cada pedido se cobra con un único método, por el total
  exacto o más.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ocultar "Liberar Mesa" / "Cerrar Mesa" del panel de mostrador mientras el pago está pendiente (Priority: P1)

El cajero selecciona una mesa cuyo pedido tiene un pago pendiente de confirmar (efectivo entregado
a la espera de contar/validar, o transferencia a la espera de aprobar el comprobante). Hoy el panel
"Pedido de mostrador" sigue mostrando "Liberar Mesa" (el mismo botón documentado en spec 028,
FR-016, como "Cerrar Mesa" / "Liberar Mesa") como si el pago ya estuviera resuelto, permitiendo
liberar la mesa antes de validar que el dinero/comprobante efectivamente corresponde.

**Why this priority**: es el riesgo operativo más directo del reporte original — liberar una mesa
antes de confirmar que el pago realmente llegó puede hacer perder el rastro de un cobro sin
validar.

**Independent Test**: se puede probar completamente seleccionando una mesa con un pago en efectivo
o transferencia pendiente de confirmar y verificando que el panel de mostrador no muestra "Liberar
Mesa" mientras ese pago siga pendiente.

**Acceptance Scenarios**:

1. **Given** una mesa seleccionada con un pedido cuyo pago en efectivo está "Pendiente de
   revisión", **When** el cajero mira el panel "Pedido de mostrador", **Then** no ve el botón
   "Liberar Mesa".
2. **Given** el mismo estado, pero con un pago por transferencia pendiente de aprobar el
   comprobante (con o sin comprobante ya subido), **When** el cajero mira ese mismo panel,
   **Then** tampoco ve "Liberar Mesa" — mismo tratamiento que el efectivo pendiente.
3. **Given** una mesa cuyo pedido **no** tiene ningún pago pendiente de confirmar (ya cobrado, o
   aún sin ningún intento de pago registrado), **When** el cajero la selecciona, **Then** el panel
   de mostrador muestra "Liberar Mesa" con el comportamiento de siempre.
4. **Given** un pago pendiente ya visible con "Liberar Mesa" oculto, **When** el cajero confirma el
   efectivo o aprueba el comprobante, **Then** "Liberar Mesa" reaparece de inmediato en el panel de
   mostrador, sin necesidad de volver a seleccionar la mesa.

---

### User Story 2 - "Dividir la cuenta entre varias personas" eliminada por completo de la aplicación (Priority: P2)

El botón y el flujo de "Dividir la cuenta entre varias personas" (reparto de ítems/unidades por
comensal, spec 010) se eliminan de la aplicación — no solo se ocultan. Ningún panel (ni "Pedido de
mostrador", spec 036; ni "Cuenta de la mesa", spec 026) vuelve a ofrecer esa opción, y ninguna otra
vía ya implementada para llegar a ella (atajos, menús, rutas directas) sigue funcionando.

**Why this priority**: es una decisión de limpieza/consistencia de negocio, no un riesgo operativo
inmediato como la Historia 1 — puede completarse después de que esa ya esté resuelta, sin bloquear
el valor que entrega.

**Independent Test**: se puede probar completamente recorriendo el panel "Pedido de mostrador" y el
panel "Cuenta de la mesa", intentando cualquier vía ya conocida para llegar al reparto por
comensal, y verificando que ninguna funciona — el botón no aparece y ninguna ruta directa ni atajo
abre ese flujo.

**Acceptance Scenarios**:

1. **Given** cualquier pedido seleccionado sin pago pendiente, **When** el cajero mira el panel
   "Pedido de mostrador", **Then** no ve el botón "Dividir la cuenta entre varias personas" en
   ningún lugar del panel.
2. **Given** una mesa con consumo abierta desde el panel "Cuenta de la mesa" (spec 026), **When**
   el cajero busca la opción de dividir la cuenta por comensal, **Then** ya no existe — el cobro
   desde ese panel se hace por el total completo con un único método (Historia 3), sin ninguna
   forma de repartirlo por comensal.
3. **Given** cualquier vía de acceso ya implementada al reparto por comensal fuera de los dos
   botones anteriores (por ejemplo, una ruta directa o un atajo ya existente), **When** el cajero
   la intenta, **Then** ya no lleva a ningún flujo de reparto por comensal — el sistema no
   mantiene ningún punto de entrada vivo hacia esa función retirada.

---

### User Story 3 - "Combinar método de pago" eliminado por completo de la aplicación (Priority: P2)

La opción de cobrar un pedido combinando más de un método de pago en un mismo cobro (spec 011;
usada hoy desde el panel "Cuenta de la mesa", spec 026 FR-008) se elimina de la aplicación. Cada
pedido se cobra con un único método, por el total exacto o más — si el monto entregado en efectivo
o declarado por transferencia no cubre el total, el cajero rechaza el pedido (spec 044) en vez de
completarlo con un segundo método.

**Why this priority**: es una decisión de limpieza/consistencia de negocio, igual que la Historia
2 — puede completarse después de que la Historia 1 (el riesgo operativo directo) ya esté resuelta,
sin bloquear el valor que entrega.

**Independent Test**: se puede probar completamente intentando cobrar un pedido, o confirmar un
pago pendiente, con un monto que no cubre el total por un solo método, y verificando que el
sistema no ofrece ninguna forma de completarlo con un segundo método — solo rechazar o esperar el
monto exacto con el mismo método.

**Acceptance Scenarios**:

1. **Given** una mesa con consumo abierta desde el panel "Cuenta de la mesa", **When** el cajero
   busca la opción de cobrar combinando varios métodos de pago, **Then** ya no existe — solo puede
   cobrar el total con un único método.
2. **Given** un pago en efectivo pendiente de revisión, o una transferencia pendiente de aprobar,
   **When** el cajero mira el bloque de confirmación de ese pago, **Then** no ve ninguna opción de
   combinar método de pago — solo los controles de confirmación/rechazo ya existentes (spec 044).
3. **Given** un pago pendiente cuyo monto no cubre el total del pedido, **When** el cajero necesita
   resolverlo, **Then** solo puede rechazar el pedido (spec 044) o esperar a que se complete el
   monto exacto con el mismo método — no hay ninguna vía para combinarlo con otro método.
4. **Given** un pago en efectivo que supera el total del pedido, **When** el cajero lo confirma,
   **Then** el cálculo de cambio sobre ese excedente sigue funcionando igual que hoy — eliminar
   "combinar método de pago" no afecta el cambio de un pago hecho con un único método.

---

### Edge Cases

- **Pedido de mostrador/mesero sin ningún intento de pago pendiente** (ya cobrado directamente, o
  aún sin ningún pago registrado): "Liberar Mesa" sigue disponible en el panel de mostrador como
  hoy (spec 036); el cobro se hace con un único método por el total exacto o más (Historia 3), sin
  ninguna opción de combinar métodos.
- **Mesa con varios pedidos activos, uno con pago pendiente y otro ya cobrado**: la condición de
  FR-001/FR-002 aplica al pedido actualmente seleccionado en el panel (spec 036, FR-005) — cambiar
  de pestaña de pedido reevalúa qué controles se muestran, sin afectar a los demás pedidos de la
  misma mesa.
- **Rechazo del pedido con pago pendiente** (spec 044): al rechazarlo, el bloque de confirmación
  desaparece; no libera la mesa por sí solo, igual que hoy (spec 044, edge case "Liberación de la
  mesa") — "Liberar Mesa" sigue siendo una acción explícita aparte, disponible una vez ya no queda
  ningún pago pendiente en esa mesa.
- **Código que implementaba "Dividir la cuenta entre varias personas"** (spec 010: asignación por
  comensal, panel de reparto) **y "Combinar método de pago"** (spec 011: múltiples métodos por
  venta): DEBEN eliminarse del código fuente (FR-006/FR-007) — no quedan como superficie viva
  alternativa ni accesibles por ninguna vía (mismo criterio ya usado en spec 045 para código sin
  punto de entrada, aquí elevado a requisito explícito por decisión del dueño del producto).
- **Pago en efectivo con excedente (cambio)**: eliminar "combinar método de pago" no elimina el
  cálculo de cambio cuando el cliente paga de más con un único método en efectivo — ese cálculo
  sigue funcionando igual que hoy; lo que se elimina es solo la posibilidad de completar un pago
  insuficiente sumando un método distinto.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Mientras el pedido seleccionado tenga un intento de pago pendiente de confirmar
  (efectivo pendiente de revisión, o transferencia pendiente de aprobar, con o sin comprobante ya
  subido — mismo alcance que spec 036 FR-004 y spec 044), el panel "Pedido de mostrador" NO DEBE
  mostrar el botón "Liberar Mesa" (amend spec 036 FR-010 en su parte de "Liberar Mesa").
- **FR-002**: En cuanto el cajero confirma el efectivo o aprueba el comprobante del pago pendiente
  de FR-001, "Liberar Mesa" DEBE volver a estar disponible en el panel "Pedido de mostrador" de
  inmediato y sin requerir reseleccionar la mesa (reutilizando la actualización inmediata ya
  corregida en spec 044, FR-006).
- **FR-003**: El bloque de confirmación de todo pago pendiente de confirmar (efectivo pendiente de
  revisión, o transferencia pendiente de aprobar, con o sin comprobante ya subido) NO DEBE ofrecer
  ninguna opción de combinar método de pago — solo sus controles ya existentes (p. ej. "Confirmar
  efectivo"/"Rechazar pedido", o "Aprobar"/"Rechazar" del comprobante — spec 044). Si el monto
  declarado no cubre el total del pedido, el cajero DEBE rechazar el pedido (spec 044) en vez de
  completarlo con otro método.
- **FR-004**: El panel "Cuenta de la mesa" NO DEBE permitir cobrar combinando varios métodos de
  pago en un mismo cobro (amend spec 026 FR-008) — cada cobro DEBE hacerse con un único método,
  por el total exacto o más; el cambio (cuando el método es efectivo) solo puede provenir de un
  excedente pagado con ese mismo método.
- **FR-005**: El botón "Dividir la cuenta entre varias personas" (spec 010) NO DEBE mostrarse en
  ningún panel de la aplicación — se retira del panel "Pedido de mostrador" (amend spec 036
  FR-010) y del panel "Cuenta de la mesa" (amend spec 026 FR-007).
- **FR-006**: Más allá de ocultar el botón (FR-005), el sistema DEBE eliminar por completo la
  funcionalidad de "Dividir la cuenta entre varias personas" (spec 010, FR-005): ninguna otra vía
  ya implementada (atajos de teclado, rutas directas, menús) DEBE seguir llevando a ese flujo, y el
  código que la implementaba, al quedar sin ningún punto de entrada, DEBE eliminarse del código
  fuente — no permanece como superficie viva alternativa.
- **FR-007**: Más allá de retirar el punto de entrada de FR-003/FR-004, el sistema DEBE eliminar
  por completo la funcionalidad de "Combinar método de pago" (spec 011): ninguna otra vía ya
  implementada DEBE seguir permitiendo un cobro con más de un método, y el código que la
  implementaba, al quedar sin ningún punto de entrada, DEBE eliminarse del código fuente — no
  permanece como superficie viva alternativa. El cálculo de cambio para un pago hecho con un único
  método en efectivo (FR-003, edge case) NO se ve afectado.

### Key Entities *(include if feature involves data)*

Esta spec no agrega ni modifica entidades de datos — reorganiza qué controles se muestran y dónde
(Terminal de Mesas, spec 036; confirmación de pago pendiente, spec 044; panel "Cuenta de la mesa",
spec 026) y elimina por completo dos funcionalidades ya implementadas ("Dividir la cuenta entre
varias personas", spec 010; "Combinar método de pago", spec 011), incluyendo el código de ambas
que quede sin punto de entrada (FR-006/FR-007).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 0% de los pedidos con un pago pendiente de confirmar muestran "Liberar Mesa" en el
  panel de mostrador mientras ese pago siga pendiente.
- **SC-002**: 0% de los bloques de confirmación de pago pendiente (efectivo o transferencia) y 0%
  de los cobros desde el panel "Cuenta de la mesa" muestran alguna opción de combinar método de
  pago.
- **SC-003**: Tras confirmar o aprobar un pago pendiente, "Liberar Mesa" vuelve a estar disponible
  en el panel de mostrador en el mismo instante, sin ningún clic adicional de reselección de mesa.
- **SC-004**: 0% de las pantallas de la aplicación (panel de mostrador, panel "Cuenta de la mesa")
  muestran el botón "Dividir la cuenta entre varias personas", y ninguna vía alternativa ya
  existente (atajo, ruta directa, menú) sigue abriendo ese flujo.
- **SC-005**: Cero casos donde un cajero libera una mesa mientras el pago de ese pedido sigue
  pendiente de confirmar.
- **SC-006**: 100% de los pagos pendientes cuyo monto declarado no cubre el total del pedido
  terminan rechazados (spec 044) o completados con el monto exacto del mismo método — cero casos
  completados combinando un segundo método.

## Out of Scope

- **La lógica de confirmación de pago en sí** (Confirmar efectivo, Aprobar/Rechazar comprobante,
  Rechazar pedido completo — spec 044) — sin cambios de comportamiento propios; esta spec solo
  retira de ahí la opción de combinar método de pago (FR-003).
- **El cálculo de cambio sobre un pago hecho con un único método en efectivo** — sin cambios; lo
  que se elimina es solo la posibilidad de completar un pago insuficiente con un segundo método
  (FR-003/FR-004/FR-007).
- **El botón fijo "+ Crear pedido nuevo" y su condición de aparición** (spec 045, sin ningún
  pedido seleccionado) — sin cambios, es un estado distinto al de esta spec.
- **La sección global "Pagos por confirmar" ya retirada** (spec 045, cuando no hay nada
  seleccionado) — sin cambios, sigue retirada.

## Assumptions

- **"Pago pendiente de confirmar" cubre el mismo alcance ya usado en specs 036/044**: efectivo
  "Pendiente de revisión" y transferencia pendiente de aprobar (con o sin comprobante subido). No
  incluye pedidos ya cobrados ni pedidos sin ningún intento de pago registrado todavía.
- **Eliminar "Dividir la cuenta entre varias personas" y "Combinar método de pago" (no solo
  ocultarlas) es una decisión de negocio explícita del dueño del producto**, escalada en dos
  rondas sucesivas de Clarifications el mismo día: primero se decidió que "combinar método de
  pago" cubría, en la práctica, la necesidad original de "dividir cuenta"; después se decidió
  eliminar también "combinar método de pago". El resultado final es que ningún pedido puede
  pagarse repartido entre comensales ni con más de un método — cada pedido se cobra por el total
  exacto o más, con un único método; si no alcanza, se rechaza (spec 044). Esta spec documenta la
  decisión tal como la dio el dueño del producto, sin cuestionarla.
- **"Liberar Mesa" y "Cerrar Mesa" son el mismo botón** (spec 028, FR-016) — esta spec no
  introduce una acción nueva, solo condiciona ese botón único a que no haya pago pendiente
  (FR-001/FR-002); sigue siendo una acción manual y explícita una vez deja de haber pago
  pendiente, nunca automática al confirmar un pago (mismo criterio que spec 044).
- **La eliminación de código de FR-006/FR-007 no incluye migrar ni alterar datos históricos** ya
  generados por repartos por comensal o cobros combinados completados antes de esta spec — esta
  spec es de alcance UI/código, no de datos; no se documenta ninguna migración de base de datos.
