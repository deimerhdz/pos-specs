# Feature Specification: Rediseño de Layout de la Terminal de Mesas — Grilla de Mesas, Pagos por Confirmar y Menú Central

**Feature Branch**: `036-terminal-mesas-rediseno-layout`

**Created**: 2026-08-25

**Status**: Draft

**Naturaleza de esta spec**: ajuste de **layout y distribución** de la Terminal de Mesas (pantalla ya
implementada por [spec 028](../028-terminal-mesas-modo-hibrido/spec.md) y sus correcciones en
[spec 029](../029-correccion-cobro-cierre-mesa/spec.md)), tomando como referencia dos capturas de
prototipo provistas por el usuario (sesión de clarificación 2026-08-26, que actualiza y reemplaza la
imagen de referencia original de esta spec). El comportamiento ya implementado (validación de pago QR,
cobro manual, cálculo de cambio, envío a cocina, facturación, cierre de mesa) **se mantiene sin
cambios** — esta spec reorganiza cómo se distribuyen y presentan esos mismos componentes en pantalla:
la grilla de mesas y sus filtros de ocupación se conservan (spec 028) pero se les agregan pestañas de
tipo de orden ("Mesas"/"Domicilios"/"Para llevar") y una sección "Pagos por confirmar" que agrupa
visualmente los pagos pendientes de revisión ya existentes; el panel central pasa a mostrar la lista de
ítems del pedido con un flujo de "+ Agregar producto" para el catálogo; y se agrega un botón para
colapsar/expandir el menú de navegación global de la aplicación. Las pestañas "Domicilios" y "Para
llevar" se incorporan al layout (Principio VI de la
[Constitución](../../.specify/memory/constitution.md): evolución incremental, sin mezclar clases de
cambio distintas en un mismo incremento) **sin la capacidad de crear órdenes de esos tipos todavía**:
esa creación (que, según se descubrió durante `/speckit-plan`, requeriría un campo nuevo de backend y
un flujo de creación de "venta de mostrador" que hoy no existe en el frontend — ver Clarifications) se
especifica y construye en una spec futura independiente.

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

### Session 2026-08-26 (actualización de prototipos de referencia)

- Q: Los nuevos prototipos muestran la franja superior organizada por mesas (tarjetas "Mesa 1", "Mesa
  2"... con filtros "Todas/Libres/Ocupadas/Pendientes", como en spec 028), no por órdenes con pestañas
  "Todas las Órdenes/Domicilios/Mesas" y barra de tiempo transcurrido, como describía la spec anterior
  (FR-001 a FR-003). ¿Qué organización debe reflejar la spec actualizada para la franja superior? → A:
  Adoptar el diseño de los prototipos — franja de tarjetas de **mesa** con los filtros de ocupación
  "Todas/Libres/Ocupadas/Pendientes" ya existentes (spec 028, FR-014); se abandona el concepto de
  franja de "órdenes" con barra de tiempo transcurrido introducido en la sesión de clarificación
  anterior (2026-08-25).
- Q: El prototipo muestra pestañas de tipo "Mesas / Domicilios / Para llevar" (no "Todas las
  Órdenes"), combinadas con los filtros de ocupación debajo. ¿Se adopta el set de pestañas del
  prototipo, agregando "Para llevar" como tercera pestaña de tipo de orden? → A: Se agrega "Para
  llevar" como tercera pestaña junto a "Mesas" y "Domicilios", pero queda **vacía** en esta spec —
  mismo tratamiento que "Domicilios" (sin flujo de creación nuevo, ya que la vía de "venta de
  mostrador" no existe hoy en el código). Se documenta como asunción que su futura implementación
  reutilizará el mismo flujo que se construya para Domicilio, sin el campo de dirección del cliente
  (aclarado por el usuario). Las pestañas de tipo de orden pasan a ser "Mesas / Domicilios / Para
  llevar" (reemplaza a "Todas las Órdenes/Domicilios/Mesas" de la sesión anterior) y se muestran junto
  con — no en reemplazo de — los filtros de ocupación "Todas/Libres/Ocupadas/Pendientes".
- Q: El prototipo 1 muestra una sección nueva, "Pagos por confirmar", debajo de la grilla de mesas —
  una lista separada de pedidos con pago pendiente de revisión, cada uno con su propio total y botón de
  acción. ¿Se agrega esta sección como parte del rediseño de layout? → A: Sí — se agrega como
  reorganización visual del bloque de validación de pago que ya existe hoy por mesa (spec 028/024):
  agrupa en una lista aparte todos los pedidos con pago pendiente de revisión (efectivo pendiente,
  transferencia por aprobar), sin cambiar la lógica de confirmación ya implementada (mismo botón
  "Confirmar efectivo", mismo estado "Pendiente de revisión"/"Aprobado"). Sigue siendo el mismo dato y
  la misma acción ya implementados; solo cambia dónde se listan en pantalla.
- Q: El prototipo 2 muestra el panel central mostrando la lista de ítems ya agregados al pedido de la
  mesa seleccionada ("Pedido de la mesa", con botón "+ Agregar producto" al final), no una grilla de
  productos con buscador y categorías visible de forma permanente. ¿Dónde vive la grilla de productos
  con buscador/categorías en el layout actualizado? → A: El panel central muestra por defecto la lista
  de ítems del pedido en construcción (no el catálogo); el botón "+ Agregar producto" abre, dentro del
  mismo panel central (sin superponerse a pantalla completa como el catálogo actual), la grilla de
  productos con buscador por nombre y pestañas de categoría. Agregar un producto desde esa grilla
  regresa a la vista de lista de ítems del pedido, ahora con el producto agregado.
- Q: El panel derecho del prototipo 2 incluye un botón "Dividir la cuenta entre varias personas", un
  campo "Facturar a nombre de" y un botón único "Cobrar, Facturar y Enviar a Cocina" que combina tres
  acciones hoy separadas. ¿Son funcionalidad nueva a construir en esta spec, o solo el diseño visual
  del prototipo, manteniendo el comportamiento de cobro ya implementado sin cambios? → B: Solo layout.
  "Dividir la cuenta entre varias personas" se muestra en el panel derecho pero no es funcional en esta
  spec (queda fuera de alcance, ver Out of Scope). "Facturar a nombre de" reutiliza el mismo campo de
  cliente ya existente (spec 011, FR-010) — no es un campo nuevo. El botón "Cobrar, Facturar y Enviar a
  Cocina" agrupa visualmente la misma secuencia de acciones ya implementada hoy (cobrar, facturar,
  enviar a cocina) sin cambiar su lógica, validaciones ni pasos internos.

### Session 2026-08-26 (durante `/speckit-plan`)

- Q: Al investigar el código real (`pos-heladeria`) para planear la implementación, se descubrió que
  "Dividir la cuenta entre varias personas" — que la sesión anterior de clarificación trató como un
  botón sin funcionalidad asociada (solo fidelidad visual al prototipo) — **ya está completamente
  implementado y en uso hoy** en `pos-checkout-panel.component.ts` (botón que abre
  `split-bill-panel.component.ts`, que asigna ítems/unidades por comensal vía `participant_id` y
  alimenta el cobro dividido de `session-bill-panel.component.ts`), construido en spec 010 (sesión de
  mesa, reparto, cierre, barrido). ¿Cómo se ajusta FR-010 y el resto del spec ante este hallazgo? → A:
  Se corrige el spec: "Dividir la cuenta entre varias personas" pasa de "sin funcionalidad en esta
  spec" a **funcionalidad ya existente que se conserva sin cambios de comportamiento** (mismo
  tratamiento que el resto del panel derecho, FR-009) — esta spec solo adapta su disposición visual al
  prototipo, igual que ya hace con el resto del cobro. No amplía el alcance (no hay funcionalidad nueva
  que construir); corrige un supuesto incorrecto de la sesión de clarificación anterior.

## User Scenarios & Testing _(mandatory)_

### User Story 1 - El cajero encuentra cualquier mesa y revisa los pagos pendientes de confirmar desde la parte superior de la pantalla (Priority: P1)

En la parte superior de la pantalla, el cajero ve tres pestañas de tipo de orden — "Mesas",
"Domicilios" y "Para llevar" — y, para la pestaña "Mesas", los mismos filtros de ocupación ya
existentes ("Todas"/"Libres"/"Ocupadas"/"Pendientes", spec 028 FR-014) junto con el buscador de mesa
(F2). Debajo, la grilla de tarjetas de mesa se mantiene con el mismo contenido ya implementado
(identificador, insignia de estado de texto+color, cantidad de productos y total), adaptando solo su
disposición visual a los prototipos de referencia. Debajo de la grilla, una sección "Pagos por
confirmar" agrupa en una lista aparte todos los pedidos con pago pendiente de revisión (efectivo
pendiente de confirmar, transferencia por aprobar), reutilizando exactamente el mismo dato y la misma
acción de confirmación ya implementados hoy por mesa (spec 028/024), solo reorganizados visualmente en
un listado dedicado en lugar de vivir únicamente dentro del panel de cada mesa. Al seleccionar una
tarjeta de mesa (desde la grilla o desde "Pagos por confirmar"), se abre esa orden en el centro y la
derecha exactamente como hoy.

**Why this priority**: es el cambio de mayor visibilidad del rediseño y la puerta de entrada a
cualquier otra acción de la pantalla; sin esta grilla y sin la visibilidad de pagos pendientes, el
resto de la reorganización (menú central, resumen derecho) no tiene desde dónde abrirse.

**Independent Test**: abrir la Terminal de Mesas con mesas en distintos estados de ocupación y con
pagos pendientes de revisión; alternar entre las pestañas "Mesas"/"Domicilios"/"Para llevar" y los
filtros de ocupación; confirmar un pago desde la sección "Pagos por confirmar" y verificar que produce
el mismo efecto que confirmarlo desde el panel de la mesa; seleccionar una tarjeta de mesa y confirmar
que abre la misma orden que hoy se abre al seleccionar una mesa.

**Acceptance Scenarios**:

1. **Given** varias mesas con distintos estados de ocupación, **When** el cajero selecciona un filtro
   de ocupación ("Libres"/"Ocupadas"/"Pendientes"), **Then** ve únicamente las mesas que cumplen ese
   estado, igual que en el comportamiento ya implementado (spec 028, FR-014).
2. **Given** que hoy no existe ninguna vía para crear una orden de tipo Domicilio ni Para Llevar (ver
   Clarifications), **When** el cajero selecciona la pestaña "Domicilios" o "Para llevar", **Then** ve
   un listado vacío con un mensaje claro (no un error ni una pantalla en blanco sin explicación).
3. **Given** uno o más pedidos con pago pendiente de revisión (efectivo o transferencia), **When** el
   cajero mira la sección "Pagos por confirmar", **Then** ve cada pedido con su cliente/mesa, método de
   pago, estado ("Pendiente de revisión"/"Aprobado") y total, igual a la información ya disponible hoy
   en el panel de la mesa.
4. **Given** un pedido con efectivo pendiente de confirmar en la sección "Pagos por confirmar",
   **When** el cajero pulsa "Confirmar efectivo" desde esa sección, **Then** el pedido se confirma con
   exactamente el mismo efecto que confirmarlo desde el panel de la mesa (spec 024/028), sin duplicar
   ni requerir una segunda confirmación.
5. **Given** el listado de mesas visible, **When** el cajero selecciona una tarjeta (desde la grilla o
   desde "Pagos por confirmar"), **Then** el panel central y el panel derecho muestran esa orden con el
   mismo comportamiento ya implementado (bloque de validación de pago QR, constructor de orden manual,
   o resumen de cuenta/cobro, según corresponda).
6. **Given** una mesa libre sin orden activa, **When** el cajero la busca o la selecciona, **Then**
   sigue disponible la misma vía ya implementada para iniciar una orden sobre esa mesa (spec 028,
   Historia 2), solo reubicada visualmente dentro del nuevo layout.
7. **Given** una mesa con pago QR pendiente de revisión, **When** se muestra en la grilla superior,
   **Then** conserva su insignia de estado de texto+color ya existente ("Por confirmar") sin cambios.

---

### User Story 2 - El cajero arma la orden desde la lista de ítems del pedido, agregando productos desde una grilla con búsqueda y categorías (Priority: P1)

El panel central muestra, por defecto, la lista de ítems ya agregados al pedido en construcción
("Pedido de la mesa"), cada ítem con su cantidad, nombre, precio y estado. Un botón "+ Agregar
producto" al final de la lista abre, dentro del mismo panel central (sin superponerse a pantalla
completa como el catálogo actual), una grilla de productos con pestañas de categoría y un campo de
búsqueda por nombre, tal como en los prototipos de referencia. Al seleccionar un producto desde esa
grilla, el panel central vuelve a mostrar la lista de ítems del pedido, ahora actualizada.

**Why this priority**: es el segundo cambio central del rediseño (deprecar el catálogo como panel
superpuesto) y el medio principal para construir cualquier orden manual; sin esto, el resto del
layout no tiene cómo agregar productos.

**Independent Test**: con una orden en construcción, pulsar "+ Agregar producto", escribir un texto en
el buscador y verificar que la grilla se filtra por nombre; seleccionar una categoría y verificar que
se filtra por categoría; agregar un producto y verificar que el panel central regresa a la lista de
ítems del pedido con el producto agregado, y que el resumen de la derecha refleja el mismo
comportamiento ya implementado (control de cantidad, nota, eliminar).

**Acceptance Scenarios**:

1. **Given** un pedido en construcción sin productos agregados aún, **When** el cajero abre la mesa,
   **Then** el panel central muestra la lista de ítems (vacía) junto con el botón "+ Agregar producto".
2. **Given** el panel central mostrando la grilla de productos (tras pulsar "+ Agregar producto"),
   **When** el cajero escribe parte del nombre de un producto en el buscador, **Then** la grilla
   muestra únicamente los productos cuyo nombre coincide, sin recargar ni salir de la pantalla.
3. **Given** la grilla de productos abierta, **When** el cajero selecciona una categoría, **Then** la
   grilla muestra únicamente los productos de esa categoría; seleccionar "Todos" vuelve a mostrar el
   catálogo completo.
4. **Given** la grilla de productos abierta, **When** el cajero selecciona un producto, **Then** el
   producto se agrega al pedido, el panel central regresa a la lista de ítems (mostrando el producto
   agregado) y el resumen del panel derecho refleja la misma mecánica de cantidad, nota y eliminación
   ya implementada hoy en el constructor de orden.
5. **Given** una orden de origen QR en modo "Resumen de Cuenta" (solo lectura), **When** el cajero mira
   el panel central, **Then** ve la lista de ítems de solo lectura y no se ofrece el botón "+ Agregar
   producto", igual que hoy el catálogo no está disponible para órdenes QR de solo lectura.
6. **Given** una búsqueda o filtro de categoría sin resultados dentro de la grilla de "+ Agregar
   producto", **When** el cajero lo aplica, **Then** el panel central muestra un estado vacío claro en
   vez de una grilla en blanco sin explicación.

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

- **El cajero selecciona la pestaña "Domicilios" o "Para llevar"**: no existe hoy ninguna vía para
  crear una orden de esos tipos (ver Clarifications); el sistema muestra el listado vacío con un
  mensaje claro en vez de un error o una pantalla en blanco sin explicación.
- **Un pedido aparece simultáneamente en la grilla de mesas (con su insignia "Por confirmar") y en la
  sección "Pagos por confirmar"**: ambas vistas reflejan el mismo estado subyacente; confirmar el pago
  desde cualquiera de las dos actualiza el estado en ambas sin duplicar la acción ni requerir una
  segunda confirmación.
- **El cajero pulsa "+ Agregar producto" y luego decide no agregar nada**: puede volver a la lista de
  ítems del pedido sin que se pierda ningún ítem ya agregado previamente.
- **El cajero filtra la grilla de productos (abierta desde "+ Agregar producto") por categoría y además
  escribe un término de búsqueda**: ambos filtros se combinan (solo se muestran productos que cumplen
  la categoría Y el texto buscado); si no hay coincidencias, se muestra el estado vacío de la
  Historia 2.
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

- **FR-001**: El sistema DEBE mostrar, en la parte superior de la pantalla, tres pestañas de tipo de
  orden — "Mesas", "Domicilios" y "Para llevar"; seleccionar "Mesas" DEBE mostrar la grilla de tarjetas
  de mesa ya implementada (spec 028), adaptando únicamente su disposición visual a los prototipos de
  referencia.
- **FR-002**: Para la pestaña "Mesas", el sistema DEBE conservar, sin cambios de comportamiento, los
  filtros de ocupación ya existentes ("Todas"/"Libres"/"Ocupadas"/"Pendientes" — spec 028, FR-014) y el
  buscador de mesa (F2), adaptando únicamente su disposición visual.
- **FR-003**: Las pestañas "Domicilios" y "Para llevar" DEBEN mostrarse siempre disponibles pero con un
  listado vacío y un mensaje claro, porque hoy no existe ninguna vía de creación de orden para esos
  tipos (ver Clarifications); ninguna de las dos DEBE bloquear, alterar ni reemplazar el flujo de
  creación de orden de Mesa ya implementado (spec 028, Historia 2).
- **FR-004**: El sistema DEBE agregar una sección "Pagos por confirmar" debajo de la grilla de mesas
  que agrupe, en un listado aparte, todo pedido con pago pendiente de revisión (efectivo pendiente de
  confirmar, transferencia por aprobar), reutilizando exactamente el mismo dato y la misma acción de
  confirmación ya implementados hoy dentro del panel de cada mesa (spec 024/028), sin duplicar ni
  requerir una segunda confirmación.
- **FR-005**: Seleccionar una tarjeta de mesa (desde la grilla o desde "Pagos por confirmar") DEBE
  abrir esa orden en los paneles central y derecho con exactamente el mismo comportamiento ya
  implementado hoy al seleccionar una mesa (bloque de validación de pago QR, constructor de orden
  manual, o resumen de cuenta/cobro, según el estado y origen de la orden — spec 028).
- **FR-006**: El panel central DEBE mostrar, por defecto, la lista de ítems ya agregados al pedido en
  construcción (cantidad, nombre, precio y estado de cada ítem), en lugar de la grilla de catálogo de
  productos.
- **FR-007**: El panel central DEBE ofrecer un botón "+ Agregar producto" que, sin superponerse a
  pantalla completa, abre dentro del mismo panel una grilla de productos con campo de búsqueda por
  nombre y pestañas de filtro por categoría, ambos aplicables de forma combinada; seleccionar un
  producto desde esa grilla DEBE regresar la vista a la lista de ítems del pedido, ahora con el
  producto agregado.
- **FR-008**: Agregar, quitar, ajustar cantidad o anotar un producto desde la grilla central DEBE
  producir exactamente el mismo efecto sobre la orden en construcción que el ya implementado hoy desde
  el catálogo superpuesto (sin cambios en el cálculo de precios, disponibilidad o descuento de
  inventario).
- **FR-009**: El panel derecho DEBE conservar, sin cambios de comportamiento, el resumen y cobro de la
  orden ya implementado (modos "Resumen de Cuenta" y "Terminal POS / Cobro Inmediato" de spec 028),
  adaptando únicamente su disposición visual a los prototipos de referencia, e incluyendo el campo
  "Facturar a nombre de" como el mismo campo de cliente ya existente (spec 011, FR-010) y el botón
  "Cobrar, Facturar y Enviar a Cocina" como agrupación visual de la misma secuencia de acciones ya
  implementada hoy, sin cambiar su lógica, validaciones ni pasos internos.
- **FR-010**: El panel derecho DEBE conservar, sin cambios de comportamiento, el botón y el flujo de
  "Dividir la cuenta entre varias personas" ya implementados hoy (spec 010), adaptando únicamente su
  disposición visual a los prototipos de referencia.
- **FR-011**: El resumen de pago del panel derecho DEBE mostrar una línea de "Impuesto" con valor fijo
  en $0, sin campo editable y sin alterar el cálculo de venta ya vigente (spec 011, decisión A-41).
- **FR-012**: El sistema DEBE ofrecer un botón en la parte superior de la Terminal de Mesas que
  colapsa o expande el menú de navegación global de la aplicación, sin afectar el estado de ninguna
  orden en curso en los paneles central o derecho.
- **FR-013**: El sistema NO DEBE reducir, respecto de la interfaz actual, ninguna información hoy
  disponible sobre el pedido, el pago o la factura (reutiliza spec 026, FR-012 / spec 028, FR-015).

### Key Entities _(include if feature involves data)_

- **Orden**: entidad ya existente (spec 011, spec 016, spec 028); no se le agregan atributos nuevos en
  esta spec. Las pestañas "Domicilios" y "Para llevar" (FR-003) son puramente de interfaz — no existe
  todavía ningún atributo de clasificación de tipo de orden en el modelo de datos que las respalde;
  agregarlo se especifica en una spec futura independiente (ver Clarifications). La sección "Pagos por
  confirmar" (FR-004) no introduce un nuevo estado ni entidad: reutiliza el mismo estado de pago ya
  existente en la Orden.
- **Ítem de Menú** y **Categoría de Menú**: entidades ya existentes en el catálogo; esta spec no
  cambia sus atributos, solo la forma en que se listan y filtran en pantalla (grilla accesible desde
  "+ Agregar producto" en vez de panel superpuesto).

## Success Criteria _(mandatory)_

### Measurable Outcomes

- **SC-001**: El 100% de las mesas con orden activa son localizables desde la grilla superior
  combinando pestaña de tipo y filtro de ocupación en un máximo de dos interacciones (elegir pestaña
  y, si aplica, filtro de ocupación).
- **SC-002**: El cajero puede ver todos los pagos pendientes de revisión (efectivo/transferencia) desde
  una única sección ("Pagos por confirmar") sin tener que abrir cada mesa individualmente para
  detectarlos.
- **SC-003**: El cajero puede agregar un producto a una orden en construcción sin que el catálogo se
  superponga a pantalla completa (0 pantallas superpuestas necesarias, frente al panel de catálogo
  superpuesto que existía antes), regresando a la lista de ítems del pedido tras cada selección.
- **SC-004**: Una búsqueda por nombre o un filtro de categoría dentro de la grilla de "+ Agregar
  producto" reduce la grilla a los productos coincidentes sin recargar la pantalla ni perder los ítems
  ya agregados al pedido.
- **SC-005**: El 100% del comportamiento de validación de pago QR, cobro manual, cálculo de cambio,
  envío a cocina, facturación y cierre de mesa ya implementado (spec 028) permanece disponible e
  inalterado tras el rediseño.
- **SC-006**: El usuario puede ocultar o volver a mostrar el menú de navegación global en una sola
  interacción, sin perder el estado de la orden que esté viendo en la Terminal de Mesas.

## Out of Scope

- Cualquier cambio al cálculo de impuestos: la línea "Impuesto" se muestra siempre en $0; no se
  reabre la decisión A-41 (spec 011) ni se implementa un cálculo real de impuesto.
- **Creación de órdenes de tipo Domicilio y Para Llevar**, el campo/atributo de backend que las
  clasificaría, y el flujo de "venta de mostrador" (spec 011) que las soportaría — ninguno existe hoy
  en el código (descubierto durante `/speckit-plan`); se especifican y construyen en una spec futura
  independiente. Esta spec solo agrega las pestañas visuales "Domicilios" y "Para llevar" (vacías).
- Logística de domicilios: asignación de repartidor, zonas de cobertura, seguimiento de entrega,
  dirección estructurada o costos de envío.
- Cambios al motor de facturación, promociones, combos, validación de pago QR o cierre de mesa
  (specs 010, 011, 012, 013, 024, 026, 028, 029) — todos se reutilizan sin modificación de
  comportamiento.
- Definir mockups pixel a pixel, paleta de colores exacta o tipografía — esta spec define
  distribución y comportamiento; el detalle visual final se resuelve en `/speckit-plan`.

## Assumptions

- **Las pestañas "Domicilios" y "Para llevar" son puramente visuales en esta spec** (listados siempre
  vacíos) porque, según se descubrió al investigar el código durante `/speckit-plan`, no existe hoy
  ninguna vía de backend ni de frontend para crear una orden de esos tipos; construirlas mezclaría
  nueva funcionalidad, una migración de datos y el cambio de layout en un mismo incremento (Principio
  VI de la Constitución). Se especifican en una spec futura independiente; cuando se construya, "Para
  llevar" reutilizará el mismo flujo que se defina para "Domicilio" pero sin el campo de dirección del
  cliente (aclarado por el usuario durante esta especificación).
- **El botón de "abrir/cerrar sidebar" controla el menú de navegación global del panel administrativo**
  (el mismo que existe hoy fuera de la Terminal de Mesas), no un panel interno de esta pantalla —
  aclarado explícitamente por el usuario durante esta especificación.
- **La grilla de mesas y sus filtros de ocupación (Todas/Libres/Ocupadas/Pendientes) se mantienen sin
  cambios de comportamiento** respecto de spec 028; el rediseño solo ajusta su disposición visual y
  agrega las pestañas de tipo de orden por encima. No se agregan controles de carrusel nuevos: el
  desplazamiento/scroll de la grilla reutiliza el mismo mecanismo ya implementado.
- **La sección "Pagos por confirmar" reutiliza el mismo dato y la misma acción de confirmación de pago
  ya implementados por mesa** (spec 024/028); no introduce un nuevo estado de orden ni una segunda
  fuente de verdad.
- **La búsqueda por nombre y el filtro por categoría del menú central son funcionalidad nueva
  solicitada explícitamente** (no existían en el panel de catálogo superpuesto anterior), ahora
  accesibles desde el botón "+ Agregar producto" en el panel central, limitados a filtrar la lista ya
  cargada de productos sin cambiar cómo se calculan precios, disponibilidad o stock.
- **"Dividir la cuenta entre varias personas" es funcionalidad ya implementada hoy** (spec 010,
  `split-bill-panel.component.ts`/`session-bill-panel.component.ts`) — se descubrió durante
  `/speckit-plan` que el supuesto inicial de la clarificación (botón sin funcionalidad) era incorrecto;
  esta spec solo reubica visualmente el botón junto con el resto del panel derecho, sin cambiar su
  comportamiento. El campo "Facturar a nombre de" reutiliza el campo de cliente ya existente (spec
  011, FR-010) y no es un dato nuevo.
