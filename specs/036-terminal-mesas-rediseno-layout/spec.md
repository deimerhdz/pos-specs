# Feature Specification: Rediseño de Layout de la Terminal de Mesas — Franja de Órdenes y Menú Central

**Feature Branch**: `036-terminal-mesas-rediseno-layout`

**Created**: 2026-08-25

**Status**: Draft

**Naturaleza de esta spec**: ajuste de **layout y distribución** de la Terminal de Mesas (pantalla ya
implementada por [spec 028](../028-terminal-mesas-modo-hibrido/spec.md) y sus correcciones en
[spec 029](../029-correccion-cobro-cierre-mesa/spec.md)), tomando como referencia una imagen de diseño
provista por el usuario. El comportamiento ya implementado (validación de pago QR, cobro manual,
cálculo de cambio, envío a cocina, facturación, cierre de mesa) **se mantiene sin cambios** — esta
spec solo reorganiza cómo se distribuyen y presentan esos mismos componentes en pantalla, con una
única adición puntual: un botón para colapsar/expandir el menú de navegación global de la aplicación.
El filtro visual "Domicilios" del diseño de referencia se incorpora al layout (Principio VI de la
[Constitución](../../.specify/memory/constitution.md): evolución incremental, sin mezclar clases de
cambio distintas en un mismo incremento) **sin la capacidad de crear órdenes de ese tipo todavía**: la
creación de un tipo de orden "Domicilio" (que, según se descubrió durante `/speckit-plan`, requeriría un campo
nuevo de backend y un flujo de creación de "venta de mostrador" que hoy no existe en el frontend — ver
Clarifications) se especifica y construye en una spec futura independiente.

**Input**: User description: mejorar el diseño de la Terminal de Mesas — específicamente el
componente donde se muestran las mesas — adaptando el layout y la distribución de los componentes a
una imagen de referencia, sin implementar comportamiento distinto al ya implementado salvo lo
explícitamente indicado. Debe existir un botón en la parte superior para cerrar y abrir el sidebar de
navegación; se deben listar las órdenes tal cual la imagen de referencia (tarjetas de orden con
tiempo transcurrido), con tres filtros: Todas las Órdenes, Domicilios, Mesas; en la parte derecha se
mantiene el resumen/cobro de la orden ya existente; en el centro se muestra el menú de productos con
el diseño de la imagen (grid de productos, categorías, buscador por nombre), reemplazando el catálogo
que hoy se abre como panel superpuesto; la forma de crear una orden se adapta a este nuevo layout.

## Clarifications

### Session 2026-08-25

- Q: El diseño trae un filtro "Domicilios" (Delivery), pero el sistema actual solo maneja canales
  qr/counter/waiter — no existe el concepto de "domicilio" como tipo de orden. ¿Cómo se trata ese
  filtro en este ajuste? → A: Se crea un tipo de orden nuevo "Domicilio". Para mantener el alcance
  acotado a un ajuste de layout, esta spec lo implementa como una **etiqueta de tipo de orden** sobre
  el flujo de venta de mostrador ya existente (spec 011) — no como un canal (`channel`) nuevo de
  backend ni como una integración de logística de despacho.
- Q: El diseño muestra una línea "Tax (18%)" en el resumen de pago, pero el impuesto está
  deliberadamente fijado en $0 en la Terminal de Mesas (decisión de negocio previa, ver A-41 en
  spec 011). ¿Qué se hace con esa línea? → A: Se mantiene en $0. La línea "Impuesto" se conserva
  visualmente en el resumen para igualar la disposición del diseño, pero el valor sigue fijo en $0,
  sin reabrir la decisión fiscal ya tomada en A-41 ni su cálculo.
- Q: El botón para "cerrar y abrir el sidebar" solicitado en la parte superior — ¿a cuál panel se
  refiere? → A: Al panel de menú de opciones del administrador — el menú de navegación global del
  panel/dashboard de la aplicación (no un panel interno de la Terminal de Mesas). El botón permite
  colapsarlo para ganar espacio de pantalla mientras se opera la Terminal de Mesas.
- Q: Una vez que una orden de Domicilio ya fue cobrada y facturada, ¿debe desaparecer de inmediato de
  "Órdenes Recientes", o debe permanecer visible (como una orden de Mesa) hasta que el cajero la
  cierre manualmente? → A: Desaparece automáticamente del listado en cuanto se cobra/factura, igual
  que una venta de mostrador hoy (sin paso de cierre); Domicilio no introduce ninguna acción de
  "Cerrar"/"Liberar" nueva.
- Q: El diseño de referencia muestra una barra de tiempo transcurrido (verde/roja) en cada tarjeta de
  orden, pero hoy el listado de mesas usa insignias de texto+color "Por confirmar"/"En preparación"/
  "Libre" (spec 028, FR-014), obligatorias por accesibilidad (el estado nunca depende solo del color).
  ¿Las tarjetas del nuevo listado deben mantener esas insignias además de la barra de tiempo, o la
  barra la reemplaza? → A: Se mantienen las insignias de texto+color ya existentes y se añade la barra
  de tiempo del diseño como elemento visual adicional en la misma tarjeta, sin que ninguna reemplace a
  la otra.
- Q: Para que una tarjeta de orden de tipo Domicilio muestre una "referencia de cliente" (FR-001), ¿es
  obligatorio capturar el nombre del cliente al crear la orden, o puede quedar sin nombre? → A:
  Opcional — reutiliza el dato de cliente ya existente en venta de mostrador ("Consumidor Final" por
  defecto, spec 011 FR-010); si no se captura nombre, la tarjeta muestra solo el número de orden, sin
  bloquear la creación de la orden.

### Session 2026-08-25 (durante `/speckit-plan`)

- Q: Al investigar el código real (`pos-heladeria`, `pos-backend`) para planear la implementación, se
  descubrió que el flujo de "venta de mostrador" (spec 011) que FR-008 asumía "ya implementado" para
  que Domicilio lo reutilizara **no existe en el frontend** (ningún archivo `*mostrador*`), y que el
  backend tampoco tiene un campo `channel`/`order_type` ni nada equivalente a "Domicilio" — solo
  infiere mesa-vs-mostrador por si `dining_table_id` es nulo. Agregar Domicilio tal como estaba
  especificado exigiría una migración de backend nueva y construir un flujo de creación desde cero, lo
  que mezclaría nueva funcionalidad + migración de datos + cambio de layout en un mismo incremento
  (Principio VI de la Constitución). ¿Cómo se ajusta el alcance de esta spec ante este hallazgo? → A:
  Esta spec (036) se acota estrictamente al rediseño de layout — la pestaña "Domicilios" se muestra en
  el filtro (coincidiendo con el diseño de referencia) pero permanece vacía porque todavía no existe
  ninguna vía para crear una orden de ese tipo. La creación de órdenes de Domicilio (campo de backend +
  flujo de "venta de mostrador" en el frontend) se define en una spec futura independiente.

## User Scenarios & Testing _(mandatory)_

### User Story 1 - El cajero encuentra cualquier orden activa desde un único listado con filtros (Priority: P1)

En la parte superior de la pantalla, ocupando todo el ancho disponible, en lugar de una grilla de
mesas agrupada solo por su estado de ocupación, el cajero ve una franja horizontal de "Órdenes
Recientes" — una tarjeta por orden activa, con su identificador, cliente o mesa asociada, su insignia
de estado de texto+color ya existente ("Por confirmar"/"En preparación") y el tiempo transcurrido
desde que se abrió mostrado como una barra visual, tal como en la imagen de referencia — filtrable por
tres pestañas: "Todas las Órdenes", "Domicilios" y
"Mesas". Cuando las tarjetas no caben todas en el ancho visible, dos botones tipo carrusel (flecha
izquierda / flecha derecha) permiten desplazar el listado horizontalmente. Al seleccionar una tarjeta,
se abre esa orden en el centro y la derecha exactamente como hoy se abre una mesa.

**Why this priority**: es el cambio de mayor visibilidad del rediseño y la puerta de entrada a
cualquier otra acción de la pantalla; sin esta franja el resto de la reorganización (menú central,
resumen derecho) no tiene desde dónde abrirse.

**Independent Test**: abrir la Terminal de Mesas con más órdenes activas (mesas y mostrador) que las
que caben en el ancho de pantalla, alternar entre las tres pestañas de filtro, usar los botones de
carrusel para desplazar el listado y confirmar que se revelan las tarjetas restantes, y verificar que
seleccionar cualquier tarjeta abre la misma orden que hoy se abre al seleccionar una mesa.

**Acceptance Scenarios**:

1. **Given** varias mesas con órdenes activas, **When** el cajero selecciona la pestaña "Mesas",
   **Then** ve únicamente las tarjetas de las mesas con orden activa, con el mismo detalle
   (identificador, cliente/mesa, tiempo transcurrido) que hoy se muestra en el listado de mesas.
2. **Given** que hoy no existe ninguna vía para crear una orden de tipo Domicilio (ver Clarifications,
   sesión durante `/speckit-plan`), **When** el cajero selecciona la pestaña "Domicilios", **Then** ve
   un listado vacío con un estado claro (no un error ni una pantalla en blanco sin explicación).
3. **Given** cualquier estado de las mesas activas, **When** el cajero selecciona "Todas las Órdenes",
   **Then** ve el mismo conjunto de tarjetas que vería en "Mesas" (no hay órdenes de Domicilio que
   agregar todavía).
4. **Given** el listado de órdenes visible, **When** el cajero selecciona una tarjeta, **Then** el
   panel central y el panel derecho muestran esa orden con el mismo comportamiento ya implementado
   (bloque de validación de pago QR, panel de construcción de orden manual, o resumen de cuenta/cobro,
   según corresponda).
5. **Given** una mesa libre sin orden activa, **When** el cajero la busca o la selecciona desde la
   franja superior, **Then** sigue disponible la misma vía ya implementada para iniciar una orden
   sobre esa mesa (spec 028, Historia 2), solo reubicada visualmente dentro del nuevo layout.
6. **Given** más tarjetas de orden que las que caben en el ancho visible, **When** el cajero pulsa la
   flecha derecha del carrusel, **Then** el listado se desplaza para revelar las tarjetas siguientes;
   **When** ya no hay más tarjetas hacia ese lado, **Then** esa flecha aparece deshabilitada.
7. **Given** una tarjeta de orden de tipo Mesa con pago QR pendiente de revisión, **When** se muestra
   en la franja superior, **Then** incluye tanto su insignia "Por confirmar" (texto + color) como la
   barra de tiempo transcurrido, sin que una sustituya a la otra.

---

### User Story 2 - El cajero arma la orden desde un menú central siempre visible, con búsqueda y categorías (Priority: P1)

El catálogo de productos deja de abrirse como un panel superpuesto de pantalla completa. En su lugar,
el panel central muestra permanentemente una grilla de productos, con pestañas de categoría y un
campo de búsqueda por nombre, tal como en la imagen de referencia. El cajero agrega productos a la
orden en construcción directamente desde esa grilla, sin salir de la pantalla ni abrir un panel
adicional.

**Why this priority**: es el segundo cambio central del rediseño (deprecar el catálogo como panel
superpuesto) y el medio principal para construir cualquier orden manual; sin esto, el resto del
layout no tiene cómo agregar productos.

**Independent Test**: con una orden en construcción, escribir un texto en el buscador y verificar que
la grilla se filtra por nombre; seleccionar una categoría y verificar que la grilla se filtra por
categoría; agregar un producto desde la grilla y verificar que aparece en el resumen de la derecha con
el mismo comportamiento ya implementado (control de cantidad, nota, eliminar).

**Acceptance Scenarios**:

1. **Given** el panel central con la grilla de productos, **When** el cajero escribe parte del nombre
   de un producto en el buscador, **Then** la grilla muestra únicamente los productos cuyo nombre
   coincide, sin recargar ni salir de la pantalla.
2. **Given** la grilla de productos, **When** el cajero selecciona una categoría, **Then** la grilla
   muestra únicamente los productos de esa categoría; seleccionar "Todos" vuelve a mostrar el catálogo
   completo.
3. **Given** una orden manual en construcción, **When** el cajero agrega un producto desde la grilla
   central, **Then** el producto aparece en el resumen del panel derecho con la misma mecánica de
   cantidad, nota y eliminación ya implementada hoy en el constructor de orden.
4. **Given** una orden de origen QR en modo "Resumen de Cuenta" (solo lectura), **When** el cajero mira
   el panel central, **Then** no se ofrece la grilla de agregar productos para esa orden, igual que hoy
   el catálogo no está disponible para órdenes QR de solo lectura.
5. **Given** una búsqueda o filtro de categoría sin resultados, **When** el cajero lo aplica, **Then**
   el panel central muestra un estado vacío claro en vez de una grilla en blanco sin explicación.

---

### User Story 3 - El usuario colapsa el menú de navegación global para ganar espacio de pantalla (Priority: P2)

En la parte superior de la Terminal de Mesas hay un botón que abre y cierra el menú de navegación
global del panel administrativo (el mismo menú lateral que existe hoy en el resto de la aplicación),
permitiendo que la Terminal de Mesas use el ancho completo de la pantalla mientras se opera.

**Why this priority**: es una mejora de espacio de trabajo independiente del resto del rediseño; no
bloquea ni es bloqueada por las demás historias, por lo que se prioriza al final.

**Independent Test**: con el menú de navegación global visible, pulsar el botón y verificar que se
oculta y el contenido de la Terminal de Mesas ocupa el espacio liberado; pulsarlo de nuevo y verificar
que el menú vuelve a mostrarse.

**Acceptance Scenarios**:

1. **Given** el menú de navegación global visible, **When** el usuario pulsa el botón en la parte
   superior de la Terminal de Mesas, **Then** el menú se oculta y el contenido de la terminal ocupa el
   espacio adicional.
2. **Given** el menú de navegación global oculto, **When** el usuario pulsa el mismo botón, **Then** el
   menú vuelve a mostrarse en su disposición habitual.
3. **Given** el menú colapsado, **When** el usuario navega a otra sección de la aplicación fuera de la
   Terminal de Mesas, **Then** el estado de colapso no impide acceder a las demás opciones de
   navegación (siguen accesibles aunque el menú esté colapsado, según el patrón ya usado en el resto
   del panel administrativo).

---

### Edge Cases

- **El cajero selecciona la pestaña "Domicilios"**: no existe hoy ninguna vía para crear una orden de
  ese tipo (ver Clarifications, sesión durante `/speckit-plan`); el sistema muestra el listado vacío
  con un mensaje claro en vez de un error o una pantalla en blanco sin explicación.
- **El listado de "Órdenes Recientes" crece más allá del ancho visible**: aparecen los botones de
  carrusel (flecha izquierda / flecha derecha) para desplazar horizontalmente el listado, sin perder
  la posibilidad de filtrar por las tres pestañas; cuando el listado está al inicio o al final, la
  flecha correspondiente a ese extremo se muestra deshabilitada en vez de desplazar hacia la nada.
- **El cajero cambia de pestaña de filtro mientras el listado está desplazado hacia la derecha por el
  carrusel**: al cambiar de filtro, el listado vuelve a mostrarse desde el inicio (primera tarjeta
  visible), evitando que quede desplazado sobre un conjunto de tarjetas que ya no corresponde al
  filtro activo.
- **El cajero filtra el menú central por categoría y además escribe un término de búsqueda**: ambos
  filtros se combinan (solo se muestran productos que cumplen la categoría Y el texto buscado); si no
  hay coincidencias, se muestra el estado vacío de la Historia 2.
- **El botón de colapsar el menú de navegación se pulsa mientras hay un bloque de validación de pago o
  un cobro en curso**: colapsar o expandir el menú global no interrumpe ni descarta la acción en curso
  en el panel central o derecho.
- **Una orden de tipo Mesa cuya mesa física ya no existe o fue eliminada** (caso ya contemplado por el
  sistema actual): se sigue mostrando con el mismo tratamiento que hoy tiene ese caso; esta spec no
  cambia esa regla.
- **El impuesto mostrado en el resumen de pago**: siempre se muestra en $0 y no es editable por el
  cajero, sin excepción.

## Requirements _(mandatory)_

### Functional Requirements

- **FR-001**: El sistema DEBE reemplazar la grilla de mesas agrupada por estado de ocupación (Todas /
  Libres / Ocupadas / Pendientes) por una franja de tarjetas de orden ("Órdenes Recientes") ubicada en
  la parte superior de la pantalla, ocupando todo el ancho disponible, cada tarjeta mostrando al menos:
  identificador de la orden, mesa o referencia de cliente asociada, la insignia de estado de
  texto+color ya existente ("Por confirmar"/"En preparación"/"Libre" — spec 028, FR-014) y el tiempo
  transcurrido desde su apertura representado como una barra visual; la insignia y la barra de tiempo
  coexisten en la misma tarjeta, ninguna reemplaza a la otra.
- **FR-002**: Cuando las tarjetas de orden no quepan todas en el ancho visible, el sistema DEBE ofrecer
  dos botones tipo carrusel (flecha izquierda / flecha derecha) que desplazan horizontalmente el
  listado; cada flecha DEBE deshabilitarse cuando no queden más tarjetas hacia ese lado, y cambiar de
  pestaña de filtro (FR-003) DEBE reiniciar el desplazamiento al inicio del listado.
- **FR-003**: El listado de órdenes DEBE ofrecer exactamente tres filtros: "Todas las Órdenes",
  "Domicilios" y "Mesas". El filtro "Mesas" DEBE mostrar toda orden asociada a una mesa física;
  "Todas las Órdenes" DEBE mostrar el mismo conjunto que "Mesas" hasta que exista una vía para crear
  órdenes de tipo Domicilio (fuera de alcance de esta spec — ver Clarifications); el filtro
  "Domicilios" DEBE mostrarse siempre disponible pero con un listado vacío mientras esa vía no exista.
- **FR-004**: Seleccionar una tarjeta del listado DEBE abrir esa orden en los paneles central y
  derecho con exactamente el mismo comportamiento ya implementado hoy al seleccionar una mesa (bloque
  de validación de pago QR, constructor de orden manual, o resumen de cuenta/cobro, según el estado y
  origen de la orden — spec 028).
- **FR-005**: El sistema DEBE reemplazar el catálogo de productos que hoy se presenta como panel
  superpuesto de pantalla completa por una grilla de productos visible de forma permanente en el panel
  central, mientras exista una orden manual en construcción.
- **FR-006**: La grilla de productos del panel central DEBE ofrecer un campo de búsqueda por nombre y
  pestañas de filtro por categoría, ambos aplicables de forma combinada, sin necesidad de abandonar la
  pantalla de la Terminal de Mesas.
- **FR-007**: Agregar, quitar, ajustar cantidad o anotar un producto desde la grilla central DEBE
  producir exactamente el mismo efecto sobre la orden en construcción que el ya implementado hoy desde
  el catálogo superpuesto (sin cambios en el cálculo de precios, disponibilidad o descuento de
  inventario).
- **FR-008**: El panel derecho DEBE conservar, sin cambios de comportamiento, el resumen y cobro de la
  orden ya implementado (modos "Resumen de Cuenta" y "Terminal POS / Cobro Inmediato" de spec 028),
  adaptando únicamente su disposición visual a la imagen de referencia.
- **FR-009**: El resumen de pago del panel derecho DEBE mostrar una línea de "Impuesto" con valor fijo
  en $0, sin campo editable y sin alterar el cálculo de venta ya vigente (spec 011, decisión A-41).
- **FR-010**: El sistema DEBE ofrecer un botón en la parte superior de la Terminal de Mesas que
  colapsa o expande el menú de navegación global de la aplicación, sin afectar el estado de ninguna
  orden en curso en los paneles central o derecho.
- **FR-011**: El sistema NO DEBE reducir, respecto de la interfaz actual, ninguna información hoy
  disponible sobre el pedido, el pago o la factura (reutiliza spec 026, FR-012 / spec 028, FR-015).
- **FR-012**: El filtro "Domicilios" y el listado vacío que produce (FR-003) NO DEBEN bloquear, alterar
  ni reemplazar ningún flujo de creación de orden ya implementado (spec 028, Historia 2); esta spec no
  agrega ninguna vía nueva para crear una orden de tipo Domicilio.

### Key Entities _(include if feature involves data)_

- **Orden**: entidad ya existente (spec 011, spec 016, spec 028); no se le agregan atributos nuevos en
  esta spec. El filtro "Domicilios" (FR-003) es puramente de interfaz — no existe todavía ningún
  atributo de clasificación de tipo de orden en el modelo de datos que lo respalde; agregarlo se
  especifica en una spec futura independiente (ver Clarifications).
- **Ítem de Menú** y **Categoría de Menú**: entidades ya existentes en el catálogo; esta spec no
  cambia sus atributos, solo la forma en que se listan y filtran en pantalla (grilla central en vez de
  panel superpuesto).

## Success Criteria _(mandatory)_

### Measurable Outcomes

- **SC-001**: El 100% de las órdenes activas de Mesa son localizables desde la franja superior de
  órdenes en un máximo de dos interacciones (elegir filtro y, si aplica, usar los botones de carrusel
  para desplazarse).
- **SC-002**: El cajero puede agregar un producto a una orden en construcción sin abrir ningún panel o
  pantalla adicional a la Terminal de Mesas (0 pantallas superpuestas necesarias, frente al panel de
  catálogo superpuesto que existía antes).
- **SC-003**: Una búsqueda por nombre o un filtro de categoría en el menú central reduce la grilla a
  los productos coincidentes sin recargar la pantalla ni perder la orden en construcción.
- **SC-004**: El 100% del comportamiento de validación de pago QR, cobro manual, cálculo de cambio,
  envío a cocina, facturación y cierre de mesa ya implementado (spec 028) permanece disponible e
  inalterado tras el rediseño.
- **SC-005**: El usuario puede ocultar o volver a mostrar el menú de navegación global en una sola
  interacción, sin perder el estado de la orden que esté viendo en la Terminal de Mesas.

## Out of Scope

- Cualquier cambio al cálculo de impuestos: la línea "Impuesto" se muestra siempre en $0; no se
  reabre la decisión A-41 (spec 011) ni se implementa un cálculo real de impuesto.
- **Creación de órdenes de tipo Domicilio**, el campo/atributo de backend que las clasificaría, y el
  flujo de "venta de mostrador" (spec 011) que las soportaría — ninguno existe hoy en el código
  (descubierto durante `/speckit-plan`); se especifican y construyen en una spec futura independiente.
  Esta spec solo agrega el filtro visual "Domicilios" (vacío) a la franja de órdenes.
- Logística de domicilios: asignación de repartidor, zonas de cobertura, seguimiento de entrega,
  dirección estructurada o costos de envío.
- Tipos de orden adicionales mostrados en la imagen de referencia pero no solicitados (por ejemplo,
  "Take Away" o "Dine In" como categorías separadas de "Mesa"): no se agregan en esta spec.
- Cambios al motor de facturación, promociones, combos, validación de pago QR o cierre de mesa
  (specs 010, 011, 012, 013, 024, 026, 028, 029) — todos se reutilizan sin modificación de
  comportamiento.
- Definir mockups pixel a pixel, paleta de colores exacta o tipografía — esta spec define
  distribución y comportamiento; el detalle visual final se resuelve en `/speckit-plan`.

## Assumptions

- **El filtro "Domicilios" es puramente visual en esta spec** (listado siempre vacío) porque, según se
  descubrió al investigar el código durante `/speckit-plan`, no existe hoy ninguna vía de backend ni
  de frontend para crear una orden de ese tipo; construirla mezclaría nueva funcionalidad, una
  migración de datos y el cambio de layout en un mismo incremento (Principio VI de la Constitución).
  Se especifica en una spec futura independiente.
- **El botón de "abrir/cerrar sidebar" controla el menú de navegación global del panel administrativo**
  (el mismo que existe hoy fuera de la Terminal de Mesas), no un panel interno de esta pantalla —
  aclarado explícitamente por el usuario durante esta especificación.
- **La franja superior de "Órdenes Recientes" reemplaza visualmente la grilla de mesas por estado**,
  pero toda mesa libre sigue siendo accesible desde esa misma franja para iniciar una orden nueva,
  reutilizando el mecanismo ya implementado en spec 028 (Historia 2).
- **Los botones de carrusel desplazan el listado por un tramo fijo de tarjetas** (no una tarjeta a la
  vez), de forma análoga a los controles de desplazamiento ya usados en otras listas horizontales de la
  imagen de referencia; el detalle exacto del tramo se resuelve en `/speckit-plan`.
- **La búsqueda por nombre y el filtro por categoría del menú central son funcionalidad nueva
  solicitada explícitamente** (no existían en el panel de catálogo superpuesto anterior), pero se
  limitan a filtrar la lista ya cargada de productos, sin cambiar cómo se calculan precios,
  disponibilidad o stock.
- **Los tipos de orden "Take Away" y "Dine In" que aparecen en la imagen de referencia no se
  implementan** porque no fueron solicitados.
