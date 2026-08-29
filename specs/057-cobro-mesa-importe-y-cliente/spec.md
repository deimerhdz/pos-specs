# Feature Specification: Importe fijo para pagos no efectivo y nombre de cliente en el desglose de cobro

**Feature Branch**: `057-cobro-mesa-importe-y-cliente`

**Created**: 2026-08-29

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección de comportamiento sobre pantallas de cobro ya
existentes**, encontrada por el dueño/desarrollador del proyecto al probar manualmente en el
navegador el flujo de "Cuenta de la mesa" (verificación pendiente tras implementar spec 056).
Ninguno de los dos ajustes toca el modelo de datos ni agrega campos nuevos: ambos son correcciones
de UI sobre un componente de pago ya compartido por dos pantallas distintas de cobro.

**Alcance concreto sobre el sistema actual**: el campo de importe del pago vive en un único
componente compartido, `PaymentInputComponent`
(`pos-heladeria/src/app/modules/tables/components/payment-input.component.ts`), usado en los dos
únicos puntos donde el sistema captura hoy el importe de un cobro: dentro de
`session-bill-panel.component.ts:144-148` ("Cuenta de la mesa", cierre de sesión) y directamente en
`pos-checkout-panel.component.ts:155-159` ("Cobrar pedido"/"Pedido de mostrador", `checkout-and-
send`). El componente ya distingue efectivo de no-efectivo (`isCash(methodId)`,
`payment-input.component.ts:92-94`) y ya precarga el importe con el total exacto al elegir un
método (`setMethod()`, línea 97-99) — pero el campo `<app-money-input>` (líneas 52-56) queda
editable siempre, sin ninguna condición sobre el método elegido. La regla de negocio de que un
cobro no efectivo nunca debe superar el total ya existe y ya se valida
(`payment-draft.util.ts:51-54,71-73`: *"Un cobro electrónico por encima del total dejaría un
faltante fantasma en el cajón"*, `paymentIssue()` ya rechaza ese caso) — lo que falta es impedir
directamente que el cajero escriba un valor distinto al total cuando el método no es efectivo, en
vez de solo rechazarlo después de escrito.

Por separado, en el desglose por comensal de "Cuenta de la mesa" (`bill.split`,
`session-bill-panel.component.ts`), la línea de ítems que el mesero agregó sin asignarlos a ningún
comensal (`participant_id` nulo) siempre se rotula como *"Sin asignar (mesero)"*
(`lineLabel()`, líneas 271-273), sin importar si la orden correspondiente sí tiene un nombre de
cliente guardado. El componente **ya recibe** ese nombre por `@Input customerName`
(línea 174), que ambos llamadores le pasan como `store.customerName()`
(`pos-checkout-panel.component.ts:79,101`) — el mismo signal que las specs 054/055/056 ya usan para
mostrar/editar el campo "Cliente" al crear una orden, siempre derivado del `customer_name` real de
la orden seleccionada (`pos-terminal.store.ts:1022,1042,1054`). Hoy ese `@Input` solo se usa para
armar el `customer_name` que se envía al backend al cobrar (`buildPayload()`, línea ~309) —
nunca para reemplazar la etiqueta "Sin asignar (mesero)" que ve el cajero.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-29, tras probar en vivo el flujo de cobro de mesa.
Introduce dos decisiones de negocio: (1) un cobro no efectivo deja de ser editable — el cajero ya
no puede escribir manualmente el importe de una transferencia u otro método distinto a efectivo, el
sistema siempre cobra el total exacto; y (2) el desglose de "Cuenta de la mesa" prioriza mostrar el
nombre de cliente de la orden por encima de la etiqueta genérica "Sin asignar (mesero)" cuando ese
nombre existe. Ninguna reabre ninguna decisión de precio, inventario ni facturación de specs
anteriores — ambas ajustan cómo se presenta/edita información que el sistema ya calcula y ya
guarda.

**Input**: User description (verbatim): "en esta parte del flujo no deberia poder editar el
importe cuando la opcion de pago es diferente de efectivo y lo otro es que debe de salir donde
dice sin asignar(mesero) el nombre del cliente customer name de la orden"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El importe de un pago no efectivo no se puede editar (Priority: P1)

Un cajero está cobrando una mesa (o un pedido de mostrador) y elige "Nequi" (o cualquier método
distinto a efectivo) como forma de pago. El sistema ya precarga el campo "Importe" con el total
exacto a cobrar, pero hoy el cajero todavía puede escribir encima y cambiarlo por error a un valor
distinto — con efectivo, en cambio, sí necesita poder escribir "con cuánto paga" el cliente para
calcular el vuelto. Con esta mejora, en cuanto el cajero elige un método distinto a efectivo, el
campo de importe queda fijo en el total: no hay nada que escribir ni que equivocar.

**Why this priority**: es el ajuste explícitamente pedido por el dueño del proyecto para prevenir
un error real de cobro (un importe de transferencia mal escrito descuadra el arqueo del turno, como
ya documenta el propio código del sistema) — protege dinero real, no solo una molestia de UI.

**Independent Test**: puede probarse por completo abriendo el cobro de una mesa o de un pedido de
mostrador, eligiendo un método de pago distinto a efectivo, y verificando que el campo "Importe"
no admite edición y muestra siempre el total exacto; y que, al elegir efectivo, el campo "Con
cuánto paga" sigue siendo editable como hoy.

**Acceptance Scenarios**:

1. **Given** el cajero está cobrando una mesa o un pedido de mostrador, **When** elige un método de
   pago distinto a efectivo (p. ej. "Nequi"), **Then** el campo de importe muestra el total exacto
   a cobrar y no permite escribir ni modificar ese valor.
2. **Given** el campo de importe está fijo por haber elegido un método no efectivo, **When** el
   cajero intenta hacer clic o escribir sobre el campo, **Then** el campo no cambia de valor.
3. **Given** el cajero eligió un método no efectivo y luego cambia a "Efectivo", **When** observa
   el campo, **Then** vuelve a quedar editable, con el mismo comportamiento de "con cuánto paga"
   que ya existe hoy (precargado con el total, editable para calcular el vuelto).
4. **Given** el cajero está en el flujo de "Cobrar pedido"/"Pedido de mostrador" (no en "Cuenta de
   la mesa"), **When** elige un método no efectivo, **Then** el mismo comportamiento de campo fijo
   aplica igual, porque ambas pantallas comparten el mismo componente de importe.

---

### User Story 2 - El desglose de "Cuenta de la mesa" muestra el nombre del cliente en vez de "Sin asignar (mesero)" (Priority: P2)

Un cajero abre la "Cuenta de la mesa" para cobrar. En el desglose por comensal, la línea de los
ítems que el mesero agregó directamente (sin asignarlos a ningún comensal) hoy siempre dice "Sin
asignar (mesero)", aunque la orden correspondiente ya tenga un nombre de cliente guardado (por
ejemplo "Consumidor final", o un nombre real editado al crear el pedido). Con esta mejora, esa
línea muestra ese nombre de cliente en su lugar, y solo cae de vuelta a "Sin asignar (mesero)"
cuando la orden no tiene ningún nombre de cliente guardado.

**Why this priority**: es una mejora de claridad para el cajero (saber a nombre de quién factura,
no solo que "fue el mesero") — útil pero no protege dinero como la Historia 1, de ahí la prioridad
menor.

**Independent Test**: puede probarse por completo abriendo la "Cuenta de la mesa" de una orden con
nombre de cliente guardado y con ítems sin comensal asignado, y verificando que esa línea del
desglose muestra el nombre del cliente en vez de "Sin asignar (mesero)".

**Acceptance Scenarios**:

1. **Given** una mesa con una orden que tiene un nombre de cliente guardado (p. ej. "Consumidor
   final" o un nombre editado) y con ítems sin comensal asignado, **When** el cajero abre "Cuenta
   de la mesa", **Then** la línea de esos ítems muestra el nombre del cliente de la orden, no "Sin
   asignar (mesero)".
2. **Given** una mesa con ítems sin comensal asignado cuya orden **no** tiene ningún nombre de
   cliente guardado, **When** el cajero abre "Cuenta de la mesa", **Then** esa línea sigue
   mostrando "Sin asignar (mesero)" (mismo comportamiento de hoy, sin regresión).
3. **Given** una línea del desglose que sí corresponde a un comensal identificado (no al mesero),
   **When** el cajero la observa, **Then** sigue mostrando el nombre/etiqueta de ese comensal, sin
   ningún cambio (esta mejora solo afecta la línea "sin asignar").

---

### Edge Cases

- ¿Qué pasa si el cajero selecciona un método no efectivo y el total de la cuenta cambia después
  (p. ej. la cuenta se actualiza)? Fuera de alcance definir un comportamiento distinto al que ya
  existe hoy para ese caso (el componente ya reinicia el pago cuando cambia la cuenta) — el importe
  fijo simplemente vuelve a precargarse con el nuevo total.
- ¿Qué pasa si una mesa tiene más de una orden activa, cada una con un nombre de cliente distinto,
  y ambas tienen ítems sin comensal asignado? Fuera de alcance resolver cuál nombre mostrar en ese
  caso particular — esta mejora reutiliza el nombre de cliente ya disponible para la orden
  actualmente seleccionada, mismo criterio que ya usa este mismo panel para el nombre de
  facturación; no introduce un desglose por orden que no existe hoy.
- ¿Aplica este mismo criterio de "importe fijo" a la cuenta dividida por comensal? Sin efecto
  adicional que declarar — spec 046 ya retiró esa opción del frontend (un único método/importe por
  cobro), así que esta mejora sigue aplicando sobre ese mismo cobro único, sin reabrir esa decisión.
- ¿Qué pasa con el campo "Con cuánto paga" (efectivo) cuando el total es $0? Fuera de alcance —
  mismo comportamiento que ya existe hoy para ese caso límite, sin cambios de esta mejora.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE impedir editar el campo de importe de un cobro cuando el método de
  pago elegido no es efectivo — el campo permanece siempre en el total exacto a cobrar.
- **FR-002**: El sistema DEBE seguir permitiendo editar libremente el campo "con cuánto paga"
  cuando el método de pago elegido es efectivo, sin ningún cambio respecto al comportamiento
  actual.
- **FR-003**: Este comportamiento (importe fijo para no efectivo) DEBE aplicar en todos los puntos
  donde el sistema captura hoy el importe de un cobro (cobro de "Cuenta de la mesa" y cobro de
  pedido de mostrador), sin excepciones entre pantallas.
- **FR-004**: Si el cajero cambia el método de pago elegido de no-efectivo a efectivo (o viceversa)
  antes de confirmar el cobro, el sistema DEBE aplicar de inmediato el comportamiento (fijo o
  editable) correspondiente al método recién elegido.
- **FR-005**: En el desglose por comensal de "Cuenta de la mesa", el sistema DEBE mostrar el nombre
  de cliente de la orden en la línea de ítems sin comensal asignado, cuando ese nombre exista.
- **FR-006**: Cuando la orden de esos ítems no tenga ningún nombre de cliente guardado, el sistema
  DEBE seguir mostrando "Sin asignar (mesero)" en esa línea, sin dejarla vacía.
- **FR-007**: Esta mejora NO DEBE cambiar la etiqueta de ninguna línea del desglose que sí
  corresponda a un comensal identificado.
- **FR-008**: Esta mejora NO DEBE cambiar qué valor se envía al backend como `customer_name` al
  cobrar — solo cambia qué texto ve el cajero en esa línea del desglose.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los cobros con un método de pago distinto a efectivo se registran por el
  total exacto — ningún cajero puede escribir manualmente un importe distinto en ese caso.
- **SC-002**: El cajero elige un método no efectivo y ve el cobro listo para confirmar sin tener
  que escribir ningún importe (cero interacciones adicionales sobre el campo de importe).
- **SC-003**: El desglose de "Cuenta de la mesa" muestra el nombre real del cliente (cuando existe)
  en el 100% de las líneas de ítems sin comensal asignado, en vez de la etiqueta genérica "Sin
  asignar (mesero)".

## Assumptions

- Los dos únicos puntos de captura de importe de cobro en el sistema son
  `session-bill-panel.component.ts` (vía `PaymentInputComponent`) y el uso directo de
  `PaymentInputComponent` en `pos-checkout-panel.component.ts` — corregir el componente compartido
  corrige ambos puntos a la vez, sin necesidad de tocarlos por separado.
- El nombre de cliente que se reutiliza para la línea "Sin asignar (mesero)" es el mismo que ya
  está disponible como `@Input customerName` en `session-bill-panel.component.ts` (derivado de la
  orden actualmente seleccionada) — no se introduce ninguna consulta ni cálculo nuevo, solo se usa
  un dato que el componente ya recibe.
- El caso de una mesa con varias órdenes con nombres de cliente distintos queda fuera de alcance
  (ver Edge Cases) — esta mejora no introduce un desglose de nombre por orden dentro de la línea
  "sin asignar", solo reutiliza el nombre ya disponible para la orden seleccionada.
- No se agrega ningún campo nuevo al modelo de datos ni al contrato de ningún endpoint — ambos
  ajustes son de presentación/edición en el frontend, sobre datos que el sistema ya calcula
  (`total`) o ya guarda (`customer_name`).
