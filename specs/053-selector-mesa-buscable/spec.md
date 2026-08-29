# Feature Specification: Selector de mesa buscable en la creación de orden manual

**Feature Branch**: `053-selector-mesa-buscable`

**Created**: 2026-08-28

**Status**: Draft

**Naturaleza de esta spec**: **spec de mejora de experiencia sobre una pantalla ya existente**,
igual que las dos specs inmediatamente anteriores sobre la misma pantalla
(`manual-order-page.component.ts`): [051](../051-imagen-producto-tipo-orden/spec.md) (imagen de
producto + título "Mesas") y [052](../052-panel-derecho-orden-manual/spec.md) (unificación de
controles en el panel derecho). Amplía, no reemplaza, ninguna de las dos — solo cambia cómo se
elige la mesa dentro del bloque "Mesas" que 052 ya dejó en el panel derecho.

**Alcance concreto sobre la pantalla actual**: tras 052, el bloque "Mesas" del panel derecho
muestra una rejilla de 4 columnas (`grid grid-cols-4 gap-2`) con un botón por mesa (`M{{ t.number
}}` + su estado), donde las mesas ocupadas quedan deshabilitadas (salvo la ya seleccionada). Con
pocas mesas esto es cómodo, pero no ofrece ninguna forma de buscar una mesa por nombre — el mesero
tiene que ubicarla visualmente en la rejilla.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-28. Es un cambio de control de UI sobre una pantalla ya
existente — no reabre ninguna regla de negocio de disponibilidad de mesas, precio ni facturación;
no aplica una nueva entrada en `registro-de-anomalias.md`.

**Input**: User description (verbatim): "me gustaria que remplazaras la seleccion de mesas de la
vista de orden manual por un componente select, con la opcion de buscar por nombre, el listado
debera mostrar el nombre de la mesa y el estado"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Elegir una mesa desde un select buscable (Priority: P1)

Un mesero abre la creación de orden manual. Hoy debe ubicar visualmente la mesa deseada en una
rejilla de botones. Con un select buscable, puede en cambio abrir un desplegable, escribir parte
del nombre de la mesa y elegirla de una lista filtrada — más rápido cuando hay muchas mesas o el
mesero ya sabe cuál número busca.

**Why this priority**: es el cambio central pedido por el usuario; sin él, el resto de la historia
(mostrar nombre y estado en el listado) no tiene dónde vivir.

**Independent Test**: puede probarse por completo abriendo la creación de orden manual, abriendo el
select de mesas, escribiendo parte del nombre de una mesa y confirmando que la selecciona
correctamente, con el mismo efecto que tenía elegirla desde la rejilla anterior.

**Acceptance Scenarios**:

1. **Given** la pantalla de creación de orden manual está abierta, **When** el mesero observa el
   bloque "Mesas" del panel derecho, **Then** ve un control tipo select (no una rejilla de botones)
   para elegir la mesa.
2. **Given** el select de mesas está cerrado, **When** el mesero hace clic sobre él, **Then** se
   abre un listado con un campo de búsqueda y todas las mesas disponibles para este tenant.
3. **Given** el listado de mesas está abierto, **When** el mesero escribe parte del nombre de una
   mesa, **Then** el listado se filtra a las mesas cuyo nombre coincide, sin distinguir mayúsculas
   ni tildes.
4. **Given** el listado filtrado muestra la mesa buscada, **When** el mesero la selecciona, **Then**
   la mesa queda elegida con el mismo efecto que tenía seleccionarla desde la rejilla anterior (la
   sesión de armado de pedido pasa a esa mesa).

---

### User Story 2 - Ver el nombre y el estado de cada mesa en el listado (Priority: P1)

Un mesero abre el select de mesas para elegir una. Necesita ver, para cada mesa del listado, tanto
su nombre como su estado actual (libre, ocupada, etc.) para no intentar elegir una que ya está en
uso.

**Why this priority**: sin esta información el select sería un downgrade respecto a la rejilla
anterior, que ya mostraba el estado de cada mesa — es igual de crítico que la Historia 1, con la
que se implementa en conjunto.

**Independent Test**: puede probarse por completo abriendo el select de mesas y verificando que
cada fila del listado muestra el nombre de la mesa y su estado, y que las mesas ocupadas no se
pueden seleccionar.

**Acceptance Scenarios**:

1. **Given** el listado de mesas está abierto, **When** el mesero lo observa, **Then** cada fila
   muestra el nombre de la mesa y su estado actual (p. ej. "Libre", "Ocupada").
2. **Given** una mesa está ocupada (y no es la mesa ya seleccionada), **When** el mesero la ve en el
   listado, **Then** sigue visible con su nombre y estado, pero no se puede seleccionar — mismo
   comportamiento que tenía deshabilitada en la rejilla anterior.
3. **Given** la mesa actualmente seleccionada aparece en el listado, **When** el mesero la ve,
   **Then** sigue siendo seleccionable (para poder confirmarla sin cambiarla), igual que en la
   rejilla anterior.

---

### Edge Cases

- ¿Qué pasa si la búsqueda no coincide con ninguna mesa? El listado debe mostrar un estado vacío
  claro, sin quedar en blanco sin explicación.
- ¿Qué pasa con las mesas ocupadas dentro del listado filtrado? Siguen las mismas reglas de
  disponibilidad que en la rejilla anterior: visibles, pero no seleccionables (salvo que sea la
  mesa ya seleccionada) — este cambio no reabre esa regla de negocio, solo cambia el control visual
  que la aplica.
- ¿Qué pasa si el mesero busca por el estado (p. ej. "Libre") en vez de por el nombre? Fuera de
  alcance garantizar ese caso — el pedido explícito del usuario es "buscar por nombre"; si el
  estado queda visible en el mismo texto de la fila y la búsqueda también lo alcanza a filtrar, es
  un efecto secundario aceptable, no un requisito de esta spec.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE reemplazar la rejilla de botones de selección de mesa por un control
  tipo select con buscador, en la pantalla de creación de orden manual.
- **FR-002**: El select de mesas DEBE permitir filtrar el listado escribiendo parte del nombre de
  la mesa, sin distinguir mayúsculas/minúsculas ni tildes.
- **FR-003**: Cada fila del listado del select DEBE mostrar el número de mesa, su nombre
  personalizado cuando exista, y su estado actual — el número nunca se omite ni se reemplaza por el
  nombre personalizado.
- **FR-004**: Las mesas no disponibles para selección (ocupadas y no seleccionadas) DEBEN seguir
  siendo visibles en el listado, mostrando su nombre y estado, pero DEBEN permanecer
  no-seleccionables — mismo criterio que existía en la rejilla que se reemplaza.
- **FR-005**: Seleccionar una mesa desde el nuevo select DEBE producir exactamente el mismo efecto
  que seleccionarla desde la rejilla anterior (cambiar la mesa de la sesión de armado de pedido).
- **FR-006**: El sistema NO DEBE cambiar ninguna otra regla de negocio de disponibilidad de mesas,
  tipo de orden, carrito o confirmación de pedido como parte de este cambio.

### Key Entities *(include if feature involves data)*

- Ninguna entidad de datos nueva ni modificada — esta spec cambia el control de UI usado para
  seleccionar una mesa ya existente, sin tocar el modelo de mesa, orden ni producto.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un mesero puede localizar y seleccionar una mesa específica escribiendo parte de su
  nombre, sin tener que ubicarla visualmente entre todas las demás.
- **SC-002**: El 100% de las mesas configuradas para el tenant siguen siendo visibles (con nombre y
  estado) desde el nuevo select, filtradas o no.
- **SC-003**: El 0% de los intentos de seleccionar una mesa ocupada (que no sea la ya seleccionada)
  tiene efecto — se mantiene el mismo bloqueo que existía en la rejilla anterior.
- **SC-004**: Elegir una mesa desde el nuevo select produce el mismo resultado observable (mesa
  activa en la sesión de armado de pedido) que elegirla desde la rejilla que reemplaza, el 100% de
  las veces.

## Assumptions

- "El nombre de la mesa" se interpreta como el número de mesa que ya usa el resto de la pantalla
  para referirse a una mesa (p. ej. "Mesa 3"), seguido del nombre personalizado de la mesa
  (`Table.name`) cuando esté configurado — el número nunca se reemplaza por el nombre personalizado,
  solo se le agrega. No se introduce ningún campo de datos nuevo.
- El componente de select con buscador ya existente en el proyecto (usado hoy en los formularios de
  producto, insumos, compras y grupos de opciones) se reutiliza para este selector, en vez de
  construir uno nuevo desde cero — la decisión técnica exacta de cómo mostrar "nombre + estado"
  dentro de ese componente compartido se resuelve en la fase de planeación técnica.
- No se solicita ningún cambio a la disponibilidad de mesas en sí (qué mesas están libres/ocupadas)
  ni a los demás controles de la pantalla (tipo de orden, catálogo, carrito, confirmación).
