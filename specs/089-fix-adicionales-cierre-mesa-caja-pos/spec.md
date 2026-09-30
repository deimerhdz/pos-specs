# Feature Specification: Correcciones de Adicionales del Menú QR, Cierre de Mesa, Impresión de Caja y Total Inmediato en el POS

**Feature Branch** (rama de la spec en `pos-specs`): `089-fix-adicionales-cierre-mesa-caja-pos`. Las ramas de código de `pos-backend` y `pos-heladeria` siguen el Principio XIV y se definen en `tasks.md` (T003)

**Created**: 2026-09-30

**Status**: Draft

**Input**: User description: "Corregir errores críticos de cálculo de adicionales en el carrito del
Menú QR, habilitar la edición de los adicionales ya elegidos, cerrar automáticamente la sesión QR
cuando se cierra la mesa, solucionar la impresión en blanco del reporte de cierre de caja y eliminar
la alerta bloqueante \"El total cambió\" para que el total de una orden abierta se recalcule al
instante al agregar productos en el POS.

Usuarios: **Cliente (Menú QR)** — necesita un carrito transparente donde ajuste sus adicionales y vea
el cálculo exacto, y una transición clara cuando se cierra su mesa; **Cajero / Mesero (POS)** —
necesita que al agregar productos a una orden abierta el total se actualice de inmediato y sin
popups, y que la impresión del cierre de caja sea legible.

Historias: (1) cantidad de adicionales cobrada exactamente igual a la elegida, sin multiplicarla por la
cantidad de productos; (2) editar o quitar los adicionales de un ítem del carrito sin eliminar el
producto; (3) pantalla \"¡Gracias por tu visita!\" e invalidación de la sesión cuando el personal cierra
la mesa; (4) cierre de caja legible al imprimir (negro sobre blanco, sin hoja en blanco); (5) total
recalculado al instante en una orden activa (mesa, domicilio o para llevar), sin mensaje ni modal
intermedio.

Reglas de negocio: total de línea = (precio del producto × cantidad del producto) + (precio del
adicional × cantidad del adicional). Ejemplo: 2 hamburguesas de $15.000 + 1 adicional de tocino de
$3.000 = **$33.000** (hoy se cobra $36.000, incorrecto). Ejemplo POS: orden abierta con total $20.000,
se agrega 1 gaseosa de $5.000 → el total visible pasa de inmediato a **$25.000**, sin diálogos."

## Clarifications

### Sesión previa a la especificación (2026-09-30)

Respuestas del usuario/negocio, previas a redactar esta spec:

- **Adicionales — cantidad propia**: el cliente elige la cantidad de cada adicional con un selector
  +/−; esa cantidad es independiente de la cantidad del producto y **nunca se auto-escala** (pasar de
  2 a 3 hamburguesas mantiene el adicional en las unidades que el cliente eligió).
- **Alcance de los adicionales**: **solo el Menú QR**. La terminal POS del cajero y el pedido manual
  no cambian su forma de armar líneas con adicionales.
- **Impacto aguas abajo**: la cantidad independiente debe reflejarse igual en el **precio**, el
  **ticket de cocina** y el **descuento de inventario**, y en los **combos/promociones con
  adicionales** (spec 083), para que las tres cuentas cuadren.
- **Datos existentes**: los carritos y pedidos ya creados **no se recalculan**; el cambio aplica solo
  a líneas nuevas o editadas.
- **Edición de adicionales**: solo líneas del carrito **aún sin enviar**; cada comensal edita
  únicamente **sus propias** líneas.
- **Qué significa "cerrar mesa"**: que en esa mesa ya no hay clientes y la mesa queda disponible. Al
  cerrar, el cliente **siempre** ve "¡Gracias por tu visita!" y se bloquea cualquier envío nuevo,
  incluso si hubiera cuentas pendientes.
- **Caminos que cierran la sesión**: **todos** los que pasen la sesión a cerrada (botón del cajero,
  liberación por mesero de la spec 075, liberación automática del scheduler, cobro completo).
- **Reingreso**: reabrir la mesa en el POS **no** reactiva la sesión del cliente anterior; solo un
  nuevo escaneo del QR crea una sesión nueva.
- **Pantalla de gracias**: solo el mensaje, sin historial ni recibo, y con el carrito local limpio.
- **Mecanismo**: reusar el tiempo real existente del comensal (spec 077) con respaldo por validación
  de sesión al reanudar/reconectar.
- **Impresión**: solo el reporte de cierre de caja; el usuario no sabe de dónde sale la hoja en
  blanco, por lo que la causa real se diagnostica en el plan.
- **Total en el POS**: el flujo reportado es "agregar productos → elegir método de pago → clic en
  Cobrar → aparece un modal que dice que el valor cambió → al confirmar, el total se actualiza". Es
  incorrecto: el total debe actualizarse desde el momento en que se guarda el pedido con los
  productos nuevos. Se aprueba una actualización local instantánea que el servidor luego confirma o
  reconcilia, y, si un producto está agotado, un mensaje que **indique cuál**.
- **Superficies**: mesa, domicilio y para llevar, y todas las superficies de cobro que validan el
  total (incluida "Pagos por confirmar", la revisión del pago QR por el cajero).
- **Formato**: una sola spec.

### Session 2026-09-30 (/speckit-clarify)

- Q: ¿Los adicionales del Menú QR son las opciones de cualquier grupo, o solo las que tienen recargo?
  → A: Son las opciones de los grupos **con recargo** (los que suman dinero al precio). Las opciones
  de grupos **incluidos** (sin cargo, p. ej. la elección de sabores) conservan su comportamiento por
  unidad de producto: son la configuración del producto, no un extra cobrado. La lógica de selección
  del grupo (uno o varios, con o sin selector de cantidad, mínimos y máximos) **no cambia**; lo que
  cambia es a cuántas unidades del producto se aplica el cobro.
- Q: ¿Cómo se comporta hoy el cobro del adicional? → A: (hallazgo del reconocimiento) el precio de la
  línea se guarda como "precio de una unidad" (presentación + adicionales × su cantidad) y luego se
  multiplica por la cantidad del producto en el carrito, el checkout, la venta y la factura. Por eso
  2 hamburguesas con 1 tocino cobran 2 tocinos. El consumo de inventario y el ticket de cocina
  siguen la misma lógica por unidad.
- Q: ¿Por qué el cliente no ve siempre "¡Gracias por tu visita!" al cerrar la mesa? → A: (hallazgo)
  (1) el aviso en tiempo real `session.closed` ya existe y lo emiten el cobro completo, el barrido
  del scheduler y el cierre desde el POS con cobro, pero **"Liberar mesa" del POS y el cierre
  automático de una sesión sin órdenes cierran la sesión sin emitirlo**; (2) cuando el aviso sí
  llega, el cliente cae en la pantalla de **ingreso de nombre** con un mensaje, y solo ve la pantalla
  de gracias si había salido voluntariamente. Se corrige que **todo** cierre notifique y que la
  pantalla sea siempre la de gracias.
- Q: ¿De dónde sale el modal "El total cambió"? → A: (hallazgo) es la doble verificación de la spec
  073 (FR-007, D11): al pulsar Cobrar se vuelve a pedir el total y, si difiere del último que la
  pantalla mostró, se pide confirmación. Aparece porque, al agregar productos a una orden ya
  guardada, la vista previa del total **no se refresca**, así que la pantalla muestra un total
  viejo hasta que Cobrar lo descubre. Esta spec corrige la causa (refrescar al guardar) y retira el
  modal como mecanismo de descubrimiento.
- Q: ¿La impresión del cierre de caja genera un PDF en el servidor? → A: (hallazgo) no: usa la
  impresión del navegador (que también permite "Guardar como PDF") tras ocultar el menú lateral y el
  encabezado; el nombre del archivo sale del título del documento (spec 087, US4). La hoja en
  blanco es un defecto de esa impresión, cuya causa exacta se reproduce y se determina en el plan.
- Q: Al cerrarse la mesa, ¿el cliente debe dejar una marca mínima por pestaña (igual que "Salir") para
  que el F5 lo bloquee, o no debe quedar ninguna marca en el navegador? → A: **Marca mínima por
  pestaña, igual que "Salir"** (hallazgo del bug de persistencia: hoy el cierre limpia el token pero
  no deja la marca de acceso cerrado, así que al recargar reaparece el ingreso de nombre y se abre una
  sesión nueva sin volver a escanear). El cierre de mesa MUST ejecutar exactamente el mismo borrado que
  "Salir": purgar todo dato de sesión y de comensal (token de sesión, datos del usuario, carrito,
  `localStorage`, `sessionStorage` y cookies asociadas) y dejar **solo** la marca de acceso cerrado de
  la pestaña (únicamente el token público de la URL de la mesa, sin datos de sesión ni de usuario). El
  enlace del QR es fijo por mesa, de modo que recarga y escaneo nuevo abren la misma URL; la marca por
  pestaña es lo único que los distingue (una pestaña nueva o un escaneo nuevo entran normal). El
  rechazo estricto del backend aplica a la **sesión anterior** (token de sesión o parámetro `s`), no a
  la URL de la mesa: una mesa libre debe poder escanearse para iniciar una sesión.
- Q: Después de un F5, ¿la pantalla "Por favor, escanea nuevamente el código QR de la mesa para
  ingresar al menú" debe reemplazar también el texto de la pantalla de "Salir" voluntario? → A: **Sí,
  una única pantalla de acceso denegado** con ese texto para el cierre de mesa y para "Salir" tras
  recargar; reemplaza el texto actual "Acceso finalizado, vuelve a escanear el código QR de tu mesa"
  (y se actualizan sus pruebas). La pantalla de gracias solo se ve en el momento del cierre o de
  "Salir", sin recargar.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Adicionales cobrados por las unidades elegidas, no por producto (Priority: P1)

Como cliente que arma su pedido en el Menú QR, quiero que cada adicional se cobre exactamente por la
cantidad que yo elegí, aunque pida varias unidades del producto, para que el total del carrito
coincida con lo que entendí que estaba comprando.

**Why this priority**: hoy el cliente paga de más (2 hamburguesas con 1 tocino cobran 2 tocinos) y
el cajero, la cocina y el inventario reciben una cantidad de adicionales que el cliente nunca pidió.
Es un error de dinero en producción, por eso comparte P1.

**Independent Test**: en el Menú QR, agregar 2 hamburguesas de $15.000 con 1 adicional de tocino de
$3.000 y verificar que el total de la línea y del carrito sea $33.000, que el pedido llegue al POS
con ese mismo total y que la comanda de cocina indique 2 hamburguesas y 1 tocino.

**Acceptance Scenarios**:

1. **Given** el Menú QR con hamburguesa a $15.000 y adicional tocino a $3.000, **When** el cliente
   agrega 2 hamburguesas y 1 unidad de tocino, **Then** el total de la línea es $33.000
   ((15.000 × 2) + (3.000 × 1)) y el detalle muestra "Tocino x1".
2. **Given** una línea con 2 hamburguesas y 1 tocino, **When** el cliente sube la cantidad del
   producto a 3, **Then** el adicional sigue en x1 y el total de la línea es $48.000
   ((15.000 × 3) + (3.000 × 1)).
3. **Given** una línea con 1 hamburguesa y 2 unidades del mismo adicional elegidas con el selector,
   **When** se calcula el total, **Then** el adicional aporta 2 × su precio, una sola vez por línea.
4. **Given** un producto en promoción con adicionales (spec 083), **When** el cliente lo agrega,
   **Then** el total es (precio promocional × cantidad del producto) + (adicionales × sus
   cantidades); la promoción nunca descuenta los adicionales.
5. **Given** un pedido del Menú QR con adicionales, **When** el mesero o el cajero lo ven en la
   terminal, el checkout, "Pagos por confirmar" o el detalle de venta, **Then** todas esas pantallas
   muestran el mismo total que vio el cliente.
6. **Given** un pedido de 2 hamburguesas con 1 tocino, **When** se confirma, **Then** la comanda de
   cocina indica las 2 hamburguesas y **1** tocino, y el inventario descuenta **1** unidad del
   insumo del adicional (no 2).
7. **Given** un grupo de opciones **incluido** (sin recargo, p. ej. sabores) en un producto con
   cantidad 2, **When** se calcula el consumo, **Then** su comportamiento por unidad de producto no
   cambia.
8. **Given** carritos y pedidos creados antes del despliegue, **When** se consultan, **Then** sus
   totales, comandas y descuentos de inventario son exactamente los de antes.

---

### User Story 2 - Total inmediato y sin modales al agregar productos a una orden abierta en el POS (Priority: P1)

Como cajero o mesero, quiero que al agregar o modificar productos de una orden abierta (mesa,
domicilio o para llevar) el total se actualice de inmediato en pantalla, para poder cobrar en un solo
paso sin que aparezca un mensaje que interrumpa el cobro.

**Why this priority**: el aviso "El total cambió" aparece en cada cobro de una orden ampliada, frena
la caja en horas pico y, peor, es síntoma de que la pantalla muestra un valor viejo justo antes de
cobrar.

**Independent Test**: abrir una orden con total $20.000, agregar 1 gaseosa de $5.000, verificar que
el total visible pasa a $25.000 sin ningún diálogo, elegir el método de pago y pulsar Cobrar: la
venta se registra por $25.000 sin ninguna confirmación intermedia.

**Acceptance Scenarios**:

1. **Given** una orden abierta con total $20.000, **When** el cajero agrega 1 gaseosa de $5.000,
   **Then** el total visible pasa de inmediato a $25.000, sin diálogos de confirmación, y sigue en
   ese valor una vez el servidor confirma.
2. **Given** una orden a la que se acaban de agregar productos, **When** el cajero elige el método de
   pago y pulsa Cobrar, **Then** el cobro se ejecuta por el total que la pantalla ya mostraba, sin
   el modal "El total cambió".
3. **Given** la misma situación en una orden de mesa, de domicilio y de para llevar, **Then** el
   comportamiento es idéntico en las tres.
4. **Given** que el servidor calcula un total distinto del estimado al instante (p. ej. por una
   promoción o un impuesto), **When** llega su respuesta, **Then** la pantalla reemplaza la cifra
   por la del servidor sin diálogos y el cobro usa siempre la cifra confirmada.
5. **Given** que se agrega un producto agotado, **When** el servidor lo rechaza, **Then** la pantalla
   muestra un mensaje que **nombra el producto agotado**, revierte el total al valor anterior a ese
   intento y no queda ningún importe fantasma.
6. **Given** que el cajero abre "Pagos por confirmar" para revisar un pago QR, **Then** el total que
   se valida es el vigente, con las mismas reglas de esta historia (sin modal de total cambiado).
7. **Given** que el total real cambia entre la última actualización y el instante del cobro por una
   causa externa (p. ej. venció una promoción), **When** el cajero pulsa Cobrar, **Then** la pantalla
   actualiza la cifra con un aviso **no bloqueante** en el propio panel y **no cobra** hasta que el
   cajero vuelva a pulsar Cobrar viendo el total nuevo (nunca se cobra un importe que el cajero no
   vio).

---

### User Story 3 - Editar los adicionales de un producto ya agregado al carrito (Priority: P2)

Como cliente del Menú QR, quiero editar o quitar los adicionales (y la nota) de un producto que ya
puse en mi carrito, sin tener que eliminarlo y volver a armarlo.

**Why this priority**: completa la corrección de la Historia 1 (el cliente ve y ajusta lo que pagará)
y evita pedidos errados por tener que rehacer la línea, pero el carrito funciona sin ella.

**Independent Test**: agregar un producto con un adicional, abrir "Editar adicionales" desde el
carrito, cambiar la cantidad o quitar el adicional, guardar y verificar que la misma línea conserva
producto, presentación y cantidad, con los adicionales nuevos y el total recalculado.

**Acceptance Scenarios**:

1. **Given** un carrito con una línea que tiene adicionales y nota, **When** el cliente pulsa
   "Editar adicionales" en esa línea, **Then** se abre el mismo selector del producto con los
   adicionales y la nota actuales ya seleccionados.
2. **Given** el selector abierto, **When** el cliente cambia cantidades, agrega o quita adicionales y
   guarda, **Then** la línea se actualiza en su lugar (mismo producto, presentación y cantidad) y su
   total, y el del carrito, se recalculan con la regla de la Historia 1.
3. **Given** el selector abierto, **When** el cliente quita todos los adicionales y guarda, **Then** la
   línea queda sin adicionales y sin haberse eliminado.
4. **Given** el selector abierto, **When** el cliente cancela o cierra sin guardar, **Then** la línea
   queda exactamente como estaba.
5. **Given** una línea que ya se envió (forma parte de un pedido), **Then** no se ofrece "Editar
   adicionales" para ella.
6. **Given** una mesa con varios comensales, **Then** cada uno ve y edita únicamente las líneas de su
   propio carrito.
7. **Given** que la selección viola una regla del grupo (mínimo, máximo o cantidad máxima por
   opción), **When** el cliente intenta guardar, **Then** no se guarda y se le indica qué corregir,
   igual que al agregar el producto por primera vez.

---

### User Story 4 - Al cerrar la mesa, el cliente ve "¡Gracias por tu visita!" y su sesión queda invalidada (Priority: P2)

Como cliente sentado en una mesa, cuando el personal cierra la mesa quiero ver de inmediato una
pantalla de agradecimiento y no poder enviar más pedidos con esa sesión, para no hacer pedidos a una
mesa que ya no es mía.

**Why this priority**: hoy el cliente sigue con el menú abierto, puede seguir armando pedidos que la
mesa ya no atiende y, según cómo se cerró la mesa, no se entera. Es un problema de operación, pero
no de dinero, por lo que va después de la Historia 1 y 2.

**Independent Test**: con un teléfono conectado a la sesión de una mesa, cerrar la mesa desde el POS
por cada camino posible y verificar en el teléfono que aparece la pantalla de gracias, que no se
puede enviar ningún pedido y que solo un nuevo escaneo del QR permite pedir de nuevo.

**Acceptance Scenarios**:

1. **Given** un cliente con la sesión abierta en el Menú QR, **When** el cajero cierra la mesa desde el
   POS, **Then** el teléfono muestra de inmediato "¡Gracias por tu visita! Esperamos verte pronto.",
   sin historial, sin recibo y sin botón para continuar pidiendo.
2. **Given** el mismo cliente, **When** la mesa se cierra por **cualquier otro camino** (liberación
   por el mesero, liberación automática por inactividad, cobro completo de la mesa, cierre de una
   sesión sin órdenes), **Then** se ve la misma pantalla de gracias.
3. **Given** un cliente con carrito con productos, **When** se cierra la mesa, **Then** el carrito
   local queda vacío y nada de él se envía.
4. **Given** una sesión cerrada, **When** el teléfono intenta enviar un pedido, un pago o cualquier
   otra acción con ese token, **Then** el sistema lo rechaza y la pantalla vuelve a la de gracias.
5. **Given** un cliente cuyo teléfono estuvo sin conexión cuando se cerró la mesa, **When** vuelve a
   conectarse o recarga la página en la misma pestaña, **Then** no queda con el menú operativo: ve
   la pantalla de gracias (reconexión) o la de acceso denegado (recarga, escenario 10).
6. **Given** una mesa cerrada que el personal vuelve a abrir o a ocupar, **When** el cliente anterior
   intenta usar su sesión, **Then** sigue viendo la pantalla de gracias: reabrir la mesa no
   reactiva su sesión.
7. **Given** la pantalla de gracias, **When** el cliente escanea de nuevo el QR de la mesa, **Then**
   entra por el flujo normal (nombre) y se crea una sesión nueva.
8. **Given** una mesa con varios comensales conectados, **When** se cierra, **Then** todos ven la
   pantalla de gracias.
9. **Given** una mesa cerrada con un pago por transferencia en curso de un comensal (comprobante sin
   revisar), **When** se cierra, **Then** el comensal igualmente ve la pantalla de gracias y el
   pago pendiente sigue visible para el cajero.
10. **Given** una mesa cerrada y la pantalla de gracias en el teléfono, **When** el cliente recarga la
    página (F5) o usa "Atrás"/"Adelante" en esa misma pestaña, **Then** es llevado a una pantalla
    estática de acceso denegado que dice "Por favor, escanea nuevamente el código QR de la mesa para
    ingresar al menú", sin campo de nombre, sin menú y sin botón para continuar; no se crea ninguna
    sesión nueva. Lo mismo ocurre al recargar tras "Salir" (misma pantalla y mismo texto).
11. **Given** una mesa liberada desde el POS y otra cuyo comensal presionó "Salir", **When** se
    inspeccionan el navegador y el resultado tras recargar, **Then** el borrado es idéntico: ningún
    token de sesión, dato de comensal, carrito ni cookie asociada, y solo la marca mínima de acceso
    cerrado de la pestaña.
12. **Given** una pestaña con la marca de acceso cerrado, **When** el cliente abre el enlace del QR en
    una pestaña nueva o lo escanea de nuevo, **Then** entra por el flujo normal (nombre) y se crea
    una sesión nueva.
13. **Given** un enlace o almacenamiento con el token de una sesión anterior (parámetro `s`), **When**
    se abre con la mesa libre, cerrada o reocupada, **Then** el backend lo rechaza (401 con
    `X-Session-State: closed`) y el cliente muestra la pantalla de acceso denegado.

---

### User Story 5 - Reporte de cierre de caja legible al imprimir (Priority: P3)

Como cajero, quiero que al imprimir el reporte de cierre de turno el texto salga negro sobre fondo
blanco y completo, sin hojas en blanco, para poder entregarlo o archivarlo.

**Why this priority**: es un defecto de presentación con salida alternativa (el cajero ya ve el
reporte en pantalla), pero impide entregar el reporte en papel o PDF.

**Independent Test**: cerrar un turno de caja, pulsar "Imprimir / Exportar reporte" y comprobar en la
vista previa de impresión que todo el reporte se ve, en negro sobre blanco, con el tema claro y con
el oscuro.

**Acceptance Scenarios**:

1. **Given** el reporte del cierre de turno en pantalla, **When** el cajero pulsa "Imprimir /
   Exportar reporte", **Then** la vista previa de impresión muestra todo el contenido del reporte
   (no una hoja en blanco), con texto negro sobre fondo blanco.
2. **Given** el sistema con el tema oscuro, **When** se imprime, **Then** el resultado es idéntico al
   del tema claro: ningún texto blanco, transparente o gris claro sobre blanco.
3. **Given** un reporte extenso que ocupa más de una hoja, **When** se imprime, **Then** se ven todas
   las hojas con contenido y sin hojas en blanco intermedias.
4. **Given** la impresión, **Then** siguen sin aparecer el menú lateral, el encabezado ni los botones
   de la pantalla, y el archivo conserva el nombre `<tenant-slug>-<DD-MM-YYYY>` de la spec 087.
5. **Given** cualquier otra pantalla imprimible (recibo de venta, recibo de mesa, hoja de QR),
   **Then** su impresión no cambia.

---

### Edge Cases

- **Adicional con cantidad 0**: si el cliente lleva un adicional a 0 con el selector, se trata como
  quitarlo; no queda una fila "x0" ni se cobra.
- **Cantidad máxima por opción**: el tope de unidades de un adicional (definido en el catálogo) se
  sigue respetando, ahora independiente de la cantidad del producto.
- **Adicional dado de baja mientras está en el carrito**: al abrir "Editar adicionales", el
  adicional inactivo o sin stock se marca como no disponible y el cliente debe quitarlo o
  reemplazarlo para guardar; al enviar el pedido se sigue rechazando con el mensaje habitual.
- **Editar una línea igual a otra**: si tras editar dos líneas quedan idénticas, permanecen como
  líneas separadas (no se fusionan).
- **Línea del carrito con producto y presentación desactivados**: no se ofrece edición; solo quitarla.
- **Líneas históricas frente a líneas nuevas**: dentro de un mismo pedido pueden convivir líneas con
  la regla anterior (por unidad, creadas antes del despliegue o desde la terminal POS) y líneas con
  la regla nueva (una vez por línea, creadas en el Menú QR); cada línea conserva la regla con la que
  se creó y el total del pedido es la suma de líneas.
- **Producto sin adicionales ni notas**: la línea se comporta igual que hoy; no aparece contenedor
  vacío (spec 087).
- **Cierre de mesa con el cliente en medio de un envío**: si el envío llega al servidor después del
  cierre, se rechaza y no se crea el pedido; si llegó antes, el pedido existe y la mesa no se
  cierra (regla vigente de "Liberar mesa": no hay órdenes sin cerrar).
- **Recarga tras el cierre (misma pestaña)**: F5, "Atrás" o "Adelante" en la pestaña que vivió el cierre
  NO arrancan un ingreso nuevo: muestran la pantalla de acceso denegado (FR-015a). **Pestaña nueva o
  escaneo nuevo** del QR: sí arrancan un ingreso nuevo, porque la marca de acceso cerrado es por
  pestaña. La pantalla de gracias persiste mientras la misma página siga abierta.
- **Almacenamiento bloqueado (modo privado)**: si el navegador no deja escribir la marca, el cierre
  igualmente purga todo y muestra la pantalla de gracias; la recarga no se puede bloquear en el
  cliente (limitación aceptada, igual que hoy con "Salir"), pero cualquier token de sesión anterior
  sigue rechazado por el backend.
- **Cliente que había salido voluntariamente antes del cierre**: sigue en su estado de acceso cerrado,
  sin duplicarse; tras recargar ve la misma pantalla de acceso denegado que tras el cierre
  (FR-015b).
- **Sesión vencida sin cierre de mesa** (inactividad o duración máxima): no es un cierre; el
  cliente conserva el ingreso por nombre con su mensaje actual y solo el cierre de la mesa lleva a
  la pantalla de gracias (valor por defecto del plan, por confirmar con el negocio; ver A-95).
- **Barrido automático con pedidos por cobrar**: cierra a los comensales pero no la sesión ni emite
  aviso; el cliente ve la pantalla de gracias en el siguiente sondeo o al reusar la sesión
  (SC-005). No se cambia el barrido (Principio V).
- **Varias altas seguidas en el POS**: agregar tres productos rápidamente deja el total final
  correcto (la suma de los tres) sin cifras intermedias que se queden.
- **Servidor sin respuesta al agregar un producto**: la pantalla no debe dejar el total estimado
  como si estuviera confirmado; muestra el error habitual, revierte al total confirmado y no cobra.
- **Impresión sin turno cerrado o sin datos**: el reporte en blanco esperado (sin movimientos) se
  imprime con sus encabezados y totales en cero, no como hoja vacía.

## Requirements *(mandatory)*

### Functional Requirements

**A. Adicionales del Menú QR — cálculo y edición**

- **FR-001**: En una línea del carrito del Menú QR, el sistema MUST calcular el total como
  (precio de la presentación o promoción × cantidad del producto) + Σ (precio de cada adicional ×
  cantidad elegida de ese adicional). La cantidad del producto MUST NOT multiplicar el precio de
  los adicionales.
- **FR-002**: La cantidad de un adicional MUST ser una propiedad propia de la línea; cambiar la
  cantidad del producto MUST NOT modificarla.
- **FR-003**: Se consideran adicionales las opciones de los grupos **con recargo**. Las opciones de
  los grupos **incluidos** (sin cargo) MUST conservar su comportamiento por unidad de producto.
- **FR-004**: La regla de FR-001 MUST reflejarse igual en todas las pantallas y cálculos donde el
  pedido del Menú QR muestre o valide su total: carrito del Menú QR, resumen e historial del
  comensal, panel del pedido de la terminal, checkout, "Pagos por confirmar" y detalle de venta
  (regla ya establecida por la spec 087, US1).
- **FR-005**: El adicional MUST calcularse sobre el precio ya definido de la presentación o
  promoción sin ser descontado por esta (Precio Final = precio de promoción × cantidad + adicionales,
  spec 087 FR-011 y spec 083).
- **FR-006**: La comanda de cocina de una línea del Menú QR MUST mostrar los adicionales con la
  cantidad exacta elegida (p. ej. "2 hamburguesas + 1 tocino"), no multiplicada por la cantidad del
  producto.
- **FR-007**: El descuento de inventario del insumo de un adicional MUST ser igual a su cantidad
  elegida × el consumo de la opción, sin multiplicarse por la cantidad del producto; el consumo de
  la presentación y de los grupos incluidos MUST seguir siendo por unidad de producto.
- **FR-008**: El sistema MUST mantener exactos, sin recalcular, los totales, comandas y descuentos de
  inventario de carritos, pedidos, ventas y facturas creados antes del despliegue, y de las líneas
  creadas desde la terminal POS o el pedido manual, que conservan la regla por unidad
  (Principio VII).
- **FR-009**: El resumen del carrito MUST seguir mostrando cada adicional con su multiplicador exacto
  ("Queso extra x1"), incluso cuando es x1 (spec 087 FR-013).
- **FR-010**: En el carrito del Menú QR, cada línea MUST ofrecer una acción "Editar adicionales"
  mientras la línea no se haya enviado.
- **FR-011**: La acción de FR-010 MUST abrir el mismo selector de opciones del producto, con los
  adicionales y la nota actuales preseleccionados, y al guardar MUST actualizar la línea en su
  lugar (mismo producto, presentación y cantidad), sin eliminarla ni duplicarla, recalculando sus
  totales según FR-001.
- **FR-012**: La edición MUST validar las mismas reglas de selección del producto (mínimos, máximos,
  cantidad máxima por opción, opciones activas y con stock) y MUST permitir dejar la línea sin
  adicionales.
- **FR-013**: Cada comensal MUST poder editar únicamente las líneas de su propio carrito; las
  líneas ya enviadas MUST NOT ser editables desde el carrito.

**B. Cierre de mesa y sesión del Menú QR**

- **FR-014**: Todo camino que pase la sesión de una mesa a cerrada MUST notificar en tiempo real a
  los comensales conectados a esa sesión: el cierre del cajero, "Liberar mesa" (cajero o mesero),
  la liberación automática del scheduler, el cobro completo de la mesa y el cierre automático de una
  sesión sin órdenes.
- **FR-015**: Al recibir la notificación, o al detectar en cualquier interacción o reconexión que su
  sesión está cerrada, el Menú QR MUST mostrar siempre la pantalla "¡Gracias por tu visita! Esperamos
  verte pronto.", sin historial ni recibo, y MUST limpiar el carrito local y el estado de la sesión.
- **FR-015a**: Al recibir `session.closed`, o al detectar una sesión cerrada por 401, el cliente MUST
  ejecutar exactamente el mismo borrado de sesión que el botón "Salir" del Menú QR: purgar de
  inmediato todo almacenamiento local (`localStorage`, `sessionStorage` y cookies) que contenga el
  token de sesión, datos del comensal o del carrito, y dejar únicamente la marca mínima de acceso
  cerrado de la pestaña (solo el token público de la URL de la mesa). Las dos rutas MUST compartir la
  misma lógica, no dos copias.
- **FR-015b**: Con la marca de acceso cerrado presente, al cargar la URL del menú (F5, "Atrás",
  "Adelante") el cliente MUST mostrar una pantalla estática de acceso denegado con el texto "Por favor,
  escanea nuevamente el código QR de la mesa para ingresar al menú", sin campo de nombre, sin menú y
  sin botón o enlace para continuar, y MUST NOT crear una sesión nueva. Es **una sola pantalla** para
  el cierre de mesa y para "Salir" (reemplaza el texto actual de acceso finalizado).
- **FR-015c**: La marca de acceso cerrado MUST ser por pestaña (`sessionStorage`) y MUST NOT contener
  token de sesión ni datos del comensal; una pestaña nueva o un escaneo nuevo MUST entrar por el flujo
  normal (nombre).
- **FR-016**: Una sesión cerrada MUST rechazar cualquier acción del comensal (enviar pedido, iniciar o
  adjuntar un pago, cancelar, actualizar el carrito), sin excepciones por cuentas pendientes.
- **FR-017**: Reabrir u ocupar de nuevo la mesa MUST NOT reactivar la sesión del cliente anterior;
  solo un nuevo ingreso por el QR crea una sesión nueva.
- **FR-017a**: Al abrir el menú con un token de sesión anterior (almacenado o en el parámetro `s`), el
  backend MUST rechazarlo (401 con `X-Session-State: closed`) si la mesa está libre, cerrada o
  reocupada por otra sesión, y el cliente MUST mostrar la pantalla de acceso denegado. La URL de la
  mesa sin token de sesión MUST seguir resolviendo el menú, porque es el punto de entrada de un
  escaneo nuevo en una mesa libre.
- **FR-018**: La pantalla de gracias MUST NOT ofrecer un botón o enlace para volver a pedir; la única
  vía es volver a escanear el QR de la mesa (comportamiento ya vigente para la salida voluntaria).
- **FR-019**: El cierre de la sesión MUST NOT eliminar ni alterar los pedidos, pagos y comprobantes
  ya registrados; un pago pendiente del comensal MUST seguir visible para el cajero.

**C. Impresión del cierre de caja**

- **FR-020**: Al imprimir el reporte de cierre de turno, el sistema MUST mostrar todo su contenido
  en negro (#000000) sobre fondo blanco (#ffffff), sin hojas en blanco, tanto con el tema claro como
  con el oscuro.
- **FR-021**: La impresión MUST NOT heredar del tema oscuro ni de la pantalla estilos que dejen texto
  blanco, transparente o de bajo contraste, ni contenedores que oculten o recorten el reporte
  (alturas fijas, desbordes ocultos, elementos superpuestos).
- **FR-022**: La impresión MUST conservar lo ya vigente de la spec 087: sin menú lateral, encabezado
  ni botones, y nombre de archivo `<tenant-slug>-<DD-MM-YYYY>`.
- **FR-023**: El alcance se limita al reporte de cierre de caja; la impresión de recibos de venta,
  recibos de mesa y hoja de QR MUST NOT cambiar.
- **FR-024**: La causa real de la hoja en blanco MUST reproducirse y confirmarse antes de corregir
  (impresión del navegador o su vista previa; navegador y tema afectados), y la corrección MUST
  verificarse imprimiendo en la vista previa, no solo leyendo el código.

**D. Total inmediato en el POS**

- **FR-025**: Al agregar, modificar o quitar productos de una orden abierta (mesa, domicilio, para
  llevar) en la terminal POS, la pantalla MUST actualizar de inmediato el subtotal, los impuestos y el
  total, sin esperar la respuesta del servidor y sin ningún diálogo.
- **FR-026**: El servidor MUST confirmar o reconciliar esa cifra; si difiere de la estimada, la
  pantalla MUST reemplazarla por la del servidor sin diálogos, y el cobro MUST usar siempre la cifra
  confirmada por el servidor.
- **FR-027**: El sistema MUST NOT mostrar el modal "El total cambió" al pulsar Cobrar ni como
  mecanismo para descubrir un total nuevo; el total visible antes de Cobrar MUST ser ya el vigente.
- **FR-028**: Si aun así el total real difiere en el instante de cobrar (causa externa), la pantalla
  MUST actualizarlo con un aviso no bloqueante y MUST NOT cobrar hasta que el cajero pulse Cobrar de
  nuevo sobre el total nuevo; MUST NOT cobrar un importe que el cajero no vio.
- **FR-029**: Si el servidor rechaza un producto por estar agotado, la pantalla MUST mostrar un
  mensaje que nombre el producto agotado, MUST revertir el total al valor confirmado previo a ese
  intento y MUST NOT dejar importes ficticios en el pedido.
- **FR-030**: Lo anterior MUST aplicar a las órdenes de mesa, domicilio y para llevar, y a todas las
  superficies de cobro que validan el total, incluida "Pagos por confirmar".
- **FR-031**: Cuando el servidor no responde, la pantalla MUST NOT mantener el total estimado como si
  estuviera confirmado; MUST revertir al confirmado, mostrar el error habitual y no cobrar.

### Key Entities *(include if feature involves data)*

- **Línea de carrito / de pedido**: producto + presentación + cantidad del producto + adicionales
  elegidos + nota. Cada línea debe poder distinguirse entre la regla **por unidad** (histórica y la
  de la terminal POS) y la regla **una vez por línea** (Menú QR nuevo), para que los datos
  existentes no cambien de significado (Principio VIII).
- **Adicional elegido**: opción de un grupo con recargo, con su propia cantidad, independiente de la
  cantidad del producto de su línea.
- **Sesión de mesa / participante**: la sesión de una mesa y la de cada comensal; al pasar a
  cerrada, el participante deja de poder actuar y su pantalla pasa a la de gracias.
- **Marca de acceso cerrado**: dato mínimo por pestaña (token público de la URL de la mesa) que deja
  "Salir" y, desde esta spec, todo cierre de mesa; es lo único que sobrevive al borrado y solo sirve
  para mostrar la pantalla de acceso denegado al recargar.
- **Total de la orden en pantalla**: cifra visible en la terminal, con dos estados: estimado (recién
  calculado al instante) y confirmado (respuesta del servidor); solo el confirmado se cobra.
- **Reporte de cierre de turno**: el documento imprimible del cierre de caja.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: En el 100 % de los pedidos nuevos del Menú QR, el total de la línea equivale a
  (precio × cantidad del producto) + (precio del adicional × cantidad del adicional); con 2
  hamburguesas de $15.000 y 1 tocino de $3.000 el total es exactamente $33.000.
- **SC-002**: Para un mismo pedido del Menú QR, el total es idéntico en las cinco superficies (carrito,
  terminal, checkout, "Pagos por confirmar", detalle de venta), y la comanda de cocina y el descuento
  de inventario coinciden con las unidades de adicional que el cliente eligió.
- **SC-003**: Ningún carrito, pedido, venta ni factura creado antes del despliegue cambia de total,
  comanda o descuento de inventario.
- **SC-004**: Un cliente cambia o quita los adicionales de una línea desde el carrito en no más de 3
  interacciones (abrir, modificar, guardar), sin eliminar el producto.
- **SC-005**: Con el teléfono conectado, la pantalla "¡Gracias por tu visita!" aparece en menos de 5
  segundos desde que el personal cierra la mesa, por cada camino que cierra la sesión (aviso en
  tiempo real). Cuando el barrido automático vence a los comensales de una mesa con pedidos por
  cobrar (la sesión sigue activa y no hay aviso), la pantalla aparece en el siguiente sondeo (10 s
  como máximo con la pestaña visible) o al volver a usar la sesión. El 100 % de los intentos
  posteriores de enviar pedidos con esa sesión son rechazados.
- **SC-006**: Con el teléfono sin conexión al cierre, la pantalla de gracias (reconexión) o la de
  acceso denegado (recarga en la misma pestaña) aparece en el primer intento de usar la sesión al
  volver.
- **SC-011**: Tras cerrar la mesa por cualquier camino, el 100 % de las recargas (F5, "Atrás",
  "Adelante") en esa pestaña muestran la pantalla de acceso denegado y 0 crean una sesión nueva; el
  estado del navegador tras el cierre es idéntico al de "Salir" (sin token, datos de comensal, carrito
  ni cookies de sesión; solo la marca mínima).
- **SC-007**: El reporte de cierre de caja se ve completo, en negro sobre blanco y sin hojas en
  blanco en la vista previa de impresión, con tema claro y con tema oscuro, en Chrome y Firefox.
- **SC-008**: En una orden abierta, el total en pantalla refleja el producto agregado sin espera
  perceptible (menos de 0,2 s) y sin ningún diálogo; el cobro de una orden ampliada se completa en un
  solo paso (método de pago + Cobrar).
- **SC-009**: Cuando un producto está agotado, el mensaje nombra el producto en el 100 % de los casos
  y el total queda igual al de antes del intento.
- **SC-010**: Cero apariciones del modal "El total cambió" en el flujo de Cobrar de una orden abierta.

## Assumptions

- Los adicionales son las opciones de los grupos **con recargo**; la elección de sabores u otras
  opciones incluidas sin cargo no forma parte de esta corrección (ver Clarifications).
- La selección de opciones (una o varias, con o sin selector de cantidad, mínimos y máximos) ya está
  definida en el catálogo (spec 065); esta spec no cambia esa configuración ni la interfaz de
  selección para grupos que ya tienen selector de cantidad. Las opciones que no tienen selector
  cuentan como 1 unidad de la línea.
- La terminal POS y el pedido manual **no cambian** su forma de armar líneas con adicionales
  (decisión del negocio): conservan la regla por unidad; dentro de un mismo pedido pueden convivir
  ambas reglas (ver Edge Cases). Esta divergencia se acepta para no ampliar el alcance y queda
  registrada en A-94.
- Cambiar la regla del adicional exige poder distinguir las líneas nuevas de las históricas; la
  forma de hacerlo (y la estrategia de migración y de reversa) se define en el plan, respetando el
  Principio VIII y sin reescribir datos históricos (Principio VII).
- El mensaje de agradecimiento puede conservar el nombre y el logo del negocio que ya muestra la
  pantalla de salida voluntaria; el texto exigido es "¡Gracias por tu visita! Esperamos verte pronto."
- "Volver a escanear el QR" equivale a abrir el enlace del QR de la mesa en una **pestaña nueva**. El
  enlace es fijo por mesa, así que el servidor no distingue una recarga de un escaneo; esa distinción
  la hace únicamente la marca de acceso cerrado por pestaña (FR-015c), la misma que ya usa "Salir".
- El tiempo real del comensal (spec 077) es el canal principal; el respaldo (validación de sesión al
  reanudar o reconectar) cubre teléfonos sin conexión en el momento del cierre. No se agrega ninguna
  dependencia nueva (Principio IX).
- El total "inmediato" es una estimación local calculada con los precios ya conocidos por la
  pantalla; el servidor sigue siendo la autoridad del total cobrado. Las promociones e impuestos
  que la pantalla no pueda estimar se reconcilian con la respuesta del servidor.
- Se retira el modal "El total cambió" del cobro de una orden abierta (spec 073, FR-007/D11). Para no
  dejar dos comportamientos distintos, el mismo aviso del alta de pedido manual (spec 073, FR-015a)
  recibe el mismo tratamiento no bloqueante (FR-028). La protección de fondo se mantiene: nunca se
  cobra un importe que el cajero no vio. Esta es la decisión de negocio registrada en A-96.
- La hoja en blanco del cierre de caja se atribuye a la impresión del navegador (no a un PDF
  generado por el servidor); la causa exacta es un entregable del plan (FR-024).
- Tests de caracterización que congelen el cobro por unidad del adicional, el consumo o la comanda
  (p. ej. `test_catalog_line_pricing`, `test_catalog_consumption_plan`, `test_cart_service`,
  `test_orders_consolidation`, `test_orders_service`) se actualizan explícitamente citando A-94,
  nunca en silencio (Principio III).
- Alcance por repositorio: `pos-backend` (cálculo de línea, consumo, comanda, cierre de sesión y
  emisión de eventos), `pos-heladeria` (carrito y selector del Menú QR, pantalla de gracias, panel
  de cobro y terminal POS, impresión del reporte) y `pos-specs` (esta spec y el registro de
  anomalías A-94 a A-97; A-97 es un hallazgo documental sin cambio de comportamiento).
