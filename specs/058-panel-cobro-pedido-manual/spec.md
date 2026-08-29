# Feature Specification: Ajustes al panel de cobro de pedido manual (nombre de facturación, texto de cobro y acciones post-cobro)

**Feature Branch**: `058-panel-cobro-pedido-manual`

**Created**: 2026-08-29

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección/ajuste de UI sobre una pantalla de cobro ya
existente**, encontrada por el dueño/desarrollador del proyecto al probar en el navegador el panel
de cobro que aparece al crear o seleccionar un pedido de mostrador/mesa antes de enviarlo a cocina.
No toca el modelo de datos ni agrega campos nuevos: los tres ajustes son de presentación sobre un
panel que ya existe y ya funciona.

**Alcance concreto sobre el sistema actual**: el panel en cuestión es la rama "Cobrar
pedido"/"Pedido de mostrador" de `PosCheckoutPanelComponent`
(`pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.ts`), la que se
muestra cuando el pedido activo aún no se ha enviado a cocina (`sidebarMode() === 'terminal-pos'` y
`!showSessionCharge()`, es decir `selectedOrder()` inexistente o con `status === 'recibida'`).
Dentro de esa rama:

- El campo "Facturar a nombre de" (líneas 142-153) es hoy un `<input>` de texto simple, siempre
  editable, ligado a `store.billingCustomerName()`
  (`pos-terminal.store.ts:344`, inicializado en `'Consumidor Final'`). No sigue el mismo patrón que
  ya usa el campo "Cliente" de la vista de armado de pedido manual
  (`manual-order-page.component.ts:164-186`, spec 054): ahí el campo nace en modo solo-lectura
  (`[readOnly]="!editandoCliente()"`, con estilo atenuado `bg-gray-50`/`text-gray-500`) y un botón
  separado (`toggleEditarCliente()`) activa la edición; al perder foco (`onClienteBlur()`) vuelve a
  solo-lectura. El campo del panel de cobro nunca adoptó ese patrón — queda editable todo el tiempo,
  sin ningún estado de solo-lectura ni botón "editar".
- El botón de acción principal (líneas 167-175) muestra el texto literal "Cobrar, Facturar y Enviar
  a Cocina" (o "Cobrando…" mientras `store.checkoutSubmitting()` es verdadero). Al hacer clic
  dispara `checkout()` → `store.checkoutAndSend(...)`
  (`pos-terminal.store.ts:1634-1667`), que cobra, factura y envía el pedido a cocina en una sola
  llamada al backend (`POST /orders/{id}/checkout-and-send`) — esa lógica no cambia, solo el rótulo
  visible del botón.
- Los botones "🧾 Imprimir Factura" y "🔓 Liberar Mesa" (líneas 190-227) viven en un footer que se
  renderiza siempre que `store.sessionBill()` tiene valor, sin distinguir si el pedido de la rama
  activa ya fue cobrado o todavía está pendiente. Cuando el panel muestra la rama "Cobrar
  pedido"/"Pedido de mostrador" (pedido con `status === 'recibida'`, aún sin cobrar), ambos botones
  aparecen igual, aunque ninguna de las dos acciones tiene sentido todavía: "Imprimir Factura"
  reimprime una factura que en este punto no existe (`printOrderInvoice`, que hace *lookup* de una
  venta ya registrada), y "Liberar Mesa" cierra una mesa cuyo pedido ni siquiera se ha facturado.
  Tras un cobro exitoso, `checkoutAndSend()` limpia la selección (`cancelSelection()`,
  `pos-terminal.store.ts:1658`), así que esta rama nunca vuelve a mostrarse para ese mismo pedido ya
  cobrado — el ajuste no le quita ninguna acción legítima al cajero después de cobrar, solo evita
  mostrarla antes de tiempo. En los demás modos del mismo panel (`resumen` de QR ya pagado,
  `showSessionCharge` de "Cuenta de la mesa") ambos botones siguen aplicando y no cambian.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-29, tras revisar en vivo el panel de cobro de un pedido
manual. Introduce tres ajustes de presentación, ninguno reabre una decisión de negocio de specs
anteriores: (1) el nombre de facturación pasa a comportarse igual que el campo "Cliente" ya
existente (solo-lectura por defecto, editable a demanda); (2) el texto del botón principal se
simplifica a "Cobrar"; (3) las acciones que solo aplican después de un cobro ya efectuado dejan de
mostrarse mientras el cobro sigue pendiente.

**Input**: User description (verbatim): "primera la factura a nombre deberia mostrar el input con
la opcion de editar, segundo quiero cambiar el texto del boton cobrar,facturar y enviar a cocina,
por cobrar, y tercero no se deberia mostrar aun las acciones de cerrar mesa e imprimir factura
porque aun no se ha efectuado o validado el cobro"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El nombre de facturación se muestra como texto con un botón para editarlo (Priority: P1)

Un cajero abre el panel de cobro de un pedido de mostrador o de mesa. Hoy ve un campo de texto
"Facturar a nombre de" que puede editar libremente en cualquier momento, sin ninguna diferencia
visual respecto a un campo pensado para escribirse. Con esta mejora, el campo nace mostrando el
nombre actual como texto (por defecto "Consumidor Final"), con apariencia de solo lectura, y un
botón junto a él permite pasar a modo edición cuando el cajero necesita facturar a nombre de otra
persona — el mismo comportamiento que ya existe hoy para el campo "Cliente" al crear un pedido
manual.

**Why this priority**: es el primer ajuste pedido por el dueño del proyecto y unifica la experiencia
de edición de nombre de cliente entre las dos pantallas donde hoy se captura (armado de pedido y
cobro), evitando que una se sienta inconsistente con la otra.

**Independent Test**: puede probarse por completo abriendo el panel de cobro de un pedido de
mostrador o de mesa y verificando que el campo "Facturar a nombre de" se muestra como texto no
editable con un botón "editar" visible, que al pulsarlo habilita escribir sobre el campo, y que al
salir de ese campo (perder foco) vuelve a mostrarse como texto no editable con el valor actualizado.

**Acceptance Scenarios**:

1. **Given** el cajero abre el panel de cobro de un pedido que aún no se ha enviado a cocina,
   **When** observa el campo "Facturar a nombre de", **Then** lo ve con apariencia de solo lectura
   (estilo atenuado) y un botón para activar la edición, sin poder escribir directamente sobre él.
2. **Given** el campo está en modo solo lectura, **When** el cajero pulsa el botón de editar,
   **Then** el campo se vuelve editable y el cajero puede escribir un nombre distinto.
3. **Given** el cajero terminó de escribir un nuevo nombre y el campo pierde el foco (p. ej. hace
   clic en otra parte del panel), **When** observa el campo, **Then** vuelve a mostrarse en modo
   solo lectura, mostrando el valor recién escrito.
4. **Given** el cajero no edita nada, **When** confirma el cobro, **Then** el pedido se factura con
   el valor vigente del campo ("Consumidor Final" si nunca se editó), sin ningún cambio respecto al
   comportamiento actual de facturación.

---

### User Story 2 - El botón de cobro dice simplemente "Cobrar" (Priority: P2)

Un cajero está por confirmar el cobro de un pedido de mostrador o de mesa. Hoy el botón principal
dice "Cobrar, Facturar y Enviar a Cocina", un texto largo que describe internamente todo lo que
ocurre en una sola llamada al backend. Con esta mejora, el botón dice simplemente "Cobrar" — la
acción que dispara sigue siendo exactamente la misma (cobrar, facturar y enviar a cocina a la vez),
solo cambia el texto que ve el cajero.

**Why this priority**: es un ajuste de claridad/longitud de texto explícitamente pedido, sin ningún
riesgo operativo — de ahí la prioridad menor a la Historia 1.

**Independent Test**: puede probarse por completo abriendo el panel de cobro de un pedido pendiente
y verificando que el botón principal dice "Cobrar" (y "Cobrando…" mientras se procesa), y que al
pulsarlo el pedido se cobra, factura y envía a cocina igual que hoy.

**Acceptance Scenarios**:

1. **Given** el cajero abre el panel de cobro de un pedido que aún no se ha enviado a cocina,
   **When** observa el botón principal, **Then** el texto dice "Cobrar" (no "Cobrar, Facturar y
   Enviar a Cocina").
2. **Given** el cajero pulsa "Cobrar", **When** la operación está en curso, **Then** el botón
   muestra "Cobrando…" igual que hoy, sin ningún cambio de comportamiento.
3. **Given** el cajero pulsa "Cobrar", **When** la operación termina con éxito, **Then** el pedido
   queda cobrado, facturado y enviado a cocina exactamente igual que con el texto anterior del
   botón — este ajuste no cambia ninguna lógica de cobro, solo el rótulo.

---

### User Story 3 - "Imprimir Factura" y "Liberar Mesa" no se muestran mientras el cobro sigue pendiente (Priority: P1)

Un cajero abre el panel de cobro de un pedido que todavía no se ha cobrado. Hoy, junto al formulario
de cobro, ya aparecen los botones "Imprimir Factura" y "Liberar Mesa" — acciones que solo tienen
sentido después de que el pedido se facturó (reimprimir esa factura) o de que la mesa quedó libre de
cuentas pendientes. Con esta mejora, esos dos botones no se muestran mientras el cobro de ese
pedido sigue pendiente; solo vuelven a aparecer (para ese mismo pedido, ya facturado, o para
cualquier otro pedido/mesa donde sí correspondan) una vez que el cobro se efectuó.

**Why this priority**: evita que el cajero intente imprimir una factura que aún no existe o libere
una mesa cuya cuenta no se ha cobrado — mismo nivel de prioridad que la Historia 1 porque previene
una acción confusa/incorrecta durante el cobro, no solo una molestia visual.

**Independent Test**: puede probarse por completo abriendo el panel de cobro de un pedido pendiente
de cobrar y verificando que ni "Imprimir Factura" ni "Liberar Mesa" se muestran en ese momento; y
que, en los demás modos del mismo panel (cuenta de mesa ya en cobro por sesión, o pedido de QR ya
pagado en modo resumen), ambos botones se siguen mostrando exactamente igual que hoy.

**Acceptance Scenarios**:

1. **Given** el cajero abre el panel de cobro de un pedido de mostrador o de mesa que aún no se ha
   enviado a cocina (aún no cobrado), **When** observa el panel, **Then** no ve los botones
   "Imprimir Factura" ni "Liberar Mesa".
2. **Given** el cajero confirma el cobro con éxito, **When** observa lo que ocurre después (la
   selección se limpia, como ya pasa hoy), **Then** no queda un estado intermedio donde estos
   botones se muestren para ese pedido recién cobrado sin que el cajero haya vuelto a seleccionarlo.
3. **Given** el cajero abre "Cuenta de la mesa" (pedido ya enviado a cocina, cobro por cierre de
   sesión) o el modo resumen de un pedido QR ya pagado, **When** observa el panel, **Then**
   "Imprimir Factura" y "Liberar Mesa" se siguen mostrando exactamente igual que hoy, sin ningún
   cambio de esta mejora.
4. **Given** el cajero selecciona más tarde un pedido que ya fue cobrado anteriormente (para
   reimprimir su factura), **When** observa el panel, **Then** "Imprimir Factura" se muestra
   normalmente, porque ese pedido ya no está en la rama de cobro pendiente.

---

### Edge Cases

- ¿Qué pasa si el cajero abre el modo edición del nombre de facturación y luego cambia de pedido
  seleccionado sin haber perdido el foco del campo? Fuera de alcance definir un comportamiento
  distinto al que ya existe hoy para el resto del panel al cambiar de selección (el panel completo
  se reconstruye para el nuevo pedido) — el campo simplemente vuelve a nacer en modo solo lectura
  con el nombre de facturación del pedido recién seleccionado.
- ¿Qué pasa si el cajero deja el campo de nombre de facturación vacío tras editarlo? Sin cambios:
  igual que hoy, `checkoutAndSend()` ya usa `'Consumidor Final'` como valor de respaldo cuando el
  campo queda vacío (`billing_customer_name: this.billingCustomerName().trim() || 'Consumidor
  Final'`) — este ajuste no modifica esa regla.
- ¿Aplica el ajuste de "Imprimir Factura"/"Liberar Mesa" a la rama "Pedido de mostrador" sin ningún
  pedido todavía seleccionado (botón "+ Crear pedido nuevo")? Sin efecto adicional que declarar —
  hoy tampoco se muestran en ese caso porque `store.sessionBill()` no tiene valor sin un pedido
  seleccionado; esta mejora no cambia esa condición previa, solo agrega la nueva para cuando sí hay
  un pedido seleccionado pero aún sin cobrar.
- ¿Qué pasa con el botón "Rechazar pedido" que convive en el mismo panel? Fuera de alcance — esta
  mejora no lo toca; sigue mostrándose y comportándose exactamente igual que hoy en ambas ramas.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: En el panel de cobro de un pedido que aún no se ha enviado a cocina, el sistema DEBE
  mostrar el campo "Facturar a nombre de" en modo solo lectura por defecto (apariencia atenuada,
  igual criterio visual que el campo "Cliente" de armado de pedido manual), junto con un control
  visible para activar su edición.
- **FR-002**: El sistema DEBE permitir editar el campo "Facturar a nombre de" únicamente después de
  que el cajero active el control de edición, y DEBE devolverlo a modo solo lectura cuando el campo
  pierde el foco.
- **FR-003**: El valor que el cajero escriba en el campo "Facturar a nombre de" (o el valor por
  defecto si nunca lo edita) DEBE seguir siendo el que se envía a facturación al confirmar el cobro,
  sin ningún cambio respecto al comportamiento actual.
- **FR-004**: El sistema DEBE mostrar el texto "Cobrar" en el botón de acción principal del panel de
  cobro de un pedido pendiente de enviar a cocina, en lugar del texto actual "Cobrar, Facturar y
  Enviar a Cocina".
- **FR-005**: El sistema DEBE seguir mostrando "Cobrando…" en ese mismo botón mientras la operación
  de cobro está en curso, sin ningún cambio respecto al comportamiento actual.
- **FR-006**: Confirmar el cobro con este botón DEBE seguir cobrando, facturando y enviando el
  pedido a cocina en una sola operación, exactamente igual que hoy — este ajuste no cambia ninguna
  lógica ni llamada al backend, solo el texto visible del botón.
- **FR-007**: Mientras el panel muestra un pedido pendiente de cobrar (aún no enviado a cocina), el
  sistema NO DEBE mostrar los botones "Imprimir Factura" ni "Liberar Mesa".
- **FR-008**: En los demás modos/ramas del mismo panel de cobro donde estos botones ya se muestran
  hoy (cuenta de mesa en cobro por cierre de sesión, o pedido de canal QR ya pagado en modo
  resumen), el sistema DEBE seguir mostrándolos exactamente igual que hoy, sin ningún cambio.
- **FR-009**: Una vez que un pedido se cobra con éxito, el sistema DEBE seguir aplicando su
  comportamiento actual de limpiar la selección activa, de modo que ese pedido ya cobrado no
  permanece mostrando la rama de cobro pendiente (sin las acciones post-cobro) tras confirmarse el
  pago.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las aperturas del panel de cobro de un pedido pendiente muestran el campo
  "Facturar a nombre de" en modo solo lectura, sin que el cajero pueda escribir sobre él sin pulsar
  antes el control de edición.
- **SC-002**: El 100% de los paneles de cobro de un pedido pendiente muestran el texto "Cobrar" (o
  "Cobrando…" durante el envío) en el botón principal, nunca el texto largo anterior.
- **SC-003**: El 100% de las aperturas del panel de cobro de un pedido pendiente no muestran ni
  "Imprimir Factura" ni "Liberar Mesa", mientras que el 100% de las aperturas de los demás modos del
  panel (cuenta de mesa, resumen QR) los siguen mostrando sin regresión.

## Assumptions

- El patrón de solo-lectura + botón "editar" a reutilizar para "Facturar a nombre de" es exactamente
  el que ya existe para el campo "Cliente" en `manual-order-page.component.ts` (spec 054) — no se
  introduce un patrón visual nuevo, solo se aplica el mismo a un segundo campo.
- El único punto del sistema donde hoy se muestra el texto "Cobrar, Facturar y Enviar a Cocina" y el
  footer de "Imprimir Factura"/"Liberar Mesa" junto al formulario de cobro pendiente es
  `PosCheckoutPanelComponent` — corregir ese componente cubre el ajuste por completo, sin
  necesidad de tocar otras pantallas.
- No se agrega ningún campo nuevo al modelo de datos ni al contrato de ningún endpoint — los tres
  ajustes son de presentación/edición en el frontend, sobre datos y acciones que el sistema ya
  calcula, ya guarda o ya ejecuta.
