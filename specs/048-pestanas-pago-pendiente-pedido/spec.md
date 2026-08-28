# Feature Specification: Pestañas para Ver el Pedido Pagado Junto al Pago Pendiente de la Misma Mesa

**Feature Branch**: `048-pestanas-pago-pendiente-pedido`

**Created**: 2026-08-28

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección**, no una funcionalidad nueva desde cero. Igual
que las specs [019](../019-correccion-cuenta-mesas-fusionadas/spec.md),
[020](../020-correccion-validacion-opciones-mesero/spec.md),
[021](../021-correccion-orden-borrado-imagen-r2/spec.md),
[041](../041-correccion-bugs-menu-qr/spec.md),
[044](../044-rechazo-pedido-pago-pendiente/spec.md),
[045](../045-simplificacion-terminal-mesas/spec.md),
[046](../046-dividir-cuenta-pago-pendiente/spec.md) y
[047](../047-pedido-pagado-visible-listo/spec.md), cita nombres de archivo, función y línea del
código actual (`pos-heladeria`) porque son el contrato observable que se está corrigiendo, no una
fuga de detalles de implementación. **Amend explícito** de spec
[036](../036-terminal-mesas-rediseno-layout/spec.md), FR-005: esa spec definió que seleccionar una
mesa abre, en el panel central, **uno** de tres comportamientos posibles ("bloque de validación de
pago QR, constructor de orden manual, o resumen de cuenta/cobro, según el estado y origen de la
orden") — un "o" excluyente. Esta spec agrega el caso que faltaba: cuando la mesa tiene **a la vez**
un pago pendiente de confirmar y un pedido ya pagado/activo (típico de un comensal con sesión
abierta que ya pagó una ronda y envía una ronda nueva por QR), el cajero necesita ver ambos, no
solo uno. Es también consecuencia directa de spec [047](../047-pedido-pagado-visible-listo/spec.md):
antes de esa corrección, un pedido pagado con toda su cocina lista desaparecía de la mesa por sí
solo; ahora que permanece visible, quedó en evidencia que, mientras haya además un pago pendiente
de confirmar, no hay ninguna forma de alcanzarlo desde la pantalla.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-28, con una captura mostrando una mesa "Por confirmar"
con "2 pedidos" (uno pagado, otro pendiente) donde solo el pendiente es visible. El dueño del
producto confirmó explícitamente, ante una pregunta de diseño con dos opciones, la solución
preferida (pestañas para alternar, ver Clarifications). Es reordenamiento de navegación sobre una
pantalla ya existente — no reabre ninguna regla de negocio de precio, inventario o facturación; no
aplica una nueva entrada en `registro-de-anomalias.md`.

**Input**: User description (verbatim, con una captura de la Terminal de Mesas): "cuando la mesa
tiene ordenes que ya estan pagadas y pertenecen a la sesion del comensal y ese comensal envia una
nueva orden aparece nuevamente la ventana para validar el pago, pero para este caso no tengo forma
de volver a visualizar la ordenes, podrias mejorarlo". La causa raíz se identificó leyendo el
código real antes de escribir esta spec (ver Assumptions): `pos-terminal.store.ts`, `centralState`
(línea ~447) le da prioridad absoluta a `'validar-pago'` en cuanto hay algún pago pendiente, y
`table-sessions.component.ts` (línea ~125) renderiza uno u otro bloque según ese único estado, sin
ningún mecanismo para alternar.

## Clarifications

### Session 2026-08-28

- Q: Cuando una mesa tiene a la vez un pedido ya pagado y otro pendiente de confirmar, ¿cómo debe
  poder verlos ambos el cajero? → A: Pestañas para alternar ("Pagos por confirmar" / "Pedido de la
  mesa") — la pestaña "Pedido de la mesa" abre el mismo panel completo que ya existe hoy para
  pedidos activos (reimprimir factura, liberar mesa, etc.), no un resumen de solo lectura aparte.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ver el pedido ya pagado mientras hay un pago pendiente de confirmar en la misma mesa (Priority: P1)

Un comensal con sesión de mesa abierta ya pagó una ronda de productos (pedido ya `'pagada'`, sigue
visible gracias a spec 047) y envía, por QR, una ronda nueva que llega pendiente de confirmar el
pago. Hoy, en cuanto ese pago queda pendiente, el panel central de la Terminal de Mesas muestra
únicamente el bloque "Pagos por confirmar" — el cajero pierde por completo el acceso al pedido ya
pagado (sus ítems, el total, "Imprimir Factura", "Liberar Mesa") mientras dure esa confirmación.

**Why this priority**: es el único comportamiento reportado — sin esta corrección, el cajero no
puede reimprimir la factura de la ronda ya pagada ni liberar la mesa mientras haya cualquier pago
nuevo pendiente de esa misma mesa, aunque sea de otra ronda distinta.

**Independent Test**: se puede probar completamente teniendo una mesa con un pedido ya pagado (con
o sin toda su cocina lista) y, en la misma mesa, un segundo pedido QR recién llegado y pendiente de
confirmar; seleccionar esa mesa y verificar que aparecen dos pestañas, que el cajero puede alternar
entre ambas sin perder ninguna, y que la pestaña del pedido pagado ofrece exactamente las mismas
acciones que ya tiene hoy fuera de este escenario (reimprimir factura, liberar mesa).

**Acceptance Scenarios**:

1. **Given** una mesa con un pedido ya pagado/activo y, a la vez, un pago pendiente de confirmar,
   **When** el cajero la selecciona, **Then** el panel central muestra dos pestañas: "Pagos por
   confirmar" y "Pedido de la mesa".
2. **Given** ese mismo estado, **When** el cajero mira qué pestaña está activa por defecto,
   **Then** es "Pagos por confirmar" — la más urgente, sin que eso le impida cambiar a la otra.
3. **Given** las dos pestañas visibles, **When** el cajero pulsa "Pedido de la mesa", **Then** ve
   el mismo panel completo que ya existe hoy para un pedido activo (ítems, total, "Imprimir
   Factura", "Liberar Mesa"; si hay más de un pedido pagado/activo, incluye la navegación entre
   ellos ya existente) — no un resumen aparte ni de solo lectura.
4. **Given** el cajero en la pestaña "Pedido de la mesa", **When** confirma o aprueba el pago
   pendiente (desde la otra pestaña), **Then** la pestaña "Pagos por confirmar" deja de tener
   contenido y, si ya no queda ningún pago pendiente de esa mesa, las pestañas desaparecen y el
   panel central vuelve a mostrar directamente el pedido — mismo comportamiento de siempre, sin
   pestañas de por medio.
5. **Given** una mesa con **solo** un pago pendiente de confirmar (sin ningún pedido pagado/activo
   todavía), **When** el cajero la selecciona, **Then** no aparece ninguna pestaña — se muestra
   directamente el bloque "Pagos por confirmar", igual que hoy.
6. **Given** una mesa con **solo** pedidos pagados/activos (sin nada pendiente de confirmar),
   **When** el cajero la selecciona, **Then** no aparece ninguna pestaña — se muestra directamente
   el pedido, igual que hoy.

---

### Edge Cases

- **Varios pedidos pagados/activos a la vez que un pago pendiente**: la pestaña "Pedido de la
  mesa" reutiliza la navegación entre pedidos ya existente (spec 036) — no es responsabilidad de
  esta spec resolver cuál de varios pedidos activos se ve primero, solo que la pestaña completa sea
  alcanzable.
- **Se confirma el único pago pendiente mientras el cajero está en la pestaña "Pedido de la
  mesa"**: no lo saca de esa pestaña de golpe — las pestañas simplemente desaparecen y el panel
  queda mostrando el pedido, que es exactamente lo que ya estaba viendo (Escenario 4).
- **Llega un nuevo pago pendiente mientras el cajero está en la pestaña "Pedido de la mesa" (antes
  no había ninguno)**: las pestañas aparecen, pero no le cambian la vista activa sin que él lo
  pida — sigue en "Pedido de la mesa" hasta que la cambie manualmente.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Cuando la mesa seleccionada tenga **a la vez** al menos un pago pendiente de
  confirmar y al menos un pedido pagado/activo, el panel central DEBE mostrar dos pestañas:
  "Pagos por confirmar" y "Pedido de la mesa" (amend spec 036, FR-005, que hasta ahora solo
  contemplaba mostrar uno de los dos según el estado, nunca ambos).
- **FR-002**: La pestaña "Pagos por confirmar" DEBE mostrar exactamente el mismo bloque ya
  implementado hoy (spec 044/046), sin cambios de comportamiento.
- **FR-003**: La pestaña "Pedido de la mesa" DEBE mostrar exactamente el mismo panel ya
  implementado hoy para un pedido activo/pagado de esa mesa (ítems, total, "Imprimir Factura",
  "Liberar Mesa", y la navegación entre varios pedidos si aplica) — sin ningún cambio de
  comportamiento ni una vista de solo lectura alternativa.
- **FR-004**: Cuando ambas pestañas existan, la seleccionada por defecto DEBE ser "Pagos por
  confirmar".
- **FR-005**: Cuando la mesa tenga únicamente pago(s) pendiente(s) de confirmar (sin ningún pedido
  pagado/activo), o únicamente pedido(s) pagado(s)/activo(s) (sin nada pendiente de confirmar), el
  sistema NO DEBE mostrar ninguna pestaña — se mantiene el comportamiento ya implementado de
  mostrar directamente el bloque correspondiente.
- **FR-006**: En cuanto deje de haber algún pago pendiente de confirmar para esa mesa (se confirma,
  se aprueba o se rechaza), las pestañas DEBEN desaparecer si ya no aplica FR-001, sin sacar al
  cajero de la pestaña que tenía activa cuando esa vista siga teniendo contenido (Edge Cases).

### Key Entities *(include if feature involves data)*

Esta spec no agrega ni modifica entidades de datos — reorganiza exclusivamente la navegación de una
pantalla ya existente (Terminal de Mesas, spec 036), reutilizando dos bloques ya implementados
("Pagos por confirmar", spec 044/046; panel de pedido activo, spec 036) sin cambiar su
comportamiento interno.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% de las mesas con un pago pendiente de confirmar y, a la vez, un pedido
  pagado/activo, muestran ambas pestañas — cero casos donde el pedido pagado queda inalcanzable.
- **SC-002**: El cajero puede reimprimir la factura o liberar la mesa de un pedido ya pagado, en un
  máximo de un clic adicional (cambiar de pestaña), incluso mientras haya un pago nuevo pendiente
  de confirmar en la misma mesa.
- **SC-003**: 0% de las mesas con un único tipo de contenido (solo pendiente, o solo pagado/activo)
  muestran alguna pestaña — el comportamiento actual para esos casos no cambia.
- **SC-004**: Confirmar/aprobar/rechazar el último pago pendiente de una mesa hace desaparecer las
  pestañas sin que el cajero pierda la pestaña que tenía activa, en el 100% de los casos donde esa
  vista sigue teniendo contenido.

## Out of Scope

- **La lógica de confirmación de pago en sí** (Confirmar efectivo, Aprobar/Rechazar comprobante,
  Rechazar pedido completo — spec 044) — sin cambios, esta spec solo la hace alcanzable junto al
  pedido pagado.
- **El panel de pedido activo/pagado en sí** (ítems, total, Imprimir Factura, Liberar Mesa,
  navegación entre pedidos — spec 036) — sin cambios de comportamiento, solo se vuelve alcanzable
  como una pestaña más en este escenario.
- **Cualquier otro estado del panel central** (mesa libre, un único pedido sin pago pendiente) —
  sin cambios (Escenarios 5 y 6).
- **Reordenar o priorizar entre varios pedidos pagados/activos de una misma mesa** — sigue usando
  la navegación ya existente (spec 036), sin cambios.

## Assumptions

- **Causa raíz identificada antes de escribir esta spec, leyendo el código real**
  (`pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`,
  `pages/table-sessions.component.ts`): `centralState` resuelve a un único valor
  (`'validar-pago' | 'mesa-libre' | 'pedido'`) que decide, de forma excluyente, qué bloque se
  renderiza; el panel de pedido activo ya excluido de `pendingOfSelectedTable` sigue disponible
  hoy en un computed privado (`ordersOfTable`), simplemente no se expone a la plantilla mientras el
  estado es `'validar-pago'`.
- **La pestaña "Pedido de la mesa" reutiliza tal cual el componente/estado ya usado para el estado
  `'pedido'`** (spec 036) — no se construye ninguna vista nueva ni resumen alternativo; solo se
  vuelve alcanzable en un escenario donde antes era invisible.
- **Decisión de diseño confirmada explícitamente por el dueño del producto** (pestañas para
  alternar, no una vista combinada de solo lectura) — documentada en Clarifications.
