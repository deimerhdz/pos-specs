# Feature Specification: Eliminación de Carritos al Liberarse la Mesa

**Feature Branch**: `039-eliminacion-carritos-cierre-mesa`

**Created**: 2026-08-26

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad adicional sobre `specs/038-vaciado-carrito-pedido/`
(Principio I de la [Constitución](../../.specify/memory/constitution.md)) — decisión de negocio
tomada "al momento de hacer pruebas" de esa spec, según indica quien encarga esta spec, ejerciendo
el rol de negocio en esta misma conversación (Principio XI). No corrige una anomalía ya registrada
en `registro-de-anomalias.md`: no existe ahí ninguna entrada sobre carritos huérfanos al liberar
mesa. Su registro formal (Principio II) queda **pendiente como paso previo a implementar**,
exigido sin condicional por la Constitución; esta spec documenta la decisión en su propio texto
como insumo para esa entrada, que le correspondería el número A-54 (la última registrada hoy es
A-53, de la spec 038).

La spec 038 solo eliminó físicamente el carrito del comensal en el único momento en que su pedido
se confirma con éxito (`submit_cart`, `app/api/v1/cart/service.py`). Dejó fuera de alcance, de
forma explícita, "la purga de carritos abandonados que nunca se convirtieron en pedido" (spec 038,
Fuera de Alcance). Esta spec cubre exactamente ese vacío, pero acotado a un único disparador: el
instante en que una mesa (`DiningTable`) pasa a `libre`. Hoy, ese instante lo produce
indistintamente cualquiera de estos cinco caminos, todos ya caracterizados por la spec 010
(`specs/010-sesion-mesa-reparto-cierre-barrido/spec.md`):

1. `try_release_if_empty` (`app/api/v1/table_sessions/service.py:88-130`) — liberación automática
   cuando el último comensal activo sale, su token expira, o cancela su último pedido vivo
   (`RN-CART-15`).
2. `close_session` (`app/api/v1/table_sessions/service.py:244-332`) — el cajero cobra y cierra la
   mesa manualmente.
3. `release_paid_session` (`app/api/v1/table_sessions/service.py:335-398`) — libera una mesa cuya
   sesión ya está completamente pagada (spec 028).
4. `release_table` (`app/api/v1/orders/checkout.py:697-736`, endpoint
   `POST /orders/tables/{table_id}/release`) — "Liberar Mesa" de staff, regla dura de cero órdenes
   no terminales.
5. `_sweep_schema` (`app/core/scheduler.py:88-165`) — el barrido automático, únicamente en la rama
   donde no queda nada por cobrar y no hay huérfanos (`RN-SCHED-01`/`RN-SCHED-02`, sin bloqueo de
   `RN-SCHED-03`/`RN-SCHED-04`).

En los cinco caminos, la mesa se libera **después** de que `close_table_sessions`/
`close_participants` (`app/api/v1/orders/checkout.py:635-693`) ya cerraron a los comensales y
marcaron sus carritos `abierto` como `abandonado` — nunca los borra. Esta spec **modifica
deliberadamente** el comportamiento protegido por un test `"CONGELA comportamiento actual:"` que
ejercita exactamente el primero de esos cinco caminos:

- `test_leave_session_cierra_participante_abandona_carrito_y_libera_mesa`
  (`app/characterization_tests/test_cart_service.py:466-481`) — asume que, cuando el último
  comensal sale y la mesa se libera en el acto, su carrito queda en `status='abandonado'`
  (línea 480) y sigue existiendo. Esta spec vuelve esa aserción falsa a propósito
  (FR-001/FR-002): el carrito deja de existir en el mismo instante en que la mesa queda `libre`.
  Se actualiza citando esta spec, siguiendo el mismo patrón que la spec 038 usó para sus dos tests
  CONGELA.

Ningún otro test `"CONGELA comportamiento actual:"` relacionado con cierre de mesa se ve afectado:
`test_close_participants_cierra_activos_y_devuelve_conteo`
(`app/characterization_tests/test_orders_checkout.py:435-456`) y
`test_close_table_sessions_no_valida_pendientes_rn_ord_31`
(`app/characterization_tests/test_orders_checkout.py:460-490`) ejercitan `close_participants`/
`close_table_sessions` de forma aislada, sin que la mesa llegue a liberarse en esos escenarios, así
que siguen protegiendo exactamente lo que protegen hoy — no forman parte de la decisión de esta
spec. `test_release_table_409_con_ordenes_activas_y_libera_sin_ellas`
(`app/characterization_tests/test_orders_checkout.py:494-519`) sí libera la mesa, pero no crea
ningún participante ni carrito en el escenario, así que tampoco se ve afectado.

**Input**: User description: en el flujo de trabajo actual, cuando se cierra una sesión de mesa y,
como consecuencia, el estado de la mesa pasa a quedar libre, los registros de la tabla `carts` de
esa sesión deben eliminarse — hoy quedan huérfanos y no sirven de nada guardados ahí. Deben
eliminarse únicamente cuando la sesión de la mesa haya terminado (la mesa quede libre); de lo
contrario, el flujo de trabajo actual debe seguir funcionando exactamente igual.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - La mesa se libera sola y sus carritos desaparecen con ella (Priority: P1)

Un comensal termina de comer y sale de la sesión (o su token expira, o cancela el único pedido que
tenía sin cobrar). Al ser el último activo y no quedar nada por cobrar, la mesa se libera
automáticamente para el siguiente cliente. En ese mismo instante, cualquier registro de carrito
que hubiera quedado de esa sesión —el suyo, el de otros comensales que ya se habían ido antes, o
el que el mesero hubiera consolidado en una comanda— deja de existir.

**Why this priority**: es el camino de liberación más frecuente (nadie de staff lo dispara a
propósito) y el que hoy acumula más carritos huérfanos sin que nadie lo note.

**Independent Test**: abrir una sesión de mesa, agregar ítems a un carrito sin confirmar pedido,
hacer que el comensal salga siendo el último activo sin nada por cobrar, y verificar que (a) la
mesa queda `libre` y (b) no existe ninguna fila de `Cart` para los participantes de esa sesión.

**Acceptance Scenarios**:

1. **Given** una mesa con un único comensal con un carrito sin confirmar, **When** el comensal sale
   de la sesión y no queda nada por cobrar, **Then** la mesa queda `libre` y el carrito de ese
   comensal deja de existir en la base de datos.
2. **Given** una mesa con dos comensales, uno con un carrito ya `abandonado` (salió antes) y otro
   con un carrito consolidado por el mesero (`confirmado`), **When** el segundo comensal sale
   siendo el último activo y no queda nada por cobrar, **Then** la mesa queda `libre` y ninguno de
   los dos carritos sigue existiendo.
3. **Given** una sesión cuyo único comensal cancela su último pedido activo, **When** esa
   cancelación deja la mesa sin nadie activo y sin nada por cobrar, **Then** la mesa se libera y el
   carrito de ese comensal se elimina en la misma operación.

---

### User Story 2 - La liberación manual de staff también limpia los carritos (Priority: P1)

El cajero cobra la mesa y la cierra, o usa "Liberar Mesa" sobre una sesión ya pagada, o el barrido
automático encuentra una mesa vencida sin nada pendiente. En cualquiera de estos casos, la mesa
queda `libre` exactamente igual que hoy, y además los carritos de esa sesión dejan de existir.

**Why this priority**: sin esto, la limpieza solo cubriría el camino más común (User Story 1) y
dejaría acumulando huérfanos exactamente los mismos caminos que hoy los generan a través de staff
y del barrido — el problema de negocio seguiría existiendo en la práctica.

**Independent Test**: para cada uno de los tres caminos (cobro manual, liberar mesa ya pagada,
barrido sin nada por cobrar), preparar una sesión con al menos un carrito huérfano y verificar que,
tras la liberación, la mesa queda `libre` y el carrito ya no existe.

**Acceptance Scenarios**:

1. **Given** una mesa con pedidos facturables y un carrito huérfano de un comensal ya cerrado,
   **When** el cajero cobra y cierra la sesión, **Then** la mesa queda `libre` y ese carrito deja
   de existir.
2. **Given** una sesión ya completamente pagada con un carrito huérfano, **When** staff la libera
   sin generar una venta nueva, **Then** la mesa queda `libre` y el carrito deja de existir.
3. **Given** una mesa vencida por inactividad sin nada por cobrar y con un carrito huérfano,
   **When** corre el barrido automático, **Then** la mesa queda `libre` y el carrito deja de
   existir.

---

### User Story 3 - Si la mesa no queda libre, ningún carrito se elimina (Priority: P1)

Una sesión se cierra o sus comensales se echan, pero la mesa **no** vuelve a `libre` porque todavía
queda algo por cobrar, o porque persiste un pedido huérfano de la misma mesa física. En cualquiera
de esos casos, ningún carrito se elimina — el comportamiento sigue siendo exactamente el de hoy
(los carritos pueden quedar `abandonado` por el cierre de comensales, que ya ocurre hoy y no
cambia, pero nunca se borran), para que el cajero pueda seguir viendo y cobrando la cuenta sin que
nada relacionado con esa sesión desaparezca antes de tiempo.

**Why this priority**: es la garantía negativa que evita que esta spec borre algo que el cajero
todavía necesita — sin ella, el "adicional" pedido podría convertirse en una pérdida de datos
sobre una cuenta todavía viva.

**Independent Test**: forzar cada uno de los dos frenos existentes (pedido facturable pendiente;
`CustomerOrder` huérfano no terminal de la misma mesa) y verificar que, aunque la sesión se cierre
o se eche a los comensales, la mesa sigue `ocupada` y ningún carrito de esos comensales se elimina.

**Acceptance Scenarios**:

1. **Given** una sesión vencida con un pedido `abierta` sin cobrar y un carrito huérfano de un
   comensal ya cerrado, **When** corre el barrido, **Then** el barrido solo cierra a los
   comensales, la mesa sigue `ocupada`, y el carrito huérfano sigue existiendo (con el mismo
   comportamiento de hoy: si estaba `abierto` pasa a `abandonado`, pero no se borra).
2. **Given** una sesión vencida sin nada por cobrar pero con un `CustomerOrder` no terminal
   huérfano de la misma mesa física (sin `table_session_id`), **When** corre el barrido, **Then**
   la sesión se cierra pero la mesa no vuelve a `libre`, y ningún carrito de esa sesión se elimina.
3. **Given** cualquier intento de liberar una mesa que falla y revierte su transacción (por
   cualquier motivo), **When** ocurre el rollback, **Then** ningún carrito de esa sesión queda
   eliminado ni modificado — todo permanece exactamente como estaba antes del intento.

---

### User Story 4 - La liberación de una mesa nunca toca los carritos de otra (Priority: P2)

En un local con varias mesas activas al mismo tiempo, cuando una de ellas se libera, los carritos
huérfanos de las demás mesas —todavía ocupadas, o ya libres desde antes— no se ven afectados en
absoluto.

**Why this priority**: es una garantía de aislamiento necesaria para que esta limpieza sea segura
en el caso real (varias mesas simultáneas), pero no bloquea la entrega de las Historias 1-3 para
una sola mesa.

**Independent Test**: con dos mesas activas, cada una con su propio carrito huérfano, liberar una
de las dos y verificar que el carrito de la otra mesa sigue existiendo sin cambios.

**Acceptance Scenarios**:

1. **Given** dos mesas activas, cada una con un carrito huérfano de un comensal ya cerrado,
   **When** una de las dos se libera, **Then** el carrito de la mesa que sigue ocupada permanece
   exactamente igual, sin eliminarse ni cambiar de `status`.

---

### Edge Cases

- **Carrito `abierto` que llegara a sobrevivir hasta el instante de liberar** (no debería ocurrir
  hoy porque `close_participants` ya lo marca `abandonado` antes de liberar la mesa, pero la regla
  no depende de ese orden de ejecución): también se elimina — la eliminación no distingue por
  `status`; cualquier fila de `Cart` de un participante de la sesión que se cierra desaparece junto
  con la liberación.
- **Comensal agregado por staff sin carrito propio** (sin QR, spec 010 FR-024): no tiene ninguna
  fila de `Cart` que eliminar; sin efecto.
- **Dos `TableSession` `active` cerrándose sobre la misma mesa en la misma operación**
  (`close_table_sessions` puede cerrar más de una sesión activa de la misma mesa a la vez): se
  eliminan los carritos de los participantes de todas las sesiones cerradas en esa operación, no
  solo de una.
- **Carritos huérfanos de mesas que ya están `libre` desde antes de desplegar esta spec**: no se
  purgan retroactivamente (ver Fuera de Alcance) — la eliminación solo aplica a la próxima vez que
  cada mesa se libere.
- **Participante de otra sesión, otra mesa o otro tenant**: nunca se ve afectado por la liberación
  de una mesa distinta (User Story 4).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Historia 1/2, decisión de esta spec]**: Cuando el sistema hace que el `status` de una
  `DiningTable` pase a `libre` como consecuencia de cerrar la(s) sesión(es) de mesa asociada(s) —
  por cualquiera de los caminos que hoy producen esa liberación (liberación automática al vaciarse
  la sesión, cierre y cobro manual por el cajero, liberación de una sesión ya pagada, "Liberar
  Mesa" de staff, o el barrido automático cuando no queda nada por cobrar) — el sistema DEBE
  eliminar de forma permanente (no marcar, no archivar) todas las filas de carrito y sus líneas
  asociadas que pertenezcan a los participantes de la(s) sesión(es) que se cerraron en esa misma
  operación, sin importar el `status` en el que se encuentre cada carrito (`abierto`, `abandonado`
  o `confirmado`). *(Modifica el comportamiento protegido por
  `test_leave_session_cierra_participante_abandona_carrito_y_libera_mesa`, citado en "Naturaleza
  de esta spec".)*
- **FR-002 [Regla crítica, Principio VIII — atomicidad]**: La eliminación de esos carritos DEBE
  ejecutarse dentro de la misma transacción que libera la mesa. Si la operación de cierre/
  liberación falla por cualquier motivo y se revierte, ningún carrito debe quedar eliminado ni
  modificado — el estado de los carritos vuelve a ser exactamente el que tenían antes del intento.
- **FR-003 [Historia 3, negativo explícito, sin cambio]**: Cuando una sesión de mesa se cierra (o
  sus comensales se cierran) pero la mesa **no** pasa a `libre` —porque todavía queda un pedido
  facturable pendiente de cobro (`RN-SCHED-03`, spec 010 FR-031) o porque persiste un
  `CustomerOrder` no terminal huérfano de la misma mesa física (`RN-SCHED-04`, spec 010 FR-032)—
  el sistema NO DEBE eliminar ningún carrito. Los carritos de esos participantes DEBEN seguir
  comportándose exactamente igual que hoy (pueden quedar marcados `abandonado`, nunca borrados).
- **FR-004 [Historia 4, aislamiento]**: La eliminación DEBE alcanzar únicamente los carritos de los
  participantes de la(s) sesión(es) que efectivamente pasaron a `closed` como parte de esa
  liberación de mesa. Los carritos de participantes de cualquier otra sesión de mesa, de otra mesa
  física o de otro tenant NO DEBEN verse afectados en su contenido, su cantidad de filas ni su
  `status`.
- **FR-005 [Sin cambio, condición previa]**: Esta spec NO DEBE alterar ninguna de las condiciones
  que hoy determinan si una mesa pasa o no a `libre` (`RN-CART-15`/spec 010 FR-028,
  `RN-SCHED-01` a `RN-SCHED-04`/spec 010 FR-029 a FR-032) — el barrido, el cierre manual, "Liberar
  Mesa" y la liberación automática siguen decidiendo exactamente igual que hoy si la mesa se
  libera; la eliminación de carritos es una consecuencia añadida sobre esa decisión ya existente,
  nunca una condición adicional para tomarla.

### Key Entities *(include if feature involves data)*

- **Carrito (Cart)**: a partir de esta spec, ninguna fila de `Cart` sobrevive a la mesa cuya sesión
  la originó una vez que esa mesa vuelve a `libre` — complementa a la spec 038, que ya eliminaba el
  carrito en el momento en que su pedido se confirmaba con éxito; esta spec elimina lo que quedara
  pendiente (carritos nunca confirmados, o confirmados por la consolidación del mesero) en el
  momento en que la sesión de mesa a la que pertenecen realmente termina.
- **TableSession**: sesión de mesa cuyo cierre (`status='closed'`) es condición necesaria pero no
  suficiente para que se eliminen los carritos de sus participantes — solo ocurre si, además, la
  mesa física asociada pasa a `libre` en esa misma operación (FR-003).
- **SessionParticipant**: comensal de una sesión; sus carritos (vía `participant_id`) son la unidad
  que esta spec elimina cuando su sesión termina con la mesa libre.
- **DiningTable**: mesa física cuyo `status` pasando a `libre` es el único disparador de esta spec
  — gobernado, sin cambios, por los mismos cinco caminos ya caracterizados por la spec 010.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las mesas que pasan a `libre` quedan, en la misma operación, sin ninguna
  fila de carrito asociada a los participantes de la sesión que se cerró.
- **SC-002**: El 100% de los cierres de sesión en los que la mesa permanece ocupada (algo pendiente
  por cobrar, o huérfanos sin resolver) conservan sus carritos exactamente en el mismo estado y
  cantidad que tenían antes del intento.
- **SC-003**: El 100% de los intentos de liberar una mesa que terminan en error dejan los carritos
  de esa sesión intactos, sin ninguna eliminación parcial.
- **SC-004**: El 0% de las liberaciones de una mesa modifican el contenido, la cantidad de filas o
  el `status` de los carritos de cualquier otra sesión, mesa o tenant.

## Out of Scope

- **Purgar retroactivamente los carritos huérfanos que ya existen hoy** en mesas que ya están
  `libre` desde antes de desplegar esta spec — esta spec solo actúa hacia adelante, en la próxima
  vez que cada mesa se libere; un backfill de los ya existentes, si se decide, sería trabajo
  independiente.
- **Cambiar cuándo o bajo qué condiciones una mesa pasa a `libre`** — las reglas de liberación
  (`RN-CART-15`, `RN-SCHED-01` a `RN-SCHED-04`) ya definidas por la spec 010 no se tocan; esta spec
  solo añade una consecuencia sobre una decisión que el sistema ya toma igual que hoy.
- **El vaciado del carrito al confirmar un pedido con éxito** — ya resuelto por la spec 038
  (`submit_cart`); esta spec no modifica ese camino.
- **Registrar formalmente esta decisión en `registro-de-anomalias.md`** — sigue pendiente como paso
  previo a la implementación, exigido sin condicional por el Principio II de la Constitución; esta
  spec documenta la decisión en su propio texto ("Naturaleza de esta spec") como insumo para esa
  entrada.

## Assumptions

- **"Los cinco caminos que hoy liberan una mesa" son los mismos y los únicos que existen en el
  código a la fecha de esta spec** (`try_release_if_empty`, `close_session`, `release_paid_session`,
  `release_table`, `_sweep_schema`, citados en "Naturaleza de esta spec") — si en el futuro aparece
  un sexto camino que asigne `DiningTable.status = 'libre'`, esta spec no lo cubre automáticamente;
  debería extenderse explícitamente. Riesgo si es incorrecta: un camino nuevo dejaría carritos
  huérfanos sin que esta spec lo detecte.
- **La eliminación no depende de que `close_participants` haya marcado antes los carritos
  `abierto` como `abandonado`** — se elimina cualquier fila de `Cart` del participante sin importar
  su `status` en ese instante, para no depender de un orden de ejecución interno que esta spec no
  controla.
- **No existe ningún consumidor que dependa de que un carrito huérfano siga existiendo después de
  que su mesa se libera** — verificado revisando los usos de `Cart` fuera del módulo `cart`
  (`orders/consolidation.py`, `orders/checkout.py`), que solo consultan carritos `abierto` de
  sesiones todavía activas.
