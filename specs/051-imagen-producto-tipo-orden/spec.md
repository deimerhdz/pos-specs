# Feature Specification: Imagen de producto en el catálogo y organización del tipo de orden — creación de orden manual

**Feature Branch**: `051-imagen-producto-tipo-orden`

**Created**: 2026-08-28

**Status**: Draft

**Naturaleza de esta spec**: **spec de mejora de experiencia sobre una pantalla ya existente**,
no una funcionalidad nueva desde cero. Igual que las specs
[036](../036-terminal-mesas-rediseno-layout/spec.md),
[045](../045-simplificacion-terminal-mesas/spec.md),
[048](../048-pestanas-pago-pendiente-pedido/spec.md) y
[049](../049-rediseno-panel-pedido-mesa/spec.md), cita nombres de archivo y componente del código
actual (`pos-heladeria`) porque son el contrato observable que se está ajustando, no una fuga de
detalles de implementación.

**Alcance concreto sobre la pantalla actual**: la pantalla de creación de orden manual
(`manual-order-page.component.ts`, dentro de la Terminal de Mesas) hoy:

- Muestra la grilla de catálogo de productos (líneas 124–141) con solo nombre y precio "Desde $",
  sin imagen, aunque el producto ya tiene el campo `image_url` disponible (el modal de detalle de
  producto que se abre al seleccionar uno, `product-select.component.ts`, ya usa ese mismo campo y
  sí muestra la imagen cuando existe).
- Muestra un encabezado "Tipo de Orden" (línea 44) sobre la fila de pestañas "🍽️ En Mesa" /
  "🛍️ Para Llevar" / "🛵 Domicilio", y justo debajo, sin ningún título propio, el listado
  horizontal de mesas (líneas 74–93) que aparece cuando "En Mesa" está seleccionado. El único
  encabezado visible del bloque ("Tipo de Orden") queda leyéndose como si describiera solo las
  pestañas, no el listado de mesas que viene después.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-28. Es un ajuste de visualización y de organización
visual sobre una pantalla ya existente — no reabre ninguna regla de negocio de precio, inventario,
disponibilidad de tipos de orden ("Para Llevar" y "Domicilio" siguen deshabilitados, sin cambios)
ni facturación; no aplica una nueva entrada en `registro-de-anomalias.md`.

**Input**: User description (verbatim): "cuando creo una orden manual, quiero que se visualize la
imagen del producto, en la tarjeta del catalogo y en el detalle cuando se selecciona y quiero que
las opciones del tipo de orden se muestren de forma organizada con el titulo en la parte superior
del listado de mesas"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ver la imagen del producto en la tarjeta del catálogo (Priority: P1)

Un mesero está creando una orden manual y recorre la grilla de productos del catálogo. Hoy cada
tarjeta solo muestra el nombre y el precio, por lo que debe adivinar o recordar de memoria a qué
producto corresponde cada tarjeta. Al agregar la imagen del producto a cada tarjeta del catálogo,
el mesero puede reconocer visualmente el producto sin abrir su detalle.

**Why this priority**: es el cambio explícitamente priorizado primero por el usuario y el que
tiene mayor impacto en la velocidad de selección de productos durante la atención al cliente.

**Independent Test**: puede probarse por completo abriendo la pantalla de creación de orden
manual, seleccionando cualquier tipo de orden, y verificando que las tarjetas de productos con
imagen configurada la muestren, sin necesidad de tocar ninguna otra parte de la pantalla.

**Acceptance Scenarios**:

1. **Given** la pantalla de creación de orden manual está abierta y el catálogo de productos está
   visible, **When** un producto listado tiene una imagen configurada, **Then** la tarjeta de ese
   producto en el catálogo muestra dicha imagen.
2. **Given** la pantalla de creación de orden manual está abierta, **When** un producto listado no
   tiene ninguna imagen configurada, **Then** la tarjeta de ese producto muestra un estado visual
   neutro (sin imagen rota ni espacio vacío inconsistente) y sigue mostrando su nombre y precio con
   claridad.

---

### User Story 2 - Ver la imagen del producto en el detalle al seleccionarlo (Priority: P2)

Un mesero toca una tarjeta del catálogo para configurar las opciones del producto antes de
agregarlo a la orden. La vista de detalle/opciones que se abre debe seguir mostrando la imagen del
producto (comportamiento que ya existe hoy) para que, junto con el cambio de la Historia 1, la
identificación visual del producto sea consistente en todo el flujo de selección.

**Why this priority**: es el segundo punto pedido por el usuario; hoy ya funciona, por lo que el
valor de esta historia es de verificación y no regresión al tocar la tarjeta del catálogo, más que
de construcción de algo nuevo.

**Independent Test**: puede probarse por completo seleccionando, desde el catálogo de la orden
manual, un producto con imagen configurada y confirmando que la vista de detalle/opciones que se
abre la muestra, sin depender de los cambios de la Historia 1 ni de la Historia 3.

**Acceptance Scenarios**:

1. **Given** un producto con imagen configurada, **When** el mesero lo selecciona desde el
   catálogo de la orden manual, **Then** la vista de detalle/opciones de ese producto muestra su
   imagen.
2. **Given** un producto sin imagen configurada, **When** el mesero lo selecciona desde el catálogo
   de la orden manual, **Then** la vista de detalle/opciones se muestra correctamente sin imagen
   rota ni espacio vacío inconsistente.

---

### User Story 3 - Título organizado sobre el listado de mesas (Priority: P3)

Un mesero está creando una orden manual con el tipo de orden "En Mesa" seleccionado. Bajo las
pestañas de tipo de orden aparece un listado horizontal de mesas, pero ese listado no tiene ningún
título propio, por lo que la sección se percibe desorganizada: no queda claro, a simple vista, que
esa fila de botones es el listado de mesas disponibles y no parte de otra cosa. Al agregar un
título propio en la parte superior del listado de mesas, el mesero identifica de inmediato qué está
viendo.

**Why this priority**: es el tercer punto pedido por el usuario; mejora la claridad de la pantalla
pero no bloquea la creación de la orden, ya que el listado de mesas ya es funcional hoy.

**Independent Test**: puede probarse por completo abriendo la pantalla de creación de orden manual,
seleccionando el tipo de orden "En Mesa", y verificando que el listado de mesas muestra un título
propio inmediatamente encima de él, distinguible del encabezado de las pestañas de tipo de orden.

**Acceptance Scenarios**:

1. **Given** la pantalla de creación de orden manual está abierta, **When** el tipo de orden
   "En Mesa" está seleccionado, **Then** aparece un título encima del listado de mesas que lo
   identifica claramente como tal, distinto del encabezado que agrupa las opciones de tipo de
   orden.
2. **Given** el listado de mesas está visible con su título, **When** el mesero cambia entre las
   pestañas de tipo de orden (incluyendo las deshabilitadas), **Then** el encabezado de las
   opciones de tipo de orden y el título del listado de mesas se mantienen visualmente separados y
   organizados, sin superponerse ni generar ambigüedad sobre a cuál sección pertenece cada uno.

---

### Edge Cases

- ¿Qué pasa cuando un producto no tiene imagen configurada, tanto en la tarjeta del catálogo como
  en el detalle? El sistema debe mostrar un estado visual neutro consistente, sin imagen rota ni
  layout inconsistente entre tarjetas con y sin imagen.
- ¿Qué pasa con el listado de mesas cuando el tipo de orden seleccionado es "Para Llevar" o
  "Domicilio" (ambos deshabilitados hoy)? El listado de mesas y su nuevo título no deben mostrarse,
  igual que hoy el listado de mesas ya no se muestra cuando esas opciones son las seleccionadas.
- ¿Qué pasa si el catálogo tiene muchos productos con imágenes pesadas? La carga de imágenes no
  debe bloquear la interacción con el catálogo ni degradar perceptiblemente la velocidad de
  selección de productos.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE mostrar la imagen del producto en cada tarjeta del catálogo de la
  pantalla de creación de orden manual, cuando el producto tenga una imagen configurada.
- **FR-002**: El sistema DEBE mostrar un estado visual neutro y consistente (sin imagen rota ni
  espacio en blanco desalineado respecto a las tarjetas con imagen) en la tarjeta del catálogo de
  cualquier producto sin imagen configurada.
- **FR-003**: El sistema DEBE continuar mostrando la imagen del producto en la vista de
  detalle/opciones que se abre al seleccionar un producto desde el catálogo de la orden manual,
  preservando el comportamiento ya existente.
- **FR-004**: El sistema DEBE mostrar un título propio inmediatamente encima del listado de mesas
  cuando el tipo de orden "En Mesa" está seleccionado en la pantalla de creación de orden manual.
- **FR-005**: El título del listado de mesas y el encabezado que agrupa las opciones de tipo de
  orden DEBEN presentarse como dos elementos visualmente distinguibles entre sí, de modo que no se
  perciban como un único encabezado compartido.
- **FR-006**: El sistema NO DEBE alterar el comportamiento actual de las opciones de tipo de orden
  ("En Mesa" habilitada, "Para Llevar" y "Domicilio" deshabilitadas), ni la lógica de qué mesas se
  listan o su estado (disponible/ocupada), como parte de este cambio.

### Key Entities *(include if feature involves data)*

- **Producto**: entidad ya existente en el catálogo; ya cuenta con un campo de imagen asociado que
  hoy se usa en la vista de detalle. Esta spec no agrega ni modifica campos del producto, solo
  extiende dónde se muestra la imagen que ya existe.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los productos con imagen configurada muestran dicha imagen en su tarjeta
  del catálogo de la orden manual, sin necesidad de abrir el detalle del producto.
- **SC-002**: El 100% de los productos con imagen configurada muestran dicha imagen en la vista de
  detalle/opciones al ser seleccionados desde el catálogo de la orden manual.
- **SC-003**: En una revisión visual de la pantalla de creación de orden manual con "En Mesa"
  seleccionado, el listado de mesas se identifica correctamente por su título en el 100% de los
  casos, sin confundirse con el encabezado de tipo de orden.
- **SC-004**: Ningún producto sin imagen configurada presenta una imagen rota o un espacio vacío
  visualmente inconsistente en la tarjeta del catálogo ni en el detalle.

## Assumptions

- El campo de imagen del producto ya existe en el modelo de datos y ya se usa en al menos dos
  puntos del sistema (detalle de producto en la orden manual y catálogo del menú público por QR);
  esta spec no requiere ningún cambio de modelo de datos ni migración.
- Para el estado sin imagen, se sigue el mismo criterio visual ya establecido en el catálogo del
  menú público por QR (un ícono neutro de "sin imagen" en el mismo espacio que ocuparía la
  fotografía), en vez de introducir un patrón visual nuevo.
- "El título en la parte superior del listado de mesas" se interpreta como un encabezado propio y
  distinguible para esa sección, separado del encabezado "Tipo de Orden" que agrupa las pestañas;
  no se interpreta como un pedido de reordenar la posición relativa de las pestañas respecto al
  listado de mesas.
- No se solicita ningún cambio a los tipos de orden "Para Llevar" y "Domicilio" (siguen
  deshabilitados) ni a la lógica de qué mesas se muestran o su estado.
