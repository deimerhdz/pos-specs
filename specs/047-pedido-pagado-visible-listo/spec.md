# Feature Specification: Pedido de Mostrador Pagado Sigue Visible Hasta Liberar la Mesa

**Feature Branch**: `047-pedido-pagado-visible-listo`

**Created**: 2026-08-28

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección**, no una funcionalidad nueva desde cero. Igual
que las specs [019](../019-correccion-cuenta-mesas-fusionadas/spec.md),
[020](../020-correccion-validacion-opciones-mesero/spec.md),
[021](../021-correccion-orden-borrado-imagen-r2/spec.md),
[041](../041-correccion-bugs-menu-qr/spec.md),
[044](../044-rechazo-pedido-pago-pendiente/spec.md),
[045](../045-simplificacion-terminal-mesas/spec.md) y
[046](../046-dividir-cuenta-pago-pendiente/spec.md), cita nombres de archivo, método y línea del
código actual (`pos-heladeria`) porque son el contrato observable que se está corrigiendo, no una
fuga de detalles de implementación. **Amend explícito** de spec
[035](../035-estado-pagado-formato-moneda/spec.md) (decisión A-52 del registro de anomalías): esa
spec agregó, en `pos-terminal.store.ts`, la condición que mantiene visible un pedido `'pagada'`
**mientras cocina todavía trabaja** — pero dejó un vacío que esta spec cierra: en cuanto cocina
termina, esa misma condición hace que el pedido desaparezca de la Terminal de Mesas aunque la
sesión de la mesa siga abierta y nadie haya liberado la mesa todavía.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-28, con dos capturas de pantalla mostrando el estado
antes y después de marcar un pedido como listo. Es una corrección de un vacío dejado por spec 035
(A-52) sobre una pantalla ya existente — no reabre ninguna regla de negocio de precio, inventario o
facturación; no aplica una nueva entrada en `registro-de-anomalias.md` más allá de referenciar la
A-52 ya existente como el origen del vacío.

**Input**: User description (verbatim, con dos capturas de la Terminal de Mesas: antes y después de
pulsar "Marcar pedido listo" sobre un pedido de mostrador ya cobrado por adelantado): "cuando marco
el pedido como listo desaparece de la mesa, el pedido debería poder seguir visualizándose con el
cambio de estado, para poder seguir viendo el resumen de órdenes que tiene el cliente cuando hay
una sesión activa, por favor arreglalo". La causa raíz se identificó leyendo el código real antes de
escribir esta spec (ver Assumptions): `pos-terminal.store.ts`, funciones `activeOrders` (línea
~377) y `tableOrders(tableId)` (línea ~401) excluyen un pedido `'pagada'` en cuanto
`hasPendingKitchenWork(o)` se vuelve falso — justo al terminar de cocinar ("Marcar pedido listo").

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El pedido pagado sigue visible hasta que el cajero libera la mesa (Priority: P1)

El cajero cobró por adelantado un pedido de mostrador (el cliente pagó antes de que cocina
empezara a prepararlo). Mientras cocina trabaja, la Terminal de Mesas muestra correctamente el
resumen del pedido (ítems, total, "Imprimir Factura", "Liberar Mesa"). En cuanto el cajero marca el
pedido como listo, hoy el pedido desaparece por completo de la pantalla — el panel central cae al
mensaje de "mesa libre" y el panel derecho pierde el resumen y ofrece en su lugar "+ Crear pedido
nuevo", como si no hubiera nada que cobrar ni liberar — aunque la sesión de la mesa sigue abierta.

**Why this priority**: es el único comportamiento reportado y el que bloquea la operación diaria —
el cajero pierde de vista un pedido ya cobrado que todavía necesita imprimir/liberar, justo en el
momento en que más lo necesita.

**Independent Test**: se puede probar completamente cobrando por adelantado un pedido de mostrador,
marcándolo como listo antes de liberar la mesa, y verificando que el pedido sigue viéndose (con su
resumen) en el panel central y derecho, y que la tarjeta de la mesa refleja el nuevo estado sin
volverse "mesa libre".

**Acceptance Scenarios**:

1. **Given** un pedido de mostrador ya cobrado por adelantado, con cocina todavía preparándolo,
   **When** el cajero lo mira en la Terminal de Mesas, **Then** ve el resumen del pedido (ítems,
   total) y las acciones "Imprimir Factura"/"Liberar Mesa" — comportamiento ya correcto hoy, sin
   cambios.
2. **Given** ese mismo pedido, **When** el cajero pulsa "Marcar pedido listo", **Then** el panel
   central sigue mostrando el pedido (no cae al mensaje de "mesa libre") y el panel derecho sigue
   ofreciendo el resumen con "Imprimir Factura"/"Liberar Mesa" — no "+ Crear pedido nuevo".
3. **Given** el mismo pedido ya marcado como listo, **When** el cajero mira la tarjeta de la mesa en
   la grilla, **Then** ve un estado "Listo" (no "Ocupada" genérico ni "Libre").
4. **Given** el pedido pagado y listo todavía visible, **When** el cajero pulsa "Liberar Mesa",
   **Then** el pedido y la mesa se liberan con el mismo comportamiento ya implementado (sin
   cambios) — solo en ese momento deja de mostrarse.
5. **Given** una mesa con dos pedidos activos, uno pagado-y-listo y otro todavía en preparación,
   **When** el cajero la mira, **Then** ambos siguen contando como consumo de la mesa — ninguno
   desaparece antes de tiempo.

---

### Edge Cases

- **Pedido cancelado**: sigue sin contar como consumo de la mesa — sin cambios.
- **Pedido QR pendiente de confirmar** (`'recibida'` + canal `qr`): sigue su propio tratamiento ya
  existente (sección/bloque de pagos por confirmar) — esta spec no le cambia nada.
- **Pedido de mesero cobrado al cerrar la sesión** (no por adelantado): ese camino no pasa por
  `status === 'pagada'` antes de liberar — sin cambios, esta spec no le aplica.
- **Varios pedidos de la misma mesa en distinto estado de cocina**: cada uno se evalúa por
  separado; uno pagado-y-listo no oculta ni reemplaza a otro que siga en preparación (Escenario 5).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE seguir contando un pedido con `status = 'pagada'` como consumo activo
  de su mesa sin importar si la cocina ya terminó de prepararlo o no — deja de excluirlo en cuanto
  todos sus ítems queden marcados como listos (amend spec 035, A-52, que solo cubría el caso
  mientras cocina seguía trabajando).
- **FR-002**: Mientras la sesión de la mesa siga abierta, el panel central de la Terminal de Mesas
  DEBE seguir mostrando el pedido pagado en vez de caer al estado informativo de "mesa libre".
- **FR-003**: Mientras la sesión de la mesa siga abierta, el panel derecho DEBE seguir mostrando la
  vista de resumen/cobro de la mesa (total, "Imprimir Factura", "Liberar Mesa") para ese pedido, en
  vez de caer al panel de "Pedido de mostrador"/"+ Crear pedido nuevo".
- **FR-004**: La tarjeta de la mesa en la grilla DEBE reflejar el estado "Listo" (no "Ocupada"
  genérico) cuando el pedido está pagado y todos sus ítems ya están listos — el mismo criterio ya
  definido para ese estado (spec 029, Historia 3), hoy inalcanzable por el vacío de FR-001.
- **FR-005**: El pedido pagado y su mesa asociada DEBEN dejar de mostrarse en la Terminal de Mesas
  únicamente cuando el cajero libera la mesa explícitamente ("Liberar Mesa", ya implementado) —
  nunca antes, y nunca solo porque la cocina haya terminado.

### Key Entities *(include if feature involves data)*

Esta spec no agrega ni modifica entidades de datos ni ningún endpoint de backend — corrige
exclusivamente una condición de filtrado en el frontend (qué pedidos cuenta como "consumo vivo" de
una mesa) para que el comportamiento ya definido por spec 035 (A-52) y spec 029 (Historia 3, estado
"Listo") funcione también una vez que la cocina termina, no solo mientras trabaja.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% de los pedidos de mostrador pagados por adelantado permanecen visibles, con su
  resumen completo, en la Terminal de Mesas después de marcarse como listos — hasta que el cajero
  libera la mesa.
- **SC-002**: 0% de los casos en los que un pedido pagado y listo hace que el panel central muestre
  el mensaje de "mesa libre" mientras la sesión de esa mesa sigue abierta.
- **SC-003**: La tarjeta de mesa muestra el estado "Listo" (no "Ocupada" genérico) en el 100% de los
  casos de un pedido pagado con toda la cocina terminada.
- **SC-004**: El cajero puede imprimir la factura y liberar la mesa sin ningún paso adicional (como
  volver a seleccionar la mesa) inmediatamente después de marcar el pedido como listo.

## Out of Scope

- **Cualquier cambio de backend o de contrato de API** — la fuente de verdad sobre cuándo un pedido
  pagado deja de pertenecer a una mesa (al liberarla) ya es correcta hoy en el backend; esta spec
  es exclusivamente una corrección de frontend.
- **El mecanismo de "Marcar pedido listo" en sí** (qué endpoint llama, qué campo cambia) — sin
  cambios, sigue marcando ítems como listos sin tocar el estado del pedido.
- **Cualquier otro estado de la tarjeta de mesa** no relacionado con "pagado y listo" (p. ej. "Por
  confirmar", "Pago pendiente", "En preparación") — sin cambios.
- **El flujo de pedidos QR pendientes de confirmar** — sin cambios, sigue su propio tratamiento.

## Assumptions

- **Causa raíz identificada antes de escribir esta spec, leyendo el código real**
  (`pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`): las funciones
  `activeOrders` y `tableOrders(tableId)` excluyen un pedido `'pagada'` en cuanto
  `hasPendingKitchenWork(o)` se vuelve falso; el backend (`GET /orders?active_sessions_only=true`)
  ya es la autoridad correcta sobre cuándo un pedido pagado deja de pertenecer a una mesa (recién
  al liberarla) — el frontend no necesita duplicar ese criterio mirando el estado de cocina.
- **El comportamiento correcto ya estaba diseñado, solo era inalcanzable**: el estado visual
  "Listo" (spec 029, Historia 3) y el criterio de `active_sessions_only` del backend ya cubrían
  exactamente este caso — esta spec no introduce ningún concepto de negocio nuevo, corrige el
  filtro de frontend que impedía que ambos se activaran juntos.
- **Aplica a pedidos de canal mostrador y mesero que lleguen a `status = 'pagada'` antes de que
  cocina termine** (el camino "cobra primero, envía después"); los pedidos QR pendientes de
  confirmar no se ven afectados.
