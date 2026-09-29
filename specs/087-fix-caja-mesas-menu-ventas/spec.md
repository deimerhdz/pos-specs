# Feature Specification: Correcciones de Caja, Terminal de Mesas, Menú QR y Ventas

**Feature Branch**: `087-fix-caja-mesas-menu-ventas`

**Created**: 2026-09-28

**Status**: Implemented (US1–US7); adenda US8–US11 pendiente de implementar

**Input**: User description: "## Objetivo y Contexto de Negocio
Resolver fallos de usabilidad, persistencia de órdenes, cálculos de precios y visualización
de reportes en los módulos de Caja, Terminal de Mesas, Menú de Usuario (QR/Terminal) y
Ventas. El objetivo es garantizar el flujo correcto de pedidos abiertos, la precisión en
los cobros con adicionales y la generación limpia de reportes de cierre.

## Usuarios
- **Cajero / Administrador:** Encargado del cierre de turno de caja y consulta de detalles
  de ventas.
- **Mesero / Operador de Terminal:** Encargado de tomar y actualizar pedidos en mesas, para
  llevar y a domicilio.
- **Cliente (Menú QR):** Usuario final que visualiza el menú y configura sus productos con
  presentaciones y adicionales.

## Escenarios de Usuario (Historias)
- **Historia 1 (Cierre de Caja Limpio):** Como cajero, quiero que al cerrar el turno la
  opción de imprimir genere un documento PDF enfocado exclusivamente en el resumen
  financiero (con el nombre del tenant y fecha actual) sin capturar elementos de la
  interfaz como el sidebar.
- **Historia 2 (Consolidación de Órdenes):** Como mesero, quiero agregar productos
  adicionales a un pedido abierto existente (en mesa, para llevar o domicilio) sin que el
  sistema cree órdenes duplicadas o separadas para la misma solicitud.
- **Historia 3 (Transparencia en Menú y Precios):** Como cliente o mesero, quiero ver
  claramente la presentación, los adicionales y las notas de un producto, asegurando que
  el precio total refleje la suma correcta de la promoción más los adicionales
  seleccionados, en cualquier pantalla donde se muestre o valide el total.
- **Historia 4 (Auditoría de Ventas):** Como administrador, quiero consultar en el detalle
  de cada venta el valor exacto del cambio (devuelta) entregado al cliente, cuando
  aplique.

## Requisitos Funcionales

### A. Módulo de Caja
1. Eliminar por completo la opción de "Arqueo Parcial" de la interfaz de caja, incluyendo
   su historial, manteniendo únicamente el flujo de caja única.
2. Al imprimir el cierre de turno, renderizar únicamente el resumen financiero, excluyendo
   sidebar/header/navegación. Nombre de archivo: `<tenant-slug>-<DD-MM-YYYY>.pdf`, con el
   slug en minúsculas, sin tildes/eñes/símbolos, espacios reemplazados por guiones.

### B. Terminal de Mesas
1. Mostrar de forma destacada el nombre del cliente en la tarjeta y detalle del pedido;
   este dato pasa a ser obligatorio para crear cualquier pedido nuevo.
2. Corregir el bug de numeración: hoy el pedido más reciente se muestra como #1 y empuja a
   los anteriores hacia abajo; debe corregirse para que el número se asigne una sola vez
   por hora de creación ascendente y no cambie después.
3. Botón "Crear pedido manual" en el detalle de una mesa, incluso si ya tiene un pedido
   abierto (pedidos en paralelo).
4. Al agregar productos a un pedido no pagado, anexarlos al mismo order_id; si el pedido ya
   fue pagado/cerrado, no se debe permitir editarlo.

### C. Menú de Usuario (QR y Terminal)
1. Mostrar la presentación del producto junto al nombre en resumen e historial.
2. Corregir la sumatoria del precio total cuando un producto en promoción incluye
   adicionales: Precio Final = Precio Promocional + Σ Precio Adicionales.
3. Resaltar visualmente adicionales y notas, mostrando siempre el multiplicador exacto por
   adicional (ej. "Queso extra x1", "Choco-chips x2").

### D. Módulo de Ventas
1. Incluir el campo Cambio / Devuelta en el detalle de una venta.

## Reglas de Negocio (con Ejemplos)
- Si el Pedido A de mesa se crea a las 10:00 AM y el Pedido B a las 10:05 AM, A mantiene
  #1 y B mantiene #2 de forma estable.
- Ejemplo de suma de adicionales: Granizado de Mora (promo $5.000) + Leche condensada
  ($2.000) + Choco-chips ($1.500) = $8.500 COP.
- El slug del PDF limpia tildes/eñes/símbolos a alfanumérico.
- Una orden "abierta" es la que no está pagada/cerrada.

## Casos Límite
- Agregar productos a una orden pagada/cerrada requiere abrir una nueva orden.
- Generar el PDF para un tenant con eñes/tildes/símbolos: el slug se limpia.
- Pedidos sin adicionales ni notas no deben mostrar contenedores vacíos.

## Clarificaciones resueltas antes de esta especificación
- El fix de cálculo de adicionales sobre promociones aplica a TODAS las superficies de
  cobro (armado del pedido, checkout, Pagos por confirmar, detalle de venta).
- El bug de numeración es de reordenamiento (el más nuevo se vuelve #1); se corrige para
  que sea estable por orden de creación, y aplica solo a pedidos de mesa.
- Arqueo Parcial se elimina incluyendo su historial completo.
- El campo Cambio/Devuelta se oculta por completo cuando el pago no incluyó componente en
  efectivo.
- El nombre del cliente pasa a ser obligatorio para pedidos nuevos (mesas, para llevar,
  domicilio); no se migra en pedidos históricos, que muestran un placeholder.
- Si la orden ya fue pagada, agregar productos crea una nueva orden; si sigue abierta
  (no pagada), los nuevos items se anexan a la misma orden.
- "Crear pedido manual" permite un pedido adicional en paralelo aunque la mesa ya tenga uno
  abierto; "Agregar producto" siempre opera sobre el pedido específico ya identificado en
  pantalla, sin ambigüedad de a cuál pedido pertenece.
- El multiplicador de cantidad de un adicional se muestra siempre, incluso cuando es x1."

## Clarifications

### Session 2026-09-28

- Q: ¿La corrección del cálculo de adicionales sobre promociones (Regla 2) debe aplicarse
  en todas las superficies que muestran o validan el total, o solo al armar el pedido? →
  A: En todas las superficies de cobro: armado del pedido (Menú QR/Terminal), checkout,
  "Pagos por confirmar" (revisión de pago QR por el cajero) y detalle de venta.
- Q: ¿Cómo se define el alcance de la numeración secuencial ascendente de pedidos? → A: Es
  un bug de reordenamiento en los chips de la Terminal de Mesas: el pedido más reciente se
  muestra como #1 y empuja a los anteriores hacia abajo. Debe corregirse para que el número
  se asigne una sola vez por hora de creación y permanezca estable, y aplica únicamente a
  pedidos de mesa (no a para llevar ni domicilio).
- Q: Al deprecar "Arqueo Parcial", ¿qué alcance tiene sobre datos e historial existentes? →
  A: Se elimina también el historial; ya no se podrá consultar arqueos parciales pasados.
- Q: ¿Cómo debe comportarse el campo "Cambio / Devuelta" cuando el pago no fue en efectivo
  o fue dividido entre varios métodos? → A: El campo se oculta por completo del detalle de
  venta cuando el método de pago no incluyó ningún componente en efectivo.
- Q: ¿Es obligatorio el nombre del cliente y qué se muestra si falta? → A: Es obligatorio
  para crear cualquier pedido nuevo (mesas, para llevar, domicilio); aplica solo hacia
  adelante — los pedidos históricos sin nombre no se migran y se muestran con un
  placeholder (ej. "Cliente sin nombre").
- Q: Al agregar productos a una orden ya enviada a cocina/impresa, ¿qué debe pasar? → A: Si
  la orden ya fue pagada, se crea una nueva orden con esos items; si sigue abierta (no
  pagada), los nuevos items se anexan a la orden ya abierta, sin importar si ya fue enviada
  a cocina.
- Q: ¿En qué escenario está disponible el botón "Crear pedido manual" en el detalle de una
  mesa? → A: Incluso si la mesa ya tiene un pedido abierto, permitiendo un segundo pedido
  independiente en paralelo sobre la misma mesa.
- Q: Con pedidos en paralelo por mesa, ¿cómo sabe el sistema a cuál pedido abierto anexar
  productos al usar "Agregar producto"? → A: No aplica ambigüedad: esa acción siempre se
  accede desde el detalle de un pedido específico ya identificado.
- Q: ¿Se muestra el multiplicador "xN" de un adicional también cuando la cantidad es 1? →
  A: Sí, siempre se muestra el multiplicador, sin excepción (ej. "Queso extra x1").
- Q: El nombre de cliente obligatorio, ¿aplica también a pedidos históricos (migración)? →
  A: Solo aplica hacia adelante, a pedidos nuevos; los históricos no se migran.
- Q: Al eliminar "Arqueo Parcial" con su historial, ¿los registros deben borrarse
  físicamente de la base de datos o solo ocultarse/archivarse (datos conservados, sin
  acceso desde la UI)? → A: Eliminación física: los registros de arqueo parcial se borran
  permanentemente de la base de datos mediante una migración de datos.
- Q: El número secuencial de los pedidos de mesa, ¿se reinicia en algún momento (cada día,
  cada turno de caja) o es continuo para siempre? → A: Se reinicia en cada apertura de un
  nuevo turno de caja; el primer pedido de mesa de cada turno vuelve a ser #1.
- Q: Si dos personas agregan productos casi al mismo tiempo al mismo pedido abierto, ¿deben
  conservarse siempre ambas adiciones? → A: Una mesa es atendida por un solo mesero, por lo
  que no hay colisión mesero-contra-mesero sobre el mismo pedido. El caso adicional de
  mesero y cliente (Menú QR) agregando al mismo tiempo sobre el mismo pedido queda fuera de
  alcance de esta especificación: no se exige una garantía explícita de fusión/no-pérdida
  para esa colisión.
- Q: Si un producto no tiene ninguna presentación definida, ¿qué debe mostrarse junto a su
  nombre? → A: Nada; se omite por completo la etiqueta de presentación, igual que se omiten
  los contenedores vacíos de adicionales y notas.
- Q: El fix de FR-011, al mostrarse en el "detalle de venta", ¿debe recalcular y mostrar el
  total corregido también para ventas ya emitidas antes del despliegue, o solo aplica a
  ventas nuevas a partir del despliegue? → A: Solo aplica a ventas nuevas; ninguna venta ya
  emitida se recalcula retroactivamente (Principio VII — inmutabilidad de facturas).
- Q: El nombre del archivo PDF de cierre de turno (`<tenant-slug>-<DD-MM-YYYY>.pdf`), ¿usa la
  fecha del momento en que se imprime o la fecha de cierre del turno? → A: La fecha de cierre
  del turno (`closed_at`), para que el nombre identifique siempre al mismo turno sin importar
  cuándo se descargue o reimprima el reporte.

### Session 2026-09-29

- Q (segunda ronda, mismo día): en el modal del Menú QR solo se veía la promoción y **nunca** el
  nombre de la presentación, incluso tras corregir el truncado de US8 → A: la causa no era CSS.
  Desde spec 084 (A-79) el backend serializa el nombre de la variante como `presentation_name`
  y el mapper del comensal (`diner.service.ts::mapCategory`) leía `name` (`undefined`); la
  Terminal (`menu.service.ts`) sí lo mapeaba bien. Se corrige en esta misma US8 (T093–T096) y se
  amplía A-91; sin cambio de contrato de API.

- Q: Las 4 correcciones reportadas tras la implementación (presentación + promoción en el
  Menú QR, adicionales en la Terminal de Mesas, total al editar un pedido manual, tamaño de
  fuente de las notas), ¿se agregan a esta spec 087 o van en una spec nueva 088? → A: Se
  agregan a la spec 087 (mismo directorio y rama), como nuevas historias y requisitos
  (US8 en adelante, FR-014 en adelante), con `/speckit-plan` y `/speckit-tasks`
  incrementales sobre 087.
- Q: Tras editar un pedido manual (agregar o quitar productos) y guardar, ¿en qué pantalla se
  ve el total desactualizado? → A: En el "TOTAL ORDEN" del panel del pedido manual de la
  Terminal de Mesas, justo después de guardar. El pedido no persiste `subtotal`/`total`
  (se calculan al vuelo desde los ítems), por lo que la corrección apunta al recálculo del
  estado local y de la vista previa del cobro, no a un campo guardado.
- Q: En la Terminal de Mesas, ¿en qué vista no aparecen los adicionales/toppings de los
  productos? → A: En la vista de detalle y pedidos de la mesa (panel del pedido de la
  terminal). El requisito aplica a todas las líneas de esa vista — tanto ítems ya guardados
  del pedido como ítems recién agregados en borrador —, sin distinguir entre ambos.
- Q: En el Menú QR del cliente, ¿en qué pantalla se pierde el nombre de la presentación y
  solo se ve el texto de la promoción? → A: En el modal del producto, en la lista "Elige tu
  presentación" (cada fila muestra el nombre de la presentación junto al texto de la
  promoción). Es la superficie de alcance del problema; carta, carrito e historial no se
  modifican por este requisito.
- Q: Al agrandar y destacar las notas, ¿cuáles notas y en qué pantallas deben cambiar? → A:
  La nota por producto en todas las pantallas que usan el componente compartido
  `app-cart-item-options` (panel del pedido de la Terminal de Mesas, página de pedido manual
  y Menú QR del cliente), con un único estilo reforzado. La nota general del pedido
  (`order.notes`) no cambia.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Cobro correcto de adicionales sobre promociones (Priority: P1)

Como cliente que arma su pedido desde el Menú QR (o como mesero desde la Terminal), y como
cajero que revisa un pago, quiero que el precio total mostrado en cualquier pantalla de
cobro refleje siempre la suma exacta del precio promocional más el valor de cada
adicional seleccionado, para que a nadie se le cobre de más ni de menos por un error de
cálculo del sistema.

**Why this priority**: Es un error que afecta directamente el dinero cobrado a los
clientes y el dinero registrado en caja. Es el de mayor severidad de negocio: un cálculo
incorrecto de precios erosiona la confianza del cliente y descuadra la caja, sin importar
qué tan bien funcionen las demás historias.

**Independent Test**: Puede probarse por completo armando un pedido con un producto en
promoción y dos adicionales, y verificando que el mismo total correcto aparece de forma
consistente en el resumen del pedido, en el checkout, en "Pagos por confirmar" y en el
detalle de venta final.

**Acceptance Scenarios**:

1. **Given** el producto "Granizado de Mora" tiene precio promocional de $5.000 COP,
   **When** un cliente le agrega "Leche condensada" ($2.000 x1) y "Choco-chips" ($1.500
   x1) desde el Menú QR, **Then** el total mostrado antes de enviar el pedido es $8.500
   COP.
2. **Given** un pedido con un producto en promoción y adicionales ya fue enviado por el
   cliente, **When** el cajero lo revisa en la pantalla "Pagos por confirmar", **Then** el
   total mostrado ahí coincide exactamente con el total mostrado al cliente al armar el
   pedido.
3. **Given** una venta realizada **después del despliegue de este fix** que incluyó un
   producto en promoción con adicionales, **When** se abre el detalle de esa venta en el
   módulo de Ventas, **Then** el total registrado coincide con Precio Promocional + Σ
   Precio Adicionales. Las ventas emitidas **antes** del despliegue conservan su total
   histórico tal como fue cobrado, sin recalcularse (Principio VII de la constitución).

---

### User Story 2 - Consolidación de pedidos abiertos sin duplicados (Priority: P1)

Como mesero, quiero agregar productos adicionales a un pedido que sigue abierto (en mesa,
para llevar o domicilio) y que el sistema los anexe al mismo pedido existente, para no
generar registros duplicados que descuadren el conteo de pedidos y los reportes de venta.

**Why this priority**: Los pedidos duplicados corrompen los reportes de cierre de caja y
de ventas, y generan confusión operativa en cocina. Es tan crítico como la Historia 1
porque también compromete la integridad de los datos financieros del negocio.

**Independent Test**: Puede probarse creando un pedido en una mesa, usando "Agregar
producto" sobre ese pedido (mientras siga sin pagar) y verificando que el total del pedido
aumenta manteniendo el mismo número/order_id, sin que aparezca un segundo pedido nuevo.

**Acceptance Scenarios**:

1. **Given** un pedido de mesa abierto (no pagado) con un producto ya registrado, **When**
   el mesero selecciona "Agregar producto", añade un ítem y presiona "Guardar", **Then**
   el total de la mesa aumenta y el pedido conserva su mismo número, sin crear un pedido
   nuevo.
2. **Given** un pedido para llevar o a domicilio abierto (no pagado), **When** se le
   agregan productos adicionales y se guarda, **Then** los ítems se anexan al mismo
   pedido existente.
3. **Given** un pedido que ya fue pagado y cerrado, **When** alguien intenta agregarle
   productos, **Then** el sistema lo impide y guía a crear un pedido nuevo en su lugar.

---

### User Story 3 - Numeración estable y cronológica de pedidos de mesa (Priority: P2)

Como mesero u operador de la Terminal, quiero que el número asignado a cada pedido de mesa
se mantenga fijo desde el momento en que se crea, en estricto orden de llegada, para poder
identificar y comunicar cada pedido sin que su número cambie cada vez que entra uno nuevo.

**Why this priority**: Es un bug de usabilidad operativa (genera confusión al comunicar
"el pedido 1" entre mesero y cocina) pero no compromete directamente el dinero cobrado ni
la integridad de los datos de venta, a diferencia de las Historias 1 y 2.

**Independent Test**: Puede probarse creando dos pedidos de mesa consecutivos y verificando
que sus números no se alteran al crear un tercero.

**Acceptance Scenarios**:

1. **Given** el Pedido A de mesa se crea a las 10:00 a. m., **When** cinco minutos después
   se crea el Pedido B en otra mesa, **Then** A se muestra como #1 y B como #2.
2. **Given** los pedidos A (#1) y B (#2) ya existen, **When** se crea un tercer pedido C,
   **Then** C se muestra como #3 y los números de A y B no cambian.

---

### User Story 4 - Cierre de caja limpio, sin Arqueo Parcial (Priority: P2)

Como cajero, quiero que al cerrar el turno la opción de imprimir genere un PDF enfocado
solo en el resumen financiero, con un nombre de archivo predecible, y que la interfaz de
caja ya no ofrezca la opción de "Arqueo Parcial", para trabajar únicamente con el flujo de
caja única y entregar un reporte limpio y presentable.

**Why this priority**: Mejora la calidad del reporte que se entrega o archiva por cierre
de turno y reduce una opción de flujo que ya no debe usarse, pero no bloquea la operación
diaria de cobro como sí lo hacen las Historias 1 y 2.

**Independent Test**: Puede probarse abriendo el módulo de Caja y confirmando que no existe
ningún botón de Arqueo Parcial, y luego cerrando un turno y descargando el PDF de cierre
para revisar su nombre y contenido.

**Acceptance Scenarios**:

1. **Given** un usuario con acceso al módulo de Caja, **When** navega por su interfaz y su
   historial, **Then** no encuentra ningún botón, acceso ni registro consultable de
   "Arqueo Parcial".
2. **Given** un cajero cierra el turno de un tenant llamado "Tenant de Prueba" el
   28-09-2026, **When** presiona "Imprimir" en el resumen de cierre, **Then** se descarga
   un PDF llamado `tenant-de-prueba-28-09-2026.pdf` que contiene solo el resumen
   financiero, sin sidebar ni navegación.
3. **Given** un tenant cuyo nombre contiene eñes, tildes o símbolos especiales, **When** se
   genera el PDF de cierre, **Then** el nombre del archivo usa un slug limpio, solo con
   caracteres alfanuméricos y guiones.

---

### User Story 5 - Nombre de cliente visible y obligatorio, y pedidos en paralelo por mesa (Priority: P2)

Como mesero, quiero ver de forma destacada el nombre del cliente en cada pedido y poder
abrir un pedido adicional independiente en una mesa que ya tiene uno abierto, para atender
correctamente mesas compartidas o solicitudes que llegan por separado.

**Why this priority**: Resuelve una carencia operativa concreta (identificar al cliente
correcto y atender pedidos paralelos en la misma mesa) pero es de menor severidad
financiera que las Historias 1 y 2.

**Independent Test**: Puede probarse creando un pedido de mesa con nombre de cliente
obligatorio, confirmando que se muestra destacado en la tarjeta y el detalle, y luego
usando "Crear pedido manual" sobre esa misma mesa para verificar que se abre un segundo
pedido independiente.

**Acceptance Scenarios**:

1. **Given** un mesero intenta guardar un pedido nuevo sin diligenciar el nombre del
   cliente, **When** presiona guardar, **Then** el sistema bloquea el guardado hasta que
   se capture el nombre.
2. **Given** un pedido con nombre de cliente capturado, **When** se visualiza su tarjeta o
   detalle, **Then** el nombre aparece de forma destacada y visualmente explícita.
3. **Given** una mesa que ya tiene un pedido abierto, **When** el mesero presiona "Crear
   pedido manual" desde el detalle de esa mesa, **Then** se crea un segundo pedido
   independiente asociado a la misma mesa, con su propio número y total.
4. **Given** un pedido histórico creado antes de este cambio y sin nombre de cliente
   registrado, **When** se visualiza su tarjeta o detalle, **Then** se muestra un
   placeholder (ej. "Cliente sin nombre") en vez de bloquear su consulta.

---

### User Story 6 - Adicionales, notas y presentación explícitos en el resumen del pedido (Priority: P3)

Como cliente o mesero, quiero ver claramente la presentación del producto, cada adicional
con su cantidad exacta y las notas del cliente en el resumen del pedido, para confirmar
antes de enviarlo que refleja exactamente lo que se pidió.

**Why this priority**: Es una mejora de claridad visual que reduce errores de
interpretación, pero no corrige un cálculo de dinero ni un bug de duplicación de datos, por
lo que tiene menor severidad que las historias anteriores.

**Independent Test**: Puede probarse armando un pedido con presentación, dos adicionales
distintos y una nota, y verificando que el resumen muestra los tres de forma explícita y
resaltada, y que un pedido sin adicionales ni notas no muestra contenedores vacíos.

**Acceptance Scenarios**:

1. **Given** un producto con presentación "16oz", **When** se agrega al pedido, **Then**
   el resumen y el historial del pedido muestran la presentación junto al nombre del
   producto.
2. **Given** un pedido con el adicional "Queso extra" seleccionado una vez y "Choco-chips"
   seleccionado dos veces, **When** se revisa el resumen del pedido, **Then** se muestra
   "Queso extra x1" y "Choco-chips x2".
3. **Given** un pedido sin adicionales ni notas, **When** se revisa su resumen, **Then**
   no se muestra ningún contenedor vacío de adicionales o notas.

---

### User Story 7 - Desglose de cambio en el detalle de venta (Priority: P3)

Como administrador, quiero consultar en el detalle de una venta pagada en efectivo el valor
exacto del cambio entregado al cliente, para poder auditar los cobros realizados en caja.

**Why this priority**: Es un dato de auditoría útil pero no bloquea ninguna operación de
cobro ni de toma de pedidos, por lo que es la de menor severidad del conjunto.

**Independent Test**: Puede probarse abriendo el detalle de una venta pagada en efectivo y
verificando que aparece la línea de cambio, y abriendo el detalle de una venta pagada 100%
con un método distinto a efectivo para verificar que esa línea no aparece.

**Acceptance Scenarios**:

1. **Given** una venta pagada en efectivo donde el cliente entregó más dinero del valor de
   la venta, **When** se abre su detalle, **Then** se visualiza la línea "Cambio: $X.XXX"
   con el valor exacto entregado como devuelta.
2. **Given** una venta pagada al 100% con un método distinto a efectivo (tarjeta, QR),
   **When** se abre su detalle, **Then** no aparece la línea de Cambio.

---

### User Story 8 - Presentación siempre visible junto a la promoción en el modal del producto (Priority: P2)

Como cliente que escanea el Menú QR, quiero que cada fila de "Elige tu presentación"
muestre siempre el nombre de la presentación (ej. 8 oz, Mediano, Grande) y, aparte, la
promoción como etiqueta complementaria, para saber qué tamaño estoy eligiendo aunque tenga
una promoción aplicada.

**Why this priority**: No altera cobros ni datos, pero un cliente que no distingue el
tamaño puede pedir el producto equivocado; es una confusión al momento de la decisión.

**Independent Test**: Abrir en el Menú QR un producto con varias presentaciones y una
promoción vigente en al menos una, y verificar que en cada fila el nombre de la
presentación se lee completo y la promoción aparece como etiqueta secundaria.

**Acceptance Scenarios**:

1. **Given** una presentación "8 oz" con una promoción "2x1", **When** el cliente abre el
   modal del producto, **Then** la fila muestra "8 oz" como etiqueta principal y "2x1" como
   etiqueta secundaria, nunca "2x1" en lugar de "8 oz".
2. **Given** una presentación "Mediano" con un descuento del 15%, **When** se abre el modal,
   **Then** la fila muestra "Mediano" como etiqueta principal y el descuento como etiqueta
   secundaria.
3. **Given** una presentación sin promoción, **When** se abre el modal, **Then** la fila
   muestra solo el nombre de la presentación, sin etiqueta de promoción vacía.
4. **Given** el Menú QR del comensal, cargado desde `GET /menu/qr-token/{token}` (que entrega el
   nombre como `presentation_name`), **When** se abre el modal de un producto con varias
   presentaciones, **Then** cada fila muestra su nombre ("Grande", "Mediano"…) exactamente igual
   que en la Terminal de Mesas, tenga o no promoción (corrección 2026-09-29, T093–T094).

---

### User Story 9 - Adicionales visibles en el detalle y pedidos de la mesa (Priority: P1)

Como mesero, quiero ver debajo de cada producto del detalle de la mesa los adicionales
(toppings) que el cliente eligió, para preparar y entregar el pedido completo.

**Why this priority**: Sin los adicionales visibles el personal puede preparar un pedido
incompleto; es un error operativo directo, del mismo nivel que los cobros incorrectos.

**Independent Test**: Crear un pedido con un producto con dos adicionales (uno con
cantidad 2), guardarlo, y verificar en el panel del pedido de la Terminal de Mesas que
ambos aparecen debajo del producto tanto antes de guardar (borrador) como después de
guardado.

**Acceptance Scenarios**:

1. **Given** un producto con los adicionales "Queso extra" (x1) y "Choco-chips" (x2)
   recién agregado al borrador del pedido, **When** se ve el panel del pedido, **Then**
   ambos adicionales aparecen debajo del producto en formato "Nombre xN".
2. **Given** el mismo pedido ya guardado y recargado, **When** se ve el panel del pedido,
   **Then** los mismos adicionales siguen apareciendo debajo del producto, en el mismo
   formato.
3. **Given** un adicional que ya no está activo en el menú actual, **When** se ve un pedido
   guardado que lo incluye, **Then** el adicional sigue apareciendo (no se omite en
   silencio).

---

### User Story 10 - Total del pedido actualizado al editar un pedido manual (Priority: P1)

Como mesero, quiero que el "TOTAL ORDEN" del pedido manual se actualice al agregar o quitar
productos y al guardar, para no cobrar ni informar un valor desactualizado.

**Why this priority**: Un total desactualizado es un error de dinero visible al cliente,
igual de crítico que el cálculo de adicionales sobre promociones (US1).

**Independent Test**: Abrir un pedido manual existente con un total conocido, agregar un
producto, quitar otro, guardar, y verificar que el "TOTAL ORDEN" del panel coincide con la
suma de los ítems vigentes sin recargar la pantalla.

**Acceptance Scenarios**:

1. **Given** un pedido manual con total $10.000, **When** se agrega un producto de $4.000
   y se guarda, **Then** el "TOTAL ORDEN" muestra $14.000 sin recargar la pantalla.
2. **Given** ese pedido con ítems agregados, **When** se anula/quita un producto y se
   guarda, **Then** el "TOTAL ORDEN" descuenta el valor del producto quitado.
3. **Given** un pedido con una promoción aplicada, **When** se agregan o quitan ítems y se
   guarda, **Then** el total refleja la promoción reevaluada sobre los ítems vigentes.

---

### User Story 11 - Notas por producto legibles y destacadas (Priority: P2)

Como personal de la terminal, quiero que la nota de cada producto (instrucciones de
cocina y observaciones) se lea con facilidad y destaque sobre el resto del pedido, para no
pasar por alto una instrucción especial.

**Why this priority**: Mejora de legibilidad; una nota pasada por alto genera errores de
preparación pero no de cobro.

**Independent Test**: Crear un pedido con una nota en un producto y verificar, en las tres
pantallas afectadas, que la nota se ve con texto de al menos 16px, en negrita o semibold, y
sobre un fondo de alto contraste.

**Acceptance Scenarios**:

1. **Given** un producto con la nota "sin azúcar", **When** se ve el panel del pedido de la
   terminal, la página de pedido manual o el historial del Menú QR, **Then** la nota se
   muestra con texto de al menos 16px, peso semibold o bold y fondo de alto contraste.
2. **Given** un producto sin nota, **When** se ve el pedido, **Then** no se muestra ningún
   contenedor de nota vacío.
3. **Given** una nota larga, **When** se ve el pedido, **Then** el texto completo es
   legible (se ajusta en varias líneas) sin desbordar la tarjeta ni ocultar el producto.

---

### Edge Cases

- Intentar agregar productos a un pedido ya pagado/cerrado: el sistema lo impide y guía a
  crear un pedido nuevo; no se edita el pedido cerrado.
- Generar el PDF de cierre de caja para un tenant cuyo nombre contenga eñes, tildes o
  símbolos especiales: el slug del archivo se limpia a caracteres alfanuméricos y guiones.
- Pedidos que no tienen adicionales ni notas: no se muestran contenedores vacíos en el
  resumen.
- Un producto sin presentación definida (no maneja variantes de tamaño): no se muestra
  ninguna etiqueta de presentación junto a su nombre.
- Una mesa con más de un pedido abierto en paralelo: cada uno se distingue claramente
  (número, hora, total), y "Agregar producto" siempre opera sobre el pedido específico
  abierto en pantalla, nunca sobre la mesa en general.
- Un pedido histórico sin nombre de cliente registrado: se muestra con un placeholder, sin
  bloquear su consulta ni requerir completarlo retroactivamente.
- Una venta con pago dividido donde ninguna porción fue en efectivo: no muestra la línea de
  Cambio, igual que un pago 100% no efectivo.
- Un adicional guardado en un pedido que después se desactiva o se elimina del menú: sigue
  mostrándose en el panel del pedido de la terminal (no se omite en silencio).
- Editar un pedido manual y dejarlo sin ningún ítem vigente: el "TOTAL ORDEN" pasa a $0, sin
  mostrar un total anterior obsoleto.
- Una nota de producto muy larga: se ajusta en varias líneas sin desbordar la tarjeta ni
  ocultar el nombre del producto.
- Una presentación con nombre largo y promoción a la vez: el nombre no se trunca hasta
  volverse ilegible; la etiqueta de promoción baja a otra línea si hace falta.
- Reimprimir el resumen de un turno ya cerrado, o imprimir dos turnos distintos cerrados el
  mismo día calendario: el nombre del archivo usa la fecha de cierre del turno, no la fecha
  de impresión, para identificar siempre al turno correcto sin colisionar con otro reporte.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST eliminar por completo la opción "Arqueo Parcial" de la
  interfaz de caja — sin botón, acceso ni forma de consultar arqueos parciales creados
  antes de este cambio — manteniendo únicamente el flujo de caja única. Los registros de
  arqueo parcial existentes MUST eliminarse físicamente de la base de datos mediante una
  migración de datos (borrado permanente, no archivado).
- **FR-002**: Al presionar "Imprimir" en el resumen de cierre de turno, el sistema MUST
  generar un PDF que contenga únicamente el componente de resumen financiero, excluyendo
  el sidebar, header y navegación del POS.
- **FR-003**: El PDF de cierre de caja MUST nombrarse `<tenant-slug>-<DD-MM-YYYY>.pdf`,
  donde `<DD-MM-YYYY>` es la fecha de **cierre del turno** (`closed_at`), no la fecha en que
  se imprime o reimprime el reporte, para que el nombre identifique siempre al mismo turno
  sin importar cuándo se descargue; el slug del tenant está en minúsculas, sin
  tildes/eñes/símbolos especiales (limpiados a caracteres alfanuméricos) y con espacios
  reemplazados por guiones.
- **FR-004**: El sistema MUST mostrar el nombre del cliente de forma destacada y
  visualmente explícita en la tarjeta y en el detalle de cada pedido.
- **FR-005**: El sistema MUST exigir el nombre del cliente como campo obligatorio al crear
  cualquier pedido nuevo (mesa, para llevar o domicilio), bloqueando el guardado si no se
  diligencia. Este requisito aplica solo a pedidos creados a partir de esta funcionalidad;
  los pedidos históricos sin nombre no se migran y se muestran con un placeholder (ej.
  "Cliente sin nombre").
- **FR-006**: El sistema MUST asignar a cada pedido de mesa un número secuencial ascendente
  según su hora de creación, y ese número MUST permanecer estable una vez asignado, sin
  recalcularse ni desplazarse al crearse pedidos nuevos. Este requisito aplica únicamente a
  pedidos de mesa. El contador MUST reiniciarse en #1 en cada apertura de un nuevo turno de
  caja (el número es único y estable dentro de un turno, no a través de todo el histórico).
- **FR-007**: El sistema MUST ofrecer un botón "Crear pedido manual" en la vista detallada
  de una mesa, disponible incluso si esa mesa ya tiene un pedido abierto, permitiendo crear
  un segundo pedido independiente en paralelo sobre la misma mesa.
- **FR-008**: Cuando se seleccione "Agregar producto" sobre un pedido (mesa, para llevar o
  domicilio) que no ha sido pagado, el sistema MUST anexar los nuevos ítems a ese mismo
  pedido al guardar, sin crear un registro de pedido nuevo.
- **FR-009**: El sistema MUST impedir agregar productos a un pedido que ya fue pagado o
  cerrado, y MUST guiar al usuario a crear un pedido nuevo en su lugar.
- **FR-010**: El sistema MUST mostrar la presentación del producto (ej. 8oz, 16oz, Grande)
  junto a su nombre, tanto en el resumen del pedido como en su historial. Si el producto no
  tiene ninguna presentación definida, el sistema MUST omitir por completo la etiqueta de
  presentación (sin mostrar texto genérico ni contenedor vacío).
- **FR-011**: El sistema MUST calcular el precio total de un producto en promoción con
  adicionales como Precio Promocional + Σ Precio Adicionales, y MUST mostrar ese mismo total
  correcto de forma consistente en toda superficie que presente o valide el total: armado
  del pedido (Menú QR/Terminal), checkout, "Pagos por confirmar" y detalle de venta. Este
  cálculo corregido aplica únicamente a pedidos y checkouts realizados a partir del
  despliegue de esta funcionalidad; ninguna venta ya emitida (`Sale.total`,
  `Sale.change_given`) se recalcula retroactivamente.
- **FR-012**: El sistema MUST resaltar visualmente, en el resumen del pedido, cada
  adicional seleccionado junto con su cantidad exacta usando el formato "Nombre xN"
  (mostrando el multiplicador incluso cuando N=1) y las notas/observaciones del cliente, y
  MUST omitir cualquier contenedor de adicionales o notas cuando el pedido no tiene ninguno.
- **FR-013**: El sistema MUST incluir en el detalle de una venta el campo "Cambio /
  Devuelta" únicamente cuando el método de pago de esa venta incluyó un componente en
  efectivo; MUST omitirlo cuando el pago fue 100% con un método distinto a efectivo.

- **FR-014**: En el modal del producto ("Elige tu presentación", componente compartido
  `app-product-select`, por lo que aplica al Menú QR y también a la Terminal de Mesas / pedido
  manual / catálogo de la terminal), el sistema MUST
  mostrar siempre el nombre de la presentación como etiqueta principal de cada fila, y MUST
  mostrar la promoción vigente de esa presentación únicamente como etiqueta secundaria
  complementaria (ej. "8 oz" + "2x1", "Mediano" + "15% OFF"), sin reemplazar, truncar ni
  ocultar el nombre de la presentación. Si la presentación no tiene promoción, MUST omitir la
  etiqueta secundaria.
- **FR-015**: El sistema MUST mostrar, debajo de cada producto del panel del pedido de la
  Terminal de Mesas (detalle y pedidos de la mesa), los adicionales seleccionados con su
  cantidad en formato "Nombre xN" (FR-012), tanto para ítems ya guardados como para ítems
  recién agregados en borrador. Un adicional ya guardado en el pedido MUST seguir mostrándose
  aunque ya no esté activo en el menú vigente; MUST omitirse el contenedor si el ítem no
  tiene adicionales.
- **FR-016**: Al agregar o quitar productos de un pedido manual existente y guardar, el
  sistema MUST recalcular subtotal, descuento (incluidas promociones reevaluadas sobre los
  ítems vigentes), impuestos y total, y MUST mostrar el "TOTAL ORDEN" actualizado en el panel
  del pedido sin recargar la pantalla. El recálculo MUST ejecutarse cada vez que cambie la
  lista de ítems, y el total mostrado tras guardar MUST coincidir con el que calcule el
  backend al cobrar, también cuando el pedido incluye combos: los combos suman a su precio de
  combo, sin descuento promocional ni impuestos adicionales. El pedido no persiste
  `subtotal`/`total`; el total se deriva de los ítems vigentes.
- **FR-017**: El sistema MUST mostrar la nota de cada producto con un estilo reforzado
  único — tamaño de texto de al menos 16px, peso semibold o bold, y fondo de alto contraste —
  en todas las pantallas que usan el componente compartido `app-cart-item-options` (panel
  del pedido de la Terminal de Mesas, página de pedido manual y Menú QR del cliente). La
  nota general del pedido (`order.notes`) queda fuera de este requisito. MUST omitirse el
  contenedor cuando el producto no tiene nota.

### Key Entities *(include if feature involves data)*

- **Pedido / Orden**: registro de una solicitud de consumo asociada a una mesa, a un
  cliente para llevar o a un cliente de domicilio. Atributos relevantes para esta
  funcionalidad: nombre del cliente, estado (abierta/no pagada vs. pagada/cerrada), hora de
  creación, número secuencial (solo para pedidos de mesa, reiniciado en #1 en cada apertura
  de turno de caja), y la mesa a la que pertenece (una mesa puede tener más de un pedido
  abierto en paralelo).
- **Producto del pedido**: ítem dentro de un pedido, con su presentación, su promoción
  aplicada (si tiene) y sus adicionales seleccionados, cada uno con su cantidad y precio.
- **Adicional**: complemento seleccionable de un producto, con precio propio y cantidad
  elegida por el cliente o mesero.
- **Venta**: registro de un pedido ya cobrado; incluye el método de pago (o combinación de
  métodos) y, cuando aplica, el valor exacto del cambio entregado al cliente.
- **Turno de Caja**: periodo de operación de caja que se cierra generando un resumen
  financiero imprimible; ya no admite el sub-flujo de Arqueo Parcial.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los pedidos **nuevos, creados a partir del despliegue de esta
  funcionalidad**, con productos en promoción y adicionales muestran el mismo total correcto
  (promoción + suma de adicionales) en el armado del pedido, en el checkout, en "Pagos por
  confirmar" y en el detalle de venta, sin discrepancias entre pantallas.
- **SC-002**: El 100% de los pedidos abiertos a los que se les agregan productos
  adicionales conservan su mismo número de pedido y no generan un registro duplicado.
- **SC-003**: El número asignado a un pedido de mesa no cambia en el 100% de los casos tras
  la creación de pedidos posteriores.
- **SC-004**: 0% de los cierres de turno de caja generan un PDF que incluya elementos de
  navegación del sistema (sidebar, header), y el 100% de los archivos descargados cumplen
  el patrón de nombre `<tenant-slug>-<DD-MM-YYYY>.pdf`.
- **SC-005**: 0% de las pantallas de caja ofrecen acceso a crear o consultar un Arqueo
  Parcial tras esta funcionalidad.
- **SC-006**: El 100% de los pedidos nuevos creados exigen y muestran el nombre del cliente
  de forma destacada antes de poder guardarse.
- **SC-007**: El 100% de los detalles de venta pagados con algún componente en efectivo
  muestran la línea de Cambio, y el 0% de los pagados sin ningún componente en efectivo la
  muestran.
- **SC-008**: El 100% de las filas de presentación del modal del producto en el Menú QR (cargado por
  el token del comensal, no solo por la Terminal)
  muestran el nombre de la presentación, incluso las que tienen promoción vigente.
- **SC-009**: El 100% de los ítems con adicionales del panel del pedido de la Terminal de
  Mesas (guardados y en borrador) muestran todos sus adicionales con cantidad "Nombre xN".
- **SC-010**: Tras agregar o quitar productos y guardar un pedido manual, el "TOTAL ORDEN"
  mostrado coincide en el 100% de los casos con la suma de los ítems vigentes (menos
  descuento, más impuestos) sin recargar la pantalla.
- **SC-011**: El 100% de las notas por producto se muestran con texto de al menos 16px, peso
  semibold o bold y fondo de alto contraste en las tres pantallas afectadas, sin errores de
  tipo ni de consola durante el renderizado o el recálculo de montos.

## Assumptions

- (Sesión 2026-09-29) El pedido no persiste `subtotal`/`total`: se derivan de los ítems
  vigentes; por eso la corrección de US10 es de recálculo y refresco del estado mostrado, no
  de un campo guardado.
- (Sesión 2026-09-29) Las correcciones US8–US11 aplican a pedidos nuevos y existentes por
  igual, porque solo cambian cómo se presenta o recalcula la información; no modifican
  ventas ya emitidas.

- La eliminación de "Arqueo Parcial" incluye su historial: tras esta funcionalidad no
  queda ninguna forma de consultar arqueos parciales creados antes del cambio, y los
  registros correspondientes se borran físicamente de la base de datos vía migración de
  datos (no se conservan archivados).
- El nombre de cliente obligatorio aplica solo hacia adelante; los pedidos históricos sin
  nombre no se migran ni se bloquean, y se muestran con un placeholder al visualizarlos.
- La corrección de numeración secuencial aplica únicamente a pedidos de mesa; los pedidos
  de para llevar y domicilio no están dentro del alcance de esta corrección específica. El
  contador se reinicia en #1 en cada apertura de turno de caja.
- Una mesa puede tener más de un pedido abierto en paralelo (por el botón "Crear pedido
  manual"); la acción "Agregar producto" siempre se ejecuta desde el detalle de un pedido
  específico ya identificado, por lo que no requiere un selector adicional de "a cuál
  pedido agregar".
- Una orden se considera "abierta" cuando su estado no es pagada/cerrada, sin importar si
  ya fue enviada o impresa hacia cocina.
- El campo Cambio/Devuelta se rige por si hubo o no componente en efectivo en el pago de
  esa venta; no se especifica un desglose adicional para pagos divididos entre varios
  métodos donde ninguno fue efectivo.
- Cada mesa es atendida por un solo mesero, por lo que no se espera colisión de ediciones
  simultáneas entre distintos meseros sobre el mismo pedido de mesa. La colisión entre un
  mesero y un cliente (Menú QR) agregando productos al mismo pedido casi al mismo tiempo
  queda fuera de alcance: no se exige una garantía explícita de fusión/no-pérdida para ese
  caso.
