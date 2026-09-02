# Feature Specification: Corrección — la Terminal de mesas no detecta un turno de caja que sí está abierto

**Feature Branch**: `072-fix-deteccion-turno-caja`

**Created**: 2026-09-02

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección**, no una funcionalidad nueva desde cero. Igual
que las specs [019](../019-correccion-cuenta-mesas-fusionadas/spec.md),
[029](../029-correccion-cobro-cierre-mesa/spec.md),
[050](../050-correccion-liberar-mesa-pedido-cancelado/spec.md) y
[069](../069-fix-creacion-tenant/spec.md), cita nombres de archivo, función y línea del código
actual (`pos-heladeria`) porque son el contrato observable que se está corrigiendo, no una fuga
de detalles de implementación. Es un bug preexistente, introducido por la spec
[059](../059-terminal-mesas-carga-y-pedidos/spec.md) (Historia 1, "carga diferida" del turno de
caja): el disparador que decide cuándo pedir el turno de caja solo cubre la selección **activa**
de una mesa/pedido, no la llegada **reactiva** de un pedido nuevo a una mesa que ya estaba
seleccionada — y se agrava con un mecanismo más antiguo (`localStorage`) que tampoco cubre el caso
de un dispositivo que nunca "operó" una caja desde ese mismo navegador.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-09-02, con tres capturas de pantalla que reproducen el
bug en el tenant real "springfield granizados" (usuario `stremighomero`, Administrador). No
reabre ninguna regla de negocio de cobro, caja ni facturación: el fix es exclusivamente sobre
**cómo se detecta** si hay un turno de caja abierto, no sobre si debe o no exigirse uno para
cobrar (esa exigencia, introducida por la spec 028, se mantiene intacta — ver FR-003). No aplica
una nueva entrada en `registro-de-anomalias.md`: restaura el comportamiento pretendido, no cambia
una regla de negocio deliberada.

**Input**: User description (verbatim, con tres capturas de pantalla): "cuando recibo un pedido
por primera vez desde una mesa, no me deja confirmar el pago porque dice que necesito abrir una
caja, pero cuando voy a la caja veo que la sesión esta abierta, entonces al regresar de nuevo a
la terminal me sale ya como si estuviera habilitado para poder cobrar el pago de la orden". Las
capturas muestran, en orden: (1) la Terminal de mesas con un pedido "Por confirmar" de $16.000 en
efectivo, el botón "Confirmar efectivo" deshabilitado y el mensaje "Abre un turno de caja para
poder confirmar el pago."; (2) el módulo de Caja mostrando la caja "Principal" con "Turno abierto"
(cajero Erian Espitia, apertura 26/08/2026 15:58, efectivo esperado $289.000); (3) la misma
Terminal de mesas, mismo pedido, ahora con "Confirmar efectivo" habilitado.

Investigación posterior (leyendo el código real de `pos-heladeria`) confirmó **dos causas que se
combinan**: un disparador que no cubre el caso reportado, y un mecanismo de respaldo que puede
quedarse pegado en "no hay turno" sin reintentar.

**Causa principal — el pedido llega de forma reactiva, no por una selección activa**:

- El panel de pago ("Pagos por confirmar") se pinta a partir de un signal puramente **reactivo**:
  `PosTerminalStore.pendingOfSelectedTable` (`src/app/modules/tables/services/
  pos-terminal.store.ts:520-523`) se recalcula solo, sin ninguna condición, cada vez que cambia
  `orders()` — y `orders()` se actualiza sola por polling (`reloadOrders()`,
  `pos-terminal.store.ts:1074-1081`) y por eventos en tiempo real (`connectRealtime()`,
  líneas 1026-1047) cuando el comensal envía un pedido por QR. `table-sessions.component.ts:162`
  liga ese resultado directo al panel de cobro (`[orders]="store.pendingOfSelectedTable()"`).
- El turno de caja, en cambio, **solo** se pide de forma imperativa desde
  `ensureCheckoutDataLoaded()` (`pos-terminal.store.ts:963-971`, spec 059 Historia 1), y **solo**
  la disparan `selectTable()` (línea 1153, únicamente `if (list.length > 0)`, es decir, solo si la
  mesa **ya tenía** un pedido en el instante del clic) y `selectStandaloneOrder()` (línea 1182).
  Ninguna de las dos se vuelve a ejecutar cuando `orders()` cambia por sí sola.
- Consecuencia: si el cajero ya tenía seleccionada una mesa **vacía** (esperando el pedido, el caso
  típico de "primera vez que llega un pedido de esa mesa") y el comensal envía su pedido por QR, el
  panel de cobro aparece solo por la vía reactiva — sin que `ensureCheckoutDataLoaded()` se haya
  ejecutado nunca para ese pedido —, así que `cashShiftId`
  (`this.cash.isOpen() ? (this.cash.shift()?.id ?? null) : null`, líneas 1597-1598) sigue
  evaluando sobre lo que `cash.shift()` ya tuviera (típicamente `null`, porque en una pantalla
  recién cargada nada más lo puebla) y muestra "Abre un turno de caja para poder confirmar el
  pago." (`payment-attempt-review-panel.component.ts:74-75`) aunque el turno sí esté abierto.

**Causa que se suma — el respaldo de `restoreShift()` puede fallar en silencio y no reintentar**:

- `CashService.restoreShift()` (`src/app/modules/cash-register/services/cash.service.ts:50-63`),
  que es lo que `ensureCheckoutDataLoaded()` sí invoca cuando llega a ejecutarse, lee
  `localStorage.getItem('cash.register')` y, si esa clave está vacía (dispositivo que nunca
  "operó" una caja desde este navegador), **retorna de inmediato sin llamar al backend y sin tocar
  la señal `shift`**, que queda en `null`.
- Por la regla de caché de la spec 059 (FR-003, "una vez cargado no se vuelve a pedir"), aunque el
  cajero seleccione después esa misma mesa (ahora con el pedido ya visible) y
  `ensureCheckoutDataLoaded()` sí se ejecute, el chequeo `this.cash.shift() ? null :
  this.cash.restoreShift()` ya vio `shift()` en `null` una vez y **no vuelve a intentarlo** dentro
  de esa sesión de pantalla si el primer intento no encontró nada.
- `localStorage['cash.register']` solo se escribe al hacer clic en "Operar" sobre una caja desde
  el módulo de Caja (`CashSessionStore.operateRegister()`,
  `src/app/modules/cash-register/services/cash-session.store.ts:299-314`) — es un recordatorio por
  **navegador/dispositivo** de "la última caja que operó desde aquí", no una consulta de "hay
  algún turno abierto ahora".

**Por qué visitar Caja "arregla" la Terminal**: `operateRegister()` escribe
`localStorage['cash.register']` y llama `loadShiftData()` → `this.shift.set(shift)`
(`cash-session.store.ts:399-400`), sin ninguna de las dos limitaciones anteriores. Como
`CashSessionStore.shift` es literalmente la misma señal que `CashService.shift`
(`readonly shift = this.api.shift;`, línea 95), esa acción puebla por primera vez, desde una vía
totalmente distinta, el mismo estado compartido que la Terminal lee — dando la impresión de que
"revisar la caja" fue lo que lo arregló, cuando en realidad cualquier acción que poblara
`cash.shift()` habría funcionado igual.

**Evidencia de que es un vacío real, no una decisión ya tomada**: la prueba existente
`pos-terminal.store.spec.ts:1499-1507` ("seleccionar una mesa libre (sin pedido) tampoco los pide
(FR-001)") confirma **a propósito** que seleccionar una mesa vacía no dispara el chequeo de turno
— correcto en sí mismo, pero ninguna prueba cubre qué pasa cuando esa misma mesa, ya seleccionada,
recibe después un pedido por la vía reactiva (`orders.set(...)`/polling/tiempo real) en vez de por
un nuevo clic de selección. La prueba de la línea 1509-1538 también **pre-siembra**
`localStorage.setItem('cash.register', 'reg-1')` antes de ejercitar el flujo, sin cubrir el
arranque en frío de esa clave.

---

## Clarifications

### Session 2026-09-02

- Q: Una vez que aparece el panel de cobro para un pedido recién llegado, ¿qué tan rápido debe
  reflejar correctamente si se puede cobrar en efectivo: al instante, o es aceptable un momento
  breve (uno o dos segundos) mientras se verifica en silencio? → A: **Un momento breve (menos de
  ~2 segundos) es aceptable.** El botón puede pasar de deshabilitado a habilitado un instante
  después de que aparece el panel, mientras el sistema verifica el turno en ese momento — no hace
  falta que el dato ya esté disponible antes de que el panel se pinte. Esto conserva intacto el
  principio de la spec 059 (no cargar los datos de cobro por adelantado, solo cuando algo se
  vuelve cobrable) y evita revertirlo; solo se amplía **cuándo** se dispara esa carga (FR-007).

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Cobrar el pedido que acaba de llegar a una mesa ya seleccionada (Priority: P1)

Un cajero tiene una mesa vacía seleccionada en la Terminal, esperando a que el comensal pida desde
el QR. En cuanto el pedido llega y el panel de cobro aparece, con un turno de caja genuinamente
abierto (sin importar desde qué dispositivo se abrió), debe poder confirmar el pago de inmediato
— sin tener que volver a seleccionar la mesa, ni mucho menos ir primero al módulo de Caja "para
que se entere".

**Why this priority**: es el defecto reportado, y el camino más común para reproducirlo: el
cajero deja una mesa seleccionada esperando el pedido (no vuelve a hacer clic sobre ella cuando
éste llega), así que el chequeo de turno de caja —que hoy solo se dispara al seleccionar
activamente una mesa que ya tenía pedido— nunca llega a ejecutarse para ese pedido nuevo.

**Independent Test**: con un turno de caja abierto, seleccionar una mesa libre en la Terminal, y
—sin volver a tocarla— hacer que llegue un pedido nuevo a esa mesa (por QR o por polling/tiempo
real simulado). Cuando aparezca el panel "Pagos por confirmar", "Confirmar efectivo" debe estar
habilitado sin pasos previos.

**Acceptance Scenarios**:

1. **Given** un turno de caja abierto y una mesa libre ya seleccionada en la Terminal, **When**
   llega el primer pedido de esa mesa (sin que el cajero vuelva a seleccionarla) y aparece el
   panel "Pagos por confirmar", **Then** "Confirmar efectivo" queda habilitado en menos de ~2
   segundos desde que aparece el panel (puede verificarse en ese momento, no antes), sin quedar
   mostrando "Abre un turno de caja para poder confirmar el pago." más allá de ese margen.
2. **Given** el mismo escenario, **When** el cajero abre el panel de cobro directo de esa mesa
   (no solo el pedido "por confirmar" del QR), **Then** también puede seleccionar método de pago y
   cobrar, sin necesitar visitar antes el módulo de Caja.
3. **Given** un turno de caja abierto desde otro dispositivo (por otro usuario) y un
   navegador/dispositivo de Terminal que nunca visitó el módulo de Caja en esta sesión, **When**
   el cajero selecciona activamente una mesa que ya tiene un pedido y luego intenta cobrar,
   **Then** el sistema también lo permite — el defecto no depende de qué dispositivo abrió el
   turno.
4. **Given** la Terminal ya detectó correctamente el turno abierto para un pedido, **When** llega
   un segundo pedido de otra mesa en la misma sesión de pantalla, **Then** también puede cobrarse
   de inmediato, sin repetir ningún paso.

---

### User Story 2 - El bloqueo se mantiene cuando de verdad no hay turno abierto (Priority: P1)

El mismo mecanismo que hoy impide cobrar sin una caja abierta (spec 028) sigue funcionando
exactamente igual cuando **de verdad** no hay ningún turno de caja abierto en el tenant.

**Why this priority**: la corrección no puede convertir el candado en un adorno — si el fix
termina por permitir cobrar sin ningún turno abierto en ningún lado, se pierde el control de caja
que la spec 028 introdujo a propósito.

**Independent Test**: sin ningún turno de caja abierto en el tenant, recibir un pedido en una
mesa e intentar confirmar el pago — debe seguir bloqueado con el mismo mensaje de hoy.

**Acceptance Scenarios**:

1. **Given** ningún turno de caja abierto en el tenant, **When** el cajero intenta confirmar o
   aprobar el pago de un pedido, **Then** el sistema lo sigue bloqueando y muestra "Abre un turno
   de caja para poder confirmar/aprobar el pago.", igual que hoy.
2. **Given** un turno de caja que estaba abierto se cierra, **When** el cajero abre por primera
   vez, después de ese cierre, un pedido nuevo para cobrar, **Then** el sistema ya no lo deja
   cobrar sin abrir un turno nuevo.

---

### User Story 3 - Más de una caja abierta a la vez sigue pidiendo elegir cuál (Priority: P2)

Un tenant con más de una caja registrada tiene, en un momento dado, dos o más turnos abiertos
simultáneamente. El sistema no debe adivinar con cuál cobrar.

**Why this priority**: cobrar en efectivo contra la caja equivocada descuadra el arqueo de ambas
cajas — es un riesgo financiero, no solo de usabilidad; por eso el criterio por defecto debe ser
conservador (seguir pidiendo selección explícita) en vez de adivinar.

**Independent Test**: con dos cajas distintas, cada una con su propio turno abierto, intentar
cobrar desde la Terminal sin haber operado ninguna explícitamente — el sistema debe pedir elegir
cuál, en vez de cobrar contra una al azar.

**Acceptance Scenarios**:

1. **Given** dos turnos de caja abiertos a la vez (cajas distintas), **When** el cajero intenta
   cobrar sin haber operado explícitamente ninguna de las dos desde ese dispositivo, **Then** el
   sistema le pide seleccionar la caja correspondiente (mismo flujo ya existente de "Operar"),
   en vez de cobrar automáticamente contra cualquiera de las dos.
2. **Given** el cajero ya operó explícitamente una de las dos cajas desde ese dispositivo,
   **When** cobra un pedido, **Then** el cobro se atribuye a esa caja, sin ambigüedad.

---

### Edge Cases

- **Mesa vacía ya seleccionada que recibe un pedido por vía reactiva (polling o tiempo real)**: es
  el caso principal de esta corrección — el panel de cobro debe reflejar el turno de caja
  realmente abierto aunque nadie haya vuelto a hacer clic sobre la mesa (User Story 1).
- **Pedido de Domicilio/Para llevar nuevo mientras el cajero no tiene nada seleccionado**: mismo
  criterio — el panel de cobro que aparece para ese pedido debe reflejar el turno abierto sin
  exigir que el cajero lo seleccione primero para "activar" el chequeo.
- **Ningún turno de caja abierto en ningún lado**: el bloqueo se mantiene (User Story 2).
- **Dos o más turnos abiertos a la vez**: no se adivina; se exige selección explícita (User Story
  3) — evita atribuir un cobro a la caja equivocada.
- **El turno se cierra mientras la Terminal ya lo había detectado como abierto en esa sesión de
  pantalla**: fuera de alcance de esta corrección (ver Assumptions); el cajero se entera al
  intentar el siguiente cobro después de recargar o reabrir la pantalla, igual que otros datos
  que hoy tampoco se sincronizan en tiempo real entre dispositivos.
- **El navegador de la Terminal nunca visitó el módulo de Caja ni "operó" una caja en esta
  sesión**: no debe impedir detectar un turno genuinamente abierto (User Story 1, escenario 3).
- **Falla de red al verificar el turno abierto**: mismo tipo de error ya manejado hoy para esa
  petición (spec 059, Edge Cases), sin bloquear el resto de la pantalla.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE determinar si hay un turno de caja abierto consultando el estado
  real del servidor, sin depender de si el navegador/dispositivo actual registró previamente
  haber "operado" una caja.
- **FR-002**: Cuando exista exactamente un turno de caja abierto en el tenant, el sistema DEBE
  habilitar la confirmación/aprobación de pago en la Terminal de mesas (y en cualquier otra
  pantalla de cobro que dependa del mismo estado compartido) usando ese turno, sin exigir que el
  usuario lo haya seleccionado manualmente antes desde ese dispositivo.
- **FR-003**: Cuando NO exista ningún turno de caja abierto, el sistema DEBE seguir bloqueando la
  confirmación/aprobación de pago con el mismo mensaje ya existente — sin regresión sobre la
  exigencia introducida por la spec 028.
- **FR-004**: Cuando existan dos o más turnos de caja abiertos simultáneamente, el sistema DEBE
  seguir exigiendo la selección explícita ya existente (visitar Caja y "Operar" la caja
  correspondiente) en vez de elegir una automáticamente.
- **FR-005**: La verificación de turno abierto para habilitar el cobro NO DEBE quedar fija en un
  resultado obtenido en un intento anterior dentro de la misma sesión de pantalla cuando ese
  resultado no reflejaba la realidad — debe ser posible que un cobro que antes aparecía bloqueado
  se refleje como habilitado sin necesitar recargar toda la pantalla, en cuanto la información
  correcta esté disponible.
- **FR-006**: La corrección DEBE aplicar a toda pantalla de cobro que dependa de este mismo estado
  compartido de turno de caja (el panel de "Pagos por confirmar" del QR y el panel de cobro
  directo de una mesa en la Terminal de mesas, como mínimo), no solo al panel donde se reportó el
  defecto.
- **FR-007**: El sistema DEBE reflejar el turno de caja realmente abierto también cuando el panel
  de cobro aparece por la llegada de un pedido nuevo a una mesa (o pedido de Domicilio/Para
  llevar) que el cajero ya tenía seleccionada o a la vista, sin exigir que el cajero vuelva a
  seleccionarla activamente para que el sistema "se entere" de que hay algo para cobrar. La
  verificación puede resolverse en el momento en que el panel aparece (no antes) — un margen de
  hasta ~2 segundos con el control aún reflejando el estado previo mientras se verifica es
  aceptable; no hace falta cargar el turno de caja por adelantado para cumplir esto (Clarifications,
  sesión 2026-09-02).

### Key Entities *(include if feature involves data)*

- **Turno de caja (`CashShift`)**: entidad ya existente (spec 006); esta corrección no le agrega
  ningún atributo ni cambia su ciclo de vida (abrir/cerrar) — solo cambia cómo el cliente descubre
  cuál está abierto.
- **Caja (`CashRegister`)**: entidad ya existente; un tenant puede tener una o varias.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un cajero puede confirmar o aprobar el pago del primer pedido recibido en cualquier
  mesa, desde cualquier dispositivo, en el 100% de los casos en que exista un turno de caja
  abierto en el tenant — sin necesitar visitar antes el módulo de Caja, y con el control de cobro
  reflejando el estado correcto en menos de ~2 segundos desde que el panel de cobro aparece en
  pantalla.
- **SC-002**: Con ningún turno de caja abierto, el 100% de los intentos de confirmar o aprobar un
  pago siguen bloqueados con el mensaje explicativo, sin excepción.
- **SC-003**: Con más de un turno de caja abierto a la vez, el 100% de los intentos de cobro sin
  selección explícita previa piden elegir la caja, sin atribuir nunca un cobro a la caja
  equivocada.

## Assumptions

- La mayoría de los tenants operan con una sola caja ("Principal"); el caso de varias cajas
  abiertas a la vez sigue resolviéndose con la selección manual ya existente ("Operar"), en vez de
  introducir un mecanismo nuevo de elección automática (User Story 3) — evita el riesgo financiero
  de adivinar mal sin necesitar diseñar una interfaz de desambiguación nueva para un caso poco
  frecuente.
- El alcance de esta corrección es exclusivamente **cómo se detecta** si hay un turno abierto; no
  cambia el modelo de datos de `CashShift`/`CashRegister`, ni el motor de cobro, ni la exigencia de
  la spec 028 de tener un turno abierto para cobrar.
- La sincronización en tiempo real entre dispositivos cuando un turno se cierra mientras otro
  cajero ya tiene la Terminal abierta queda fuera de alcance — es un caso distinto (el estado
  vuelve a estar desactualizado, pero en la dirección contraria: de "abierto" a "cerrado") que no
  fue parte del defecto reportado.
- El backend ya expone la información necesaria para descubrir el turno abierto de una caja sin
  depender de `localStorage` (consultar las cajas del tenant y el turno actual de cada una); el
  diseño técnico concreto de cómo el cliente recompone esa información se decide en la fase de
  planeación, no en esta spec.
