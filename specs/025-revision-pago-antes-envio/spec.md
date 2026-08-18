# Feature Specification: Revisión y Pago Antes de Enviar el Pedido (Skeilopos)

**Feature Branch**: `025-revision-pago-antes-envio`

**Created**: 2026-08-18

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad **nueva**, no de ingeniería inversa (fase de
evolución funcional, Principio I de la
[Constitución](../../.specify/memory/constitution.md)). Se construye directamente sobre
[spec 024](../024-pagos-ordenes-mesa/spec.md) — que ya define métodos de pago configurables por
tenant, comprobantes de transferencia revisados por el cajero, y el bloqueo de avance a comanda
sin pago confirmado — y **cambia el momento** en que el pedido queda registrado para el staff:
hoy, enviar el carrito crea el pedido de inmediato y el pago se resuelve después, sobre un pedido
que ya existe; esta spec invierte ese orden para que el comensal resuelva primero cómo va a pagar
y el pedido solo se registre una vez ese paso está resuelto.

**Input**: User description: mejorar el flujo del menú QR para que, al presionar "Enviar pedido",
en lugar de enviarse de inmediato, se muestre una pantalla de revisión del pedido con selección de
método de pago; al confirmar el pago, si es en efectivo el pedido se registra con el pago
pendiente de recibir en caja, y si es por transferencia el comensal debe cargar el comprobante
antes de que el pedido se registre, continuando después con el flujo ya existente de verificación
del cajero y facturación. Se adjuntó una captura de un producto de referencia (entorno de pruebas
ajeno) únicamente como inspiración visual de la disposición de pantallas — ni sus textos ni su
comportamiento exacto se replican literalmente.

## Clarifications

### Session 2026-08-18

- Q: ¿Debe el comensal ver algún número o referencia de su pedido mientras todavía está en el
  paso de pago (antes de que el pedido exista formalmente), o esa referencia solo debe aparecer
  una vez el pedido ya quedó registrado para el staff? → A: No se muestra ningún número hasta que
  el pedido se cree (al confirmar efectivo o al cargar el comprobante) — confirma como decisión
  de negocio lo que ya estaba documentado como supuesto de diseño.
- Q: Si el comensal confirma el pago dos veces seguidas para el mismo envío (doble toque, o
  reintento tras una falla de red), ¿el sistema debe garantizar del lado del servidor que nunca se
  cree más de un pedido, o basta con que la interfaz reduzca el riesgo (botón deshabilitado
  mientras se procesa)? → A: Garantía del lado del servidor — un mismo envío nunca debe crear más
  de un pedido, incluso si la petición de confirmación llega duplicada.
- Q: Si el comprobante se sube correctamente pero el pedido no llega a crearse justo después (p.
  ej. se corta la conexión), ¿el comensal reintenta solo la creación del pedido sin volver a
  elegir el archivo, o repite el proceso completo desde elegir el método de pago? → A: Reintenta
  solo la creación del pedido — el sistema recuerda que el comprobante ya se subió y no se lo
  vuelve a pedir.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El comensal revisa su pedido y elige cómo pagar antes de enviarlo (Priority: P1)

Cuando el comensal arma su carrito y presiona "Enviar pedido", en lugar de que el pedido se envíe
de inmediato, se le muestra una pantalla de revisión con el resumen de lo que va a pedir (ítems,
cantidades, notas ya agregadas por ítem, subtotal y total) y la selección del método de pago entre
los que su tenant tenga activos en ese momento. El pedido todavía no existe para el staff en este
punto — es el paso previo que determina por cuál de los dos caminos de pago continuará.

**Why this priority**: es el cambio central que pide esta spec — sin esta pantalla no hay dónde
elegir el método antes de enviar, y ninguna de las otras historias tiene sentido.

**Independent Test**: armar un carrito, presionar "Enviar pedido", y verificar que aparece la
pantalla de revisión con el resumen correcto y los métodos de pago activos del tenant, sin que el
pedido aparezca todavía en el panel del staff.

**Acceptance Scenarios**:

1. **Given** un carrito con productos, **When** el comensal presiona "Enviar pedido", **Then** ve
   un resumen de su pedido (ítems, cantidades, notas por ítem, subtotal y total) y los métodos de
   pago que su tenant tiene activos, y el pedido no aparece todavía para el staff.
2. **Given** la pantalla de revisión abierta, **When** el comensal vuelve a la carta sin completar
   ningún pago, **Then** no se crea ningún pedido y su carrito conserva exactamente los mismos
   productos.
3. **Given** un tenant con solo efectivo activo, **When** el comensal abre la pantalla de
   revisión, **Then** solo ve efectivo como opción de pago — ningún método desactivado aparece.

---

### User Story 2 - El comensal paga en efectivo y el pedido queda para el cajero (Priority: P1)

El comensal elige efectivo en la pantalla de revisión y confirma. En ese momento el pedido queda
registrado y visible para el staff, con un intento de pago en efectivo pendiente de que el cajero
registre el monto recibido al recibirlo — el mismo flujo de confirmación y cálculo de cambio que
ya existe.

**Why this priority**: es uno de los dos únicos caminos de pago y, sin él, un comensal que paga en
efectivo no tendría cómo completar su pedido.

**Independent Test**: elegir efectivo en la pantalla de revisión, confirmar, y verificar que el
pedido aparece para el staff con el pago en efectivo pendiente de registrar; que el cajero pueda
confirmarlo con el flujo ya existente.

**Acceptance Scenarios**:

1. **Given** la pantalla de revisión abierta con efectivo disponible, **When** el comensal lo
   elige y confirma el pago, **Then** el pedido queda creado y visible para el staff, con el
   efectivo pendiente de registrar.
2. **Given** un pedido creado por esta vía, **When** el cajero registra el monto recibido, **Then**
   el sistema calcula el cambio y confirma el pago exactamente igual que hoy (sin cambio de
   comportamiento sobre lo ya definido).

---

### User Story 3 - El comensal paga por transferencia y el pedido solo se registra tras cargar el comprobante (Priority: P1)

El comensal elige un método de transferencia en la pantalla de revisión. Se le muestran los datos
de pago de ese método y debe cargar el comprobante de su transferencia. El pedido **no** queda
registrado para el staff mientras el comprobante no se haya cargado con éxito; una vez cargado, el
pedido se crea y queda visible para el staff con el intento de pago en revisión, continuando con
el flujo ya existente de aprobación o rechazo del cajero.

**Why this priority**: es el otro camino de pago y el que motiva el cambio de orden — evita que
existan pedidos por transferencia sin ningún comprobante asociado.

**Independent Test**: elegir un método de transferencia, ver sus datos de pago, cargar un
comprobante, y verificar que el pedido solo aparece para el staff después de completar esa carga.

**Acceptance Scenarios**:

1. **Given** la pantalla de revisión abierta, **When** el comensal elige un método de
   transferencia, **Then** ve los datos de pago de ese método específico y el control para cargar
   el comprobante, sin que el pedido se haya creado todavía.
2. **Given** la pantalla de carga del comprobante abierta sin nada cargado, **When** el comensal la
   abandona, **Then** no se crea ningún pedido.
3. **Given** un comprobante cargado con éxito, **When** la carga se completa, **Then** el pedido
   queda creado y visible para el staff con el intento de pago en revisión.
4. **Given** un pedido creado por esta vía, **When** el cajero aprueba o rechaza el comprobante
   (motivo obligatorio en el rechazo), **Then** el flujo ya existente funciona exactamente igual
   que hoy, incluyendo el reintento tras un rechazo.

---

### User Story 4 - El comensal cambia de método de transferencia antes de subir el comprobante (Priority: P2)

Mientras el comensal está en la pantalla de un método de transferencia y todavía no ha cargado
ningún comprobante, puede volver y elegir otro método (de transferencia o efectivo) sin ninguna
restricción — porque en este punto el pedido todavía no existe, así que no hay ningún intento de
pago que cancelar o gestionar.

**Why this priority**: evita fricción innecesaria en un punto donde, a diferencia de un reintento
tras un rechazo (spec 024), no hay nada creado todavía que limite el cambio de opinión. Prioridad
P2 porque es una mejora sobre el camino principal (Historia 3), no un requisito para completarlo.

**Independent Test**: abrir la pantalla de un método de transferencia sin cargar nada, volver y
elegir un método distinto, y verificar que solo termina existiendo un pedido, con el método
finalmente elegido.

**Acceptance Scenarios**:

1. **Given** el comensal eligió un método de transferencia sin cargar comprobante, **When** vuelve
   y elige otro método, **Then** ve los datos de pago del nuevo método y puede cargar el
   comprobante ahí, sin ningún rastro del método anterior.

---

### Edge Cases

- **El comensal pierde la conexión mientras carga el comprobante**: la carga falla, el pedido no
  llega a crearse, y el comensal puede reintentar sin haber perdido su carrito.
- **El comensal agrega más productos desde la pantalla de revisión**: se permite volver a la carta
  y regresar — nada se ha enviado todavía, así que no hay ningún pedido parcial que reconciliar.
- **El tenant se queda sin ningún método de pago activo**: no puede ocurrir en la práctica — spec
  024 (FR-003) ya impide desactivar el último método activo del tenant.
- **La carga del comprobante se completa pero la confirmación del pedido falla justo después**: el
  sistema no debe dejar un comprobante huérfano sin ningún pedido asociado ni un pedido sin su
  comprobante — o ambos quedan creados juntos, o ninguno de los dos. El comensal reintenta solo la
  creación del pedido, sin volver a cargar el archivo (FR-012).
- **El comensal ya tiene una orden activa sin finalizar**: sigue aplicando el mismo límite ya
  vigente (spec 024, "una orden activa a la vez") — la pantalla de revisión no es una excepción a
  esa regla.
- **El comensal toca "Pagar" dos veces seguidas, o su conexión reintenta la misma confirmación
  sola**: el sistema garantiza que esa confirmación produzca un único pedido, sin importar cuántas
  veces llegue la misma solicitud (FR-013).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Al presionar "Enviar pedido", el sistema DEBE mostrar una pantalla de revisión con
  el resumen del pedido (ítems, cantidades, notas ya agregadas por ítem, subtotal y total) y la
  selección del método de pago, en lugar de enviar el pedido de inmediato.
- **FR-002**: La pantalla de revisión DEBE ofrecer únicamente los métodos de pago que el tenant
  tenga activos en ese momento, sin mostrar ninguno desactivado.
- **FR-003**: El sistema NO DEBE registrar el pedido para el staff mientras el comensal permanezca
  en la pantalla de revisión (o en la de un método de transferencia) sin completar su paso de
  pago.
- **FR-004**: Cuando el comensal elija efectivo y confirme, el sistema DEBE crear el pedido y
  dejarlo visible para el staff con un intento de pago en efectivo pendiente de que el cajero
  registre el monto recibido, continuando con el flujo de confirmación y cambio ya definido en
  spec 024.
- **FR-005**: Cuando el comensal elija un método de transferencia, el sistema DEBE mostrarle los
  datos de pago de ese método específico y exigirle cargar un comprobante antes de continuar.
- **FR-006**: El sistema NO DEBE crear el pedido de un pago por transferencia hasta que el
  comprobante se haya cargado con éxito.
- **FR-006a**: Mientras el pedido no exista (pantalla de revisión y pantalla de un método de
  transferencia), el sistema NO DEBE mostrarle al comensal ningún número o identificador de
  pedido — esa referencia solo aparece una vez el pedido efectivamente se creó (Clarification
  Session 2026-08-18).
- **FR-007**: Una vez cargado el comprobante, el sistema DEBE crear el pedido y dejarlo visible
  para el staff con el intento de pago en revisión, continuando con el flujo de aprobación o
  rechazo del cajero ya definido en spec 024.
- **FR-008**: El sistema DEBE permitir que el comensal salga de la pantalla de revisión o de la de
  un método de transferencia sin completar el pago, sin que eso cree ningún pedido ni afecte los
  productos ya agregados a su carrito.
- **FR-009**: Mientras el comensal no haya cargado ningún comprobante, el sistema DEBE permitirle
  cambiar libremente de un método de transferencia a otro (o a efectivo) sin restricción alguna.
- **FR-010**: Sobre el pedido ya creado por esta pantalla, el sistema DEBE seguir aplicando, sin
  cambios, todas las reglas ya vigentes de spec 024: aprobación/rechazo de comprobante con motivo,
  confirmación de efectivo con cambio calculado, bloqueo de avance a comanda sin pago confirmado,
  reintento tras rechazo, y una sola confirmación válida por intento de pago.
- **FR-011**: El sistema DEBE seguir impidiendo que el comensal complete esta pantalla de revisión
  si ya tiene una orden activa sin finalizar (mismo límite ya vigente en spec 024).
- **FR-012**: Si la carga del comprobante se completa pero el pedido no llega a registrarse por
  cualquier motivo, el sistema NO DEBE dejar el comprobante asociado a ningún pedido a medio
  crear, y el comensal DEBE poder reintentar únicamente la creación del pedido — sin tener que
  volver a seleccionar ni cargar el archivo del comprobante (Clarification Session 2026-08-18).
- **FR-013**: El sistema DEBE garantizar, del lado del servidor, que una misma confirmación de
  pago (efectivo o carga de comprobante) nunca genere más de un pedido — incluso si la
  confirmación llega duplicada por un doble toque o un reintento de red (Clarification Session
  2026-08-18).

### Key Entities *(include if feature involves data)*

Esta spec no agrega entidades nuevas — reutiliza **Orden**, **Intento de Pago**, **Comprobante** y
**Método de Pago**, ya definidas en spec 024. El único cambio de comportamiento es *cuándo* nace la
Orden (y su primer Intento de Pago): antes, al enviar el carrito; ahora, al completar el paso de
pago que corresponda según el método elegido.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los pedidos que llegan al staff ya tienen un método de pago elegido y su
  primer intento de pago creado — ninguno llega "sin pagar" a la espera de que el comensal decida.
- **SC-002**: El 0% de las veces que un comensal abandona la pantalla de revisión o la de un
  método de transferencia sin completar el pago queda un pedido registrado para el staff.
- **SC-003**: El comensal puede completar el flujo de revisión y pago en efectivo (desde que
  presiona "Enviar pedido" hasta ver la confirmación) sin más de 2 toques.
- **SC-004**: El comensal que paga por transferencia nunca pierde productos de su pedido entre que
  presiona "Enviar pedido" y que su comprobante queda cargado — el 100% de las veces el resumen
  final coincide con lo que armó en el carrito.
- **SC-005**: Cambiar de método de transferencia antes de cargar el comprobante no genera ningún
  registro visible para el cajero — el 100% de las veces, solo el método finalmente elegido deja
  rastro.
- **SC-006**: El 0% de las confirmaciones de pago duplicadas (doble toque o reintento de red sobre
  el mismo envío) produce más de un pedido — verificable incluso ante confirmaciones repetidas
  deliberadamente en pruebas.

## Out of Scope

- Métodos de pago nuevos (tarjeta, PSE, pago en mostrador); se mantiene exactamente el mismo
  conjunto que spec 024 (efectivo y las transferencias que cada tenant configure).
- Un campo de nota general del pedido (a nivel de todo el pedido, distinto de las notas que ya
  existen por ítem individual) — aparece en la captura de referencia pero no se pidió
  explícitamente; queda como posible mejora futura, no como parte de esta spec.
- Cambios a cómo el cajero aprueba o rechaza comprobantes, o confirma pagos en efectivo — ya
  definidos en spec 024 y sin modificación aquí.
- Cambios a la regla de "una orden activa a la vez", a la cancelación de pedidos, o a la
  generación de la factura — permanecen exactamente iguales.
- Integración directa vía API/webhook con pasarelas o bancos — igual que spec 024, el proceso
  sigue siendo manual vía comprobante y verificación humana del cajero.

## Assumptions

- **Se construye enteramente sobre spec 024**: métodos de pago configurables, intentos de pago,
  comprobantes, aprobación/rechazo con motivo, confirmación de efectivo con cambio, y el bloqueo
  de avance a comanda sin pago confirmado no cambian — solo cambia el momento en que la Orden y su
  primer Intento de Pago se crean.
- **Cambio de comportamiento explícito** (Constitución, Principio II): antes, enviar el carrito
  creaba el pedido de inmediato (sin ningún paso de pago); ahora, el pedido nace únicamente al
  completar el pago (confirmar efectivo, o cargar el comprobante de una transferencia). Se
  autoriza aquí, como parte de esta spec de evolución funcional.
- **No se muestra ningún número de pedido antes de que exista** (FR-006a, confirmado en
  Clarifications): a diferencia de la captura de referencia (que muestra un número de pedido ya
  en la pantalla de carga del comprobante), este flujo no asigna ni muestra un identificador de
  pedido hasta que el pedido efectivamente se creó.
- **Cambiar de método de transferencia sin comprobante cargado es libre** (FR-009): es distinto
  del límite de "un intento pendiente a la vez" de spec 024, que solo aplica una vez el pedido ya
  existe (por ejemplo, al reintentar tras un rechazo, spec 024 Historia 5) — aquí, antes de la
  primera carga, no hay ningún intento de pago que gestionar.
- **La captura de referencia adjunta es solo inspiración visual** de un producto de pruebas ajeno:
  ni sus textos, ni su disposición exacta de pantallas, ni el número de opciones de pago que
  muestra (tarjeta, PSE, pago en mostrador) se replican — el conjunto de métodos sigue siendo el
  que define spec 024, y los textos deben redactarse para producción, en español de Colombia,
  consistentes con el resto del producto.
- El resto del flujo de mesa por QR (identificación del comensal solo por nombre, cancelación de
  pedidos antes de que cocina empiece, una orden activa a la vez) permanece exactamente igual.
