# Feature Specification: Campo "Cliente" con valor por defecto "Consumidor final" en la creación de orden manual

**Feature Branch**: `054-campo-cliente-orden-manual`

**Created**: 2026-08-29

**Status**: Draft

**Naturaleza de esta spec**: **spec de mejora sobre una pantalla ya existente**, en la misma línea
que las tres specs inmediatamente anteriores sobre la misma pantalla
(`manual-order-page.component.ts`): [051](../051-imagen-producto-tipo-orden/spec.md),
[052](../052-panel-derecho-orden-manual/spec.md) y
[053](../053-selector-mesa-buscable/spec.md). A diferencia de esas tres, esta sí involucra un
cambio de comportamiento observable en el dato que queda guardado en cada orden creada desde esta
pantalla (ver "Decisión de negocio" más abajo) — no un simple reordenamiento visual.

**Alcance concreto sobre la pantalla actual**: la pantalla de creación de orden manual no tiene hoy
ningún campo para capturar el nombre del cliente. `PosTerminalStore` (usado por esta pantalla)
expone `customerName` (`pos-terminal.store.ts:314`) y ya lo envía en el cuerpo de la petición de
creación de orden (`createManualOrderFromDraft()`, `pos-terminal.store.ts:1072-1078`:
`customer_name: this.customerName().trim() || null`) — pero nada en
`manual-order-page.component.ts` lee ni escribe ese signal, así que hoy siempre viaja `null`. El
backend (`pos-backend`) ya acepta y persiste ese campo sin cambios: `CustomerOrder.customer_name`
(`app/models/customer_order.py:49`, `String(255)`, nullable) y `OrderCreate.customer_name`
(`app/api/v1/orders/schemas.py:120`) ya existen y el endpoint `POST /orders`
(`app/api/v1/orders/router.py:502-504`) ya los acepta.

**Decisión de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-29. Introduce un valor por defecto nuevo
("Consumidor final") para el nombre del cliente de las órdenes creadas desde esta pantalla — hoy
siempre se guarda `null`. Esto **no reabre** la decisión documentada en
`pos-terminal.store.ts:1042-1044` (evitar que un relleno tipo "Cliente Mesa 3" quede en el
documento fiscal en vez de los comensales reales): "Consumidor final" es la designación estándar ya
usada en Colombia para ventas sin un cliente identificado, no un relleno inventado con apariencia
de nombre real; y el nuevo valor por defecto se aplica únicamente dentro de esta pantalla dedicada
de creación de orden manual, sin modificar el método compartido `selectTable()` que usan otras
pantallas (Terminal de Mesas). No aplica una nueva entrada en `registro-de-anomalias.md` más allá
de citar esta spec como origen y autorización del cambio.

**Input**: User description (verbatim): "al momento de crear la orden manual, debe tambien aparece
cuando el tipo de orden es mesa un input de tipo readonly con la opcion de editar y el valor por
defecto Consumidor final, el cual debe almacenarse en la base de datos al crear la orden"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ver el campo "Cliente" con "Consumidor final" por defecto (Priority: P1)

Un mesero abre la creación de orden manual con el tipo de orden "En Mesa" (la única habilitada
hoy). Ve un campo "Cliente" ya diligenciado con "Consumidor final", en modo de solo lectura, sin
tener que escribir nada para poder continuar armando y confirmando el pedido.

**Why this priority**: es el comportamiento base pedido por el usuario; sin el valor por defecto
visible, el resto de la historia (poder editarlo) no tiene nada que editar.

**Independent Test**: puede probarse por completo abriendo la creación de orden manual y
verificando que el campo "Cliente" ya muestra "Consumidor final" sin ninguna interacción previa.

**Acceptance Scenarios**:

1. **Given** la pantalla de creación de orden manual está abierta con "En Mesa" como tipo de orden,
   **When** el mesero la observa, **Then** ve un campo "Cliente" con el valor "Consumidor final" ya
   diligenciado, en modo de solo lectura.

---

### User Story 2 - Editar el nombre del cliente (Priority: P2)

Un mesero quiere registrar el nombre real del cliente en vez de dejar "Consumidor final". Activa el
modo de edición del campo, escribe el nombre, y ese valor queda como el nombre del cliente para
esta orden.

**Why this priority**: es la segunda parte explícita del pedido del usuario; depende de que el
campo ya exista con su valor por defecto (Historia 1).

**Independent Test**: puede probarse por completo activando la edición del campo "Cliente", escribiendo un
nombre distinto, y confirmando que el campo refleja el nuevo valor.

**Acceptance Scenarios**:

1. **Given** el campo "Cliente" muestra "Consumidor final" en modo de solo lectura, **When** el
   mesero activa la opción de editar, **Then** el campo se vuelve editable.
2. **Given** el campo "Cliente" está en modo editable, **When** el mesero escribe un nombre y deja
   de editar (p. ej. hace clic fuera del campo), **Then** el campo muestra el nombre escrito, ya no
   "Consumidor final".
3. **Given** el mesero activó la edición y borró todo el texto sin escribir un nombre nuevo,
   **When** deja de editar, **Then** el campo vuelve a mostrar "Consumidor final" (nunca queda
   vacío).

---

### User Story 3 - El nombre del cliente se guarda al crear la orden (Priority: P1)

Un mesero confirma y envía la orden manual, con el campo "Cliente" en "Consumidor final" (sin
editar) o con un nombre que sí editó. El nombre que se ve en pantalla en ese momento es el que
queda guardado como cliente de la orden creada.

**Why this priority**: es el requisito explícito de persistencia del usuario ("debe almacenarse en
la base de datos al crear la orden") — sin esto, el campo sería solo decorativo.

**Independent Test**: puede probarse por completo creando una orden manual (con el valor por
defecto o con uno editado) y confirmando que la orden resultante tiene ese mismo nombre de cliente.

**Acceptance Scenarios**:

1. **Given** el campo "Cliente" muestra "Consumidor final" sin haber sido editado, **When** el
   mesero confirma y envía la orden, **Then** la orden creada queda con "Consumidor final" como
   nombre de cliente.
2. **Given** el mesero editó el campo "Cliente" a un nombre específico, **When** confirma y envía
   la orden, **Then** la orden creada queda con ese nombre específico como cliente.

---

### Edge Cases

- ¿Qué pasa si el mesero cambia de mesa (usando el selector de la spec 053) después de haber editado
  el nombre del cliente? Fuera de alcance definir un comportamiento distinto al que ya exista hoy
  para otros campos del borrador al cambiar de mesa — no se solicita ningún cambio a ese flujo; el
  campo "Cliente" sigue el mismo criterio de "borrador de la mesa seleccionada" que ya aplica al
  carrito.
- ¿Qué pasa con los tipos de orden "Para Llevar" y "Domicilio" (deshabilitados)? Fuera de alcance —
  siguen deshabilitados sin cambios (specs 036/051/052); esta spec no define si también tendrán
  este campo cuando se habiliten en el futuro.
- ¿Qué pasa si el mesero dejó el campo en blanco pero no llegó a "perder el foco" (p. ej. confirma
  la orden mientras el campo sigue en modo edición y vacío)? El sistema NO DEBE enviar un nombre de
  cliente vacío como valor "en blanco" válido — debe aplicar el mismo valor por defecto
  ("Consumidor final") antes de guardar.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE mostrar, en la pantalla de creación de orden manual con "En Mesa"
  como tipo de orden, un campo "Cliente" con el valor "Consumidor final" ya diligenciado por
  defecto.
- **FR-002**: El campo "Cliente" DEBE iniciar en modo de solo lectura (no editable directamente).
- **FR-003**: El sistema DEBE ofrecer una opción explícita para activar la edición del campo
  "Cliente".
- **FR-004**: Una vez activada la edición, el mesero DEBE poder escribir un nombre de cliente
  distinto al valor por defecto.
- **FR-005**: Si el mesero deja el campo "Cliente" vacío al terminar de editarlo (o al confirmar la
  orden sin haber escrito nada), el sistema DEBE aplicar "Consumidor final" como valor — el campo
  nunca queda vacío ni se guarda un nombre de cliente en blanco.
- **FR-006**: Al confirmar y enviar la orden manual, el sistema DEBE guardar el valor
  actualmente mostrado en el campo "Cliente" (por defecto o editado) como el nombre del cliente de
  la orden creada.
- **FR-007**: El sistema NO DEBE cambiar ningún otro comportamiento de la pantalla de creación de
  orden manual (catálogo, selector de mesa, carrito, totales, tipos de orden deshabilitados) como
  parte de este cambio.

### Key Entities *(include if feature involves data)*

- **Orden (`CustomerOrder`)**: entidad ya existente; su campo `customer_name` (ya existente,
  opcional) pasa a recibir sistemáticamente un valor no vacío ("Consumidor final" o el nombre
  editado) desde esta pantalla, en vez del valor `null` que recibía hasta ahora. No se agrega
  ningún campo nuevo al modelo de datos.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las órdenes creadas desde la pantalla de creación de orden manual quedan
  con un nombre de cliente no vacío ("Consumidor final" o el nombre editado por el mesero) — nunca
  `null` ni cadena vacía.
- **SC-002**: Un mesero puede completar la creación de una orden manual sin necesidad de tocar el
  campo "Cliente" en ningún momento.
- **SC-003**: Un mesero puede reemplazar "Consumidor final" por un nombre específico en menos de 2
  interacciones (activar edición + escribir).

## Assumptions

- El campo aparece siempre que la pantalla está activa (hoy "En Mesa" es el único tipo de orden
  funcional; "Para Llevar" y "Domicilio" siguen deshabilitados) — no se introduce ninguna condición
  adicional de visibilidad más allá de la ya existente para el resto de la pantalla.
- "Consumidor final" se persiste literalmente tal cual, sin normalización adicional (mayúsculas,
  acentos), igual que cualquier nombre de cliente escrito a mano.
- El valor por defecto se aplica únicamente en esta pantalla dedicada de creación de orden manual;
  no se modifica el comportamiento de `selectTable()` compartido por otras pantallas (Terminal de
  Mesas), que sigue dejando el nombre de cliente vacío cuando no hay una orden ya creada
  (`pos-terminal.store.ts:1042-1044`, sin cambios).
- No se solicita ningún cambio de backend ni de modelo de datos — `customer_name` ya existe, ya es
  opcional, y el endpoint de creación de orden ya lo acepta y persiste sin modificaciones.
