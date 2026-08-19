# Feature Specification: Rediseño UX de Confirmación de Pago y Comanda en Terminal de Mesas (Skeilopos)

**Feature Branch**: `026-mejoras-ux-comanda`

**Created**: 2026-08-18

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad **nueva** de experiencia de usuario (fase de evolución
funcional, Principio I de la [Constitución](../../.specify/memory/constitution.md)). No reabre las
reglas de negocio ya definidas en [spec 024](../024-pagos-ordenes-mesa/spec.md) (métodos de pago
configurables, aprobación/rechazo de comprobantes, cálculo del cambio) ni en
[spec 010](../010-sesion-mesa-reparto-cierre-barrido/spec.md) /
[spec 011](../011-venta-mostrador-constructor-factura/spec.md) (reparto de cuenta exclusivamente
por ítem/unidad asignada a un comensal, prohibición de división porcentual o "a partes iguales",
pagos con varios métodos en una misma venta) — las reutiliza y las expone con una interfaz más
clara para el personal. **Sí cambia** un comportamiento ya definido en spec 024 (FR-017: enviar a
comanda era una acción manual separada de confirmar el pago) — a partir de esta spec, confirmar el
pago (efectivo recibido, o comprobante de transferencia aprobado) envía automáticamente el pedido a
cocina y descuenta el inventario en la misma acción, sin un paso manual adicional; se autoriza aquí
explícitamente como cambio de comportamiento (Principio II).

**Input**: User description: a partir de una captura de la Terminal de Mesas, se reportan varios
problemas de experiencia de usuario del personal (cajero/mesero): (1) al confirmar un pago en
efectivo, el pedido queda en la pestaña "Por confirmar" sin pasar automáticamente a la mesa/cocina,
generando la impresión de que el flujo quedó incompleto; (2) el cambio a entregar no se muestra en
ningún momento al confirmar un pago en efectivo; (3) se pide revisar y mejorar la forma en que se
genera la factura, se divide la cuenta y se cobra con varios métodos de pago; (4) se pide
modernizar visualmente la comanda completa, incluyendo el tamaño de los textos, para el personal
del restaurante; y se solicitan varias propuestas de UX antes de definir el diseño final.

## Clarifications

### Session 2026-08-18

- Q: La captura muestra que, tras confirmar el pago en efectivo, el pedido queda en "Por
  confirmar" esperando una segunda acción manual ("Confirmar") que lo envía a cocina — esto es hoy
  una regla de negocio explícita (spec 024, FR-017: enviar a comanda solo con pago confirmado, como
  acción separada). ¿Esta spec debe fusionar ambos pasos en uno (confirmar el pago envía
  automáticamente a cocina), mantenerlos separados pero rediseñar la interfaz para que la acción
  pendiente sea imposible de pasar por alto, u ofrecer ambos modos como una preferencia configurable
  por tenant? → A: Fusionar ambos pasos en uno — confirmar el pago (efectivo recibido, o
  comprobante aprobado) envía automáticamente el pedido a cocina y descuenta el inventario en la
  misma acción, sin ningún paso manual adicional del cajero. Cambia explícitamente el
  comportamiento de spec 024 FR-017.
- Q: Sobre la revisión de factura, división de cuenta y pago con varios métodos: ¿esta spec debe
  rediseñar y exponer la asignación de ítems a comensales y el cobro con varios métodos de pago
  directamente dentro del panel "Cuenta de la mesa" de esta misma Terminal de Mesas, o esa
  capacidad permanece en su pantalla actual de cierre de sesión de mesa (spec 010) y esta spec se
  limita a mejorar la claridad visual de lo que la Terminal de Mesas ya muestra hoy (que hoy, según
  la captura, es prácticamente nada — "Selecciona una mesa con consumo")? → A: Rediseñar y exponer
  división de cuenta y cobro con varios métodos de pago directamente en el panel "Cuenta de la
  mesa" de la Terminal de Mesas, reutilizando las reglas y datos ya definidos en spec 010/011 (sin
  reabrir la prohibición de división porcentual ni el motor de cálculo de factura).
- Q: ¿La Terminal de Mesas se usa principalmente en tablet táctil, en computador de escritorio con
  mouse/teclado, o en ambos por igual? Esto determina si el rediseño debe priorizar objetivos
  táctiles grandes o mayor densidad de información en pantalla. → A: Ambos por igual — el diseño
  debe funcionar bien tanto en tablet táctil como en escritorio con mouse/teclado, sin asumir un
  dispositivo principal.
- Q: ¿La distinción visual entre estados del pedido (pago pendiente, pago rechazado, pago
  confirmado y en cocina) debe apoyarse únicamente en el color, o siempre debe ir acompañada de una
  etiqueta de texto o ícono que no dependa del color? → A: Color + texto/ícono siempre juntos — cada
  estado lleva una etiqueta corta o ícono además de su color, en toda vista (tarjeta de mesa, lista,
  detalle), para que la distinción no dependa de percibir el color bajo cualquier condición de luz
  o percepción visual.
- Q: ¿El tamaño mínimo de texto y de los controles táctiles debe fijarse como un estándar numérico
  concreto, o debe quedar como un criterio cualitativo ("sin necesidad de hacer zoom") a validar con
  pruebas de usuario en la fase de diseño? → A: Estándar numérico mínimo — texto legible equivalente
  a 16px o más en pantalla de escritorio (con jerarquía visual mayor para totales y estado), y
  controles táctiles de al menos 44x44 puntos, verificable directamente sin depender de una prueba
  de usuario.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Confirmar el pago envía el pedido a cocina en un solo paso (Priority: P1)

Cuando el cajero confirma el pago de un pedido (registra el efectivo recibido, o aprueba un
comprobante de transferencia), el pedido debe pasar directamente a cocina — se descuenta el
inventario y queda visible para producción en la misma acción, sin ningún paso manual adicional.
Hoy existe un segundo paso ("Por confirmar" → botón "Confirmar") que obliga al cajero a volver a
intervenir sobre un pedido cuyo pago ya verificó, lo que genera la percepción de que el pedido "no
pasó a la mesa" aunque el pago ya esté correctamente registrado — el problema que originó esta
spec.

**Why this priority**: es el problema central que reportó el usuario — un pago ya verificado que
no llega solo a cocina genera la percepción de que el sistema falló y obliga a una acción manual
redundante; eliminar ese segundo paso es indispensable antes de cualquier otro cambio visual.

**Independent Test**: confirmar el pago de un pedido (efectivo o transferencia) y verificar que,
sin ninguna acción adicional del cajero, el pedido queda con el inventario descontado y visible
para cocina inmediatamente.

**Acceptance Scenarios**:

1. **Given** un pedido con pago en efectivo pendiente de confirmar, **When** el cajero registra el
   monto recibido y confirma, **Then** el pedido descuenta su inventario y queda visible para
   cocina en la misma acción, sin que el cajero deba realizar ningún paso adicional.
2. **Given** un pedido pagado por transferencia con comprobante pendiente de revisión, **When** el
   cajero aprueba el comprobante, **Then** el pedido descuenta su inventario y queda visible para
   cocina en la misma acción de aprobación.
3. **Given** que la confirmación del pago dispara automáticamente el descuento de inventario,
   **When** el inventario disponible no alcanza para cubrir lo que el pedido consume, **Then** el
   sistema lo informa de inmediato al cajero en el mismo momento de la confirmación, sin dejar el
   pedido en un estado ambiguo (ni pagado-sin-cocina, ni descontado a medias).
4. **Given** un pedido cuyo pago todavía está pendiente o fue rechazado, **When** el cajero lo
   revisa, **Then** el sistema lo distingue visualmente con claridad de un pedido cuyo pago ya fue
   confirmado y enviado a cocina — nunca deben verse igual.

---

### User Story 2 - El cajero ve el cambio a entregar al confirmar un pago en efectivo (Priority: P1)

Al registrar el monto recibido en efectivo y confirmar el pago, el sistema ya calcula el cambio
(spec 024, FR-010) pero hoy no lo muestra en ningún lugar de la interfaz — el cajero no tiene forma
de saber cuánto debe devolver sin calcularlo mentalmente. El sistema debe mostrar el cambio de
forma inmediata y prominente en el momento de confirmar, y dejarlo consultable después.

**Why this priority**: es un defecto concreto y de alto impacto operativo — un cajero que no ve el
cambio calculado puede devolver un monto incorrecto, con pérdida directa de dinero para el negocio
o el cliente.

**Independent Test**: registrar un monto recibido mayor al total de una orden en efectivo,
confirmar el pago, y verificar que el cambio a entregar aparece de forma clara e inmediata, y que
puede volver a consultarse después sin recalcularlo.

**Acceptance Scenarios**:

1. **Given** una orden de $18.000 pagada en efectivo, **When** el cajero registra $20.000 recibidos
   y confirma, **Then** el sistema muestra de inmediato y de forma prominente "Cambio: $2.000" como
   parte de la confirmación, sin que el cajero deba buscarlo en otra pantalla.
2. **Given** una orden pagada con el monto exacto en efectivo, **When** el cajero confirma,
   **Then** el sistema muestra igual de claramente que el cambio es $0 (no omite el dato solo
   porque es cero).
3. **Given** un pedido cuyo pago en efectivo ya fue confirmado, **When** el cajero vuelve a abrir
   ese pedido más tarde, **Then** puede seguir viendo el monto recibido y el cambio calculado sin
   tener que recordarlo de memoria.

---

### User Story 3 - El cajero divide la cuenta de una mesa y la cobra con varios métodos de pago desde la Terminal de Mesas (Priority: P2)

Hoy el panel "Cuenta de la mesa" de la Terminal de Mesas no muestra nada útil mientras no se
selecciona una mesa con consumo, y no ofrece ninguna forma de dividir la cuenta ni de cobrarla con
más de un método de pago sin salir de esta pantalla. El sistema debe permitir, desde este mismo
panel, ver el detalle de consumo de la mesa, asignar ítems o unidades a comensales específicos
cuando la cuenta se vaya a dividir, y registrar el cobro con uno o varios métodos de pago
combinados, reutilizando exactamente las reglas y los cálculos ya definidos en spec 010 (reparto
solo por ítem/unidad asignada, nunca porcentual) y spec 011 (una venta puede combinar varios
métodos de pago, con cambio solo desde el excedente en efectivo).

**Why this priority**: responde directamente al pedido de revisar factura, división de cuenta y
pago con varios métodos; depende de que la comanda (Historias 1 y 2) ya sea clara, por eso es P2 y
no P1.

**Independent Test**: abrir una mesa con consumo, ver el detalle de su cuenta, dividirla asignando
ítems a dos comensales distintos, cobrar cada parte con un método de pago diferente (o combinando
efectivo y otro método en una misma parte), y verificar que el total cobrado y el cambio (si aplica)
son correctos, y que se genera la factura correspondiente al cerrar.

**Acceptance Scenarios**:

1. **Given** una mesa con consumo, **When** el cajero la selecciona, **Then** el panel "Cuenta de la
   mesa" muestra el detalle completo (ítems, cantidades, subtotal, descuentos/promociones ya
   aplicados, total) sin necesidad de salir de la Terminal de Mesas.
2. **Given** el detalle de la cuenta de una mesa con varios comensales, **When** el cajero decide
   dividirla, **Then** puede asignar cada ítem o unidad a un comensal específico, y el sistema
   nunca ofrece una división porcentual o "a partes iguales" automática.
3. **Given** la cuenta ya dividida por comensal, **When** el cajero cobra la parte de un comensal
   combinando efectivo y otro método de pago, **Then** el sistema acepta ambos montos, calcula el
   cambio únicamente sobre el excedente pagado en efectivo, y dicho excedente no puede salir de un
   medio electrónico.
4. **Given** una mesa ya cobrada (completa o dividida), **When** el cierre se completa, **Then**
   la factura correspondiente queda generada automáticamente, visible desde el mismo flujo, sin
   pasos adicionales.

---

### User Story 4 - El personal lee la comanda con claridad, en tablet o en escritorio (Priority: P3)

El texto y la jerarquía visual de la Terminal de Mesas (lista de mesas, pedidos, cuenta de la mesa)
se actualizan para que el personal —trabajando tanto en tablet táctil como en computador de
escritorio— identifique de un vistazo lo esencial de cada pedido (mesa, productos, total, estado de
pago) sin acercarse a la pantalla ni pedir ayuda, y para que los botones de acción (confirmar pago,
aprobar/rechazar comprobante, cobrar) sean fáciles de tocar o pulsar sin error en cualquiera de los
dos dispositivos.

**Why this priority**: es una mejora transversal de legibilidad y usabilidad; se prioriza después
de las historias 1-3 porque no corrige ningún flujo roto por sí sola, pero mejora la experiencia
de todas ellas.

**Independent Test**: mostrar la misma pantalla de Terminal de Mesas a personal nuevo, en tablet y
en escritorio, y verificar que identifican correctamente mesa, productos, total y estado de pago de
un pedido sin ayuda ni zoom manual, y que pueden pulsar la acción correcta sin errores de toque.

**Acceptance Scenarios**:

1. **Given** la lista de mesas y pedidos de la Terminal de Mesas, **When** un usuario la observa
   por primera vez, **Then** distingue sin ayuda el estado de cada mesa/pedido (libre, ocupada,
   pago pendiente, pago confirmado y en cocina) por su jerarquía visual, no solo por texto pequeño.
2. **Given** la misma pantalla en una tablet táctil, **When** el usuario intenta tocar un botón de
   acción (confirmar pago, aprobar/rechazar comprobante, cobrar), **Then** el objetivo es lo
   bastante grande como para tocarlo sin activar por error un botón adyacente.
3. **Given** la misma pantalla en un computador de escritorio, **When** el usuario la usa con mouse
   y teclado, **Then** la densidad de información permite ver varias mesas/pedidos sin scroll
   excesivo, sin sacrificar la legibilidad lograda para tablet.

---

### Edge Cases

- **El descuento automático de inventario falla en el momento de confirmar el pago** (por ejemplo,
  stock insuficiente descubierto justo al confirmar): el sistema no debe confirmar el pago y enviar
  el pedido a cocina a medias; el cajero recibe un aviso claro e inmediato y una forma de resolverlo
  (FR-002), en vez del comportamiento anterior a esta spec, donde ese chequeo quedaba implícito en
  el paso manual separado de "enviar a comanda".
- **Cambio de $0 en un pago en efectivo con monto exacto**: debe mostrarse explícitamente como
  "$0", nunca omitirse como si no existiera el dato (Historia 2, escenario 2).
- **División de cuenta con un comensal que no consumió ningún ítem propio** (por ejemplo, compartió
  todo): el sistema permite que su parte asignada quede en $0 sin bloquear el cierre de los demás.
- **Cobro combinando métodos donde el monto electrónico por sí solo ya cubre o excede el total**:
  el excedente en ese caso no genera cambio (el cambio solo puede salir de efectivo, spec 010
  FR-020); el sistema lo rechaza si se intenta lo contrario.
- **Dos cajeros atendiendo la misma mesa casi al mismo tiempo** (por ejemplo, ambos intentan
  confirmar el mismo pago o cobrar la misma cuenta): se mantiene sin cambios la garantía ya
  existente de que solo una acción tiene efecto (spec 024 FR-018, spec 010 FR-001).
- **Pantallas o ventanas muy angostas** (tablet en orientación vertical, ventana de escritorio
  redimensionada): la información esencial (mesa, total, estado) permanece legible sin recortarse
  ni requerir scroll horizontal.
- **Personal con dificultad para distinguir colores, o pantallas con poca luz/reflejo (cocina,
  exteriores)**: el estado de un pedido sigue siendo identificable porque nunca depende solo del
  color (FR-003).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Cuando el cajero confirme un pago en efectivo (monto recibido registrado) o apruebe
  un comprobante de transferencia, el sistema DEBE, en esa misma acción y sin ningún paso manual
  adicional, descontar el inventario que el pedido consume y dejarlo visible para cocina —
  eliminando el paso separado de "enviar a comanda" que existía antes de esta spec (cambia spec
  024, FR-017).
- **FR-002**: Si el descuento automático de inventario de FR-001 no puede completarse (por ejemplo,
  stock insuficiente en el momento de confirmar), el sistema DEBE notificar el problema al cajero
  de inmediato, en el mismo paso de confirmación, y NO DEBE dejar el pedido en un estado ambiguo:
  o el pago queda confirmado y el pedido enviado a cocina con su inventario correctamente
  descontado, o ninguna de las dos cosas ocurre, con una vía clara para que el cajero resuelva el
  problema (por ejemplo, ajustar el pedido o reintentar) antes de que el cliente asuma que ya se
  está preparando.
- **FR-003**: El sistema DEBE distinguir visualmente, sin ambigüedad, entre un pedido con pago
  pendiente o rechazado y un pedido con pago ya confirmado y enviado a cocina — nunca deben
  presentarse con la misma apariencia. Esa distinción NO DEBE depender únicamente del color: cada
  estado DEBE mostrarse siempre junto con una etiqueta de texto corta o un ícono reconocible, en
  toda vista donde el estado sea visible.
- **FR-004**: Cuando el cajero confirme un pago en efectivo, el sistema DEBE mostrar de inmediato y
  de forma prominente el monto recibido, el total de la orden y el cambio a entregar (incluyendo el
  caso en que el cambio sea $0), como parte del mismo paso de confirmación.
- **FR-005**: El sistema DEBE mantener visible y consultable el monto recibido y el cambio de un
  pago en efectivo ya confirmado, en cualquier momento posterior en que el cajero vuelva a abrir ese
  pedido.
- **FR-006**: El panel "Cuenta de la mesa" DEBE mostrar, al seleccionar una mesa con consumo, el
  detalle completo de su cuenta (ítems, cantidades, subtotal, descuentos o promociones ya aplicados,
  total) sin requerir navegar a otra pantalla.
- **FR-007**: El sistema DEBE permitir, desde el panel "Cuenta de la mesa", dividir la cuenta
  asignando cada ítem o unidad de un ítem a un comensal específico, reutilizando exactamente la
  regla ya vigente (spec 010, FR-005): ninguna división porcentual o "a partes iguales" automática.
- **FR-008**: El sistema DEBE permitir, desde el mismo panel, cobrar la cuenta completa o cada parte
  ya dividida combinando uno o varios métodos de pago en un mismo cobro, reutilizando exactamente
  las reglas ya vigentes (spec 011): el pago recibido debe cubrir el total exacto o más, y el cambio
  solo puede provenir de un excedente pagado en efectivo.
- **FR-009**: Al completar el cobro de una mesa (completa o dividida) desde este panel, el sistema
  DEBE generar automáticamente la factura correspondiente, sin pasos adicionales del cajero.
- **FR-010**: El sistema DEBE presentar la lista de mesas, los pedidos y el detalle de la cuenta con
  un tamaño de texto legible equivalente a 16px o más en pantalla de escritorio (con jerarquía
  visual mayor para el total y el estado de cada pedido), de forma que se identifiquen mesa,
  productos, total y estado de pago sin necesidad de hacer zoom o acercarse a la pantalla.
- **FR-011**: El sistema DEBE dimensionar los controles de acción (confirmar pago, aprobar/rechazar
  comprobante, cobrar) con un objetivo táctil de al menos 44x44 puntos, de forma que sean operables
  con precisión tanto por toque en tablet como por clic en escritorio, sin asumir un único tipo de
  dispositivo como principal.
- **FR-012**: El sistema NO DEBE reducir, respecto de la interfaz actual, ninguna información hoy
  disponible en la comanda (ítems, notas por producto, método de pago, estado del pedido) como
  consecuencia de este rediseño.

### Key Entities *(include if feature involves data)*

Esta spec no agrega entidades nuevas — reutiliza **Orden**, **Intento de Pago**, **Comprobante** y
**Método de Pago** (spec 024), y **Sesión de Mesa / Cuenta de la Mesa**, **Venta**, **Pago** y
**Factura** (spec 010, spec 011, spec 016). El cambio es exclusivamente de presentación e
interacción: cómo y dónde se muestran los estados, el cambio en efectivo, la división de cuenta y
el cobro con varios métodos que ya existen hoy a nivel de datos y reglas de negocio.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los pagos confirmados (efectivo registrado o comprobante aprobado) dejan
  el pedido descontado en inventario y visible para cocina sin ninguna acción manual adicional del
  cajero, salvo que el sistema detecte un problema de inventario, en cuyo caso el cajero es
  notificado de inmediato (FR-002).
- **SC-002**: El 100% de las confirmaciones de pago en efectivo muestran el cambio a entregar en el
  mismo momento de la confirmación, incluyendo cuando el cambio es $0.
- **SC-003**: El personal nuevo, al ver la Terminal de Mesas por primera vez, identifica
  correctamente el estado de un pedido (pago pendiente, pago rechazado, pago confirmado y en
  cocina) sin ayuda, en una prueba de reconocimiento visual.
- **SC-004**: El cajero puede dividir la cuenta de una mesa entre varios comensales y cobrar cada
  parte con uno o más métodos de pago sin salir del panel "Cuenta de la mesa" ni recurrir a otra
  pantalla.
- **SC-005**: El 100% de los cierres de cuenta realizados desde este panel (completos o divididos)
  dejan una factura generada, verificable inmediatamente después del cobro.
- **SC-006**: El 100% del texto y los controles de acción de la comanda cumplen el mínimo definido
  (texto legible equivalente a 16px+ en escritorio, controles táctiles de al menos 44x44 puntos),
  verificable directamente sobre el diseño entregado, sin depender de una prueba de usuario.
- **SC-007**: Ninguna información hoy visible en la comanda (ítems, notas, método de pago, estado)
  deja de estar disponible tras el rediseño.

## Out of Scope

- Cambios a las reglas de negocio ya definidas en spec 024: métodos de pago configurables por
  tenant, aprobación/rechazo de comprobantes con motivo, y el cálculo mismo del cambio — se
  reutilizan sin modificación; esta spec solo corrige que el cambio ya calculado no se mostraba.
- Reabrir la prohibición de dividir la cuenta de forma porcentual o "a partes iguales" (spec 010,
  RN-MESA-05) — el reparto sigue siendo exclusivamente por ítem o unidad asignada a un comensal.
- Cambios al motor de cálculo de factura, promociones, combos o descuentos (spec 011, spec 012,
  spec 013) — se reutilizan tal como están.
- Nuevos métodos de pago o integraciones con pasarelas/bancos.
- Cambios a la regla de "una orden activa a la vez", a la cancelación de pedidos, o a la revisión
  del comprobante de transferencia por el cajero (spec 024) — permanecen exactamente iguales.
- Definir o construir la o las propuestas visuales concretas (mockups) como parte de esta spec de
  negocio — esta especificación define el comportamiento y los criterios de éxito que cualquier
  propuesta de diseño debe cumplir; las alternativas visuales concretas se exploran en la fase de
  planeación/diseño posterior.

## Assumptions

- **El descuento automático de inventario al confirmar el pago (Historia 1)** reutiliza la misma
  operación de descuento que hoy ejecuta el paso manual "enviar a comanda" (spec 024) — esta spec
  solo cambia el momento en que se dispara (automáticamente, dentro de la confirmación del pago) y
  exige el manejo explícito del caso de fallo (FR-002); no cambia cómo se calcula ni qué se
  descuenta.
- **La pestaña "Por confirmar"** deja de tener sentido como cola de pedidos pendientes de enviar a
  cocina, ya que ese envío ahora es automático; su rediseño (o eliminación) concreto se resuelve en
  la fase de planeación, junto con las propuestas de UX que el usuario pidió explorar.
- **El cambio en efectivo se muestra únicamente al personal (cajero/mesero) en la Terminal de
  Mesas**, no al comensal en el flujo QR — coherente con que el registro del monto recibido y la
  confirmación del pago en efectivo ya son, según spec 024, una acción exclusiva del cajero.
- **La división de cuenta y el cobro con varios métodos de pago expuestos en el panel "Cuenta de la
  mesa"** reutilizan exactamente los mismos cálculos y reglas ya implementados para el cierre de
  sesión de mesa (spec 010) y la construcción de venta/factura (spec 011); esta spec no introduce
  una segunda forma de calcular la cuenta de una mesa, para no reabrir la inconsistencia ya
  documentada y corregida entre distintas implementaciones del "total a cobrar".
- **El diseño debe funcionar igual de bien en tablet táctil y en escritorio** (confirmado en
  Clarifications), por lo que los tamaños de texto y controles se definen pensando en el caso más
  exigente (toque en tablet) sin sacrificar la densidad de información útil en escritorio.
- **Esta spec no prescribe una única propuesta visual final**: el usuario pidió explícitamente
  "varias propuestas de UX"; esta especificación define el comportamiento esperado y los criterios
  de éxito medibles que cualquiera de esas propuestas debe cumplir, dejando la comparación de
  alternativas visuales concretas para la fase de planeación (`/speckit-plan`).
