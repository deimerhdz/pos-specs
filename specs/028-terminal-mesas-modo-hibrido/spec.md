# Feature Specification: Rediseño Híbrido de la Terminal de Mesas — Validación QR y Cobro Manual (Skeilopos)

**Feature Branch**: `028-terminal-mesas-modo-hibrido`

**Created**: 2026-08-20

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad **nueva** de experiencia de usuario (fase de evolución
funcional, Principio I de la [Constitución](../../.specify/memory/constitution.md)). Se construye
directamente sobre [spec 026](../026-mejoras-ux-comanda/spec.md) — que ya define el envío
automático a cocina al confirmar el pago, la visualización del cambio en efectivo, y la división de
cuenta con varios métodos de pago en el panel "Cuenta de la mesa" — y sobre
[spec 024](../024-pagos-ordenes-mesa/spec.md) (métodos de pago configurables, aprobación/rechazo de
comprobantes) y [spec 010](../010-sesion-mesa-reparto-cierre-barrido/spec.md) /
[spec 011](../011-venta-mostrador-constructor-factura/spec.md) (reparto por ítem, venta y factura).
Ninguna de esas specs está implementada todavía en código (la captura de referencia muestra
exactamente el estado que spec 026 se proponía corregir: pestañas redundantes y el botón genérico
"Cobrar y cerrar mesa" produciendo el error "La orden no tiene un pago confirmado" /
"No se pudo aprobar el comprobante"), así que esta spec **amplía y reemplaza el alcance de diseño de
spec 026 para la misma pantalla** en lugar de coexistir con ella: reutiliza sus reglas de negocio ya
definidas (auto-envío a cocina, cálculo y visualización del cambio, reglas de reparto y cobro
múltiple) y agrega lo que spec 026 no cubría — que la Terminal de Mesas debe comportarse de forma
distinta según el **origen de la orden** (pedido por QR ya pre-pagado, vs. orden creada manualmente
por el cajero en caja) — porque tratar ambos orígenes con la misma interfaz es precisamente la causa
raíz de la inconsistencia y el error reportado en la captura.

**Input**: User description: rediseño de la UI y el flujo de la Terminal de Mesas para soportar un
modelo híbrido: (1) flujo principal por menú QR/auto-pago, donde el comensal pide y reporta su pago
y el cajero valida el comprobante, libera la orden a cocina y emite la factura; (2) flujo manual de
atención en caja, donde el cajero crea la orden para clientes que no usan QR, cobra directamente y
envía a cocina. Se adjuntó una captura del estado actual de la Terminal de Mesas mostrando las
inconsistencias a corregir: pestañas redundantes "Pedido de la mesa" / "Pagos por confirmar", y un
botón genérico "Cobrar y cerrar mesa" en la barra lateral que, al usarse sobre una orden ya pagada
por QR, produce un error ("La orden no tiene un pago confirmado" / "No se pudo aprobar el
comprobante"). Se pide: un único bloque central de "Validación de Pago Requerida" para mesas con
pedido QR pendiente (con ver comprobante, rechazar con motivo, y confirmar-y-enviar-a-cocina, con
impresión automática de factura/comanda activable); un estado vacío con acción destacada
"+ Crear Orden Manual" (atajo F3) para mesas libres; una barra lateral derecha reactiva al origen de
la orden — modo "Resumen de Cuenta" de solo lectura para órdenes QR (sin selector de método de pago
ni botón de cobro genérico, con accesos a "Imprimir Pre-cuenta" y "Reimprimir Factura POS"), y modo
"Terminal POS / Cobro Inmediato" para órdenes manuales (selector de método de pago con cálculo de
cambio, datos de facturación, y un botón único "Cobrar, Facturar y Enviar a Cocina"); mantener el
módulo de facturación e impresión de tirilla térmica (80mm/58mm) tal como funcionaba antes, tanto
para la validación QR como para el cobro manual; corregir el bug de raíz eliminando el botón
genérico de cobro para órdenes QR; y mostrar en el sidebar de mesas insignias de estado (Por
confirmar en amarillo, En preparación en azul, Libre en gris/verde).

## Clarifications

### Session 2026-08-20

- Q: Una vez que un pedido (QR o manual) ya está pagado y enviado a cocina, ¿qué debe devolver esa
  mesa al estado "Libre" — una acción manual explícita "Cerrar Mesa / Liberar Mesa" disponible para
  el cajero, o únicamente el barrido automático ya existente (spec 010), que puede tardar hasta 6
  horas y hoy depende sobre todo de que los comensales pulsen "Salir" en su propia sesión QR? → A:
  Se mantiene una acción manual explícita "Cerrar Mesa / Liberar Mesa", disponible en ambos modos
  (Resumen de Cuenta y Terminal POS), además del barrido automático ya existente como respaldo — sin
  reabrir las reglas de cierre ya definidas en spec 010 (rechazo si queda algo por cobrar o ítems sin
  terminar en cocina).
- Q: Si una mesa ya tiene una orden manual activa (el cajero la está atendiendo directamente) y un
  comensal de esa misma mesa escanea el QR para pedir por su cuenta, ¿el sistema debe bloquear ese
  pedido QR, permitir que ambos orígenes convivan como órdenes independientes en la misma sesión, o
  convertir automáticamente toda la sesión a modo QR? → A: Bloquear — el sistema no permite que se
  cree un pedido QR mientras exista una orden manual activa en esa mesa, en simetría exacta con
  FR-013 (que ya bloquea la dirección opuesta); el mesero registra cualquier ítem adicional
  manualmente.
- Q: En una mesa QR con varios comensales pagando por separado, si el cajero confirma el pago de un
  comensal mientras el de otro sigue pendiente de revisión, ¿los ítems de ese comensal deben pasar a
  cocina de inmediato, o toda la mesa espera a que se resuelvan todos los pagos antes de enviar algo?
  → A: Por comensal — cada ítem pasa a cocina en cuanto se confirma el pago de ese comensal
  específico, sin esperar a los demás; la insignia de la mesa (FR-014) muestra "Por confirmar"
  mientras quede al menos un comensal pendiente, y pasa a "En preparación" solo cuando todos quedan
  resueltos.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El cajero valida un pago QR desde un único bloque, sin botones que fallen (Priority: P1)

Cuando una mesa tiene un pedido hecho por QR con el pago pendiente de revisión, el cajero ve un
único bloque central "Validación de Pago Requerida" — sin pestañas redundantes que dupliquen la
misma información — con la tarjeta de pago del comensal (método, comprobante si aplica), y desde
ahí puede ver el comprobante, rechazarlo con un motivo, o confirmarlo. Al confirmar, el pedido pasa
de inmediato a cocina y la factura/tirilla se genera, sin que exista en la barra lateral ningún
botón de cobro genérico que el cajero pueda pulsar por error sobre esta misma orden y que produzca
un error de "pago no confirmado".

**Why this priority**: es el problema central reportado — el botón "Cobrar y cerrar mesa" de la
barra lateral, pensado para cobro manual, se ofrece también sobre órdenes ya pre-pagadas por QR y al
pulsarlo falla con un error confuso ("La orden no tiene un pago confirmado"); mientras ese botón
siga expuesto en ese contexto, el bug puede repetirse sin importar cuánto se mejore el bloque
central.

**Independent Test**: abrir una mesa con un pedido QR pendiente de validación, confirmar que solo
existe un bloque de validación (no pestañas separadas), que la barra lateral no ofrece ningún
selector de método de pago ni botón de cobro para esa orden, y que aprobar el comprobante envía el
pedido a cocina y genera la factura sin ningún error.

**Acceptance Scenarios**:

1. **Given** una mesa con un pedido QR cuyo pago está pendiente de revisión, **When** el cajero abre
   la mesa, **Then** ve un único bloque "Validación de Pago Requerida" (no pestañas separadas de
   "Pedido de la mesa" y "Pagos por confirmar" mostrando la misma orden).
2. **Given** el bloque de validación con un comprobante de transferencia adjunto, **When** el
   cajero pulsa "Ver Comprobante", **Then** el sistema muestra la imagen del comprobante en una
   vista ampliada (modal o previsualización) sin salir de la pantalla de la mesa.
3. **Given** el mismo bloque, **When** el cajero pulsa "Rechazar", **Then** el sistema exige
   seleccionar o escribir un motivo antes de completar el rechazo (reutilizando la regla ya vigente
   de spec 024) y no permite rechazar sin motivo.
4. **Given** el comprobante o el efectivo reportado son correctos, **When** el cajero pulsa
   "Confirmar y Enviar a Cocina", **Then** el pedido queda pagado y visible para cocina en la misma
   acción (spec 026, FR-001), y el sistema genera el documento de venta/factura correspondiente.
5. **Given** una mesa con una orden de origen QR en cualquier estado (pendiente de validar, o ya
   pagada), **When** el cajero mira la barra lateral derecha, **Then** no encuentra en ningún
   momento el botón genérico "Cobrar y cerrar mesa" ni un selector manual de método de pago
   asociado a esa orden — de forma que ya no es posible reproducir el error "La orden no tiene un
   pago confirmado" ni "No se pudo aprobar el comprobante" sobre una orden QR.
6. **Given** una mesa con dos comensales pagando por separado, uno con el pago ya confirmado y otro
   todavía pendiente de revisión, **When** el cajero mira la insignia de la mesa, **Then** los
   ítems del comensal confirmado ya están en cocina, la insignia de la mesa sigue mostrando "Por
   confirmar" mientras el segundo comensal siga pendiente, y pasa a "En preparación" solo cuando
   también se resuelve el segundo.

---

### User Story 2 - El cajero crea y cobra una orden manual para un cliente sin QR (Priority: P1)

Para una mesa libre, el cajero puede iniciar una orden manual con una acción destacada
("+ Crear Orden Manual" o el atajo F3), construir el pedido desde el catálogo, y cobrarlo
directamente desde la barra lateral derecha —ahora en modo "Terminal POS / Cobro Inmediato"— sin
pasar por el flujo de validación de comprobantes que solo aplica a pedidos QR.

**Why this priority**: es el otro flujo principal descrito por el usuario (atención en caja para
clientes que no usan QR) y hoy la Terminal de Mesas no ofrece ninguna forma clara de iniciarlo desde
una mesa libre; sin esto, el cajero no tiene una vía consistente para atender a ese tipo de cliente
desde esta misma pantalla.

**Independent Test**: seleccionar una mesa libre, crear una orden manual, agregar productos desde el
catálogo, cobrar en efectivo con cálculo de cambio (o con transferencia/datafono), y verificar que
el pedido se marca pagado, se envía a cocina y queda una factura generada, todo sin usar el flujo de
validación de comprobantes.

**Acceptance Scenarios**:

1. **Given** una mesa libre, **When** el cajero la selecciona, **Then** ve un estado vacío claro con
   una acción destacada "+ Crear Orden Manual", también accionable con el atajo de teclado F3.
2. **Given** una orden manual en construcción, **When** el cajero agrega productos desde el
   catálogo, **Then** la barra lateral derecha muestra el desglose de la cuenta actualizado en modo
   "Terminal POS / Cobro Inmediato" (no en modo "Resumen de Cuenta" de solo lectura).
3. **Given** una orden manual con productos agregados, **When** el cajero selecciona "Efectivo" como
   método de pago e ingresa el monto recibido, **Then** el sistema calcula y muestra el cambio a
   entregar antes de confirmar el cobro.
4. **Given** la misma orden, **When** el cajero selecciona "Transferencia/Datafono" en lugar de
   efectivo, **Then** el sistema no exige comprobante ni pasa por un paso de revisión — el cajero
   confirma directamente que el pago fue recibido, porque en el cobro manual el cajero presencia la
   transacción.
5. **Given** una orden manual lista para cobrar, **When** el cajero pulsa "Cobrar, Facturar y Enviar
   a Cocina", **Then** el sistema registra el pago, genera la factura, y envía el pedido a cocina en
   la misma acción.
6. **Given** el paso de cobro manual, **When** el cajero revisa los datos de facturación, **Then**
   por defecto la venta queda a nombre de "Consumidor Final" y el cajero puede reemplazarlo por los
   datos de un cliente específico antes de confirmar el cobro.

---

### User Story 3 - El cajero reimprime la factura, adelanta la pre-cuenta, y libera la mesa cuando ya está todo pagado (Priority: P2)

Desde la cabecera de la mesa o la barra lateral, el cajero tiene acceso directo a "Imprimir
Pre-cuenta" (antes de que el pago quede confirmado), a "Reimprimir Factura POS" (después de que la
venta ya quedó registrada), y a "Cerrar Mesa" / "Liberar Mesa" (una vez todo está pagado y los
comensales se han retirado), sin que ninguna de esas acciones reprocese el pago o intente cobrar de
nuevo.

**Why this priority**: soporta la operación diaria (atasco de papel, solicitud del cliente de una
copia) y depende de que los modos "Resumen de Cuenta" y "Terminal POS" (Historias 1 y 2) ya existan
como contextos distintos donde ofrecer estas acciones; por eso es P2.

**Independent Test**: sobre una mesa con orden QR ya pagada, usar "Reimprimir Factura POS" y
verificar que se reimprime el mismo documento sin alterar el pago ni la orden; sobre una mesa con
orden aún sin pagar, usar "Imprimir Pre-cuenta" y verificar que imprime el detalle de consumo sin
marcar la orden como pagada.

**Acceptance Scenarios**:

1. **Given** una orden QR pendiente de validación, **When** el cajero pulsa "Imprimir Pre-cuenta",
   **Then** el sistema imprime el desglose de consumo actual sin cambiar el estado de pago de la
   orden.
2. **Given** una orden (QR o manual) ya pagada y facturada, **When** el cajero pulsa "Reimprimir
   Factura POS", **Then** el sistema reimprime la misma tirilla/factura ya emitida, sin generar un
   nuevo documento de venta ni volver a cobrar.
3. **Given** una impresora sin papel o con atasco durante la reimpresión, **When** el cajero
   reintenta, **Then** el sistema le permite reintentar la impresión sin duplicar el registro de
   venta.
4. **Given** una mesa (QR o manual) ya pagada, facturada y con los comensales retirados, **When**
   el cajero pulsa "Cerrar Mesa" / "Liberar Mesa" desde la cabecera o la barra lateral, **Then** la
   mesa vuelve de inmediato al estado "Libre", sin esperar al barrido automático (spec 010).
5. **Given** una mesa con algo todavía por cobrar o con ítems sin terminar en cocina, **When** el
   cajero intenta "Cerrar Mesa" / "Liberar Mesa", **Then** el sistema rechaza la acción con el mismo
   motivo ya definido en spec 010 (nada que cobrar pendiente, ítems de cocina sin terminar), sin
   liberar la mesa.

---

### User Story 4 - El personal identifica el estado de cada mesa por su insignia en el listado (Priority: P3)

En el listado lateral de mesas, cada mesa muestra una insignia de estado clara: "Por confirmar"
(amarillo) para mesas con pago QR pendiente de revisión, "En preparación" (azul) para mesas con
pedido ya pagado y enviado a cocina, y "Libre" (gris/verde) para mesas sin consumo activo.

**Why this priority**: es una mejora de reconocimiento visual que depende de que los estados de las
Historias 1 y 2 ya existan con nombres y transiciones consistentes; no corrige por sí sola ningún
flujo roto, por lo que se prioriza al final.

**Independent Test**: mostrar el listado de mesas con al menos una mesa en cada estado (por
confirmar, en preparación, libre) y verificar que cada una muestra la insignia de color y texto
correspondiente, sin ambigüedad entre estados.

**Acceptance Scenarios**:

1. **Given** una mesa con un pago QR pendiente de revisión, **When** se muestra en el listado,
   **Then** su insignia dice "Por confirmar" en amarillo.
2. **Given** una mesa con un pedido ya pagado (QR o manual) y enviado a cocina, **When** se muestra
   en el listado, **Then** su insignia dice "En preparación" en azul.
3. **Given** una mesa sin ningún consumo activo, **When** se muestra en el listado, **Then** su
   insignia dice "Libre" en gris o verde.
4. **Given** cualquiera de las tres insignias, **When** se observa bajo poca luz o sin distinguir
   colores, **Then** el estado sigue siendo identificable porque la insignia siempre combina color y
   texto (regla ya vigente en spec 026, FR-003).

---

### Edge Cases

- **Una mesa QR recibe un nuevo intento de pago mientras el cajero ya tiene abierto el bloque de
  validación de un intento anterior rechazado**: el bloque central se actualiza para mostrar el
  intento vigente; no deben coexistir dos tarjetas de validación activas para el mismo comensal
  (regla ya vigente en spec 024: solo un intento pendiente a la vez).
- **El cajero intenta crear una orden manual sobre una mesa que en ese momento ya tiene un pedido QR
  entrante**: el sistema no permite mezclar ambos orígenes en la misma sesión de mesa activa; debe
  completarse o resolverse la orden existente antes de habilitar "+ Crear Orden Manual" sobre esa
  mesa, o la orden manual debe abrirse sobre una mesa distinta.
- **Un comensal escanea el QR de una mesa que ya tiene una orden manual activa** (el cajero la está
  atendiendo directamente): el sistema bloquea la creación de ese pedido QR, en simetría exacta con
  el caso anterior; cualquier ítem adicional para esa mesa se registra manualmente por el mesero
  hasta que la orden manual se resuelva.
- **El toggle "Imprimir factura/comanda automáticamente al confirmar" está desactivado por el
  cajero**: al confirmar el pago QR, el pedido igual pasa a cocina de inmediato (eso no es opcional,
  spec 026 FR-001), pero la impresión física de la tirilla/comanda queda pendiente de una acción
  manual posterior del cajero en vez de dispararse sola.
- **El cajero pulsa "Cobrar, Facturar y Enviar a Cocina" dos veces seguidas sobre la misma orden
  manual** (doble clic, o reintento tras una respuesta lenta): el sistema garantiza que la orden se
  cobra y factura una sola vez, sin generar dos ventas ni descontar el inventario dos veces (misma
  garantía ya exigida en spec 024 FR-018 y spec 025 para envíos duplicados).
- **Falla el descuento de inventario al confirmar un cobro manual** (stock insuficiente descubierto
  en el momento de cobrar): el sistema no cobra ni envía a cocina a medias; informa al cajero de
  inmediato con una vía clara para resolverlo, igual que ya exige spec 026 FR-002 para el flujo QR.
- **Se intenta reimprimir una factura de una orden que nunca llegó a pagarse** (por ejemplo, fue
  cancelada o el pago fue rechazado sin reintento): el sistema no ofrece "Reimprimir Factura POS"
  para esa orden porque nunca existió un documento de venta que reimprimir; solo "Imprimir
  Pre-cuenta" permanece disponible mientras la orden siga activa.
- **Datos de facturación de una orden manual sin nombre de cliente específico**: la venta se registra
  válidamente a nombre de "Consumidor Final" sin bloquear el cobro.
- **El cajero pulsa "Cerrar Mesa" / "Liberar Mesa" mientras todavía queda algo por cobrar o hay
  ítems sin terminar en cocina**: el sistema rechaza la acción con el mismo motivo ya definido en
  spec 010, y la mesa permanece en su estado actual.
- **El cajero pulsa "Cerrar Mesa" dos veces seguidas casi al mismo tiempo (doble clic, o dos
  cajeros distintos)**: el sistema garantiza que solo una liberación tiene efecto, reutilizando el
  bloqueo de fila ya exigido en spec 010 (`RN-MESA-01`) — nunca se generan dos cierres ni se pierde
  el registro de a quién se le cobró.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Para una mesa con un pedido QR pendiente de validación de pago, el sistema DEBE
  mostrar un único bloque "Validación de Pago Requerida" en el área central — el sistema NO DEBE
  presentar esa misma información duplicada en pestañas separadas.
- **FR-002**: El bloque de validación DEBE ofrecer, por cada intento de pago pendiente, las acciones
  "Ver Comprobante" (previsualización de la imagen subida), "Rechazar" (exige motivo, reutilizando
  spec 024) y "Confirmar y Enviar a Cocina" (reutilizando el envío automático a cocina y el descuento
  de inventario ya definidos en spec 026, FR-001/FR-002). En una mesa con varios comensales pagando
  por separado, esta acción opera por comensal: confirmar el pago de un comensal envía solo sus
  ítems a cocina de inmediato, sin esperar a que los demás comensales de la misma mesa resuelvan su
  propio pago.
- **FR-003**: El bloque de validación DEBE incluir una opción de facturación, activa por defecto,
  para imprimir automáticamente la factura/comanda al confirmar el pago; el cajero DEBE poder
  desactivarla antes de confirmar, en cuyo caso la orden pasa igualmente a cocina pero la impresión
  física queda pendiente de una acción manual posterior.
- **FR-004**: Para una mesa libre (sin pedido activo), el sistema DEBE mostrar un estado vacío con
  una acción destacada "+ Crear Orden Manual", accionable también mediante el atajo de teclado F3.
- **FR-005**: La barra lateral derecha DEBE presentarse en uno de dos modos exclusivos, determinado
  por el origen de la orden activa de la mesa:
  - **Modo "Resumen de Cuenta"** (orden de origen QR): de solo lectura — muestra comensal(es),
    desglose de ítems, total y método de pago elegido, sin selector manual de método de pago ni
    botón de cobro.
  - **Modo "Terminal POS / Cobro Inmediato"** (orden de origen manual): editable — selector de
    catálogo, desglose de cuenta, selector de método de pago (efectivo con cálculo de cambio, o
    transferencia/datafono), datos de facturación, y una única acción de cobro.
- **FR-006**: El sistema NO DEBE mostrar el botón "Cobrar y cerrar mesa" (ni ningún botón de cobro
  genérico) en el modo "Resumen de Cuenta" — eliminando la vía que hoy produce el error de "pago no
  confirmado" al intentar cobrar una orden QR que ya se valida y confirma desde el bloque central.
- **FR-007**: En modo "Resumen de Cuenta", el sistema DEBE ofrecer como acciones secundarias
  "Imprimir Pre-cuenta" (antes del pago) y "Reimprimir Factura POS" (después de facturada) —esta
  última solo disponible si ya existe un documento de venta emitido para esa orden—, ninguna de las
  dos DEBE reprocesar el pago ni generar un nuevo documento de venta.
- **FR-008**: En modo "Terminal POS / Cobro Inmediato", al seleccionar efectivo como método de pago,
  el sistema DEBE calcular y mostrar el cambio a entregar antes de confirmar el cobro (reutilizando
  spec 026, FR-004).
- **FR-009**: En modo "Terminal POS / Cobro Inmediato", al seleccionar transferencia o datafono como
  método de pago, el sistema NO DEBE exigir comprobante ni un paso de revisión —el cajero confirma
  directamente que el pago fue recibido, dado que en el cobro manual el cajero presencia la
  transacción en el momento—.
- **FR-010**: En modo "Terminal POS / Cobro Inmediato", el sistema DEBE permitir capturar datos de
  facturación del cliente, con "Consumidor Final" como valor por defecto cuando el cajero no
  proporciona datos específicos.
- **FR-011**: La acción de cobro del modo "Terminal POS / Cobro Inmediato" ("Cobrar, Facturar y
  Enviar a Cocina") DEBE, en un solo paso, registrar el pago, generar el documento de venta/factura
  y enviar el pedido a cocina con su inventario descontado — con la misma garantía de no dejar la
  orden en un estado ambiguo si el descuento de inventario falla (spec 026, FR-002) y la misma
  protección contra doble ejecución (spec 024, FR-018).
- **FR-012**: El sistema DEBE ofrecer un acceso directo a "Reimprimir Factura" desde la cabecera de
  la mesa o la barra lateral, disponible para cualquier orden (QR o manual) que ya tenga un
  documento de venta emitido, sin límite de veces ni efecto sobre el pago o el inventario ya
  registrados.
- **FR-013**: El sistema NO DEBE permitir iniciar una orden manual sobre una mesa que ya tiene un
  pedido de origen QR activo en la misma sesión, ni viceversa —NO DEBE permitir que se cree un
  pedido de origen QR sobre una mesa que ya tiene una orden manual activa—; ningún origen se mezcla
  dentro de una misma sesión activa.
- **FR-014**: El listado de mesas DEBE mostrar, para cada mesa, una insignia de estado con color y
  texto: "Por confirmar" (amarillo) cuando existe al menos un intento de pago QR pendiente de
  revisión en esa mesa (aunque otros comensales de la misma mesa ya estén confirmados y enviados a
  cocina), "En preparación" (azul) cuando el pedido ya está pagado y visible para cocina —o, en una
  mesa con varios comensales, cuando todos ellos ya quedaron resueltos—, y "Libre" (gris/verde)
  cuando la mesa no tiene consumo activo — reutilizando el requisito de que el estado nunca dependa
  solo del color (spec 026, FR-003).
- **FR-015**: El sistema NO DEBE reducir, respecto de la interfaz actual, ninguna información hoy
  disponible sobre el pedido, el pago o la factura (reutiliza spec 026, FR-012).
- **FR-016**: El sistema DEBE ofrecer una acción manual explícita "Cerrar Mesa" / "Liberar Mesa",
  accesible desde la cabecera de la mesa o la barra lateral en ambos modos (Resumen de Cuenta y
  Terminal POS), que el cajero usa una vez el consumo ya está pagado y los comensales se han
  retirado, sin depender de ni esperar al barrido automático (spec 010). Esta acción reutiliza
  exactamente el mecanismo de cierre ya definido en spec 010 (`close_session`) y sus condiciones de
  rechazo (nada pendiente por cobrar, ningún ítem sin terminar en cocina) sin reabrirlas.

### Key Entities *(include if feature involves data)*

Esta spec no agrega entidades nuevas — reutiliza **Orden**, **Intento de Pago**, **Comprobante** y
**Método de Pago** (spec 024), y **Sesión de Mesa / Cuenta de la Mesa**, **Venta**, **Pago** y
**Factura** (spec 010, spec 011, spec 016). Se introduce el concepto de negocio **origen de la
orden** (QR vs. manual/caja) como criterio que determina qué modo de la barra lateral y qué acciones
se muestran — este origen ya se conoce hoy por cómo se creó la orden (vía menú QR del comensal, o
vía acción del cajero en la Terminal de Mesas) y esta spec no cambia cómo se determina, solo cómo se
usa para decidir la interfaz.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las mesas con pago QR pendiente muestran un único bloque de validación (sin
  pestañas duplicadas) y el 0% de los intentos de cobro sobre una orden QR desde la barra lateral
  producen el error "pago no confirmado" o "no se pudo aprobar el comprobante", porque ese botón ya
  no existe en ese contexto.
- **SC-002**: El 100% de las mesas libres ofrecen la acción "+ Crear Orden Manual" visible y
  accionable en menos de 2 pasos (clic o atajo F3).
- **SC-003**: El 100% de las órdenes manuales cobradas en efectivo muestran el cambio a entregar
  antes de confirmar el cobro, y el 100% de las cobradas por transferencia/datafono se confirman sin
  exigir carga de comprobante.
- **SC-004**: El 100% de los cobros manuales completados ("Cobrar, Facturar y Enviar a Cocina")
  dejan, en la misma acción, un documento de venta emitido y el pedido visible para cocina.
- **SC-005**: El 100% de las reimpresiones de factura ("Reimprimir Factura POS") entregan el mismo
  documento ya emitido sin crear un segundo registro de venta ni alterar el pago.
- **SC-006**: El personal identifica correctamente, en una prueba de reconocimiento visual sobre el
  listado de mesas, los tres estados (por confirmar, en preparación, libre) por su insignia de color
  y texto, sin ayuda.
- **SC-007**: Ninguna información hoy visible sobre pedido, pago o factura deja de estar disponible
  tras el rediseño.
- **SC-008**: El 100% de las mesas ya pagadas y con comensales retirados pueden liberarse mediante
  "Cerrar Mesa" en menos de 10 segundos, sin esperar al barrido automático; el 100% de los intentos
  de liberar una mesa con algo pendiente por cobrar o cocina sin terminar son rechazados con el
  motivo correspondiente.

## Out of Scope

- Las reglas de negocio ya definidas en spec 024 (métodos de pago configurables por tenant,
  aprobación/rechazo de comprobantes con motivo) y spec 026 (envío automático a cocina al confirmar
  el pago, cálculo del cambio, reparto de cuenta y cobro con varios métodos, tamaños mínimos de
  texto y controles táctiles) — se reutilizan sin modificación; esta spec solo decide cuándo y a
  quién se le muestra cada pieza según el origen de la orden.
- Reabrir la prohibición de dividir la cuenta de forma porcentual o "a partes iguales" (spec 010,
  RN-MESA-05).
- Integración con proveedores de facturación electrónica (DIAN) o generación de factura electrónica
  con validez fiscal ante terceros — la captura de "datos de facturación" en el cobro manual (FR-010)
  es únicamente para identificar al cliente en el ticket/POS emitido, no para una integración de
  facturación electrónica nueva.
- Nuevos métodos de pago o integraciones con pasarelas, bancos o datáfonos — "Transferencia/Datafono"
  en el cobro manual sigue siendo un registro manual del cajero, no una integración con el
  dispositivo físico del datáfono.
- Cambios al motor de cálculo de factura, promociones, combos o descuentos (spec 011, spec 012,
  spec 013).
- Definir las propuestas visuales concretas (mockups, colores exactos, disposición pixel a pixel) —
  esta especificación define comportamiento y criterios de éxito verificables; las alternativas
  visuales se resuelven en la fase de planeación (`/speckit-plan`).

## Assumptions

- **Esta spec reemplaza el alcance de diseño de spec 026 para la Terminal de Mesas** en lugar de
  coexistir con ella, porque ninguna de las dos está implementada todavía y ambas describen la misma
  pantalla; las reglas de negocio ya definidas en spec 026 (auto-envío a cocina, cálculo de cambio,
  reparto de cuenta) se mantienen intactas y se incorporan aquí por referencia.
- **El origen de la orden (QR vs. manual) ya es un dato disponible** en el momento en que se crea la
  orden (según se originó desde el menú QR del comensal o desde una acción del cajero en la Terminal
  de Mesas); esta spec no cambia cómo se registra ese origen, solo lo usa para decidir el modo de la
  barra lateral.
- **El modo "Terminal POS / Cobro Inmediato" no ofrece división de cuenta entre varios comensales**
  como paso obligatorio — una orden manual normalmente corresponde a un único cliente en caja; si en
  el futuro se requiere dividir una orden manual entre varios comensales, se reutiliza el mecanismo
  ya definido en spec 026 (Historia 3) sin duplicarlo aquí.
- **El pago por transferencia/datafono en el cobro manual no requiere comprobante fotográfico**
  porque el cajero presencia la transacción directamente (a diferencia del flujo QR, donde el
  comensal reporta el pago de forma remota y por eso sí requiere comprobante revisable).
- **"Datos de facturación" en el cobro manual** se limita a identificar al cliente en el documento
  emitido (nombre, y datos básicos si aplica), con "Consumidor Final" como valor por defecto; no
  implica construir una integración de facturación electrónica nueva (ver Out of Scope).
- **El toggle de impresión automática (FR-003)** solo controla si el documento se imprime físicamente
  de inmediato; no controla si el pedido se envía a cocina, lo cual ya no es opcional desde spec 026.
- **Una mesa nunca mezcla, dentro de la misma sesión activa, un pedido de origen QR con una orden
  manual**: si conviven ambos casos operativamente (por ejemplo, un cliente llega sin QR a una mesa
  que ya tenía un pedido QR en curso), se resuelve como dos sesiones o se completa una antes de
  iniciar la otra, evitando que la barra lateral deba mostrar ambos modos a la vez.
