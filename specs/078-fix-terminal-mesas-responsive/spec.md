# Feature Specification: Correcciones responsive y de presentación de la Terminal de Mesas (panel de detalle, tarjetas de domicilio, botón de crear pedido, navegación en tablet)

**Feature Branch**: `078-fix-terminal-mesas-responsive`

**Created**: 2026-09-07

**Status**: Draft

**Naturaleza de esta spec**: spec de **corrección de defectos y ajuste de presentación** sobre la
Terminal de Mesas ya rediseñada por spec 076, en la línea de spec 073
(`073-fix-descuento-cobro-terminal`). Recoge siete problemas concretos observados por el negocio
después de poner en uso el rediseño responsive (spec 076) y las tarjetas de Domicilio/Para llevar
(spec 059). Seis son defectos de presentación/responsive que dejan datos fuera de pantalla, muestran
un total equivocado o comunican mal una acción; el séptimo (total de la tarjeta de Domicilio) es un
defecto que muestra al cajero un importe distinto del que realmente se cobra. Dos de los siete
puntos introducen además un comportamiento nuevo acotado (tipo de pedido preseleccionado al crear
desde una pestaña, y colapso del menú de navegación en tablet), explícitamente autorizado por el
negocio (ver Clarifications e Impacto sobre comportamiento existente).

Esta spec **no** rediseña la Terminal de Mesas otra vez: el layout de grilla responsive de 3
variantes, la barra superior operativa, los contadores de ocupación y "Atendido por" definidos por
spec 076 no cambian. Esta spec corrige cómo se comportan, a distintos anchos, el **panel derecho de
detalle/cobro** y algunos elementos de las **tarjetas y pestañas**, y ajusta la densidad de
información del panel de detalle.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-09-07, enumerando siete defectos y ajustes concretos
detectados al usar la Terminal de Mesas tras el rediseño de spec 076, con instrucción explícita de
resolver las ambigüedades antes de redactar la especificación. La sesión de aclaración del
2026-09-07 (ver Clarifications) fijó las seis decisiones abiertas del pedido original y, en una
segunda ronda, otras tres decisiones de detalle (comportamiento del panel en tablet, ubicación del
botón de crear pedido en los anchos angostos y densidad mínima de la lista de productos por vista).

**Input**: descripción del usuario (verbatim):

> en la terminal de mesas:
> - el panel derecho con los detalles del pedido en la terminal de mesas se pierde en la pantalla a
>   medida que esta reduce su tamaño
> - el boto de crear pedido desaparece cuando el usuario entra en la vista para domicilios y para
>   llevar
> - en vista mobile el boton para crear pedido no comunica correctamente la opcion solo muestra un
>   icono con un simbolo + la idea es que deje bien claro y explicito para que sirve ese boton
> - en la seccion de los domicilios los precios se estan mostrando mal el total en las tarjetas,
>   solo muestra el total por producto mas no tiene en cuenta el valor del domicilio
> - en la seccion donde estan los detalles del pedido, hay muy poco espacio en la seccion donde se
>   muestran los productos lo ahce una mala experiencia de usuario, hay que ajustarlo
> - la informacion del domicilio (direccion, telefono y el valor del domicilio) esta ocupando
>   demasiado espacio se debe mostrar de forma inline al lado de el estado del domicilio
> - en la vista de tablet dale el mismo comportamiento al sidebar que en la vista mobile

## Clarifications

### Session 2026-09-07

- Q: En "en la vista de tablet dale el mismo comportamiento al sidebar que en la vista mobile", ¿a
  qué "sidebar" se refiere? → A: **El menú de navegación global de la aplicación** (la barra lateral
  de módulos), no el panel derecho de detalle de la Terminal.
- Q: El panel derecho de detalle "se pierde" al reducir el ancho. ¿Cuál es el comportamiento real y
  qué se espera? → A: El panel **aparece incompleto: no se adapta al ancho de la pantalla en
  ninguna de las tres vistas** (móvil, tablet, escritorio) — parte de su contenido queda fuera del
  área visible. Se espera que se ajuste al ancho disponible en las tres vistas, mostrando todo su
  contenido.
- Q: El botón "Crear pedido" debe seguir visible en las pestañas "Domicilios" y "Para llevar". Al
  pulsarlo estando en esas pestañas, ¿qué tipo de pedido inicia? → A: **Con el tipo preseleccionado**
  — en "Domicilios" abre la creación con tipo Domicilio ya elegido; en "Para llevar", con tipo Para
  llevar; en "Mesas" y en el estado sin selección, sin preselección (como hoy).
- Q: La tarjeta de un pedido de Domicilio debe incluir el valor del domicilio en el total. ¿Cómo se
  muestra el total en la tarjeta? → A: **Solo el total corregido** — un único total = productos +
  valor del domicilio (con los descuentos/impuestos/propina que ya se consideran hoy), sin desglose
  en la tarjeta. El desglose se ve en el panel de detalle.
- Q: El colapso del menú de navegación en tablet, ¿aplica en toda la aplicación a ancho de tablet o
  solo mientras se está en la Terminal de Mesas? → A: **En toda la aplicación a ancho de tablet** —
  el menú de navegación global adopta en tablet (768px–1023px) el mismo comportamiento colapsable
  que ya tiene en móvil, en todas las pantallas.
- Q: En la fila compacta con la información del domicilio junto al estado del pedido, si la dirección
  es larga y no cabe en una línea, ¿qué se prefiere? → A: **Siempre completa aunque envuelva** — la
  dirección se muestra completa, envolviendo en 2–3 líneas si hace falta; no se trunca ni se
  esconde tras una interacción.
- Q: En tablet (768–1023px), al seleccionar una mesa o un pedido, ¿el panel de detalle/cobro se
  muestra al lado de la grilla (como en escritorio) o la reemplaza a todo el ancho (como en móvil)?
  → A: **La reemplaza a todo el ancho, como en móvil** — al cerrar el detalle el cajero vuelve a la
  grilla en el mismo estado (pestaña, filtro, scroll). Solo en escritorio (≥ 1024px) la grilla y el
  panel se muestran lado a lado.
- Q: En "Domicilios"/"Para llevar" sin tarjeta seleccionada, ¿dónde aparece el botón de crear pedido
  en los tres anchos? → A: **En el mismo lugar que en "Mesas" sin selección** — en escritorio, fijo
  al final del panel derecho de bienvenida (spec 076 FR-027); en tablet y móvil, como botón de
  acción fijo sobre la grilla/lista de tarjetas mientras el cajero navega la pestaña sin una tarjeta
  seleccionada. Al seleccionar una tarjeta se comporta como hoy en "Mesas" (el botón no forma parte
  del panel de cobro).
- Q: SC-005 pide "ver al menos 4 productos a la vez"; ¿contra qué alto de pantalla se cumple? → A:
  **≥ 4 productos en escritorio y tablet**; en móvil se relaja a "los que quepan tras encabezado,
  totales y acciones — al menos 2".

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ver en la tarjeta de un pedido de Domicilio el mismo total que se va a cobrar (Priority: P1)

El cajero abre la pestaña "Domicilios" de la Terminal de Mesas y mira las tarjetas de los pedidos
pendientes de cobro. Hoy la tarjeta muestra como "total" solo la suma de los productos y no incluye
el valor del domicilio (`delivery_fee`, spec 056), de modo que el número de la tarjeta es menor que
el que aparece en el panel de cobro al seleccionar ese mismo pedido y menor que el que finalmente se
factura. Con esta corrección, la tarjeta muestra el total real a cobrar, incluyendo el valor del
domicilio.

**Why this priority**: es un defecto que muestra un importe equivocado en una superficie que el
cajero usa para decidir cuánto cobrar — riesgo directo de cobrar de menos en pedidos a domicilio.
Es el único de los siete puntos que afecta dinero visible, no solo disposición visual.

**Independent Test**: se puede probar completamente creando un pedido de Domicilio con un valor de
domicilio distinto de cero, abriendo la pestaña "Domicilios", anotando el total de la tarjeta,
seleccionando el pedido y verificando que el total del panel de cobro coincide exactamente con el de
la tarjeta y que ambos incluyen el valor del domicilio.

**Acceptance Scenarios**:

1. **Given** un pedido de Domicilio con subtotal de productos de $25.000 y valor de domicilio de
   $6.000, sin descuento/impuesto/propina, **When** el cajero mira su tarjeta en la pestaña
   "Domicilios", **Then** la tarjeta muestra un total de $31.000 (no $25.000).
2. **Given** ese mismo pedido, **When** el cajero lo selecciona y mira el panel de detalle/cobro,
   **Then** el total mostrado ahí es idéntico al de la tarjeta ($31.000).
3. **Given** un pedido de Domicilio con un descuento aplicado, **When** el cajero mira su tarjeta,
   **Then** el total sigue la misma fórmula que el preview de cobro para ese tipo de pedido
   (`max(0, subtotal − descuento por promoción + valor de domicilio)`, sin impuesto ni propina —
   spec 073), no una fórmula distinta solo para la tarjeta.
4. **Given** un pedido "Para llevar" o un pedido de mesa (sin valor de domicilio), **When** el
   cajero mira su tarjeta, **Then** el total mostrado es exactamente el mismo que hoy — esta
   corrección no cambia el total de los pedidos que no son de Domicilio.
5. **Given** la tarjeta de un pedido de Domicilio, **When** el cajero la mira, **Then** ve un único
   total combinado, sin una línea aparte de "productos" y "domicilio" (ese desglose vive en el panel
   de detalle).

---

### User Story 2 - Poder crear un pedido nuevo desde cualquier pestaña, con el tipo correcto y un botón que se entienda (Priority: P1)

El cajero está en la pestaña "Domicilios" o "Para llevar" de la Terminal de Mesas y quiere registrar
un pedido nuevo de ese tipo. Hoy el botón de crear pedido nuevo desaparece al entrar en esas
pestañas: solo está disponible desde la pestaña "Mesas" (o desde el panel derecho sin selección),
obligando al cajero a volver atrás para poder crear el pedido. Además, en móvil ese botón se muestra
como un simple ícono "+" sin ningún texto, y no comunica para qué sirve. Con esta corrección el
botón está siempre disponible, en móvil lleva un texto que explica su función, y al pulsarlo desde
"Domicilios" o "Para llevar" el formulario de creación se abre con ese tipo de pedido ya
seleccionado.

**Why this priority**: es funcionalidad que hoy está rota por contexto — la acción principal de
"registrar un pedido" no está donde el cajero la necesita, y en móvil ni siquiera se entiende qué
hace el botón. Además, preseleccionar el tipo elimina un paso y una fuente de error (crear un
domicilio como "para llevar" por descuido).

**Independent Test**: se puede probar completamente abriendo cada una de las tres pestañas y
verificando que el botón de crear pedido está visible en todas y, en móvil, que muestra un texto que
comunica su función; luego pulsándolo desde "Domicilios" y verificando que el formulario abre con
tipo "Domicilio", y desde "Para llevar" con tipo "Para llevar".

**Acceptance Scenarios**:

1. **Given** la Terminal de Mesas en la pestaña "Domicilios", **When** el cajero mira la pantalla,
   **Then** el botón para crear un pedido nuevo está visible y habilitado (con las mismas reglas de
   permiso que hoy).
2. **Given** la Terminal de Mesas en la pestaña "Para llevar", **When** el cajero mira la pantalla,
   **Then** el botón para crear un pedido nuevo está visible y habilitado.
3. **Given** el cajero en la pestaña "Domicilios", **When** pulsa el botón de crear pedido (o usa su
   atajo), **Then** se abre el flujo de creación manual de pedido con el tipo "Domicilio"
   preseleccionado.
4. **Given** el cajero en la pestaña "Para llevar", **When** pulsa el botón de crear pedido, **Then**
   se abre el flujo de creación manual con el tipo "Para llevar" preseleccionado.
5. **Given** el cajero en la pestaña "Mesas" (o en el panel derecho sin ninguna selección), **When**
   pulsa el botón de crear pedido, **Then** se abre el flujo de creación manual sin ningún tipo
   preseleccionado — exactamente como hoy.
6. **Given** el formulario de creación abierto con un tipo preseleccionado, **When** el cajero cambia
   el tipo dentro del formulario, **Then** el cambio se acepta — la preselección es un valor inicial,
   no un bloqueo.
7. **Given** cualquiera de los casos anteriores, **When** el pedido se crea, **Then** el resto del
   comportamiento (a dónde navega, cómo se guarda, qué permisos exige) es el mismo que hoy.
8. **Given** la Terminal de Mesas en ancho de móvil, **When** el cajero mira el botón de crear
   pedido, **Then** el botón muestra un texto visible que comunica su función (por ejemplo "Crear
   pedido") y no únicamente un ícono "+".
9. **Given** la Terminal de Mesas en ancho de escritorio, **When** el cajero mira el botón de crear
   pedido, **Then** su presentación es la misma que hoy (fijo al final del panel de bienvenida con
   su etiqueta completa), más allá de estar visible en todas las pestañas.
10. **Given** la Terminal de Mesas en ancho de tablet sin una tarjeta seleccionada, en cualquiera de
    las tres pestañas, **When** el cajero mira la pantalla, **Then** el botón de crear pedido
    aparece como acción fija sobre la grilla/lista, con una etiqueta de texto visible (no solo el
    ícono "+").

---

### User Story 3 - Ver el panel de detalle del pedido completo, sin importar el ancho de la pantalla (Priority: P1)

El cajero selecciona una mesa o un pedido y el panel derecho muestra el detalle y el flujo de cobro.
Hoy ese panel no se ajusta al ancho disponible: en escritorio, tablet y móvil parte de su contenido
queda fuera del área visible (recortado por el borde, o empujado fuera de la pantalla), y en algunos
anchos obliga a un desplazamiento horizontal de toda la página. Con esta corrección el panel se
adapta al ancho de las tres vistas y muestra todo su contenido dentro del área visible.

**Why this priority**: sin esto, el cajero no puede ver ni usar de forma fiable los controles de
cobro (totales, método de pago, botones de acción) en anchos que no sean el ideal — es un bloqueo
de la tarea central de la pantalla en tablet y móvil.

**Independent Test**: se puede probar completamente seleccionando un pedido y abriendo la Terminal
de Mesas en los tres anchos (escritorio ≥ 1024px, tablet 768–1023px, móvil < 768px), verificando en
cada uno que todo el contenido del panel (encabezado, lista de productos, totales, método de pago,
botones de acción) queda dentro del área visible y que la página no necesita desplazamiento
horizontal.

**Acceptance Scenarios**:

1. **Given** un pedido seleccionado en ancho de escritorio (≥ 1024px), **When** el cajero mira el
   panel derecho, **Then** todo su contenido queda dentro del panel — nada recortado por el borde ni
   fuera de la pantalla — y la página no tiene desplazamiento horizontal.
2. **Given** un pedido seleccionado en ancho de tablet (768–1023px), **When** el cajero mira el
   panel derecho, **Then** el panel reemplaza a la grilla y ocupa todo el ancho disponible,
   mostrando todo su contenido, sin recorte lateral ni desplazamiento horizontal de la página; al
   cerrarlo, el cajero vuelve a la grilla en el mismo estado (pestaña, filtro, scroll).
3. **Given** un pedido seleccionado en ancho de móvil (< 768px), **When** el cajero mira el panel
   derecho, **Then** el panel ocupa el ancho disponible del dispositivo y muestra todo su contenido
   dentro de la pantalla.
4. **Given** cualquiera de los tres anchos, **When** el cajero busca los controles de cobro (totales,
   método de pago, botones de acción), **Then** todos están visibles y se pueden accionar sin salir
   del área visible.
5. **Given** el contenido del panel más alto que el espacio disponible, **When** el cajero necesita
   ver lo que queda abajo, **Then** se desplaza verticalmente dentro del panel — nunca en horizontal
   y nunca con contenido recortado sin forma de alcanzarlo.
6. **Given** un pedido con nombres de producto largos, una dirección de domicilio larga o un nombre
   de cliente largo, **When** el cajero mira el panel en cualquier ancho, **Then** ese texto se
   ajusta al ancho del panel (envuelve o se acomoda) sin ensancharlo ni empujar contenido fuera de
   la vista.
7. **Given** una mesa con un pago QR pendiente de confirmar (pestaña "🔔 Pagos por confirmar"),
   **When** el cajero abre esa superficie en cualquiera de los tres anchos, **Then** todo su
   contenido queda dentro del área visible y se desplaza solo verticalmente dentro del panel — sin
   recorte lateral ni scroll horizontal de página.

---

### User Story 4 - Ver varios productos del pedido a la vez en el panel de detalle (Priority: P2)

El cajero selecciona un pedido con varios productos y revisa el detalle en el panel derecho. Hoy la
sección que lista los productos recibe muy poco alto: las secciones fijas del panel (encabezado,
datos del pedido, totales, acciones) la comprimen a una franja mínima donde solo se ven uno o dos
productos, y revisar el pedido completo obliga a un desplazamiento incómodo dentro de un espacio muy
pequeño. Con este ajuste, la lista de productos recibe el espacio vertical disponible restante y se
ven varios productos a la vez.

**Why this priority**: es un problema de experiencia de uso real (revisar un pedido antes de
cobrarlo es una tarea frecuente), pero no bloquea la tarea ni muestra datos equivocados —
por eso va después de las tres correcciones P1.

**Independent Test**: se puede probar completamente seleccionando un pedido de al menos 6 productos
y verificando que en el panel de detalle se ven varios productos a la vez (no solo uno o dos), que
la lista se desplaza dentro de su propia área, y que el encabezado del pedido y la zona de
totales/acciones permanecen visibles mientras se desplaza la lista.

**Acceptance Scenarios**:

1. **Given** un pedido con 6 o más productos seleccionado, **When** el cajero mira el panel de
   detalle, **Then** la lista de productos ocupa el espacio vertical disponible restante del panel y
   muestra varios productos a la vez (no queda comprimida a una franja de uno o dos).
2. **Given** esa lista con más productos de los que caben, **When** el cajero se desplaza por ella,
   **Then** el desplazamiento ocurre dentro del área de la lista, manteniendo visibles el encabezado
   del pedido y la zona de totales y acciones.
3. **Given** un pedido con pocos productos (1–2), **When** el cajero mira el panel, **Then** la lista
   no fuerza un alto artificial ni deja un hueco vacío con aspecto de error — el resto del panel se
   acomoda con normalidad.
4. **Given** ancho de móvil, **When** el cajero revisa un pedido largo, **Then** la lista de
   productos también recibe el espacio disponible y los botones de cobro siguen siendo alcanzables
   sin tener que desplazar toda la pantalla.

---

### User Story 5 - Ver la información del domicilio de forma compacta, junto al estado del pedido (Priority: P2)

El cajero selecciona un pedido de Domicilio y revisa su detalle. Hoy la dirección de entrega, el
teléfono de contacto y el valor del domicilio ocupan un bloque vertical extenso en el panel, que
empuja hacia abajo la lista de productos y los totales. Con este ajuste, esos tres datos se muestran
de forma compacta, en línea junto al estado del pedido, liberando alto para el resto del panel
(apoya directamente la Historia 4).

**Why this priority**: mismo tipo de mejora de experiencia que la Historia 4 y muy ligada a ella
(ambas buscan devolver alto a la lista de productos); se puede implementar de forma independiente
pero su valor se nota junto con la Historia 4.

**Independent Test**: se puede probar completamente seleccionando un pedido de Domicilio y
verificando que dirección, teléfono y valor del domicilio aparecen de forma compacta al lado del
estado del pedido (no como un bloque vertical extenso), que la dirección se muestra completa aunque
tenga que envolver en varias líneas, y que el alto que antes ocupaba ese bloque ahora está
disponible para la lista de productos.

**Acceptance Scenarios**:

1. **Given** un pedido de Domicilio seleccionado, **When** el cajero mira el panel de detalle,
   **Then** la dirección de entrega, el teléfono de contacto y el valor del domicilio se presentan
   de forma compacta, en línea junto al estado del pedido, no como un bloque vertical separado y
   extenso.
2. **Given** esa presentación compacta, **When** el cajero la lee, **Then** los tres datos siguen
   completos y legibles — no se elimina ninguno.
3. **Given** una dirección de entrega larga, **When** el cajero mira esa fila, **Then** la dirección
   se muestra completa, envolviendo en 2–3 líneas si hace falta, sin truncarse ni quedar escondida
   tras una interacción.
4. **Given** el valor del domicilio mostrado en esa fila, **When** el cajero lo compara con el total
   del pedido, **Then** es el mismo valor de domicilio que está sumado en ese total (Historia 1) —
   nunca dos cifras distintas para el mismo concepto.
5. **Given** un pedido que no es de Domicilio (mesa o "Para llevar"), **When** el cajero mira su
   detalle, **Then** no aparece esta fila de información de domicilio — el panel se ve como hoy para
   esos casos.

---

### User Story 6 - Aprovechar el ancho de la tablet ocultando el menú de navegación como en móvil (Priority: P2)

El cajero (o cualquier usuario) usa la aplicación en una tablet. Hoy, a ese ancho, el menú de
navegación global (la barra lateral de módulos) se mantiene fijo ocupando ancho de forma permanente,
igual que en escritorio, lo que deja poco espacio para el contenido de cada pantalla —
particularmente la Terminal de Mesas. En móvil ese menú ya está oculto por defecto y se despliega
bajo demanda. Con este cambio, la tablet adopta el mismo comportamiento que móvil: el menú se oculta
por defecto y se despliega con el mismo control, en toda la aplicación.

**Why this priority**: es el pedido explícito del punto 7 y beneficia a toda la app en tablet, pero
es un cambio de disposición sin riesgo sobre datos ni sobre la lógica de negocio — puede ir después
de las correcciones que sí bloquean tareas.

**Independent Test**: se puede probar completamente abriendo varias pantallas de la aplicación
(incluida la Terminal de Mesas) en un ancho de tablet (768–1023px) y verificando que el menú de
navegación global está oculto por defecto y se despliega con el mismo control que en móvil; luego
repitiendo en escritorio (≥ 1024px) y verificando que ahí el menú sigue mostrándose fijo como hoy.

**Acceptance Scenarios**:

1. **Given** la aplicación abierta en un ancho de tablet (768–1023px) en cualquier pantalla, **When**
   el usuario la mira, **Then** el menú de navegación global está oculto por defecto (no reserva
   ancho fijo permanente).
2. **Given** ese estado, **When** el usuario activa el control de menú (el mismo botón/hamburguesa
   que ya se usa en móvil), **Then** el menú se despliega superponiéndose al contenido, con las
   mismas opciones que en escritorio y móvil.
3. **Given** el menú desplegado en tablet, **When** el usuario elige una opción o toca fuera del
   menú, **Then** el menú se vuelve a ocultar — mismo comportamiento que en móvil.
4. **Given** la Terminal de Mesas abierta en tablet con el menú oculto, **When** el usuario la mira,
   **Then** el ancho liberado queda disponible para la grilla y el panel de detalle.
5. **Given** la aplicación abierta en escritorio (≥ 1024px), **When** el usuario la mira, **Then** el
   menú de navegación global se muestra fijo como hoy — este cambio no afecta el escritorio.
6. **Given** cualquiera de los tres anchos, **When** el usuario abre el menú, **Then** su contenido y
   sus opciones son los mismos — solo cambia cómo se muestra u oculta, no qué contiene.

---

### Edge Cases

- **Pedido de Domicilio con valor de domicilio cero** (envío gratis o promoción): la tarjeta y el
  detalle muestran el total sin sumarle nada por domicilio — el resultado coincide con el subtotal
  de productos, y esto es correcto, no el defecto que corrige la Historia 1.
- **Pedido de Domicilio histórico anterior a spec 056** (`delivery_fee` nulo): se trata como valor
  cero a efectos de mostrar el total de la tarjeta — mismo criterio que ya aplica el cobro para
  esos pedidos (spec 056, compatibilidad con datos históricos); no se recalcula ninguna venta ya
  emitida (Principio VII).
- **Promoción pausada, eliminada o con reglas cambiadas después de confirmar un pedido de Domicilio
  pendiente de cobro**: la tarjeta muestra el total con el descuento congelado en las líneas al
  confirmar (`discounted_unit_price`); el panel de cobro recalcula el descuento en vivo (spec 073) y
  puede mostrar un total distinto. El panel manda: es el importe que se factura. La tarjeta no se
  recalcula por sí sola (no hace peticiones por tarjeta, Assumptions); la diferencia se resuelve al
  abrir el pedido y cobrarlo. Cerrar esta brecha por completo exigiría revisar la restricción de
  "ninguna petición nueva al servidor" o "ningún cambio de backend" — decisión de negocio, fuera del
  alcance de esta spec.
- **El cajero pulsa el atajo de crear pedido estando en la pestaña "Domicilios" o "Para llevar"**: se
  comporta igual que pulsar el botón desde esa pestaña — abre la creación con el tipo
  correspondiente preseleccionado.
- **El cajero tiene una mesa seleccionada y cambia a la pestaña "Domicilios" y pulsa crear pedido**:
  la creación abre con tipo "Domicilio"; la selección previa de mesa se resuelve con el mismo
  criterio que ya aplica hoy al navegar a la creación manual con una mesa seleccionada (spec 059,
  Edge Cases).
- **Ancho de pantalla justo en el límite entre dos vistas** (768px o 1024px exactos): se resuelve con
  el mismo criterio de breakpoint ya usado hoy en la aplicación (`md` ≈ 768px, `lg` ≈ 1024px), sin
  zona muerta ni salto brusco; el panel de detalle y el menú de navegación adoptan de forma
  consistente el comportamiento de la vista que corresponde a ese ancho.
- **Panel de detalle con muy poco contenido** (pedido de un solo producto, sin datos de domicilio):
  se adapta al ancho igual que un panel lleno, sin dejar franjas vacías con aspecto de error.
- **Dirección de domicilio extremadamente larga** (varias líneas): se muestra completa envolviendo;
  si aun así empuja la lista de productos, el panel resuelve el exceso con desplazamiento vertical
  dentro del panel (Historia 3), nunca recortando la dirección ni desbordando en horizontal.
- **El usuario rota la tablet de vertical a horizontal cruzando el umbral de 1024px**: el menú de
  navegación y el panel de detalle pasan de forma consistente al comportamiento de la vista que
  corresponde al nuevo ancho, sin perder la selección ni el estado actual de la pantalla.
- **El menú de navegación se despliega en tablet mientras hay un pedido seleccionado en la Terminal
  de Mesas**: el menú se superpone sin cerrar la selección ni el panel de detalle; al cerrarse, el
  cajero vuelve exactamente al mismo estado.
- **Pestaña "Domicilios"/"Para llevar" sin ningún pedido pendiente**: sigue mostrando el mismo
  mensaje de listado vacío ya existente (spec 036 FR-003, spec 059 FR-009); el botón de crear pedido
  (Historia 2) igualmente está visible en ese estado.

## Requirements *(mandatory)*

### Functional Requirements — Total de la tarjeta de un pedido de Domicilio (Historia 1)

- **FR-001**: La tarjeta de un pedido de Domicilio (pestaña "Domicilios") DEBE mostrar como total el
  monto total a cobrar del pedido, incluyendo el valor del domicilio (`delivery_fee`, spec 056)
  además del subtotal de productos y de los descuentos, impuestos y propina que ya se consideran hoy
  para ese total.
- **FR-002**: El total mostrado en la tarjeta de un pedido de Domicilio DEBE representar el mismo
  importe que el panel de cobro muestra para ese pedido — el `total` de
  `GET /orders/{id}/checkout-preview` (spec 073,
  `073-fix-descuento-cobro-terminal/contracts/preview-cobro-pedido.md`):
  `max(0, subtotal de productos − descuento por promoción + valor de domicilio)`. Estos pedidos no
  llevan impuesto ni propina y el preview los fija en cero; spec 056 (`orders-checkout-total.md`)
  define la fórmula del `Sale.total` del backend, no la que pinta el panel. NO se usa una fórmula
  propia solo para la tarjeta ni se llama al backend una vez por tarjeta (Assumptions).
- **FR-003**: El total de la tarjeta y el total del panel de detalle/cobro para el mismo pedido de
  Domicilio DEBEN coincidir mientras la promoción que aplicó al pedido conserve el mismo estado y
  reglas que tenía al confirmarlo. La tarjeta se compone de los mismos datos que alimentan el panel:
  el descuento por promoción congelado en cada línea al confirmar el pedido (`discounted_unit_price`,
  spec 038 FR-013) y el `delivery_fee` del pedido. Si, después de confirmar, esa promoción se pausa,
  se elimina o cambia sus reglas, el panel —que recalcula el descuento en vivo contra el instante
  congelado del pedido (spec 073 FR-009a)— es la autoridad, y la tarjeta puede mostrar
  transitoriamente el descuento congelado hasta el cobro (ver Edge Cases). Fuera de ese caso, nunca
  se muestran dos cifras distintas para el mismo pedido.
- **FR-004**: La tarjeta de un pedido de Domicilio DEBE mostrar un único total combinado, sin
  desglose de "productos" y "domicilio" por separado — el desglose vive en el panel de detalle
  (Historia 5).
- **FR-005**: El total de las tarjetas de pedidos "Para llevar" y de pedidos de mesa NO DEBE cambiar
  con esta spec — se sigue calculando y mostrando exactamente como hoy (para estos tipos el valor de
  domicilio es cero/nulo).
- **FR-006**: Un pedido de Domicilio con valor de domicilio cero o nulo (envío gratis, o pedido
  histórico anterior a spec 056) DEBE mostrar un total igual al que resulta de no sumar nada por
  domicilio — sin error ni descuadre.

### Functional Requirements — Botón de crear pedido en todas las pestañas (Historia 2)

- **FR-007**: El botón para crear un pedido nuevo DEBE estar visible y disponible en las tres
  pestañas de la Terminal de Mesas ("Mesas", "Domicilios", "Para llevar"), en escritorio, tablet y
  móvil — con las mismas reglas de permiso y de habilitación que hoy. Su ubicación es la misma que
  hoy tiene en la pestaña "Mesas" sin selección: en escritorio, fijo al final del panel derecho de
  bienvenida (spec 076 FR-027); en tablet y móvil, como botón de acción fijo sobre la grilla/lista
  de tarjetas mientras el cajero navega cualquiera de las tres pestañas sin una tarjeta
  seleccionada. Con una tarjeta seleccionada, el botón NO forma parte del panel de cobro — mismo
  comportamiento que hoy en "Mesas".
- **FR-008**: Al activar el botón de crear pedido (o su atajo) desde la pestaña "Domicilios", el
  flujo de creación manual de pedido DEBE abrirse con el tipo de pedido "Domicilio" preseleccionado.
- **FR-009**: Al activarlo desde la pestaña "Para llevar", DEBE abrirse con el tipo "Para llevar"
  preseleccionado.
- **FR-010**: Al activarlo desde la pestaña "Mesas" o desde el panel derecho sin ninguna selección,
  DEBE abrirse sin ningún tipo preseleccionado — comportamiento idéntico al actual.
- **FR-011**: El tipo preseleccionado (FR-008, FR-009) DEBE poder cambiarse dentro del formulario de
  creación — es un valor inicial, no un bloqueo.
- **FR-012**: Salvo la visibilidad en todas las pestañas (FR-007) y el tipo preseleccionado (FR-008
  a FR-010), el resto del comportamiento del botón de crear pedido NO cambia: misma navegación
  destino, mismo atajo de teclado, mismos permisos, mismo resultado al crear el pedido.

### Functional Requirements — Comunicación del botón de crear pedido en móvil (Historia 2)

- **FR-013**: En ancho de móvil, el botón de crear pedido DEBE incluir un texto visible que comunique
  su función (por ejemplo "Crear pedido" / "Crear pedido nuevo"), no únicamente un ícono "+".
- **FR-014**: El texto del botón en móvil DEBE usar el mismo vocabulario ya establecido para esta
  acción en el resto de la pantalla ("Crear pedido nuevo", spec 076 FR-027) — sin introducir un
  término nuevo.
- **FR-015**: En escritorio, la presentación del botón de crear pedido NO cambia respecto a hoy
  (más allá de estar visible en todas las pestañas, FR-007): sigue fijo al final del panel derecho
  de bienvenida con su etiqueta completa ("+ Crear pedido nuevo", spec 076 FR-027). En tablet, al
  no existir ya un panel de bienvenida permanente (Historia 3), el botón se presenta como acción
  fija sobre la grilla/lista (igual que en móvil) y DEBE conservar una etiqueta de texto visible
  con el mismo vocabulario — nunca solo el ícono "+".

### Functional Requirements — Panel de detalle adaptado al ancho (Historia 3)

- **FR-016**: El panel derecho de detalle del pedido / cobro DEBE ajustarse al ancho disponible en
  las tres vistas (escritorio ≥ 1024px, tablet 768–1023px, móvil < 768px), mostrando todo su
  contenido dentro del área visible. En escritorio el panel se muestra al lado de la grilla; en
  tablet y en móvil, al seleccionar una mesa o un pedido el panel reemplaza a la grilla y ocupa
  todo el ancho disponible, y al cerrarlo el cajero vuelve a la grilla en el mismo estado (pestaña,
  filtro, scroll) — mismo patrón maestro-detalle que spec 076 ya asume para móvil.
- **FR-016a**: A ancho de tablet (768–1023px), la Terminal de Mesas sin ninguna mesa ni pedido
  seleccionado NO muestra un panel de bienvenida permanente ocupando ancho fijo (a diferencia de
  escritorio, spec 076 FR-027): la grilla/lista de tarjetas ocupa todo el ancho y el botón de crear
  pedido aparece como acción fija sobre ella (FR-007). Al seleccionar, el panel de detalle reemplaza
  la grilla (FR-016). Es el mismo colapso maestro-detalle que spec 076 asume para móvil, ahora
  explícito para tablet — no un panel nuevo (Out of Scope).
- **FR-017**: Ningún elemento del panel (encabezado, lista de productos, totales, selector de método
  de pago, botones de acción, y el botón de crear pedido cuando aplique — estado sin selección en
  escritorio) DEBE quedar recortado por el borde del panel ni fuera del área visible en ninguno de
  los tres anchos.
- **FR-018**: La Terminal de Mesas con un pedido seleccionado NO DEBE requerir desplazamiento
  horizontal de la página en ninguno de los tres anchos.
- **FR-019**: Cuando el contenido del panel exceda el alto disponible, el desbordamiento DEBE
  resolverse con desplazamiento vertical dentro del panel — nunca con recorte de contenido
  inalcanzable ni con desplazamiento horizontal.
- **FR-020**: El texto largo dentro del panel (nombres de producto, dirección de domicilio, nombre de
  cliente) DEBE ajustarse al ancho del panel (envolver o acomodarse) sin ensanchar el panel ni
  empujar contenido fuera de la vista.
- **FR-021**: Todos los controles de cobro DEBEN poder accionarse (pulsarse) en los tres anchos sin
  que queden fuera del área visible.
- **FR-021a**: La superficie "🔔 Pagos por confirmar" (revisión del pago QR del cajero,
  `app-payment-validation-block` bajo `effectiveCentralView() === 'validar-pago'`, spec 073 US7)
  comparte la columna de detalle que esta Historia reestructura. Tras el cambio, su contenido DEBE
  quedar íntegro dentro del área visible en los tres anchos, con desplazamiento solo vertical dentro
  del panel — mismas garantías que FR-017 y FR-019. Esta spec no cambia qué muestra ni qué valida
  esa superficie (spec 073), solo que no quede recortada.

### Functional Requirements — Espacio para la lista de productos (Historia 4)

- **FR-022**: En el panel de detalle del pedido, la lista de productos DEBE recibir el espacio
  vertical disponible restante del panel, de modo que las secciones fijas (encabezado, datos del
  pedido, totales, acciones) no la compriman a una franja mínima de uno o dos productos.
- **FR-023**: Cuando la lista de productos tenga más elementos de los que caben en su área, DEBE
  desplazarse verticalmente dentro de esa área, manteniendo visibles el encabezado del pedido y la
  zona de totales y acciones.
- **FR-024**: Un pedido con pocos productos NO DEBE forzar un alto artificial de la lista ni dejar un
  hueco vacío con aspecto de error — el panel se acomoda con normalidad.
- **FR-025**: En ancho de móvil DEBE aplicar la misma regla (FR-022, FR-023): la lista de productos
  recibe el espacio disponible y los botones de cobro siguen siendo alcanzables sin desplazar toda
  la pantalla.

### Functional Requirements — Información del domicilio en línea (Historia 5)

- **FR-026**: En el panel de detalle de un pedido de Domicilio, la dirección de entrega, el teléfono
  de contacto y el valor del domicilio DEBEN presentarse de forma compacta, en línea junto al estado
  del pedido, en vez de como un bloque vertical separado y extenso.
- **FR-027**: Esta presentación compacta NO DEBE eliminar ninguno de los tres datos — dirección,
  teléfono y valor del domicilio siguen mostrándose completos y legibles.
- **FR-028**: Una dirección de entrega larga DEBE mostrarse completa, envolviendo en varias líneas si
  hace falta — NO DEBE truncarse ni quedar escondida tras una interacción.
- **FR-029**: El valor del domicilio mostrado en esta fila DEBE ser el mismo valor que está sumado en
  el total del pedido (FR-001) — nunca dos cifras distintas para el mismo concepto.
- **FR-030**: Para pedidos que no son de Domicilio (mesa o "Para llevar"), esta fila de información
  de domicilio NO aplica y NO DEBE mostrarse — el panel se ve como hoy para esos casos.

### Functional Requirements — Menú de navegación global en tablet (Historia 6)

- **FR-031**: En anchos de tablet (768px–1023px), el menú de navegación global de la aplicación DEBE
  adoptar el mismo comportamiento que ya tiene en móvil (< 768px): oculto por defecto y desplegable
  bajo demanda mediante el mismo control (botón/hamburguesa), superponiéndose al contenido en vez de
  reservar ancho fijo permanente.
- **FR-032**: Este comportamiento en tablet DEBE aplicar en toda la aplicación, no solo en la
  Terminal de Mesas.
- **FR-033**: Al ocultarse el menú en tablet, el ancho liberado DEBE quedar disponible para el
  contenido de cada pantalla.
- **FR-034**: En escritorio (≥ 1024px), el menú de navegación global NO DEBE cambiar — se sigue
  mostrando fijo como hoy.
- **FR-035**: El contenido y las opciones del menú de navegación DEBEN ser los mismos en las tres
  vistas — solo cambia cómo se muestra u oculta, no qué contiene ni a dónde lleva cada opción.
- **FR-036**: El control para desplegar/ocultar el menú en tablet DEBE ser el mismo que ya se usa en
  móvil — no se introduce un control nuevo distinto solo para tablet.

### Functional Requirements — No regresión

- **FR-037**: Esta spec NO DEBE cambiar el layout de grilla responsive de 3 variantes, la barra
  superior operativa, los contadores de ocupación ni la referencia "Atendido por" definidos por spec
  076 — solo corrige el comportamiento del panel derecho y de los elementos indicados en las
  tarjetas/pestañas.
- **FR-038**: Esta spec NO DEBE cambiar el vocabulario de estados ni la paleta de colores ya
  establecidos (spec 076 FR-029, FR-030).
- **FR-039**: Esta spec NO DEBE cambiar ninguna regla de negocio de cobro, facturación o creación de
  pedidos — el total de un pedido de Domicilio ya incluía el valor del domicilio en el cobro (spec
  056); esta spec solo corrige que la **tarjeta** lo muestre igual.
- **FR-040**: Ninguna acción hoy disponible en la Terminal de Mesas (buscar, filtrar, seleccionar,
  crear pedido, cobrar) DEBE dejar de estar disponible en ninguno de los tres anchos tras estos
  cambios.

### Key Entities *(include if feature involves data)*

- **Pedido (`CustomerOrder` / `DiningOrder`)**: entidad ya existente. Esta spec **no** agrega ni
  modifica ningún campo. Usa el campo `delivery_fee` que spec 056 ya agregó y que el cobro ya suma
  al total, para que la tarjeta de la pestaña "Domicilios" muestre el mismo total (FR-001) y para
  mostrarlo en la fila compacta del detalle (FR-026).
- **Venta (`Sale`)**: sin cambios — esta spec no toca facturación ni recalcula ninguna venta emitida
  (Principio VII).
- **Menú de navegación global**: elemento de interfaz de la aplicación (no una entidad de datos).
  Esta spec solo cambia cómo se muestra/oculta a ancho de tablet (FR-031 a FR-036), no sus opciones
  ni su destino.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: En el 100% de los pedidos de Domicilio con valor de domicilio distinto de cero, el
  total mostrado en la tarjeta de la pestaña "Domicilios" coincide exactamente con el total que se
  factura al cobrar ese pedido.
- **SC-002**: El cajero puede crear un pedido nuevo desde cualquiera de las tres pestañas de la
  Terminal de Mesas sin tener que cambiar de pestaña primero, y desde "Domicilios"/"Para llevar" el
  tipo llega preseleccionado en el 100% de los casos.
- **SC-003**: En ancho de móvil, un cajero que nunca ha usado el sistema identifica correctamente
  para qué sirve el botón de crear pedido con solo leerlo, sin tener que pulsarlo para descubrirlo.
- **SC-004**: Con un pedido seleccionado, en los tres anchos soportados (escritorio, tablet, móvil)
  el 100% del contenido del panel de detalle/cobro queda dentro del área visible y la página no
  requiere ningún desplazamiento horizontal.
- **SC-005**: Con un pedido de 6 o más productos seleccionado, en escritorio y en tablet el cajero ve
  al menos 4 productos a la vez en el panel de detalle sin desplazarse; en móvil ve los que quepan
  tras el encabezado, los totales y las acciones — al menos 2. En los tres anchos, los botones de
  cobro permanecen visibles.
- **SC-006**: En un pedido de Domicilio, la información de dirección, teléfono y valor del domicilio
  ocupa menos alto del panel que hoy, y ese alto recuperado queda disponible para la lista de
  productos.
- **SC-007**: A ancho de tablet, en cualquier pantalla de la aplicación, el contenido principal
  dispone del ancho que hoy ocupa el menú de navegación fijo, y el menú sigue siendo accesible en
  una sola interacción (el mismo control que en móvil).
- **SC-008**: Ninguna acción hoy disponible en la Terminal de Mesas deja de estar disponible en
  ninguno de los tres anchos tras estos cambios.

## Impacto sobre comportamiento existente y decisiones de negocio

Conforme al Principio II de la [Constitución](../../.specify/memory/constitution.md), esta spec
explicita qué comportamiento existente cambia:

1. **Total de la tarjeta de un pedido de Domicilio (FR-001)**: se considera **corrección de un
   defecto de presentación**, no un cambio de regla de negocio — la regla de que el total del pedido
   de Domicilio incluye el valor del domicilio ya fue decidida y autorizada por spec 056 para el
   cobro y la facturación; la tarjeta simplemente no la reflejaba. Se recomienda registrar este
   defecto en `specs/000-reconocimiento/registro-de-anomalias.md` como anomalía de visualización
   detectada por el dueño el 2026-09-07, con la corrección acotada a la tarjeta (ninguna venta
   emitida se recalcula, Principio VII).
2. **Tipo de pedido preseleccionado al crear desde una pestaña (FR-008, FR-009)**: comportamiento
   **nuevo** — hoy la creación manual siempre abre sin tipo preseleccionado. Autorizado por el dueño
   el 2026-09-07 (Clarifications). No cambia ninguna regla de negocio de creación de pedidos; solo
   fija un valor inicial que el usuario puede modificar (FR-011).
3. **Menú de navegación global colapsable en tablet, en toda la aplicación (FR-031, FR-032)**: cambio
   de comportamiento **fuera de la Terminal de Mesas** — a ancho de tablet, todas las pantallas de
   la aplicación pasan a ocultar el menú por defecto. Autorizado por el dueño el 2026-09-07
   (Clarifications). Se recomienda registrar esta decisión en
   `specs/000-reconocimiento/registro-de-anomalias.md` indicando: qué cambia (presentación del menú
   de navegación en tablet), por qué (liberar ancho y unificar con el comportamiento de móvil),
   quién (el dueño), cuándo (2026-09-07), y qué se ve afectado (todas las pantallas a ancho de
   tablet; ninguna opción ni destino del menú cambia).
4. **Panel de detalle/cobro en tablet como patrón maestro-detalle (FR-016, Historia 3)**: se
   considera parte de la **corrección del defecto de presentación** de la Historia 3 (hoy el panel
   en tablet aparece recortado). A ancho de tablet, al seleccionar una mesa o un pedido el panel
   reemplaza la grilla a todo el ancho — el mismo colapso maestro-detalle que spec 076 Assumptions
   ya asume para móvil, ahora explícito para tablet. Decidido con el dueño en la sesión de
   aclaración del 2026-09-07 (Clarifications). No cambia el comportamiento de escritorio (grilla y
   panel lado a lado) ni ninguna regla de negocio de cobro; solo acota cómo se muestra un panel que
   hoy no se ve completo. Incluye que, sin selección, en tablet ya no se pinta el panel de
   bienvenida permanente de escritorio (spec 076 FR-027): a ese ancho la grilla ocupa todo y el
   botón de crear pedido va sobre ella (FR-016a, FR-007) — consistente con el colapso a una sola
   vista por vez.

## Out of Scope

- **Cualquier cambio de backend**: esta spec es de frontend. `delivery_fee` ya lo expone el backend
  (spec 056); no se agrega ni modifica ningún endpoint, campo, columna ni migración.
- **Recalcular o re-facturar pedidos/ventas ya emitidos**: la corrección del total de la tarjeta
  (FR-001) aplica a la visualización de pedidos pendientes de cobro; ninguna venta emitida se toca
  (Principio VII).
- **Rediseñar otra vez la Terminal de Mesas**: la grilla responsive de 3 variantes, la barra
  superior operativa, los contadores de ocupación y "Atendido por" (spec 076) no cambian.
- **Cambiar el vocabulario de estados o la paleta de colores** (spec 076 FR-029, FR-030): se
  preservan tal cual.
- **Convertir el panel derecho en un patrón maestro-detalle nuevo**: la Historia 3 corrige que el
  panel se **ajuste al ancho** y muestre todo su contenido, y fija que en tablet siga el mismo
  patrón maestro-detalle que spec 076 ya asume para móvil (reemplaza la grilla a todo el ancho); no
  redefine el patrón para escritorio ni introduce uno nuevo distinto del de spec 076.
- **Bloquear el tipo de pedido en el formulario de creación**: el tipo preseleccionado (FR-008,
  FR-009) siempre se puede cambiar (FR-011).
- **Cambiar el contenido, el orden o los destinos del menú de navegación global**: la Historia 6
  solo cambia cómo se muestra/oculta en tablet.
- **Introducir un control nuevo para el menú en tablet**: se reutiliza el mismo control de móvil
  (FR-036).
- **Logística de domicilios** (asignación de repartidor, seguimiento, estado de entrega): sin
  cambios — no existe hoy y esta spec no lo agrega (spec 056, spec 059).

## Assumptions

- **Los breakpoints de escritorio/tablet/móvil reutilizan los mismos umbrales ya usados hoy en la
  aplicación** (`md` ≈ 768px, `lg` ≈ 1024px, citados por spec 076 Assumptions y
  `table-sessions.component.ts:54,73` en `pos-heladeria`) — no se introduce un sistema de
  breakpoints nuevo.
- **"El estado del domicilio" del punto 6 se interpreta como el estado del pedido de Domicilio** (la
  insignia de estado ya existente: "Por confirmar", "En preparación", "Listo", "Pago pendiente",
  etc., `pos-terminal.store.ts:115-123`) — no existe hoy un concepto separado de "estado de
  entrega"/"estado del domicilio" (spec 056 y spec 059 lo dejan explícitamente fuera de alcance).
- **La fórmula del total que el panel de cobro muestra para un pedido de Domicilio ya está definida
  por spec 073** (`GET /orders/{id}/checkout-preview` →
  `max(0, subtotal − descuento por promoción + valor de domicilio)`, sin impuesto ni propina —
  `073-fix-descuento-cobro-terminal/contracts/preview-cobro-pedido.md`). Spec 056
  (`orders-checkout-total.md`) autorizó incluir el valor del domicilio en el total facturado; spec
  073 fija cómo lo compone el preview que ve el cajero. La Historia 1 alinea la tarjeta con ese
  importe reutilizando datos ya cargados (líneas del pedido + `delivery_fee`), sin fórmula nueva ni
  petición por tarjeta.
- **El texto del botón de crear pedido en móvil reutiliza "Crear pedido nuevo"** (spec 076 FR-027,
  `pos-checkout-panel.component.ts:131-144` en `pos-heladeria`); si el espacio en móvil no permite
  la etiqueta completa junto al ícono, se acepta una forma corta legible ("Crear pedido") siempre
  que comunique la acción — nunca solo el ícono "+".
- **El flujo de creación manual de pedido ya soporta los tres tipos** (mesa, "Para llevar",
  Domicilio) desde specs 055 y 056 (`manual-order-page.component.ts` en `pos-heladeria`); la
  Historia 2 solo le pasa un tipo inicial según la pestaña de origen, sin cambiar ese formulario.
- **El menú de navegación global ya tiene un comportamiento colapsable en móvil** (control
  botón/hamburguesa que lo superpone); la Historia 6 extiende exactamente ese comportamiento al
  rango de tablet, sin crear una variante nueva.
- **Las tres vistas (escritorio/tablet/móvil) de la Historia 3 son las mismas tres que define spec
  076** — esta spec no agrega ni quita variantes de ancho, solo corrige que el panel derecho se
  ajuste correctamente en las que ya existen.
- **En tablet y en móvil, el panel de detalle/cobro sigue el patrón maestro-detalle que colapsa a
  una sola vista por vez** — spec 076 Assumptions lo asume explícitamente para móvil; esta spec lo
  extiende al rango de tablet tras la sesión de aclaración del 2026-09-07: al seleccionar una mesa o
  un pedido el panel reemplaza la grilla a todo el ancho, y al cerrarlo el cajero vuelve a la grilla
  en el mismo estado (pestaña, filtro, scroll). Solo en escritorio (≥ 1024px) la grilla y el panel
  se muestran lado a lado — ahí el comportamiento de aparición/desaparición del panel no cambia.
- **Ninguna de las siete correcciones requiere datos nuevos ni peticiones nuevas al servidor** — toda
  la información necesaria (valor del domicilio, dirección, teléfono, tipo de pedido, estado) ya la
  entrega el backend hoy y ya la consume la Terminal de Mesas.
