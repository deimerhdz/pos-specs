# Feature Specification: Corrección — "Liberar Mesa" bloqueada por un pedido ya cancelado

**Feature Branch**: `050-correccion-liberar-mesa-pedido-cancelado`

**Created**: 2026-08-29

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección**, no una funcionalidad nueva desde cero. Igual
que las specs [019](../019-correccion-cuenta-mesas-fusionadas/spec.md),
[020](../020-correccion-validacion-opciones-mesero/spec.md),
[021](../021-correccion-orden-borrado-imagen-r2/spec.md),
[041](../041-correccion-bugs-menu-qr/spec.md),
[044](../044-rechazo-pedido-pago-pendiente/spec.md),
[045](../045-simplificacion-terminal-mesas/spec.md),
[048](../048-pestanas-pago-pendiente-pedido/spec.md) y
[049](../049-rediseno-panel-pedido-mesa/spec.md), cita nombres de archivo, función y línea del
código actual (tanto de `pos-heladeria` como de `pos-backend`) porque son el contrato observable
que se está corrigiendo, no una fuga de detalles de implementación. A diferencia de las specs
anteriores de esta serie, esta corrección toca **backend** (`pos-backend`), no solo frontend — es
un bug preexistente, no introducido por ninguna spec anterior de esta lista.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-08-29, tras reproducir el bug en un entorno real (crear un
pedido manual, cancelarlo, e intentar liberar la mesa) y confirmar dos decisiones de alcance
resueltas más abajo. No reabre ninguna regla de negocio de precio, inventario o facturación —
`cancel_order` y su lógica de ajuste de inventario (item por item, según lo que se alcanzó a
consumir) no se tocan; el fix es exclusivamente sobre qué pedidos se toman en cuenta al validar si
una mesa puede liberarse. No aplica una nueva entrada en `registro-de-anomalias.md` más allá de
citar este mismo documento como origen y autorización del cambio.

**Input**: User description (verbatim, con una captura de pantalla): "cree un pedido manual y lo
cancele desde la terminal de mesas, cuando intente liberarla me salio ese error que sale en la
imagen" — la imagen mostraba el mismo mensaje de error "Hay ítems sin terminar en cocina; anúlalos
o espera a que estén listos." repetido 8 veces, apilado como notificaciones independientes sobre la
grilla de mesas. Investigación posterior (leyendo el código real) confirmó la causa raíz exacta:

- `cancel_order` (`pos-backend/app/api/v1/orders/checkout.py:519-566`) marca
  `CustomerOrder.status = "cancelada"` pero **deliberadamente no cambia**
  `OrderItem.estado_cocina` de sus ítems — diseño intencional documentado en el propio código, para
  no interferir con el ajuste de inventario (que sí distingue `pendiente`/`en_preparacion`/`listo`
  al revertir o no el stock). Un ítem que seguía `'pendiente'` al cancelar el pedido se queda en
  `'pendiente'` para siempre.
- `_assert_closable` (`pos-backend/app/api/v1/table_sessions/service.py:217-241`) rechaza con 409 y
  ese mensaje exacto si cualquier ítem de la lista de pedidos que recibe está en curso
  (`'pendiente'`/`'en_preparacion'`) — sin filtrar por el status del pedido dueño; confía en que
  quien la invoca ya haya filtrado.
- `close_session` (el flujo normal de cobro) la invoca correctamente, pasándole solo pedidos de
  `_billable_orders` (`service.py:141-157`), que ya excluye
  `status.notin_(("cancelada", "pagada"))`.
- `release_paid_session` (`service.py:337-370`, detrás de `POST
  /table-sessions/{table_session_id}/release`, el botón "Liberar Mesa") es el único camino que
  **no** filtra: arma su lista de pedidos con un query directo sobre "todos los pedidos de la
  sesión" (líneas ~366-369, sin condición de `status`) — a propósito, según su propio docstring,
  para seguir detectando cocina en curso sobre un pedido ya `'pagada'` (que `_billable_orders` no
  incluiría). Pero ese mismo query sin filtro arrastra también los pedidos `'cancelada'`, cuyos
  ítems nunca se anulan (punto anterior) — bloqueando la liberación de la mesa sin ninguna acción
  posible desde la interfaz, porque el pedido ya es terminal (nada en la UI ofrece "anular" un ítem
  de un pedido ya cancelado).
- Por separado, `ToastService.push()` (`pos-heladeria/src/app/shared/feedback/toast.service.ts:
  36-42`) no deduplica: cada llamada agrega una tarjeta de notificación nueva sin revisar si ya hay
  una idéntica visible. El botón "Liberar Mesa" sí evita el doble envío del mismo clic
  (`[disabled]="store.submitting()"`, `pos-checkout-panel.component.ts:218-224`), pero nada impide
  un **clic siguiente** segundos después mientras el bug de fondo sigue sin resolverse — cada
  intento fallido apila una tarjeta más, idéntica a las anteriores.

## Clarifications

### Session 2026-08-29

- Q: ¿El fix debe incluir también evitar que un error repetido apile copias idénticas del mismo
  aviso, o solo el bug de fondo que bloquea la mesa? → A: Ambas correcciones — el bug de fondo
  (prioridad) y la deduplicación de avisos repetidos (prioridad menor, mejora general aplicable a
  cualquier error, no solo a este).
- Q: ¿Esta corrección de backend debe documentarse con una spec formal antes de implementarse, o
  como un hotfix directo? → A: Spec formal, siguiendo el mismo proceso ya usado por las specs de
  corrección anteriores de esta lista — es un bug preexistente de `pos-backend`, no introducido por
  ninguna spec previa, y la Constitución exige spec antes de cambiar comportamiento.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Liberar una mesa con un pedido ya cancelado (Priority: P1)

Un cajero crea un pedido manual sobre una mesa, decide cancelarlo antes de que cocina lo termine
(un ítem se queda en `'pendiente'`), y luego intenta liberar la mesa porque ya no hay nada
pendiente por cobrar ni por preparar de verdad. Hoy el sistema rechaza la liberación con "Hay ítems
sin terminar en cocina; anúlalos o espera a que estén listos." — un mensaje que no tiene ninguna
acción posible: el pedido ya es terminal (cancelado), no hay ningún botón "Anular" disponible para
sus ítems, y cocina nunca va a terminar de preparar algo que ya se canceló. La mesa queda bloqueada
para siempre.

**Why this priority**: sin esta corrección, cualquier mesa donde se cancele un pedido con algo
todavía sin terminar en cocina queda inutilizable — ni el cajero ni ningún flujo existente en la
aplicación puede desbloquearla.

**Independent Test**: se puede probar completamente creando un pedido manual sobre una mesa,
dejando al menos un ítem sin terminar en cocina (`'pendiente'` o `'en_preparación'`), cancelando
ese pedido, y verificando que "Liberar Mesa" ahora libera la mesa sin error.

**Acceptance Scenarios**:

1. **Given** una mesa cuya sesión solo tiene un pedido ya cancelado, con al menos un ítem que
   quedó `'pendiente'` o `'en_preparación'` al cancelarlo, **When** el cajero pulsa "Liberar
   Mesa", **Then** la mesa se libera (mismo resultado que hoy ya ocurre cuando no queda ningún
   pedido con problemas) — sin el error "Hay ítems sin terminar en cocina...".
2. **Given** una mesa con un pedido `'pagada'` (cobrado por adelantado) cuya cocina todavía está en
   curso, **When** el cajero pulsa "Liberar Mesa", **Then** el sistema sigue rechazando la
   liberación con el mismo error de hoy — este escenario, ya cubierto y correcto, no cambia
   (comportamiento existente protegido).
3. **Given** una mesa con un pedido `'cancelada'` (con ítems sin terminar) **y**, a la vez, otro
   pedido todavía activo con algo pendiente de cobrar o de preparar, **When** el cajero pulsa
   "Liberar Mesa", **Then** el sistema sigue rechazando la liberación por lo que sí corresponde a
   ese otro pedido activo — el pedido cancelado deja de contar, pero cualquier otro motivo legítimo
   de bloqueo sigue aplicando igual que hoy.

---

### User Story 2 - Un error que se repite no apila avisos idénticos (Priority: P2)

Mientras el bug de la Historia 1 seguía sin corregirse, el cajero pulsó "Liberar Mesa" varias veces
seguidas (sin ninguna señal de que el primer intento ya había fallado de forma reproducible), y
cada intento agregó una tarjeta de aviso nueva, idéntica a las anteriores, hasta acumular 8 copias
del mismo mensaje sobre la pantalla.

**Why this priority**: es una mejora de claridad general (útil para cualquier error que se repita,
no solo este), de menor impacto que la Historia 1 — una vez corregida esa, este escenario concreto
ya no se puede reproducir con "Liberar Mesa", pero el problema de fondo (avisos duplicados) seguiría
latente para cualquier otro error futuro.

**Independent Test**: se puede probar completamente provocando dos veces seguidas, sin resolver la
causa entre intentos, cualquier error que hoy muestre un aviso (por ejemplo, intentar cobrar sin
elegir método de pago dos veces), y verificando que solo queda visible una tarjeta de aviso, no dos
copias idénticas.

**Acceptance Scenarios**:

1. **Given** un aviso de error ya visible en pantalla, **When** ocurre exactamente el mismo error
   (mismo tipo y mismo texto) antes de que ese aviso desaparezca, **Then** no se agrega una tarjeta
   nueva — sigue habiendo una sola.
2. **Given** un aviso de error ya visible, **When** ocurre un error de texto distinto, o del mismo
   texto pero de otro tipo (por ejemplo, uno informativo en vez de uno de error), **Then** sí se
   agrega una tarjeta nueva, independiente — la deduplicación no oculta avisos genuinamente
   distintos.
3. **Given** un aviso de error ya visible, **When** ese aviso desaparece (por su propio tiempo de
   vida) y **luego** ocurre otra vez el mismo error, **Then** sí se muestra de nuevo — la
   deduplicación no bloquea permanentemente un mensaje, solo evita duplicados simultáneos.

---

### Edge Cases

- **Mesa con varios pedidos cancelados a la vez, todos con ítems sin terminar**: ninguno de ellos
  debe contar para el chequeo de cocina en curso — la exclusión aplica por pedido, no solo al
  primero que se encuentre.
- **Pedido cancelado cuyos ítems sí llegaron a `'listo'` antes de cancelarse**: sigue sin bloquear
  la liberación hoy (no dispara `_assert_closable`) — esta corrección no cambia ese caso, solo el
  de ítems que se quedaron sin terminar.
- **El pedido cancelado es el único pedido de la sesión**: la mesa debe poder liberarse igual que si
  nunca hubiera tenido ningún pedido — mismo resultado final que hoy ya produce una mesa sin
  ningún pedido en absoluto.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema NO DEBE contar los ítems de un pedido con `status = 'cancelada'` al validar
  si una mesa tiene "ítems sin terminar en cocina" para poder liberarse
  (`release_paid_session`/`_assert_closable`).
- **FR-002**: El sistema DEBE seguir rechazando la liberación de una mesa cuando un pedido **no
  cancelado** (incluido uno `'pagada'` cuya cocina siga en curso) tenga ítems sin terminar — sin
  cambios de comportamiento en ese caso (Principio II).
- **FR-003**: El sistema DEBE seguir rechazando la liberación de una mesa por cualquier otro motivo
  ya existente (por ejemplo, que todavía quede algo por cobrar) — esta corrección no afecta esas
  otras validaciones.
- **FR-004**: El sistema NO DEBE modificar `estado_cocina` de los ítems de un pedido al cancelarlo
  — la lógica de ajuste de inventario de `cancel_order`, que depende de ese estado para decidir si
  revertir o no el stock, permanece intacta (Principio V: no mezclar esta corrección con una
  refactorización de esa lógica).
- **FR-005**: El sistema NO DEBE mostrar una tarjeta de aviso nueva cuando ya haya una visible con
  el mismo tipo (éxito/error/información) y el mismo texto.
- **FR-006**: El sistema DEBE seguir mostrando avisos distintos (por texto o por tipo) como tarjetas
  independientes — la corrección del FR-005 no oculta información genuinamente distinta.
- **FR-007**: El sistema DEBE volver a mostrar un aviso cuyo texto y tipo coincidan con uno anterior
  ya desaparecido — la deduplicación del FR-005 no es permanente, solo evita duplicados
  simultáneos.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un cajero puede liberar cualquier mesa cuyo único obstáculo sea un pedido ya
  cancelado, sin necesitar ninguna acción manual adicional ni intervención de soporte.
- **SC-002**: Una mesa con un pedido `'pagada'` cuya cocina siga en curso sigue sin poder liberarse
  — 0% de regresión sobre este comportamiento ya protegido.
- **SC-003**: Ante un mismo error repetido, el cajero ve como máximo un aviso visible a la vez para
  ese error, en vez de una copia por cada intento.

## Assumptions

- El fix vive únicamente en la consulta que arma la lista de pedidos que se le pasa a
  `_assert_closable` dentro de `release_paid_session` — no se modifica `_assert_closable` en sí
  (sigue sirviendo sin cambios a `close_session`) ni ninguna otra función que ya filtra
  correctamente (`_billable_orders`, `has_billable_orders`).
- La deduplicación de avisos (Historia 2) es un cambio general sobre el mecanismo de notificaciones
  ya existente, aplicable a toda la aplicación — no un parche puntual solo para el botón "Liberar
  Mesa".
- Fuera de alcance de esta spec: cualquier cambio a `cancel_order` y su lógica de ajuste de
  inventario; a la posibilidad de "anular" ítems de un pedido ya cancelado (no existe hoy y esta
  spec no la agrega, porque deja de hacer falta); y a cualquier otra validación de cobro/cierre de
  sesión distinta de la cubierta aquí.
