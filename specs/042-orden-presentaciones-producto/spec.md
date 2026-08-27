# Feature Specification: Orden de Presentaciones de un Producto

**Feature Branch**: `042-orden-presentaciones-producto`

**Created**: 2026-08-27

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad **nueva** de gestión de catálogo (fase de evolución
funcional, Principio I de la [Constitución](../../.specify/memory/constitution.md)). No reabre las
reglas de creación, edición o eliminación de presentaciones ya definidas en la
[spec 002](../002-catalogo-productos-variantes-y-precios/spec.md) (SKU automático, precio,
soft-delete, unicidad de nombre) — esta spec agrega un atributo nuevo (el orden de visualización) y
una forma de modificarlo, sin cambiar ninguna de esas reglas existentes. Tampoco reabre las reglas
de precio/descuento/disponibilidad del Menú QR definidas en la
[spec 007](../007-menu-carrito-qr/spec.md) — únicamente agrega una regla nueva sobre el **orden**
en que esas presentaciones ya vendibles se listan en el detalle del producto.

**Autorización de negocio (Principio I de la [Constitución](../../.specify/memory/constitution.md))**:
solicitado directamente por el dueño/desarrollador del proyecto el 2026-08-27, junto con una imagen
de referencia del componente de listado de presentaciones en el formulario de productos.

**Input**: User description: "necesito establecer el orden en que se muestran las presentaciones o
variantes de un producto en el formulario, actualmente se muestran en el mismo orden en que se
crean, pero me gustaría implementar una forma de cambiar ese orden y que de acuerdo a ese orden se
muestre en los detalles del producto en el menú QR. Adjunto la imagen del componente: este debe
permitir arrastrar las presentaciones y ordenarlas de mayor a menor de acuerdo al número de la
lista, sin perder la funcionalidad de eliminar, editar o agregar."

## Clarifications

### Session 2026-08-27

- Q: Mientras el administrador arrastra una fila (antes de presionar "Guardar"), ¿el número de
  orden junto a cada presentación debe actualizarse de inmediato como retroalimentación visual
  (aunque el guardado real ocurra después), o debe permanecer congelado en su último valor guardado
  hasta que se presione "Guardar"? → A: Las filas se reordenan y el número se actualiza de inmediato
  como retroalimentación visual/local; nada se envía al backend hasta presionar "Guardar" — el
  arrastre en sí nunca persiste nada por sí solo.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Reordenar presentaciones arrastrándolas en el formulario (Priority: P1)

Un administrador abre el formulario de edición de un producto que ya tiene varias presentaciones
(por ejemplo «Balde 24 onz», «Extra grande 20 onz», «Grande 16onz», «Mediano 12onz», «Pequeño 8onz»,
«Litro 32 onz», listadas hoy en el orden en que se crearon). El administrador arrastra una fila de
la lista y la suelta en una nueva posición. La lista se reacomoda de inmediato y el número de orden
de cada fila se actualiza para reflejar la nueva posición (1, 2, 3...) — como retroalimentación
visual del formulario, sin que esto persista nada todavía. El nuevo orden solo queda guardado
cuando el administrador presiona "Guardar" en el formulario del producto, igual que el resto de los
cambios en memoria (nombre, precio, manejo de inventario).

**Why this priority**: es el problema central que reportó el usuario — hoy el único orden posible
es el de creación, y no hay forma de destacar, por ejemplo, la presentación más vendida o de mayor
margen sin recrear las demás presentaciones desde cero.

**Independent Test**: se puede probar por completo abriendo el formulario de un producto con al
menos tres presentaciones, arrastrando una fila a otra posición, verificando que el número de orden
de cada fila cambia de inmediato en el formulario (sin necesidad de guardar para verlo), y que solo
al presionar "Guardar" y recargar el formulario el nuevo orden se mantiene — si se recarga o se sale
del formulario sin guardar, el orden vuelve a ser el último guardado.

**Acceptance Scenarios**:

1. **Given** un producto con presentaciones numeradas 1 a 6 en su orden de creación, **When** el
   administrador arrastra la presentación en la posición 3 («Grande 16onz») hasta la posición 1,
   **Then** esa presentación pasa a mostrarse con el número 1 de inmediato en el formulario, y las
   demás se renumeran de forma consecutiva (1 a 6, sin huecos ni números repetidos) para reflejar su
   nueva posición relativa — todo esto antes de guardar, únicamente como vista previa en el
   formulario.
2. **Given** un reordenamiento recién realizado, **When** el administrador presiona "Guardar" y
   luego recarga el formulario o vuelve a abrir el producto más tarde, **Then** la lista se muestra
   en el nuevo orden guardado, no en el orden original de creación.
3. **Given** un reordenamiento recién realizado que **todavía no se ha guardado**, **When** el
   administrador recarga el formulario o navega fuera sin presionar "Guardar", **Then** el orden
   vuelve a mostrarse tal como estaba guardado antes del arrastre — el reordenamiento sin guardar no
   persiste.
4. **Given** un producto con una sola presentación, **When** el administrador intenta arrastrarla,
   **Then** no ocurre ningún cambio visible ni error — no hay otra posición posible.

---

### User Story 2 - El orden definido se refleja en el detalle del producto en el Menú QR (Priority: P1)

Un comensal escanea el QR de su mesa y abre el detalle de un producto que tiene varias
presentaciones. El detalle muestra las presentaciones en el mismo orden que el administrador definió
en el formulario, no en el orden en que fueron creadas.

**Why this priority**: es la razón de negocio detrás de poder reordenar — de nada sirve reordenar en
el formulario si el comensal, que es quien realmente elige y compra, sigue viendo el orden viejo.

**Independent Test**: se puede probar reordenando las presentaciones de un producto en el formulario
y abriendo después el detalle de ese mismo producto en el Menú QR, verificando que el orden visible
coincide exactamente con el definido en el formulario.

**Acceptance Scenarios**:

1. **Given** un producto cuyas presentaciones fueron reordenadas en el formulario, **When** un
   comensal abre el detalle de ese producto en el Menú QR, **Then** las presentaciones se listan en
   el mismo orden guardado desde el formulario.
2. **Given** un producto cuyas presentaciones nunca fueron reordenadas manualmente, **When** un
   comensal abre su detalle en el Menú QR, **Then** el orden mostrado es el mismo que se veía antes
   de esta funcionalidad (orden de creación) — desplegar esta funcionalidad no debe reordenar nada
   por sí solo.

---

### User Story 3 - Reordenar convive con crear, editar y eliminar presentaciones (Priority: P2)

Un administrador sigue creando presentaciones nuevas, editando las existentes (nombre, precio) y
eliminándolas (soft-delete, según spec 002) desde el mismo listado que ahora también se puede
arrastrar. Ninguna de esas acciones se ve afectada por la nueva capacidad de reordenar.

**Why this priority**: reordenar es una capacidad agregada sobre un componente que ya cumple otras
funciones críticas del día a día (dar de alta y mantener el catálogo); si alguna se rompiera al
agregar el arrastre, el costo operativo sería mayor que el beneficio de ordenar.

**Independent Test**: se puede probar ejecutando, sobre el mismo listado ya reordenado al menos una
vez, los flujos existentes de crear, editar y eliminar una presentación, y verificando que se
comportan igual que antes de esta funcionalidad.

**Acceptance Scenarios**:

1. **Given** una lista de presentaciones ya reordenada, **When** el administrador agrega una
   presentación nueva, **Then** esta se agrega al final de la lista, con el siguiente número de
   orden consecutivo disponible, sin alterar el orden de las demás.
2. **Given** una lista de presentaciones ya reordenada, **When** el administrador elimina
   (soft-delete) una presentación intermedia, **Then** las presentaciones restantes conservan su
   orden relativo entre sí, renumeradas de forma consecutiva sin huecos.
3. **Given** una lista de presentaciones ya reordenada, **When** el administrador edita el nombre o
   el precio de una presentación existente, **Then** su posición en el orden no cambia.

---

### Edge Cases

- **Reactivar una presentación desactivada** (soft-delete, spec 002 RN-CAT-09): recupera también el
  orden que tenía asignado antes de desactivarse; se reintegra a la secuencia consecutiva de
  presentaciones activas en la posición que le corresponda.
- **Dos administradores editando el mismo producto al mismo tiempo**: el último reordenamiento
  guardado exitosamente es el que prevalece — no se define un mecanismo de fusión entre ambos
  cambios (igual que el resto de la edición de producto hoy).
- **Producto con muchas presentaciones**: la lista arrastrable debe seguir siendo utilizable con
  scroll, sin que el arrastre se limite solo a las filas visibles en pantalla.
- **Arrastrar y soltar una fila en su misma posición original**: no debe generar ningún cambio ni
  guardar nada.
- **Recargar el formulario o salir sin presionar "Guardar" después de arrastrar**: el reordenamiento
  visto en pantalla se descarta; al volver a abrir el producto, el orden es el último guardado, no
  el que se veía en la vista previa sin guardar (FR-002, FR-003).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El formulario de edición de producto DEBE permitir a un administrador cambiar el
  orden de las presentaciones de ese producto arrastrando una fila de la lista a otra posición.
- **FR-002**: El sistema DEBE persistir el orden resultante por presentación únicamente cuando el
  administrador presiona "Guardar" en el formulario del producto — el arrastre por sí solo, sin
  guardar, NO DEBE persistir ningún cambio de orden. Un orden arrastrado y no guardado NO DEBE
  sobrevivir a recargar el formulario o a salir sin guardar; en ese caso, el orden vuelve a ser el
  último guardado.
- **FR-003**: El número de orden visible junto a cada presentación en el formulario DEBE
  actualizarse de inmediato tras cada arrastre, como retroalimentación visual dentro del formulario
  (reflejando la posición 1, 2, 3... de forma consecutiva y sin huecos ni valores repetidos), **sin
  que esto implique guardarlo todavía** — ese número visible antes de guardar es una vista previa,
  no el valor persistido (FR-002).
- **FR-004**: El detalle de un producto en el Menú QR DEBE listar sus presentaciones en el mismo
  orden guardado desde el formulario de productos.
- **FR-005**: Agregar una presentación nueva a un producto que ya tiene un orden definido NO DEBE
  alterar el orden de las presentaciones existentes; la nueva se agrega al final.
- **FR-006**: Editar los datos de una presentación (nombre, precio, imagen, etc.) NO DEBE alterar
  su posición en el orden.
- **FR-007**: Eliminar (soft-delete, spec 002 `RN-CAT-10`) una presentación NO DEBE dejar huecos en
  la numeración de las presentaciones restantes de ese producto.
- **FR-008**: Reactivar una presentación previamente desactivada (spec 002 `RN-CAT-09`) DEBE
  conservar el orden que tenía asignado antes de desactivarse, reintegrándola a la secuencia
  consecutiva de presentaciones activas.
- **FR-009**: Las presentaciones que ya existen al desplegar esta funcionalidad DEBEN recibir
  automáticamente un orden inicial equivalente a su orden de creación actual, de modo que el orden
  mostrado hoy en el Menú QR no cambie por el solo hecho de introducir esta funcionalidad.
- **FR-010**: El orden es específico de cada producto — reordenar las presentaciones de un producto
  NO DEBE afectar el orden de presentaciones de ningún otro producto.

### Key Entities *(include if feature involves data)*

- **ProductVariant (presentación/variante)**: entidad ya existente (spec 002). Esta spec le agrega
  un atributo de **orden de visualización**, entero, único y consecutivo dentro de las
  presentaciones de un mismo producto, que determina tanto su posición en el listado del formulario
  como en el detalle del producto en el Menú QR.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un administrador reordena las presentaciones de un producto arrastrando, sin usar
  ningún otro control ni pantalla adicional, en menos de 30 segundos para un producto de hasta 10
  presentaciones.
- **SC-002**: El orden mostrado en el detalle del producto en el Menú QR coincide, en el 100% de los
  casos verificados, con el orden definido en el formulario de productos.
- **SC-003**: Crear, editar y eliminar una presentación siguen funcionando sin ninguna regresión
  observable después de introducir el arrastre para reordenar.
- **SC-004**: Al desplegar esta funcionalidad, el orden de presentaciones que ya ven hoy los
  comensales en el Menú QR no cambia para ningún producto existente, hasta que un administrador lo
  reordene explícitamente.

## Assumptions

- **La imagen adjunta corresponde al componente ya existente** de listado de presentaciones dentro
  del formulario de productos (icono de arrastre, ícono de tipo, nombre, editar, eliminar y número
  de orden a la derecha) — esta spec agrega la capacidad de arrastrar esa fila para cambiar dicho
  número, sin rediseñar el resto del componente.
- **El nuevo orden se guarda junto con el resto del formulario, al presionar "Guardar"**, no de
  inmediato al soltar la fila — el formulario de producto ya sigue este patrón para otros cambios en
  memoria (por ejemplo, activar/desactivar el manejo de inventario, spec 027), y el arrastre lo
  sigue por consistencia: soltar la fila solo actualiza la vista y la numeración (FR-003) hasta que
  se guarda el producto.
- **El listado arrastrable del formulario es el de presentaciones activas**; las presentaciones
  desactivadas (soft-delete, spec 002 US5) se muestran en una sección aparte para reactivarlas y no
  participan del arrastre mientras estén desactivadas. Al reactivarse, retoman el orden que tenían
  guardado antes de desactivarse (Edge Cases), sin necesidad de arrastrarlas de nuevo.
- **El alcance de esta spec es el detalle del producto en el Menú QR y el formulario de productos**;
  cualquier otro lugar del sistema que también liste presentaciones (por ejemplo, el listado interno
  de productos o la venta de mostrador) queda fuera de alcance y puede seguir usando su criterio de
  orden actual salvo que una spec futura lo extienda.
