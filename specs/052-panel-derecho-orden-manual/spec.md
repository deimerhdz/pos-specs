# Feature Specification: Panel derecho unificado — tipo de orden, mesas y pedido en la creación de orden manual

**Feature Branch**: `052-panel-derecho-orden-manual`

**Created**: 2026-08-28

**Status**: Draft

**Naturaleza de esta spec**: **spec de mejora de experiencia sobre una pantalla ya existente**,
igual que la spec inmediatamente anterior,
[051](../051-imagen-producto-tipo-orden/spec.md), sobre la misma pantalla
(`manual-order-page.component.ts`). Amplía, no reemplaza, spec 051 — spec 051 dejó el título
"Mesas" sobre el listado de mesas dentro de la barra superior; esta spec traslada ese mismo
listado (junto con las pestañas de "Tipo de Orden") desde la barra superior hacia el panel derecho,
antes de que 051 haya sido desplegada a producción (051 sigue sin commitear al momento de esta
spec).

**Alcance concreto sobre la pantalla actual**: tras 051, la pantalla de creación de orden manual
tiene: una barra superior de ancho completo (`manual-order-page.component.ts:35-96`) con el botón
"← Volver a la Terminal", el encabezado "Tipo de Orden" y sus tres pestañas (línea 45-70), y debajo
el título "Mesas" con el listado horizontal de mesas (línea 75-95); y, en una fila aparte
(línea 99-208), el catálogo de productos a la izquierda (`flex-1`, línea 100) y el panel "Nueva
orden" a la derecha (`w-full sm:w-[320px]`, línea 155) con el carrito y el resumen de totales.
Elegir un tipo de orden o una mesa exige mover el mouse hasta la barra superior (que se extiende de
extremo a extremo); elegir productos y ver/confirmar el pedido ocurre en la fila de abajo, dividida
entre el extremo izquierdo (catálogo) y el extremo derecho (pedido, en un panel de solo 320px).

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-28, sobre una captura de pantalla del estado actual
(post-051), junto con dos decisiones de diseño resueltas antes de esta spec (ver Clarifications).
Es reordenamiento/ensanchamiento visual sobre una pantalla ya existente — no reabre ninguna regla
de negocio de precio, inventario, disponibilidad de tipos de orden ni facturación; no aplica una
nueva entrada en `registro-de-anomalias.md`.

**Input**: User description (verbatim): "a nivel de diseño me gustaria que le dieras un poco mas de
valor al ancho del valor de panel de nueva orden para que no se tan pequeño, y para mejorar la
experiencia de usuario me gustaria que unificaras la opciones de seleccion en la parte derecha para
que el usuario no este desplazando el maus de un lado a otro"

## Clarifications

### Sesión 2026-08-28

- P: ¿Cómo unificar "Tipo de Orden" y "Mesas" en la parte derecha? → R: Ambos controles se
  trasladan al panel derecho, apilados justo arriba de "Nueva orden" — ese panel pasa a ser el
  único lugar donde se configura y arma el pedido completo (tipo de orden, mesa, carrito, total,
  confirmar). La barra superior queda reducida a solo "← Volver a la Terminal". El catálogo ocupa
  todo el ancho izquierdo restante.
- P: ¿Cómo mostrar el listado de mesas dentro de un panel más angosto que la barra superior
  original (que hoy usa scroll horizontal para 8 mesas)? → R: se resuelve en la fase de
  planeación técnica (research.md), sin cambiar el criterio de qué mesas se listan, cuáles están
  deshabilitadas, ni el comportamiento de selección — solo cómo se acomodan visualmente dentro del
  panel angosto.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Configurar tipo de orden, mesa y pedido sin cruzar la pantalla (Priority: P1)

Un mesero abre la creación de orden manual. Hoy, para elegir el tipo de orden y la mesa debe mirar
y hacer clic en la barra superior (que se extiende de un extremo al otro de la pantalla), y luego
mover el mouse hasta el catálogo (izquierda) o el panel de pedido (derecha, angosto) para seguir
armando la orden. Al mover "Tipo de Orden" y "Mesas" al panel derecho, junto con el carrito y el
botón de confirmar, todas las decisiones de configuración y armado del pedido quedan en una sola
columna, sin que el mesero tenga que desplazar el mouse de un extremo a otro de la pantalla entre
un paso y el siguiente.

**Why this priority**: es el cambio central pedido por el usuario y el que más reduce el
desplazamiento de mouse durante la atención al cliente.

**Independent Test**: puede probarse por completo abriendo la creación de orden manual y
verificando que "Tipo de Orden", "Mesas", "Nueva orden" (carrito), el resumen de totales y
"Confirmar y Enviar" están todos dentro del mismo panel derecho, sin necesidad de tocar el
catálogo de la izquierda.

**Acceptance Scenarios**:

1. **Given** la pantalla de creación de orden manual está abierta, **When** el mesero la observa,
   **Then** la barra superior solo contiene "← Volver a la Terminal" (sin "Tipo de Orden" ni
   "Mesas"), y el panel derecho contiene, en este orden de arriba hacia abajo: "Tipo de Orden" con
   sus tres opciones, "Mesas" con el listado de mesas, y luego "Nueva orden" con el carrito, el
   resumen de totales y "Confirmar y Enviar".
2. **Given** el panel derecho está visible, **When** el mesero elige un tipo de orden, luego una
   mesa, luego agrega un producto al carrito, **Then** puede completar las tres acciones sin que el
   mouse salga del panel derecho (salvo para elegir el producto en el catálogo).
3. **Given** la mesa seleccionada cambia desde el panel derecho, **When** se selecciona una mesa
   libre distinta, **Then** el comportamiento de selección de mesa (mesas ocupadas deshabilitadas,
   mesa seleccionada resaltada) es idéntico al que existía antes de este cambio.

---

### User Story 2 - Panel derecho con más espacio (Priority: P2)

Un mesero usa el panel derecho para revisar el pedido armado (carrito, subtotal, total) antes de
confirmar. Hoy el panel mide 320px de ancho, un espacio que ya se sentía apretado antes de este
cambio y que ahora, al recibir también "Tipo de Orden" y "Mesas", necesita más aire para que nada
se vea amontonado.

**Why this priority**: es el segundo punto pedido por el usuario; depende de la Historia 1 (el
panel solo necesita ser más ancho porque ahora contiene más secciones), por lo que se implementa en
conjunto con ella, pero se valida como su propio criterio de aceptación.

**Independent Test**: puede probarse por completo abriendo la creación de orden manual y midiendo
que el panel derecho ocupa un ancho mayor al que tenía antes de esta spec, sin que el catálogo de
la izquierda quede visualmente comprimido a un punto en que sus tarjetas de producto se vean
amontonadas en pantallas de escritorio estándar.

**Acceptance Scenarios**:

1. **Given** la pantalla de creación de orden manual está abierta en una pantalla de escritorio
   estándar, **When** el mesero observa el panel derecho, **Then** su ancho es mayor al que tenía
   antes de esta spec (320px), y ninguna de sus secciones ("Tipo de Orden", "Mesas", "Nueva orden")
   se ve recortada ni obliga a scroll horizontal interno.

---

### Edge Cases

- ¿Qué pasa con el listado de mesas dentro del panel angosto cuando hay muchas mesas (más de las
  que caben en una fila)? Deben seguir siendo todas accesibles (sin quedar ocultas ni exigir scroll
  horizontal dentro de una franja muy angosta) — el criterio visual exacto se resuelve en
  research.md, Clarification 2.
- ¿Qué pasa con las pestañas "Para Llevar" y "Domicilio" (deshabilitadas) tras el traslado? Se
  trasladan igual que "En Mesa", sin cambiar su estado deshabilitado ni el motivo (spec 036,
  Out of Scope) — ningún comportamiento de disponibilidad cambia, solo su ubicación en pantalla.
- ¿Qué pasa si la pantalla es angosta (mobile/tablet en modo retrato)? Fuera de alcance: esta
  pantalla ya está diseñada para escritorio de ancho estándar (spec 036, Target Platform); esta
  spec no introduce ni corrige ningún comportamiento responsive nuevo.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE mostrar, dentro del panel derecho de la pantalla de creación de
  orden manual, las tres opciones de tipo de orden ("En Mesa", "Para Llevar", "Domicilio"), en vez
  de en la barra superior.
- **FR-002**: El sistema DEBE mostrar, dentro del mismo panel derecho e inmediatamente después de
  las opciones de tipo de orden, el listado de mesas seleccionables, en vez de en la barra
  superior.
- **FR-003**: El sistema DEBE mantener, tras el traslado, exactamente el mismo comportamiento de
  selección de mesa que existía antes de esta spec: mesas ocupadas deshabilitadas (salvo que sea la
  mesa ya seleccionada), mesa seleccionada resaltada visualmente, y clic en una mesa libre distinta
  cambia la selección.
- **FR-004**: El sistema DEBE mantener, tras el traslado, exactamente el mismo estado deshabilitado
  de las pestañas "Para Llevar" y "Domicilio" que existía antes de esta spec.
- **FR-005**: La barra superior DEBE conservar únicamente el control "← Volver a la Terminal" tras
  el traslado de "Tipo de Orden" y "Mesas" al panel derecho.
- **FR-006**: El panel derecho DEBE mostrar, en este orden, de arriba hacia abajo: tipo de orden,
  mesas, encabezado "Nueva orden", carrito de ítems, resumen de totales y el botón "Confirmar y
  Enviar" — sin alterar el comportamiento ya existente de ninguna de esas secciones.
- **FR-007**: El sistema DEBE aumentar el ancho del panel derecho respecto al que tenía antes de
  esta spec, de modo que ninguna de sus secciones quede recortada ni exija scroll horizontal
  interno en una pantalla de escritorio estándar.
- **FR-008**: El catálogo de productos (buscador, categorías, grilla de productos) DEBE seguir
  ocupando el ancho restante a la izquierda del panel derecho, sin cambios de comportamiento
  respecto a spec 051.

### Key Entities *(include if feature involves data)*

- Ninguna entidad de datos nueva ni modificada — esta spec reordena y redimensiona controles de UI
  ya existentes, sin tocar ningún modelo de producto, mesa u orden.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un mesero puede completar la secuencia "elegir tipo de orden → elegir mesa → agregar
  un producto al carrito → confirmar" sin que el mouse tenga que recorrer todo el ancho de la
  pantalla entre pasos (salvo el paso de elegir el producto en el catálogo).
- **SC-002**: El panel derecho mide, en una pantalla de escritorio estándar, un ancho mayor al que
  tenía antes de esta spec, verificable por inspección visual/medición del layout.
- **SC-003**: El 100% de las mesas configuradas para el tenant siguen siendo visibles y
  seleccionables desde el panel derecho, sin ninguna oculta por el nuevo ancho más angosto.
- **SC-004**: Ningún comportamiento de selección de mesa, disponibilidad de tipo de orden, carrito
  o confirmación de pedido cambia respecto al que existía antes de esta spec — únicamente cambia su
  ubicación y ancho en pantalla.

## Assumptions

- Esta spec se implementa sobre el resultado de spec 051 (imagen en tarjeta del catálogo + título
  "Mesas"), aunque 051 todavía no esté commiteada — no se revierte ni se contradice ninguno de sus
  cambios, solo se traslada de ubicación el bloque que 051 ya tituló "Mesas".
- "Un poco más de ancho" para el panel derecho se interpreta como un incremento moderado (no un
  panel que ocupe la mitad de la pantalla), suficiente para que sus tres secciones nuevas quepan
  sin scroll horizontal interno en una pantalla de escritorio estándar — el valor numérico exacto
  se resuelve en la fase de planeación técnica.
- El listado de mesas, al ya no estar en una barra de ancho completo, deja de depender de scroll
  horizontal y pasa a acomodarse dentro del ancho del panel (research.md), sin cambiar cuáles mesas
  se muestran ni su estado.
- No se solicita ningún cambio de comportamiento responsive para pantallas angostas (mobile/tablet
  en retrato) — esta pantalla sigue diseñada para escritorio, igual que antes de esta spec.
