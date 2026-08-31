# Feature Specification: Notas del ítem visibles en "Mis pedidos" del Menú QR

**Feature Branch**: `061-notas-visibles-mis-pedidos`

**Created**: 2026-08-31

**Status**: Implemented

**Naturaleza de esta spec**: **spec de corrección de bug**, no funcionalidad nueva. Igual que las
specs [019](../019-correccion-cuenta-mesas-fusionadas/spec.md),
[020](../020-correccion-validacion-opciones-mesero/spec.md),
[021](../021-correccion-orden-borrado-imagen-r2/spec.md) y
[041](../041-correccion-bugs-menu-qr/spec.md), cita nombre de archivo y línea del código actual
(`pos-heladeria`) porque son el contrato observable que se está corrigiendo, no una fuga de
detalles de implementación.

**Autorización de negocio (Principio I de la [Constitución](../../.specify/memory/constitution.md))**:
solicitado directamente por el dueño/desarrollador del proyecto el 2026-08-31, mediante una
captura de pantalla de la sección "Mis pedidos" del Menú QR mostrando un pedido con dos líneas
"1× Cono sencillo" sin ninguna nota visible, y la petición explícita de que las notas se
visualicen ahí.

**Input**: User description: "Cuando envío un pedido desde el menú, no se muestran las notas,
necesito que se visualice."

**Contexto verificado en el código actual**: un comensal puede adjuntar una nota de texto libre a
un ítem del carrito antes de enviarlo (`dining-cart.service.ts`, campo `notes`), y esa nota viaja
al backend y vuelve en la respuesta del pedido (`DiningOrderItem.notes?: string | null`,
`pos-heladeria/src/app/modules/tables/interfaces/dining.interface.ts:112`). La terminal de
personal ya la muestra, concatenada junto a las opciones del ítem
(`pos-terminal.store.ts:789` y `:864`). Sin embargo, la sección "Mis pedidos" del Menú QR público
(`pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts`, bloque
`@if (section() === 'pedidos')`, líneas 241-255) itera `order.items` y muestra cantidad
(`item.quantity`), nombre de variante (`variantLabel`) y opciones (`optionLabels(item)`,
líneas 245-247) — pero nunca lee ni renderiza `item.notes`. El dato existe de punta a punta (se
captura, se envía, se guarda, se devuelve, y ya se muestra al personal); solo la plantilla que ve
el comensal lo omite.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El comensal ve la nota que escribió en su propio pedido (Priority: P1)

Un comensal agrega un producto al carrito con una nota (p. ej. "sin banana" o "para llevar") y
envía el pedido. Al revisar la sección "Mis pedidos", quiere confirmar que su nota quedó
registrada, tal como ya puede ver la cantidad, el producto y las opciones elegidas.

**Why this priority**: sin esto, el comensal no tiene ninguna forma de verificar que una
instrucción especial (alergia, preferencia, modificación) fue realmente enviada — la nota existe
en el sistema pero es invisible para quien la escribió, lo que genera incertidumbre y reduce la
confianza en el pedido.

**Independent Test**: se puede probar de forma aislada agregando un ítem con nota al carrito,
enviando el pedido, y verificando en la sección "Mis pedidos" que la nota aparece bajo la línea de
ese ítem — sin tocar el flujo de captura de la nota en el carrito ni la terminal de personal,
que ya funcionan correctamente.

**Acceptance Scenarios**:

1. **Given** un comensal que agregó un ítem al carrito con una nota de texto libre, **When** envía
   el pedido y abre la sección "Mis pedidos", **Then** ve la nota mostrada bajo la línea de ese
   ítem, junto con la cantidad, el producto y las opciones ya visibles hoy.
2. **Given** un pedido con varios ítems donde solo algunos tienen nota, **When** el comensal ve
   "Mis pedidos", **Then** la nota aparece únicamente bajo los ítems que la tienen — los ítems sin
   nota se ven exactamente igual que hoy, sin ningún elemento vacío añadido.
3. **Given** un pedido en cualquier estado visible en "Mis pedidos" (por confirmar, en cocina,
   listo, etc.), **When** el comensal lo consulta, **Then** la nota del ítem se sigue mostrando
   igual — la visibilidad de la nota no depende del estado del pedido ni del ítem.
4. **Given** dos ítems idénticos en producto y opciones pero con notas de texto distintas (p. ej.
   dos "Cono sencillo", uno con "sin banana" y otro sin nota), **When** el comensal ve "Mis
   pedidos", **Then** puede distinguir cuál línea lleva cuál nota — no se fusionan ni se pierde la
   asociación nota-línea.

---

### Edge Cases

- Ítem sin nota (`notes` es `null` o cadena vacía): no debe añadirse ningún elemento visual extra
  bajo esa línea — el resultado debe ser idéntico al comportamiento actual.
- Nota muy larga: debe seguir siendo legible sin romper el layout de la tarjeta del pedido (se
  ajusta en varias líneas igual que ya ocurre con `optionLabels` cuando la lista de opciones es
  larga).
- Ítem con nota y con opciones a la vez: ambas deben mostrarse, cada una identificable por
  separado (no concatenadas de forma que se confundan entre sí).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: En la sección "Mis pedidos" del Menú QR, el sistema DEBE mostrar la nota
  (`item.notes`) de cada ítem de pedido cuando esta no sea nula ni una cadena vacía.
- **FR-002**: Cuando `item.notes` sea nulo o una cadena vacía, el sistema NO DEBE mostrar ningún
  elemento adicional para ese ítem — el comportamiento visual debe ser idéntico al actual.
- **FR-003**: La nota mostrada DEBE estar asociada visualmente a su línea de ítem específica, de
  forma que en pedidos con ítems repetidos (mismo producto/opciones) cada nota se identifique con
  su línea correspondiente sin ambigüedad.
- **FR-004**: La visibilidad de la nota NO DEBE depender del estado del pedido (por confirmar, en
  cocina, listo, etc.) ni del estado de cocina del ítem — se muestra siempre que exista, igual que
  ya ocurre con las opciones del ítem (`optionLabels`).
- **FR-005**: Esta corrección aplica únicamente a la sección "Mis pedidos" del Menú QR del
  comensal; no modifica la captura de notas en el carrito, la terminal de personal (que ya muestra
  la nota) ni el backend (el dato ya viaja correctamente hoy).

### Key Entities

- **Ítem de pedido (`DiningOrderItem`)**: entidad ya existente; esta spec no le agrega ningún
  campo — `notes` ya existe y ya viaja del backend al frontend. Solo cambia si la vista del
  comensal lo lee y lo renderiza.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los pedidos con al menos un ítem con nota muestran esa nota en la sección
  "Mis pedidos" del comensal, sin necesidad de recargar la página ni de ninguna acción adicional.
- **SC-002**: El 100% de los ítems sin nota se siguen mostrando exactamente igual que antes de esta
  corrección (sin regresión visual).
- **SC-003**: Un comensal puede verificar visualmente, sin ayuda del personal, que la nota que
  escribió al pedir quedó registrada en su pedido.

## Assumptions

- La nota a mostrar es el mismo texto libre que el comensal ya captura hoy en el carrito
  (`dining-cart.service.ts`, campo `notes`) — esta spec no cambia su formato, longitud máxima ni
  validación.
- El campo `item.notes` ya llega correctamente desde el backend en la respuesta de pedidos
  consultada por "Mis pedidos" (confirmado por su uso ya funcional en la terminal de personal,
  `pos-terminal.store.ts:789`); esta spec no toca el backend.
- La presentación visual de la nota debe seguir el mismo patrón ya usado para `optionLabels` en el
  mismo bloque (texto pequeño, bajo el nombre del producto), para mantener consistencia dentro de
  la misma tarjeta de pedido — sin introducir un componente o estilo nuevo.
