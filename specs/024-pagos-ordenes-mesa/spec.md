# Feature Specification: Pagos de Órdenes en Mesa (Skeilopos)

**Feature Branch**: `024-pagos-ordenes-mesa`

**Created**: 2026-08-18

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad **nueva**, no de ingeniería inversa (fase de
evolución funcional, Principio I de la
[Constitución](../../.specify/memory/constitution.md)). Amplía el flujo existente de pedidos
por QR (spec 007, spec 008) para que (a) cada tenant configure sus propios métodos de pago
—incluyendo cualquier método de transferencia, no solo Nequi/Bancolombia— y (b) las
transferencias queden respaldadas por un comprobante que el cajero revisa, aprueba o rechaza
explícitamente, en lugar de una confirmación sin evidencia.

**Input**: User description: especificación de negocio "Pagos de Órdenes en Mesa (Skeilopos)
v1" — amplía el flujo QR existente para que cada tenant configure sus métodos de pago
(efectivo y transferencias propias) y para que toda transferencia quede respaldada por un
comprobante que el cajero aprueba o rechaza con motivo, antes de que la orden pueda pasar a
comanda.

## Clarifications

### Session 2026-08-18

- Q: ¿Puede un participante iniciar un nuevo intento de pago (cambiar de método, o subir otro
  comprobante) mientras el intento anterior todavía está pendiente de revisión del cajero — es
  decir, antes de que sea rechazado? → A: No, solo tras rechazo — mientras exista un intento de
  pago pendiente de revisión, el sistema no permite iniciar otro; solo se habilita un nuevo
  intento una vez el anterior queda explícitamente rechazado.
- Q: ¿Qué debe pasar si el cajero registra un monto recibido en efectivo menor al total de la
  orden? → A: El sistema lo impide — no permite confirmar el pago con un monto recibido menor al
  total; el cajero debe registrar un monto igual o mayor.
- Q: Cuando el cajero rechaza un comprobante y registra el motivo, ¿ese motivo debe ser visible
  también para el participante? → A: No, solo lo ve el cajero — el motivo de rechazo queda
  registrado en el historial de la orden visible para el cajero/back-office; el participante
  solo ve que su comprobante fue rechazado (sin el detalle del motivo) y puede reintentar el
  pago.
- Q: ¿Un comprobante de transferencia puede tener un solo archivo, o el participante puede
  adjuntar varios archivos a un mismo comprobante? → A: Un solo archivo — cada comprobante
  corresponde a un único archivo subido por el participante.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El tenant configura sus métodos de pago (Priority: P1)

Un administrador del tenant define qué métodos de pago acepta su negocio: activa o desactiva
el pago en efectivo y da de alta uno o más métodos de transferencia propios (por ejemplo, una
cuenta Nequi, una cuenta Bancolombia, o cualquier otro que use), cada uno con los datos que un
participante necesitará para pagar (cuenta, teléfono, código, etc.).

**Why this priority**: es la base de todo lo demás — sin esta configuración no existen
opciones de pago que mostrarle al participante ni datos de transferencia que enseñarle. Es
además la única historia que entrega valor de forma completamente autónoma: un administrador
puede configurar sus métodos sin que ningún participante haya pagado todavía.

**Independent Test**: se puede probar dando de alta, activando, desactivando y editando
métodos de pago (efectivo y transferencias) desde la configuración del tenant, y verificando
que los cambios quedan reflejados, sin necesidad de que exista ninguna orden.

**Acceptance Scenarios**:

1. **Given** un tenant sin métodos de transferencia configurados, **When** el administrador
   agrega un nuevo método de transferencia con nombre y datos de pago, **Then** el método
   queda disponible en la lista de métodos del tenant, inactivo o activo según se configure.
2. **Given** varios métodos de transferencia ya configurados, **When** el administrador
   desactiva uno de ellos, **Then** ese método deja de estar disponible para los participantes
   sin borrar su configuración ni su historial de pagos previos.
3. **Given** el efectivo y al menos un método de transferencia activos, **When** el
   administrador intenta desactivar el último método de pago que queda activo, **Then** el
   sistema lo impide — siempre debe quedar al menos un método de pago habilitado para el
   tenant.
4. **Given** un método de transferencia activo con datos de pago, **When** el administrador
   edita esos datos (por ejemplo, cambia el número de cuenta), **Then** los pagos ya
   confirmados con la configuración anterior no se alteran, y los nuevos participantes ven la
   información actualizada.

---

### User Story 2 - El participante paga por transferencia y el cajero revisa el comprobante (Priority: P1)

Un participante en una mesa arma su pedido y, al pagar, elige un método de transferencia entre
los que su tenant tiene habilitados. El sistema le muestra los datos de pago de ese método para
que transfiera correctamente, y el participante sube el comprobante de su transferencia. El
cajero ve el comprobante vinculado a esa orden y a ese intento de pago específico, y decide
aprobarlo o rechazarlo; si lo rechaza, debe indicar un motivo, que queda visible en el
historial de la orden.

**Why this priority**: es el objetivo central de esta especificación — reemplazar la
confirmación de transferencia sin evidencia por una verificación explícita y trazable del
cajero, para que el negocio no quede expuesto a error humano o fraude.

**Independent Test**: se puede probar creando una orden, seleccionando un método de
transferencia habilitado, subiendo un comprobante, y verificando que aparece para el cajero
vinculado a la orden y al intento de pago correcto, con la posibilidad de aprobarlo o
rechazarlo con motivo.

**Acceptance Scenarios**:

1. **Given** un tenant con efectivo, Nequi y Bancolombia habilitados, **When** el participante
   abre la pantalla de pago, **Then** ve exactamente esos tres métodos como opciones, y ningún
   método que el tenant tenga desactivado.
2. **Given** el participante elige el método "Nequi", **When** se muestra la pantalla de pago,
   **Then** aparecen los datos de pago que el tenant configuró para Nequi (por ejemplo, número
   y titular).
3. **Given** el participante sube un comprobante de su transferencia, **When** la carga se
   completa, **Then** el comprobante queda visible para el cajero, asociado tanto a la orden
   como al intento de pago específico que lo generó.
4. **Given** un comprobante visible para el cajero, **When** el cajero lo aprueba, **Then** el
   intento de pago queda confirmado y la orden deja de estar pendiente de pago.
5. **Given** un comprobante visible para el cajero, **When** el cajero intenta rechazarlo sin
   indicar un motivo, **Then** el sistema no permite completar el rechazo.
6. **Given** un comprobante visible para el cajero, **When** el cajero lo rechaza indicando el
   motivo "el monto no coincide", **Then** el intento de pago queda marcado como rechazado y
   ese motivo queda visible en el historial de la orden para el cajero; el participante ve que
   su comprobante fue rechazado, pero no ve el detalle del motivo.
7. **Given** un participante que ya subió su comprobante, **When** revisa el estado de su
   orden antes de que el cajero decida, **Then** la orden sigue mostrándose como pendiente de
   pago — el participante no puede autoconfirmarla.

---

### User Story 3 - El cajero confirma el pago en efectivo y registra el cambio (Priority: P1)

Un participante elige pagar en efectivo. La orden queda pendiente de confirmación por parte
del cajero. Cuando el cliente paga físicamente, el cajero registra el monto recibido y el
sistema calcula el cambio automáticamente, sin que el cajero tenga que hacer esa cuenta a
mano.

**Why this priority**: es el otro camino de pago del flujo (junto con la transferencia de
User Story 2) y es indispensable para que una orden en efectivo pueda avanzar a comanda con la
misma trazabilidad que una transferencia aprobada.

**Independent Test**: se puede probar creando una orden con método efectivo, registrando un
monto recibido mayor al total desde la vista de caja, y verificando que el cambio calculado es
correcto y que la orden queda confirmada.

**Acceptance Scenarios**:

1. **Given** una orden de $18.000 pagada en efectivo, **When** el cajero registra un monto
   recibido de $20.000 y confirma, **Then** el sistema calcula y muestra un cambio de $2.000, y
   el pago queda confirmado.
2. **Given** una orden pagada en efectivo, **When** el cajero registra un monto recibido igual
   al total exacto de la orden, **Then** el sistema confirma el pago con un cambio de $0.
3. **Given** una orden en efectivo aún no confirmada, **When** el participante consulta el
   estado de su orden, **Then** sigue viéndola como pendiente de pago hasta que el cajero
   confirme.
4. **Given** una orden de $18.000 pagada en efectivo, **When** el cajero intenta registrar un
   monto recibido de $15.000 (menor al total), **Then** el sistema impide confirmar el pago
   hasta que el monto recibido sea igual o mayor al total.

---

### User Story 4 - Una orden solo avanza a comanda con el pago confirmado (Priority: P1)

Cualquiera sea el método elegido, una orden solo puede enviarse a producción/cocina cuando
tiene un intento de pago confirmado. Mientras el pago esté pendiente (efectivo sin confirmar,
o transferencia sin revisar) o haya sido rechazado, la orden no está disponible para enviar a
comanda.

**Why this priority**: es la regla que materializa el objetivo de negocio completo — sin ella,
la verificación del cajero (User Story 2 y 3) sería solo informativa y no bloquearía nada. Es
además la garantía de que un intento de pago nunca se confirme dos veces por error.

**Independent Test**: se puede probar intentando enviar a comanda una orden con pago
pendiente, otra con pago rechazado, y otra con pago confirmado, y verificando que solo la
última lo permite; y confirmando dos veces seguidas el mismo intento de pago para verificar
que solo la primera tiene efecto.

**Acceptance Scenarios**:

1. **Given** una orden cuyo único intento de pago está pendiente de revisión, **When** se
   intenta enviarla a comanda, **Then** el sistema lo impide.
2. **Given** una orden cuyo único intento de pago fue rechazado, **When** se intenta enviarla
   a comanda, **Then** el sistema lo impide, incluso si el participante ya vio el rechazo.
3. **Given** una orden con un intento de pago confirmado (efectivo confirmado o comprobante
   aprobado), **When** se intenta enviarla a comanda, **Then** el sistema lo permite.
4. **Given** un intento de pago ya confirmado, **When** se intenta confirmarlo una segunda vez
   (doble clic del cajero, o dos cajeros casi simultáneos), **Then** el sistema registra una
   única confirmación válida y la segunda no tiene efecto.

---

### User Story 5 - El participante reintenta el pago tras un comprobante rechazado (Priority: P2)

Cuando el cajero rechaza el comprobante de una orden, el participante puede iniciar un nuevo
intento de pago sobre esa misma orden — eligiendo de nuevo un método y, si es transferencia,
subiendo un nuevo comprobante — sin perder los productos que ya había seleccionado.

**Why this priority**: sin esta historia, un rechazo dejaría la orden sin salida posible y
obligaría a rehacer el pedido completo; es la historia que hace que el rechazo sea una
corrección, no un callejón sin salida. Prioridad P2 porque depende de que exista al menos un
rechazo (User Story 2) para poder ejercitarse.

**Independent Test**: se puede probar rechazando el comprobante de una orden, verificando que
sus productos siguen intactos, y luego subiendo un nuevo comprobante para la misma orden y
comprobando que el cajero lo ve como un intento nuevo, separado del rechazado.

**Acceptance Scenarios**:

1. **Given** una orden con su único comprobante rechazado, **When** el participante vuelve a
   la pantalla de pago, **Then** los productos de su pedido siguen siendo los mismos que había
   seleccionado antes del rechazo.
2. **Given** una orden con un intento de pago rechazado, **When** el participante sube un
   nuevo comprobante, **Then** el sistema crea un nuevo intento de pago para esa misma orden,
   y conserva el intento anterior (rechazado) en el historial.
3. **Given** una orden con un primer intento rechazado y un segundo intento aprobado, **When**
   el cajero consulta el historial de la orden, **Then** ambos intentos son visibles con su
   estado y, en el caso del rechazado, su motivo.

---

### User Story 6 - Un participante solo puede tener una orden activa a la vez (Priority: P3)

Mientras un participante tenga una orden sin finalizar (pago aún no confirmado, o confirmado
pero la orden no ha terminado su ciclo), el sistema no le permite iniciar un segundo pedido.
Una vez esa orden finaliza, el participante puede crear una nueva sin restricciones.

**Why this priority**: evita que un mismo participante acumule órdenes de pago pendiente en
paralelo, lo que complicaría innecesariamente la revisión del cajero. Prioridad P3 porque es
una restricción de guardarraíl, no el corazón del flujo de pago.

**Independent Test**: se puede probar creando una orden para un participante, intentando crear
una segunda antes de que la primera finalice (debe rechazarse), y luego repitiendo el intento
después de que la primera finalice (debe permitirse).

**Acceptance Scenarios**:

1. **Given** un participante con una orden pendiente de pago en su mesa, **When** intenta
   crear un segundo pedido, **Then** el sistema lo rechaza.
2. **Given** un participante con una orden ya finalizada, **When** intenta crear un nuevo
   pedido, **Then** el sistema lo permite sin restricciones.

---

### Edge Cases

- **El tenant solo tiene efectivo habilitado**: el participante ve únicamente esa opción al
  pagar; no se le muestra ningún método de transferencia.
- **El tenant intenta dejar cero métodos de pago activos**: el sistema lo impide — siempre
  debe quedar al menos un método habilitado (User Story 1, escenario 3).
- **El monto del comprobante no coincide con el total de la orden**: el sistema no valida
  montos automáticamente; el cajero decide, y si rechaza por esa razón, registra el motivo
  correspondiente (por ejemplo, "el monto no coincide").
- **Reintentos sucesivos de comprobantes rechazados**: no hay límite de reintentos; el
  participante puede subir un nuevo comprobante cada vez que el anterior sea rechazado.
- **El participante intenta cambiar de método o subir otro comprobante mientras el intento
  anterior sigue pendiente de revisión**: el sistema lo impide — solo puede existir un intento
  de pago activo por orden a la vez; un nuevo intento solo se habilita tras el rechazo explícito
  del anterior (FR-015a).
- **Dos cajeros intentan aprobar/rechazar el mismo comprobante casi al mismo tiempo**: solo la
  primera decisión que llegue al sistema queda registrada; la segunda no tiene efecto, siguiendo
  la misma garantía de una sola confirmación válida por intento de pago (User Story 4).
- **Un participante intenta crear una segunda orden con un nombre distinto en la misma mesa**:
  esta especificación no agrega ninguna verificación de identidad más allá de la que ya existe
  en el flujo QR (los participantes se identifican solo por nombre, sin cuenta ni login); no es
  una limitación nueva introducida por esta funcionalidad.
- **Una orden queda con pago pendiente indefinidamente porque el participante no vuelve**: fuera
  de alcance de esta especificación (no se definen reglas de expiración o timeout).
- **Cancelación de una orden con un pago pendiente o un comprobante en revisión**: fuera de
  alcance de esta especificación; la cancelación es una funcionalidad ya existente y su
  interacción con pagos en curso no se cubre aquí.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE permitir que cada tenant configure qué métodos de pago están
  disponibles para los participantes de sus mesas, incluyendo activar y desactivar el pago en
  efectivo.
- **FR-002**: El sistema DEBE permitir configurar múltiples métodos de transferencia por
  tenant, cada uno con nombre y con la información de pago que el participante necesita
  (por ejemplo, cuenta, teléfono o código).
- **FR-003**: El sistema DEBE impedir que un tenant quede sin ningún método de pago activo —
  siempre debe existir al menos un método habilitado.
- **FR-004**: El sistema DEBE permitir que el participante seleccione, al pagar, el efectivo o
  cualquiera de los métodos de transferencia que su tenant tenga habilitados en ese momento;
  no debe mostrarle métodos desactivados.
- **FR-005**: El sistema DEBE impedir que un participante cree una nueva orden mientras tenga
  una orden activa sin finalizar.
- **FR-006**: El sistema DEBE permitir que el participante cree una nueva orden una vez su
  orden anterior haya finalizado.
- **FR-007**: El sistema DEBE mantener una orden en estado "pendiente de pago" desde su
  creación hasta que su pago quede confirmado.
- **FR-008**: Cuando el participante seleccione efectivo, el sistema DEBE dejar la orden
  pendiente de confirmación por parte del cajero.
- **FR-009**: El sistema DEBE permitir que el cajero registre el monto recibido en efectivo y
  confirme el pago.
- **FR-010**: El sistema DEBE calcular y registrar el cambio (monto recibido menos total de la
  orden) cuando el monto recibido en efectivo supere el total; el cálculo de cambio solo
  aplica a pagos en efectivo.
- **FR-010a**: El sistema DEBE impedir que el cajero confirme un pago en efectivo cuyo monto
  recibido sea menor al total de la orden; el monto recibido debe ser igual o mayor al total.
- **FR-011**: Cuando el participante seleccione un método de transferencia, el sistema DEBE
  mostrarle la información de pago que el tenant configuró para ese método específico.
- **FR-012**: El sistema DEBE permitir que el participante cargue un comprobante de
  transferencia (un único archivo por comprobante), asociándolo tanto a la orden como al
  intento de pago que lo generó.
- **FR-013**: El sistema DEBE permitir que el cajero apruebe o rechace cada comprobante
  recibido; ni el participante ni ningún otro rol pueden autoconfirmar un pago.
- **FR-014**: Cuando el cajero rechace un comprobante, el sistema DEBE exigir y registrar un
  motivo de rechazo, visible en el historial de la orden para el cajero/back-office; el
  participante ve que el comprobante fue rechazado, pero no ve el detalle del motivo.
- **FR-015**: El sistema DEBE permitir que el participante realice un nuevo intento de pago
  sobre la misma orden cuando el comprobante de su intento anterior haya sido rechazado, sin
  perder los productos ya seleccionados.
- **FR-015a**: El sistema DEBE impedir que el participante inicie un nuevo intento de pago
  (cambiar de método o subir otro comprobante) mientras la orden tenga un intento de pago
  pendiente de revisión; el nuevo intento solo se habilita después de que el intento anterior
  quede explícitamente rechazado.
- **FR-016**: El sistema DEBE conservar el historial completo de todos los intentos de pago de
  una orden (rechazados y confirmados), no solo el último.
- **FR-017**: El sistema DEBE permitir enviar una orden a comanda únicamente cuando exista un
  intento de pago confirmado para esa orden, e impedirlo mientras el pago esté pendiente o
  todos sus intentos hayan sido rechazados.
- **FR-018**: El sistema DEBE impedir que un mismo intento de pago sea confirmado (o
  aprobado/rechazado) más de una vez; ante intentos casi simultáneos, solo el primero surte
  efecto.

### Key Entities *(include if feature involves data)*

- **Método de Pago**: opción de cobro configurada por un tenant. Puede ser efectivo o
  transferencia. Tiene nombre, estado (activo/inactivo) y, si es transferencia, los datos de
  pago que el participante necesita ver (cuenta, teléfono, código, u otro identificador según
  el método).
- **Orden**: pedido de un participante en una mesa. Relevante a esta spec: su estado de pago
  ("pendiente de pago" hasta que exista un intento de pago confirmado), y su elegibilidad para
  pasar a comanda.
- **Intento de Pago**: cada vez que un participante intenta pagar una orden con un método
  específico. Tiene un estado (pendiente, confirmado, rechazado), pertenece a una única orden,
  y una orden puede tener varios intentos a lo largo del tiempo (por ejemplo, tras un rechazo).
  Si es en efectivo, registra el monto recibido y el cambio calculado. Su confirmación,
  aprobación o rechazo ocurre una sola vez.
- **Comprobante**: evidencia de transferencia subida por el participante, compuesta de un único
  archivo. Pertenece siempre a un intento de pago específico (no directamente a la orden). Si es
  rechazado, registra el
  motivo del rechazo.
- **Participante**: cliente en una mesa identificado solo por nombre. Solo puede tener una
  orden activa a la vez.
- **Cajero**: único rol autorizado para confirmar pagos en efectivo y para aprobar o rechazar
  comprobantes.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las órdenes que llegan a comanda tienen, en el momento de enviarse, un
  intento de pago confirmado — ninguna orden con pago pendiente o rechazado llega a
  producción.
- **SC-002**: Un administrador de tenant puede dar de alta un nuevo método de transferencia y
  verlo disponible para los participantes en menos de 2 minutos, sin intervención técnica.
- **SC-003**: El participante nunca ve, al pagar, un método de pago que su tenant tiene
  desactivado en ese momento.
- **SC-004**: El 100% de los rechazos de comprobante quedan con un motivo registrado y visible
  en el historial de la orden — ningún rechazo queda sin justificación trazable.
- **SC-005**: El cálculo del cambio en pagos en efectivo es siempre exacto (monto recibido
  menos total de la orden), eliminando el cálculo manual del cajero.
- **SC-006**: Tras un rechazo, el 100% de los productos previamente seleccionados por el
  participante se conservan al reintentar el pago — ningún reintento obliga a rehacer el
  pedido.
- **SC-007**: Ante confirmaciones o resoluciones simultáneas sobre el mismo intento de pago, el
  sistema registra siempre exactamente una decisión válida, nunca dos contradictorias.

## Out of Scope

- Integración directa vía API/webhook con las pasarelas o bancos (Nequi, Daviplata,
  Bancolombia, Bre-B, etc.); el proceso sigue siendo manual, vía comprobante y verificación
  humana del cajero.
- Consolidación de la cuenta por mesa (modo "cuenta completa" vs. "dividida por persona"); en
  este flujo cada orden se paga individualmente por participante.
- Cancelación de órdenes y su interacción con pagos en curso (la cancelación ya existe como
  funcionalidad aparte del sistema).
- Reglas de expiración o timeout automático de pagos pendientes.
- Definición del evento que marca una orden como "finalizada"; esta spec asume que ese evento
  existe y es reconocible por el sistema (dependencia del flujo de mesa/orden ya existente o de
  otra iteración futura), pero no lo define aquí.
- Generación del movimiento de caja e inventario asociado a la venta; se asume que se dispara
  automáticamente al confirmarse el pago, según el flujo ya existente.
- Propinas, descuentos, cupones o pagos mixtos/parciales dentro de una misma orden.
- Verificación de identidad del participante más allá del nombre (el modelo sin cuenta ni login
  del flujo QR existente se mantiene sin cambios).
- Restricciones de formato o tamaño de archivo para el comprobante cargado (se asume un manejo
  estándar de imágenes/documentos, sin límites de negocio específicos definidos en esta spec).

## Assumptions

- **El rol "administrador del tenant"** es quien configura los métodos de pago del negocio
  (User Story 1); no se especificó el nombre exacto del rol en los requerimientos recibidos,
  se asume que es quien ya administra la configuración general del tenant.
- **Siempre debe quedar al menos un método de pago activo por tenant** (FR-003): es el
  comportamiento por defecto elegido para el caso "el tenant no habilita ningún método de
  pago", ya que dejar a los participantes sin ninguna forma de pagar bloquearía el flujo
  completo sin ningún camino de salida.
- **No hay límite de reintentos de pago tras rechazos sucesivos**: se asume que el participante
  puede subir un nuevo comprobante tantas veces como sea necesario hasta que uno sea aprobado.
- **El cajero no valida montos automáticamente**: si el monto de un comprobante no coincide con
  el total de la orden, la decisión de aprobar o rechazar (con motivo) queda enteramente en
  manos del cajero; el sistema no compara montos por sí mismo.
- **El evento que marca una orden como "finalizada"** (del cual depende cuándo un participante
  puede crear una nueva orden, User Story 6) no se define en esta especificación — se asume
  que ya existe o se resolverá como parte del flujo de mesa/orden existente o de otra
  iteración, según quedó explícitamente fuera de alcance.
- **La identificación del participante solo por nombre, sin autenticación**, se hereda sin
  cambios del flujo QR existente; esta especificación no introduce ni resuelve ninguna
  verificación adicional de identidad.
- **El formato y tamaño del comprobante cargado** no tiene restricciones de negocio definidas
  en esta spec; se asume un manejo estándar de archivos de imagen/documento.
- **La cancelación de órdenes, su interacción con pagos en curso, y las reglas de expiración de
  pagos pendientes** quedan fuera de alcance, tal como se indicó explícitamente en los
  requerimientos recibidos.
