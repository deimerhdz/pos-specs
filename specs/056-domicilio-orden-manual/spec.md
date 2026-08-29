# Feature Specification: Habilitación del tipo de orden "Domicilio" en la creación manual de pedidos

**Feature Branch**: `056-domicilio-orden-manual`

**Created**: 2026-08-29

**Status**: Draft

**Naturaleza de esta spec**: **spec de evolución del modelo de datos y de reglas de negocio**, en
la misma línea que la spec inmediatamente anterior sobre la misma pantalla de creación de orden
manual ([055](../055-canal-tipo-orden/spec.md)), que estandarizó el catálogo de canal/tipo de
orden y habilitó "Para Llevar" dejando explícitamente **fuera de alcance** habilitar "Domicilio"
("Habilitar la opción 'Domicilio' en el panel de tipo de orden de la creación de pedido manual
queda fuera de alcance de esta spec"). Esta spec retoma exactamente ese punto pendiente: habilita
"Domicilio" y agrega los campos de datos nuevos que ese flujo necesita (dirección, teléfono, valor
del domicilio), además de extender el cálculo del total de la orden y de la venta/factura
resultante.

**Alcance concreto sobre el sistema actual**: hoy la pestaña "🛵 Domicilio" del panel de tipo de
orden de la pantalla de creación manual está deshabilitada en el HTML
(`manual-order-page.component.ts:136-143`) con el texto "Todavía no disponible — requiere un
cambio de backend", y el método `setOrderTypeTab()` (`manual-order-page.component.ts:298`) solo
acepta `'mesas' | 'para-llevar'`. En el backend, el valor `DELIVERY` del tipo de orden **ya existe**
en el enum (`app/api/v1/orders/schemas.py:24-28`) y en el `CheckConstraint` del modelo
(`app/models/customer_order.py:118-121`) desde la spec 055, y `service.py` ya bloquea asignar mesa
a una orden `TAKEAWAY`/`DELIVERY` (`app/api/v1/orders/service.py:142-146`) — pero ningún flujo
crea hoy una orden `DELIVERY` real: `createManualOrderFromDraft()`
(`pos-terminal.store.ts:1056-1095`) solo calcula `order_type: esParaLlevar ? 'TAKEAWAY' :
'DINE_IN'` (línea 1077), sin ninguna rama para domicilio. **No existe ningún campo** de dirección,
teléfono ni valor del domicilio en `CustomerOrder` (`app/models/customer_order.py`), en
`OrderCreate`/`OrderResponse` (`app/api/v1/orders/schemas.py:129-146,195-219`), ni en
`OrderCreatePayload` del frontend (`interfaces/dining.interface.ts:50-65`). Además, `CustomerOrder`
**no tiene ningún campo de total** — el subtotal/impuesto/propina/total solo se calculan después,
al confirmar la venta (`app/api/v1/sales/builder.py:118-166`: `total = subtotal - discount + tax +
tip`, persistido en `Sale.subtotal`/`Sale.total`, `app/models/sale.py:54-58`); en la pantalla de
creación manual, el total que ve el cajero es un cálculo aparte en el store
(`pos-terminal.store.ts:790-801`: `subtotal`, `discount` e `tax` fijos en 0, `total`), sin ningún
término para domicilio. Existe además una pestaña "Domicilios" separada en el panel de Terminal de
Mesas (`pos-tables-panel.component.ts:20-23,110-133`) que hoy solo muestra un mensaje de estado
vacío — es una pantalla distinta a la creación manual y no crea órdenes; esta spec no la modifica.

**Autorización de negocio (Principio I, Principio II, Principio VIII y Principio XI de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-29. Introduce varias decisiones de negocio nuevas: (1)
agrega tres campos nuevos a la orden (dirección de entrega, teléfono de contacto opcional, valor
del domicilio); (2) a diferencia del campo "Cliente" para "En Mesa" y "Para Llevar" (spec 054 y
055), que se diligencia por defecto con "Consumidor final" en modo de solo lectura, para
"Domicilio" el campo "Cliente" es **obligatorio y sin valor por defecto** — el cajero debe escribir
el nombre explícitamente, decisión distinta y solicitada de forma explícita por el usuario para
este tipo de orden únicamente; y (3) el valor del domicilio se suma al total tanto en la pantalla
de creación de la orden como en el total final de la venta/factura generada en el checkout
(decisión confirmada por el usuario durante la clarificación de esta spec — ver sección
"Clarifications"), lo que extiende la fórmula de cálculo de `Sale.total`. No reabre ninguna decisión
de precio, inventario ni facturación histórica: la fórmula extendida solo agrega un término nuevo
que vale 0/nulo para cualquier orden que no sea de tipo `DELIVERY`, y no recalcula ninguna venta ya
emitida.

**Input**: User description (verbatim): "ahora necesito habilitar el tipo de orden a domicilio en
la creacion manual, al seleccionar esta opcion se deberan cargar los siguientes cambios, se pedira
el nombre del cliente (obligatorio sin valor por defecto) la direccion, el numero de telefono
(opcional) y valor del domicilio, el valor del domicilio debera sumarse al total de la orden y
guardarse el la base de datos"

## Clarifications

### Sesión 2026-08-29

- P: El valor del domicilio se debe sumar al "total de la orden". Hoy `CustomerOrder` no tiene
  ningún campo de total — los totales (subtotal, impuesto, propina, total) solo se calculan
  después, al momento del checkout/factura (`Sale`, en `sales/builder.py`). ¿Hasta dónde debe
  llegar esa suma? → R: hasta la factura final — el valor del domicilio se guarda en la orden
  (nuevo campo) y se suma tanto en el total mostrado durante la creación manual como en el total
  final de la venta/factura en el checkout (junto a subtotal, descuento, impuesto y propina); el
  cliente termina pagando el domicilio como parte del importe facturado.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Crear un pedido "Domicilio" con sus datos de entrega (Priority: P1)

Un cajero está creando un pedido manual para un cliente que va a recibir su pedido en una
dirección, no en el local. Hoy la pestaña "🛵 Domicilio" existe en pantalla pero está
deshabilitada, así que el cajero no puede registrar este tipo de pedido. Con esta mejora, el
cajero selecciona "Domicilio", ve los campos "Cliente" (vacío, obligatorio), "Dirección"
(obligatorio), "Teléfono" (opcional) y "Valor del domicilio" (obligatorio), arma el pedido con el
catálogo de productos igual que siempre, diligencia los datos de entrega y confirma el pedido sin
que el sistema le pida elegir ninguna mesa.

**Why this priority**: es el resultado de negocio explícitamente solicitado por el usuario
("necesito habilitar el tipo de orden a domicilio") — es la razón de ser de toda la mejora.

**Independent Test**: puede probarse por completo abriendo la creación de orden manual,
seleccionando "Domicilio", diligenciando cliente/dirección/valor del domicilio, confirmando el
pedido sin tocar ninguna mesa, y verificando que la orden se crea correctamente con esos datos.

**Acceptance Scenarios**:

1. **Given** la pantalla de creación de orden manual está abierta, **When** el cajero observa el
   panel de tipo de orden, **Then** la pestaña "🛵 Domicilio" ya no está deshabilitada y puede
   seleccionarse igual que "🍽️ En Mesa" y "🛍️ Para Llevar".
2. **Given** el cajero seleccionó "Domicilio", **When** observa el panel, **Then** no se muestra
   ningún listado de mesas ni se exige seleccionar una para poder continuar.
3. **Given** el cajero seleccionó "Domicilio", **When** observa el campo "Cliente", **Then** lo ve
   vacío (sin "Consumidor final" ni ningún otro valor por defecto) y debe escribir un nombre antes
   de poder confirmar.
4. **Given** el cajero seleccionó "Domicilio", **When** observa el panel, **Then** ve también un
   campo "Dirección" (vacío, obligatorio), un campo "Teléfono" (vacío, opcional) y un campo "Valor
   del domicilio" (vacío, obligatorio).
5. **Given** el cajero seleccionó "Domicilio", diligenció cliente, dirección y valor del domicilio
   (con o sin teléfono), agregó productos al pedido, **When** confirma y envía el pedido, **Then**
   el pedido se crea correctamente, sin mesa asociada, con esos datos guardados.

---

### User Story 2 - Impedir confirmar un pedido "Domicilio" incompleto (Priority: P1)

Un cajero selecciona "Domicilio" pero intenta confirmar el pedido sin haber escrito el nombre del
cliente, la dirección, o el valor del domicilio. El sistema le impide enviar el pedido y le indica
con claridad qué falta por diligenciar, en vez de guardar un pedido de domicilio incompleto que
después no se pueda entregar ni cobrar correctamente.

**Why this priority**: sin esta validación, el campo "Cliente" sin valor por defecto (a diferencia
de "En Mesa"/"Para Llevar") y los campos nuevos obligatorios podrían quedar vacíos, generando
pedidos de domicilio que no se pueden despachar ni cobrar — protege la utilidad misma de los datos
que la Historia 1 introduce.

**Independent Test**: puede probarse por completo seleccionando "Domicilio", dejando el nombre del
cliente (o la dirección, o el valor del domicilio) vacío, intentando confirmar el pedido, y
verificando que el sistema bloquea el envío y señala el campo faltante.

**Acceptance Scenarios**:

1. **Given** el cajero seleccionó "Domicilio" y dejó el campo "Cliente" vacío, **When** intenta
   confirmar y enviar el pedido, **Then** el sistema bloquea el envío e indica que el nombre del
   cliente es obligatorio.
2. **Given** el cajero seleccionó "Domicilio" y dejó el campo "Dirección" vacío, **When** intenta
   confirmar y enviar el pedido, **Then** el sistema bloquea el envío e indica que la dirección es
   obligatoria.
3. **Given** el cajero seleccionó "Domicilio" y dejó el campo "Valor del domicilio" vacío, **When**
   intenta confirmar y enviar el pedido, **Then** el sistema bloquea el envío e indica que el valor
   del domicilio es obligatorio.
4. **Given** el cajero seleccionó "Domicilio", diligenció cliente/dirección/valor del domicilio
   pero dejó el campo "Teléfono" vacío, **When** confirma y envía el pedido, **Then** el pedido se
   crea sin ningún bloqueo, ya que el teléfono es opcional.

---

### User Story 3 - El valor del domicilio se suma al total y queda facturado (Priority: P1)

Un cajero está armando un pedido de domicilio y escribe el valor del domicilio. Ese valor se suma
de inmediato al total que ve en pantalla (junto al subtotal y los productos del carrito), y al
confirmar el pedido y facturarlo, el total final que paga el cliente también incluye ese valor —
no se pierde ni queda solo como un dato decorativo.

**Why this priority**: es el segundo objetivo explícito del usuario ("el valor del domicilio
debera sumarse al total de la orden y guardarse en la base de datos") y protege que el negocio
efectivamente cobre el domicilio, no solo lo registre.

**Independent Test**: puede probarse por completo escribiendo un valor de domicilio en una orden
"Domicilio", verificando que el total mostrado en pantalla aumenta en esa misma cantidad, confirmando
el pedido, facturándolo, y verificando que el total de la factura resultante incluye ese mismo
valor.

**Acceptance Scenarios**:

1. **Given** el cajero seleccionó "Domicilio" y tiene productos en el carrito, **When** escribe un
   valor en el campo "Valor del domicilio", **Then** el total mostrado en pantalla se actualiza
   sumando ese valor al subtotal de los productos.
2. **Given** un pedido "Domicilio" ya confirmado con un valor de domicilio guardado, **When** ese
   pedido se factura en el checkout, **Then** el total de la venta/factura resultante incluye el
   valor del domicilio sumado al subtotal, descuento, impuesto y propina de esa venta.
3. **Given** un pedido de cualquier otro tipo de orden ("En Mesa" o "Para Llevar"), **When** se
   calcula su total en pantalla o se factura en el checkout, **Then** su total no se ve afectado
   por esta mejora (el término de domicilio vale 0/nulo para esos pedidos).

---

### Edge Cases

- ¿Qué pasa si el cajero edita los campos "Cliente", "Dirección", "Teléfono" o "Valor del
  domicilio" y luego cambia a otro tipo de orden ("En Mesa" o "Para Llevar") antes de confirmar?
  Fuera de alcance definir un comportamiento distinto al que ya exista para el manejo de campos del
  borrador al cambiar de contexto (mismo criterio ya aplicado en spec 054/055).
- ¿El cajero puede registrar domicilio gratis (valor $0)? Sí, siempre que lo escriba
  explícitamente — el campo nunca tiene un valor por defecto (ni "Consumidor final" ni "$0"), así
  que un campo vacío se trata como faltante (Historia 2), no como $0 implícito.
- ¿Qué pasa con los pedidos existentes antes de esta mejora? No se modifican — quedan sin valores
  en los campos nuevos (dirección, teléfono, valor del domicilio), y su total ya calculado
  (histórico) no se recalcula (Principio VII de la Constitución).
- ¿Qué pasa con la pestaña "Domicilios" del panel de Terminal de Mesas
  (`pos-tables-panel.component.ts`)? No se modifica — es una pantalla distinta a la creación manual
  de orden; esta spec solo habilita "Domicilio" dentro de `manual-order-page.component.ts`.
- ¿Se puede escribir un valor de domicilio negativo? No — el sistema lo rechaza; el valor del
  domicilio debe ser un número no negativo.
- ¿Puede un pedido cambiar de tipo de orden, o editarse la dirección/teléfono/valor del domicilio,
  después de confirmado? Fuera de alcance — no se construye ningún flujo de edición posterior,
  igual que ya define spec 055 para canal y tipo de orden.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El panel de tipo de orden de la pantalla de creación de pedido manual DEBE habilitar
  la opción "Domicilio" (hoy deshabilitada), permitiendo seleccionarla igual que "En Mesa" y "Para
  Llevar".
- **FR-002**: Al seleccionar "Domicilio" en la creación de pedido manual, el sistema NO DEBE exigir
  la selección de ninguna mesa para poder confirmar y enviar el pedido.
- **FR-003**: Al seleccionar "Domicilio", el sistema DEBE mostrar el campo "Cliente" vacío y
  obligatorio, sin ningún valor por defecto — comportamiento distinto al ya definido para "En Mesa"
  y "Para Llevar" (spec 054/055), donde el campo inicia con "Consumidor final".
- **FR-004**: Al seleccionar "Domicilio", el sistema DEBE mostrar un campo "Dirección", vacío y
  obligatorio.
- **FR-005**: Al seleccionar "Domicilio", el sistema DEBE mostrar un campo "Teléfono", vacío y
  opcional.
- **FR-006**: Al seleccionar "Domicilio", el sistema DEBE mostrar un campo "Valor del domicilio",
  vacío y obligatorio, que solo acepta un valor numérico no negativo, sin ningún valor por defecto.
- **FR-007**: El sistema DEBE impedir confirmar y enviar un pedido "Domicilio" si el nombre del
  cliente, la dirección, o el valor del domicilio están vacíos, indicando con claridad cuál de esos
  campos falta.
- **FR-008**: El sistema NO DEBE impedir confirmar un pedido "Domicilio" por dejar el campo
  "Teléfono" vacío.
- **FR-009**: El total mostrado en la pantalla de creación de orden manual DEBE incluir el valor
  del domicilio como un término adicional sumado al subtotal, actualizándose en cuanto el cajero lo
  escribe.
- **FR-010**: Al confirmar un pedido "Domicilio", el sistema DEBE guardarlo con tipo de orden
  DELIVERY, canal POS, sin mesa asociada, y con el nombre de cliente, la dirección, el teléfono (si
  se diligenció) y el valor del domicilio ingresados en pantalla.
- **FR-011**: El sistema DEBE incluir el valor del domicilio guardado en una orden DELIVERY como un
  término adicional en el cálculo del total de la venta/factura generada al facturar esa orden en
  el checkout, junto al subtotal, descuento, impuesto y propina.
- **FR-012**: El sistema NO DEBE alterar el total en pantalla ni el total de la venta/factura de
  ningún pedido que no sea de tipo DELIVERY — el término de valor del domicilio no afecta esos
  cálculos.
- **FR-013**: El sistema NO DEBE recalcular ni modificar el total de ninguna venta/factura ya
  emitida antes de esta mejora.
- **FR-014**: Esta mejora NO DEBE modificar la pestaña "Domicilios" del panel de Terminal de Mesas
  (`pos-tables-panel.component.ts`), que permanece con su comportamiento actual sin cambios.

### Key Entities *(include if feature involves data)*

- **Orden (`CustomerOrder`)**: entidad ya existente. Se agregan tres atributos nuevos, aplicables
  únicamente cuando el tipo de orden es DELIVERY: **dirección de entrega** (obligatoria),
  **teléfono de contacto** (opcional) y **valor del domicilio** (obligatorio, numérico no
  negativo). Para cualquier otro tipo de orden estos tres atributos quedan sin valor, sin afectar
  su comportamiento actual.
- **Venta/Factura (`Sale`, generada al facturar la orden)**: entidad ya existente. Su fórmula de
  cálculo del total se extiende para sumar el valor del domicilio de la orden asociada cuando esa
  orden es de tipo DELIVERY; para cualquier otra orden, el término vale 0 y el cálculo del total no
  cambia respecto al comportamiento actual.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El personal del punto de venta puede crear un pedido "Domicilio" completo (nombre de
  cliente, dirección, valor del domicilio, con o sin teléfono, sin seleccionar ninguna mesa) usando
  el mismo flujo de armado de pedido que ya usa para "En Mesa" y "Para Llevar".
- **SC-002**: El 100% de los intentos de confirmar un pedido "Domicilio" con el nombre del cliente,
  la dirección, o el valor del domicilio vacíos son bloqueados por el sistema, sin crear ningún
  registro incompleto.
- **SC-003**: El 100% de los pedidos "Domicilio" confirmados quedan guardados con nombre de
  cliente, dirección y valor del domicilio no vacíos.
- **SC-004**: El 100% de las ventas/facturas generadas a partir de un pedido "Domicilio" tienen un
  total que incluye exactamente el valor del domicilio de esa orden.
- **SC-005**: El 100% de los pedidos y ventas/facturas de tipo distinto a "Domicilio" (nuevos e
  históricos) mantienen su total sin ninguna variación después de esta mejora.

## Assumptions

- La combinación canal POS + tipo de orden DELIVERY ya es una combinación válida según la matriz
  de validación definida en spec 055 (POS admite DINE_IN, TAKEAWAY y DELIVERY) — esta mejora no
  modifica esa matriz.
- Igual que "Para Llevar" (spec 055), "Domicilio" no exige seleccionar mesa, reutilizando la
  validación de backend que ya bloquea asignar `dining_table_id` a órdenes DELIVERY
  (`app/api/v1/orders/service.py:142-146`).
- El impuesto de la orden está hoy fijo en 0 tanto en la pantalla de creación manual
  (`pos-terminal.store.ts:790-801`) como en la fórmula de la venta (`sales/builder.py:118-166`); el
  valor del domicilio se suma como un término aparte, sin interactuar con el cálculo de impuesto.
- Los campos "Dirección" y "Teléfono" son campos de texto libre, sin validación de formato de
  dirección real ni de número telefónico — el mismo nivel de validación que ya tienen otros campos
  de texto libre del sistema (p. ej. "Cliente", "Notas").
- El valor del domicilio usa el mismo formato monetario que el resto de valores del sistema (pesos
  colombianos sin decimales), consistente con los precios de producto ya existentes.
- El nombre del cliente, la dirección, el teléfono y el valor del domicilio de un pedido "Domicilio"
  no se pueden editar después de creado el pedido — no se construye ningún flujo de edición
  posterior, igual que ya define spec 055 para canal y tipo de orden.
- Esta mejora no construye ningún flujo de despacho, seguimiento o asignación de repartidor para
  los pedidos de domicilio — solo su creación, validación de datos, y el impacto del valor del
  domicilio sobre el total. Ese alcance queda fuera de esta spec.
