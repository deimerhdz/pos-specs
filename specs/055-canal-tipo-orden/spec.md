# Feature Specification: Estandarización de canal y tipo de orden — habilitación de pedidos "Para Llevar"

**Feature Branch**: `055-canal-tipo-orden`

**Created**: 2026-08-29

**Status**: Draft

**Naturaleza de esta spec**: **spec de evolución del modelo de datos y de reglas de negocio**,
distinta a las cuatro specs inmediatamente anteriores sobre la misma pantalla de creación de orden
manual ([051](../051-imagen-producto-tipo-orden/spec.md),
[052](../052-panel-derecho-orden-manual/spec.md),
[053](../053-selector-mesa-buscable/spec.md) y
[054](../054-campo-cliente-orden-manual/spec.md)), que fueron ajustes visuales o de un único campo
sobre una pantalla ya existente. Esta spec sí cambia el modelo de datos de la orden (`CustomerOrder`)
y agrega una regla de negocio nueva (validación de combinaciones canal/tipo de orden), y como
consecuencia de ese cambio de modelo, habilita por primera vez la creación real de pedidos "Para
Llevar" desde la misma pantalla (`manual-order-page.component.ts`).

**Alcance concreto sobre el sistema actual**: hoy la orden (`CustomerOrder`,
`app/models/customer_order.py:51`) tiene un campo `channel` de texto libre (`String(10)`) validado
solo por un `CheckConstraint` con tres valores en minúscula sin estandarizar: `qr` (menú QR),
`counter` y `waiter` (personal del punto de venta) — ninguno tiene índice propio. No existe ningún
campo que represente el tipo de orden (mesa / para llevar / domicilio): la única señal de "tipo de
orden" que existe hoy es puramente de interfaz, sin persistencia (`OrderTypeTab` en
`pos-terminal.store.ts:99`), y en la pantalla de creación manual las pestañas "🛍️ Para Llevar" y
"🛵 Domicilio" están deshabilitadas en el HTML (`manual-order-page.component.ts:120,128`) con el
texto "Todavía no disponible — requiere un cambio de backend", porque hoy la mesa (`dining_table_id`)
es la única forma de asociar un pedido, y la ausencia de tipo de orden en el backend es justo lo que
bloquea el flujo de "Para Llevar".

**Autorización de negocio (Principio I, Principio II y Principio VIII de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-29. Introduce dos decisiones de negocio nuevas: (1) un
catálogo fijo y estandarizado de canales y tipos de orden, reemplazando los valores libres actuales
del canal; y (2) reglas de qué combinaciones de canal y tipo de orden son válidas, prohibiendo
explícitamente combinaciones que no son lógicas en la operación real del negocio (por ejemplo,
WHATSAPP + DINE_IN). No reabre ninguna decisión de precio, inventario ni facturación de specs
anteriores; sí amplía spec 054, ya que el campo "Cliente" con valor por defecto "Consumidor final"
que esa spec agregó para "En Mesa" se extiende ahora también a "Para Llevar", reutilizando el mismo
comportamiento ya construido.

**Input**: User description (verbatim): "cuando creo un pedido manual hay varias opciones en mesa,
para llevar y domicilo, de cara habilitar la opcion para llevar quiero hacer una mejora actualmente
la tabla de customer_orders tiene un campo llamado chanel quiero estandarizar esos valores, deben
ser un enum y indice para poder filtrar mas adelante los valores son POS-> creada directamente desde
el cajero, QR_MENU creada desde un cliente desde el menu qr WHATSAP -> creada desde whatsapp y API,
creada desde una integracion externa, adicional a eso quiero agregar un tipo order tipe que tambien
permitira filtar registros sera un enum y tendra los siguientes valores DINE_IN-> se consume en el
establecimiento, TAKEWAY-> para llevary DELIVERY-> domicilio, en ese orden de ideas analiza tendras
que validar las posibles combinaciones que por ejemplo no permita tener WHATSAPP Y DINE_IN porque no
parece logico en la vida real y lo otro es que habilites la opcion de orden para llevar en el panel
de tipo de orden, no sera requerido seleccionar mesa, en esa ocacion solo se pedira el nombre del
cliente en un input de lectura con la opcion de editar, pero por defecto debera mostrarse el valor
Consumidor final el cual se guardar en el customer name de la orden"

## Clarifications

### Sesión 2026-08-29

- P: ¿Qué combinaciones de canal y tipo de orden deben ser válidas, más allá del ejemplo explícito
  de que WHATSAPP + DINE_IN no es válido? → R: matriz realista por canal — POS admite los tres
  tipos de orden (DINE_IN, TAKEAWAY, DELIVERY); QR_MENU admite únicamente DINE_IN (hoy el menú QR
  siempre corresponde a una mesa física, no existe flujo de QR para recoger o domicilio); WHATSAPP
  admite TAKEAWAY y DELIVERY (nunca DINE_IN); API admite TAKEAWAY y DELIVERY (nunca DINE_IN, ya que
  una integración externa trae pedidos para recoger o de domicilio, no de consumo en el local).
- P: Las órdenes históricas no tienen ningún valor de tipo de orden (el campo es nuevo). ¿Cómo se
  debe completar ese campo para esos registros? → R: según si tienen mesa asignada — las órdenes
  históricas con una mesa asignada quedan reclasificadas como DINE_IN; las que no tengan mesa
  asignada quedan sin tipo de orden asignado.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Crear un pedido "Para Llevar" sin seleccionar mesa (Priority: P1)

Un cajero está creando un pedido manual para un cliente que va a recoger su pedido y no se va a
sentar en ninguna mesa. Hoy la pestaña "🛍️ Para Llevar" existe en pantalla pero está deshabilitada,
así que el cajero no tiene más opción que registrar el pedido como si fuera "En Mesa" (o no
registrarlo en el sistema). Con esta mejora, el cajero selecciona "Para Llevar", ve un campo
"Cliente" ya diligenciado con "Consumidor final" (editable si quiere escribir el nombre real), arma
el pedido con el catálogo de productos igual que siempre, y lo confirma sin que el sistema le pida
elegir ninguna mesa.

**Why this priority**: es el resultado de negocio explícitamente solicitado por el usuario ("de
cara habilitar la opcion para llevar") — es la razón de ser de toda la mejora.

**Independent Test**: puede probarse por completo abriendo la creación de orden manual,
seleccionando "Para Llevar", confirmando el pedido sin tocar ninguna mesa, y verificando que la
orden se crea correctamente con el nombre de cliente mostrado en pantalla.

**Acceptance Scenarios**:

1. **Given** la pantalla de creación de orden manual está abierta, **When** el cajero observa el
   panel de tipo de orden, **Then** la pestaña "🛍️ Para Llevar" ya no está deshabilitada y puede
   seleccionarse igual que "🍽️ En Mesa".
2. **Given** el cajero seleccionó "Para Llevar", **When** observa el panel, **Then** no se muestra
   ningún listado de mesas ni se exige seleccionar una para poder continuar.
3. **Given** el cajero seleccionó "Para Llevar", **When** observa el campo "Cliente", **Then** lo ve
   ya diligenciado con "Consumidor final", en modo de solo lectura, con la opción de editarlo.
4. **Given** el cajero seleccionó "Para Llevar", agregó productos al pedido y dejó el campo
   "Cliente" sin editar, **When** confirma y envía el pedido, **Then** el pedido se crea
   correctamente, sin mesa asociada, con "Consumidor final" como nombre de cliente.
5. **Given** el cajero seleccionó "Para Llevar", agregó productos y editó el campo "Cliente" con un
   nombre específico, **When** confirma y envía el pedido, **Then** el pedido se crea correctamente
   con ese nombre específico como nombre de cliente.

---

### User Story 2 - Impedir combinaciones ilógicas de canal y tipo de orden (Priority: P1)

El sistema recibe la creación de una orden desde algún punto de entrada (punto de venta, menú QR,
WhatsApp o una integración externa) junto con un tipo de orden. Antes de crear la orden, el sistema
verifica que esa combinación tenga sentido en la operación real del negocio (por ejemplo, un pedido
que llega por WhatsApp no puede registrarse como "se consume en el establecimiento", porque quien
pide por WhatsApp no está físicamente en el local) y rechaza la creación cuando no lo tiene, en vez
de guardar un registro contradictorio que después ensucie los reportes y filtros del negocio.

**Why this priority**: es la validación de negocio explícitamente pedida por el usuario ("tendras
que validar las posibles combinaciones") y protege la confiabilidad de los datos que esta misma
mejora busca habilitar para filtrar más adelante — sin esta historia, el catálogo estandarizado por
sí solo no evita que se sigan colando combinaciones sin sentido.

**Independent Test**: puede probarse por completo intentando crear una orden con una combinación no
permitida (por ejemplo canal WHATSAPP con tipo de orden DINE_IN) y verificando que el sistema
rechaza la creación con un mensaje claro, mientras que una combinación permitida (por ejemplo POS
con TAKEAWAY) sí se crea con éxito.

**Acceptance Scenarios**:

1. **Given** un intento de creación de orden con canal WHATSAPP y tipo de orden DINE_IN, **When**
   el sistema procesa la solicitud, **Then** rechaza la creación e informa que esa combinación no es
   válida, sin crear ningún registro.
2. **Given** un intento de creación de orden con canal API y tipo de orden DINE_IN, **When** el
   sistema procesa la solicitud, **Then** rechaza la creación de la misma forma.
3. **Given** un intento de creación de orden con canal QR_MENU y tipo de orden TAKEAWAY o DELIVERY,
   **When** el sistema procesa la solicitud, **Then** rechaza la creación, ya que el menú QR solo
   admite pedidos de tipo DINE_IN.
4. **Given** un intento de creación de orden con canal POS y cualquiera de los tres tipos de orden
   (DINE_IN, TAKEAWAY o DELIVERY), **When** el sistema procesa la solicitud, **Then** la crea
   normalmente, ya que POS admite los tres tipos.
5. **Given** un intento de creación de orden con canal WHATSAPP o API y tipo de orden TAKEAWAY o
   DELIVERY, **When** el sistema procesa la solicitud, **Then** la crea normalmente.

---

### User Story 3 - Filtrar y clasificar pedidos por canal y tipo de orden, incluyendo el histórico (Priority: P2)

Alguien del negocio necesita analizar cuántos pedidos vienen de cada canal de origen (punto de
venta, menú QR, WhatsApp, integraciones externas) y cuántos son de cada tipo (consumo en el local,
para llevar, domicilio), tanto para los pedidos nuevos como para los que ya existían antes de esta
mejora. Hoy no puede hacerlo de forma confiable porque el canal es texto libre sin estandarizar y no
existe ningún tipo de orden persistido.

**Why this priority**: es el segundo objetivo explícito del usuario ("indice para poder filtrar mas
adelante" / "que tambien permitira filtar registros") — depende de que el catálogo estandarizado
(Historia 1 y 2) ya exista, y de que los pedidos históricos también queden clasificados con esos
mismos valores para que el filtro sea útil sobre todo el histórico, no solo sobre pedidos nuevos.

**Independent Test**: puede probarse por completo consultando el conjunto de pedidos existentes
(históricos y nuevos) y verificando que cada uno tiene asignado uno de los valores estandarizados de
canal, y que los pedidos con mesa asignada (históricos y nuevos) tienen asignado el tipo de orden
DINE_IN.

**Acceptance Scenarios**:

1. **Given** los pedidos que existían antes de esta mejora, **When** se consulta su canal después de
   aplicada la mejora, **Then** cada uno tiene uno de los valores estandarizados (POS o QR_MENU)
   según su canal original (`counter`/`waiter` → POS, `qr` → QR_MENU), sin valores libres
   remanentes.
2. **Given** los pedidos que existían antes de esta mejora y tenían una mesa asignada, **When** se
   consulta su tipo de orden después de aplicada la mejora, **Then** aparece clasificado como
   DINE_IN.
3. **Given** los pedidos que existían antes de esta mejora y no tenían ninguna mesa asignada (si los
   hay), **When** se consulta su tipo de orden después de aplicada la mejora, **Then** no tienen
   ningún tipo de orden asignado.
4. **Given** cualquier pedido nuevo creado después de esta mejora, **When** se consulta su canal y
   su tipo de orden, **Then** ambos tienen siempre uno de los valores estandarizados (nunca quedan
   vacíos).

---

### Edge Cases

- ¿Qué pasa si el cajero edita el campo "Cliente" en "Para Llevar" y luego cambia a "En Mesa" (o
  viceversa) antes de confirmar? Fuera de alcance definir un comportamiento distinto al que ya
  exista para el campo "Cliente" al cambiar de contexto — no se solicita ningún cambio adicional al
  ya definido en spec 054 para el manejo del nombre de cliente.
- ¿Qué pasa con la pestaña "🛵 Domicilio"? Permanece deshabilitada — esta mejora agrega el valor
  DELIVERY al catálogo de tipos de orden (para que exista y pueda validarse/filtrarse desde ya),
  pero no habilita ningún flujo de creación de pedidos de domicilio en la pantalla de creación
  manual. Habilitar "Domicilio" en pantalla queda fuera de alcance de esta spec.
- ¿Qué pasa si llega una solicitud de creación de orden con un canal o un tipo de orden que no está
  en el catálogo estandarizado (por ejemplo, un valor mal escrito de una integración futura)? El
  sistema rechaza la creación, igual que rechaza una combinación no permitida.
- ¿Qué pasa con el valor de canal `waiter`, que existe en el modelo de datos actual pero que hoy
  ningún flujo del sistema envía activamente? Se reclasifica igual que `counter`, como POS — ambos
  representan personal del punto de venta creando el pedido directamente.
- ¿Puede un pedido cambiar de canal o de tipo de orden después de creado? Fuera de alcance — esta
  spec no define ningún flujo de edición posterior a la creación.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE reemplazar los valores libres actuales del canal de una orden (hoy
  `qr`, `counter`, `waiter`) por un catálogo fijo y estandarizado de valores: POS (creada
  directamente desde el punto de venta/cajero), QR_MENU (creada por un cliente desde el menú QR),
  WHATSAPP (creada desde WhatsApp), y API (creada desde una integración externa).
- **FR-002**: El sistema DEBE indexar el canal de la orden para permitir filtrar pedidos por canal
  de forma eficiente.
- **FR-003**: El sistema DEBE introducir un nuevo atributo "tipo de orden" en la orden, con un
  catálogo fijo de valores: DINE_IN (se consume en el establecimiento), TAKEAWAY (para llevar), y
  DELIVERY (a domicilio).
- **FR-004**: El sistema DEBE indexar el tipo de orden para permitir filtrar pedidos por tipo de
  forma eficiente.
- **FR-005**: El canal de una orden lo asigna siempre el sistema según su punto de entrada real de
  creación — el usuario que crea el pedido nunca elige manualmente el canal.
- **FR-006**: El sistema DEBE validar, al crear una orden, que la combinación de canal y tipo de
  orden solicitada sea una de las combinaciones permitidas, y DEBE rechazar la creación cuando la
  combinación no lo es:
  - POS admite: DINE_IN, TAKEAWAY, DELIVERY.
  - QR_MENU admite únicamente: DINE_IN.
  - WHATSAPP admite: TAKEAWAY, DELIVERY (nunca DINE_IN).
  - API admite: TAKEAWAY, DELIVERY (nunca DINE_IN).
- **FR-007**: El sistema DEBE informar de forma clara por qué se rechazó la creación de una orden
  cuando la combinación de canal y tipo de orden no es válida.
- **FR-008**: El panel de tipo de orden de la pantalla de creación de pedido manual DEBE habilitar
  la opción "Para Llevar" (hoy deshabilitada), permitiendo seleccionarla igual que "En Mesa".
- **FR-009**: Al seleccionar "Para Llevar" en la creación de pedido manual, el sistema NO DEBE exigir
  la selección de ninguna mesa para poder confirmar y enviar el pedido.
- **FR-010**: Al seleccionar "Para Llevar" en la creación de pedido manual, el sistema DEBE mostrar
  el campo "Cliente" con el mismo comportamiento ya definido para "En Mesa" (spec 054): valor por
  defecto "Consumidor final" en modo de solo lectura, con opción de editarlo, sin quedar nunca
  vacío.
- **FR-011**: Al confirmar un pedido creado con "Para Llevar", el sistema DEBE guardarlo con tipo de
  orden TAKEAWAY, canal POS, sin mesa asociada, y con el nombre de cliente mostrado en pantalla en
  ese momento (por defecto o editado).
- **FR-012**: La opción "Domicilio" del panel de tipo de orden de la creación de pedido manual
  permanece deshabilitada — esta mejora no habilita la creación de pedidos de domicilio desde esa
  pantalla.
- **FR-013**: El sistema DEBE reclasificar cada pedido existente antes de esta mejora con uno de los
  valores estandarizados de canal, según su valor original: los pedidos con canal `qr` quedan como
  QR_MENU; los pedidos con canal `counter` o `waiter` quedan como POS.
- **FR-014**: El sistema DEBE asignar tipo de orden DINE_IN a cada pedido existente antes de esta
  mejora que tenga una mesa asignada; los pedidos existentes sin ninguna mesa asignada quedan sin
  tipo de orden asignado.
- **FR-015**: Todo pedido creado a partir de esta mejora (por cualquier canal) DEBE quedar siempre
  con un tipo de orden asignado del catálogo estandarizado — nunca queda sin asignar.

### Key Entities *(include if feature involves data)*

- **Orden (pedido de un cliente)**: entidad ya existente. Se estandarizan/agregan dos atributos de
  clasificación, ambos de un catálogo fijo de valores e indexados para poder filtrar: **canal de
  origen** (de dónde se creó el pedido: punto de venta, menú QR, WhatsApp, o integración externa) y
  **tipo de orden** (cómo se atiende el pedido: consumo en el local, para llevar, o domicilio). La
  combinación de ambos atributos está sujeta a las reglas de validez de negocio descritas en
  FR-006.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El personal del punto de venta puede crear un pedido "para llevar" completo (sin
  seleccionar ninguna mesa) usando el mismo flujo de armado de pedido que ya usa para "en mesa".
- **SC-002**: El 100% de los pedidos nuevos creados después de esta mejora quedan clasificados con
  un canal y un tipo de orden del catálogo estandarizado, sin valores libres ni vacíos.
- **SC-003**: El 100% de los intentos de crear un pedido con una combinación de canal y tipo de
  orden no permitida (por ejemplo WHATSAPP + DINE_IN) son rechazados por el sistema, evitando que se
  registren pedidos con una combinación ilógica.
- **SC-004**: El 100% de los pedidos existentes antes de esta mejora quedan reclasificados con un
  canal estandarizado, y los que tenían una mesa asignada quedan además clasificados como DINE_IN.
- **SC-005**: El equipo del negocio puede filtrar y contar pedidos por canal de origen y por tipo de
  orden sobre el conjunto completo de pedidos (históricos y nuevos).

## Assumptions

- El canal de una orden nunca es una elección manual del cajero ni del cliente: se asigna
  automáticamente según el punto de entrada real de creación (terminal del punto de venta, menú QR,
  WhatsApp, integración externa).
- Los valores actuales `counter` y `waiter` no tienen una diferencia de negocio relevante entre sí —
  ambos representan personal del punto de venta creando el pedido directamente — por lo que ambos se
  reclasifican como POS.
- Los canales WHATSAPP y API todavía no tienen ningún punto real de creación de pedidos en el
  sistema (no existe hoy ni bot de WhatsApp ni una integración externa activa). Esta mejora
  estandariza el modelo de datos y las reglas de validación para cuando esos canales existan, sin
  construir dichas integraciones — quedan fuera de alcance.
- El canal y el tipo de orden de un pedido no se pueden modificar después de creado; no se
  construye ningún flujo de edición posterior.
- El comportamiento del campo "Cliente" con valor por defecto "Consumidor final" para "Para Llevar"
  reutiliza exactamente el mismo comportamiento ya construido para "En Mesa" en spec 054 (solo
  lectura, editable, nunca vacío), sin ninguna variación adicional.
- Habilitar la opción "Domicilio" en el panel de tipo de orden de la creación de pedido manual queda
  fuera de alcance de esta spec — solo se habilita "Para Llevar", tal como lo pidió el usuario de
  forma explícita.
