# Feature Specification: Sesión de mesa, reparto entre comensales, cierre unificado/dividido y barrido de sesiones abandonadas

**Feature Branch**: `010-sesion-mesa-reparto-cierre-barrido`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/table_sessions/service.py` y `app/core/scheduler.py`, para que sirva de
contrato formal de cara a la modernización (Principio I y Principio III de la
[Constitución](../../.specify/memory/constitution.md)). Es el dominio de "cuánto se le debe
cobrar a esta mesa, quién paga qué parte, y qué hace el sistema con una mesa que nadie cerró a
mano": el cierre de sesión con lock optimista (la implementación **vigente y correcta** de la
cuenta, frente a los dos caminos divergentes documentados en las specs 008 y 009), el reparto por
ítem/unidad, la regla protegida A-15 que blindó cuatro huecos de seguridad del cobro dividido
antes de dárselo a los cajeros, y el barrido automático que libera mesas abandonadas sin pisar
dinero pendiente de cobro. Incluye una regla protegida (A-15), una anomalía dudosa sin decisión
de negocio (A-17, porción de mesa) y dos anomalías `PENDIENTE` documentadas sin especificar
(A-29, A-38).

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE
de la sesión de mesa, el reparto de cuenta entre comensales, el cierre unificado/dividido, y el
barrido automático de sesiones abandonadas del sistema POS Heladería, tomado de
`reglas-de-negocio.md` (RN-MESA-01 a RN-MESA-27, RN-SCHED-01 a RN-SCHED-09) y de
`registro-de-anomalias.md` (A-11, A-15, A-17, A-28, A-29, A-38), para que sirva de contrato en la
modernización."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `table_sessions/service.py` y
  `core/scheduler.py`, no uno deseado. Las anomalías conocidas se marcan inline con su
  tratamiento acordado (registro-de-anomalias.md). A-15 es la regla protegida más importante de
  esta spec: cuatro huecos de seguridad reales, cerrados antes de dar a los cajeros la capacidad
  de armar bloques de pago manualmente, con su propio script de blindaje como base de test.
-->

### User Story 1 - `close_session` es la fuente de verdad de "cuánto se le debe cobrar a esta mesa" (Priority: P1)

Al cerrar/cobrar una mesa, el sistema bloquea la fila de la sesión (`SELECT...FOR UPDATE`) antes
de comprobar que siga activa, calcula la cuenta excluyendo pedidos anulados y ya pagados, aplica
promociones automáticamente, y rechaza el cierre si no hay nada cobrable o si queda comida sin
confirmar o en curso en cocina. Es la implementación **vigente y correcta** de esa pregunta de
negocio — dos rutas paralelas (documentadas en las specs 008 y 009) responden lo mismo de forma
distinta y divergente.

**Why this priority**: es el núcleo económico de la spec — un error aquí cobra de más, cobra de
menos, o cobra dos veces la misma mesa. El propio código lo declara explícitamente: "el lock es
obligatorio en el cierre: sin él, dos `POST /close` concurrentes leen ambos `status == 'active'` y
cobran la mesa dos veces".

**Independent Test**: se puede probar invocando `close_session(db, table_session_id, data,
cashier)` sobre una sesión con pedidos en distintos estados (confirmado y listo, sin confirmar,
en cocina) y verificando el 409/cálculo resultante, sin depender de cocina física ni del
frontend.

**Acceptance Scenarios**:

1. **Given** dos cajeros que presionan "Cobrar" casi al mismo tiempo sobre la misma mesa, **When**
   ambas peticiones `POST /close` llegan casi simultáneas, **Then** la primera toma el lock, cobra
   y comitea (`closed`); la segunda espera el lock, lo adquiere después, ve `status="closed"` y
   recibe `409` — nunca se genera una segunda venta (`RN-MESA-01`).
2. **Given** una sesión sin ningún pedido cobrable (comensal se sentó sin pedir), **When** se
   intenta cerrar, **Then** el sistema rechaza con `409` "La sesión no tiene pedidos que cobrar",
   antes de evaluar cualquier otra condición (`RN-MESA-03`).
3. **Given** una sesión con un pedido `recibida` (sin confirmar por cocina), **When** se intenta
   cerrar, **Then** el sistema rechaza con `409` y el detalle de los `order_ids` sin confirmar
   (`RN-MESA-04`).
4. **Given** una sesión con todos los pedidos confirmados pero un ítem todavía `pendiente` o
   `en_preparacion`, **When** se intenta cerrar, **Then** el sistema rechaza con `409` "Hay ítems
   sin terminar en cocina; anúlalos o espera a que estén listos" (`RN-MESA-04`).
5. **Given** una sesión con ítems anulados y ya pagados en pedidos previos, **When** se calcula la
   cuenta (`compute_bill` de este módulo, o el cobro real), **Then** ambos quedan excluidos del
   total — solo entra lo que sigue vivo y sin anular (`RN-MESA-16`, evidencia: la propia consulta
   de `_billable_orders` excluye `cancelada`/`pagada`, y `compute_bill` salta `estado_cocina ==
   "anulado"`).
6. **Given** el preview de la cuenta (`GET /table-sessions/{id}/bill`) y el cobro real dividido,
   **When** se comparan sus cálculos por comensal, **Then** usan exactamente la misma lógica —
   el preview nunca puede mostrar un total distinto del que se cobrará de verdad (`RN-MESA-16`).

**Nota — divergencia documentada con otras specs (A-01)**: existen otras dos implementaciones de
"cuánto debe una mesa": `orders/checkout.compute_bill` (camino B, sin descuentos, sin caller
confirmado — documentado en spec 008 como código muerto candidato a retiro) y
`orders/tables_advanced.group_bill` (camino C, sin descuentos, sin excluir pagadas, **en uso real
para mesas fusionadas** — documentado en spec 009). Esta spec fija explícitamente que
`table_sessions.compute_bill` (el de este módulo) es el cálculo **correcto y vigente**: excluye
anuladas/pagadas y aplica promociones. La corrección de los otros dos caminos, si el negocio la
autoriza, queda fuera del alcance de esta spec — se documenta aquí solo como referencia de cuál es
la convención de la que las otras dos divergen.

---

### User Story 2 - El reparto de cuenta es por ítem/unidad, nunca por división automática (Priority: P1)

No existe ninguna función de "dividir el total entre N comensales" matemáticamente. Repartir la
cuenta exige asignar explícitamente cada línea, o cada unidad de una línea, a un comensal
concreto. La suma de lo repartido debe cuadrar exactamente con la cantidad original de la línea,
y solo se puede partir por unidades un ítem que cocina ya terminó.

**Why this priority**: es el mecanismo base sobre el que se construye todo cobro dividido — si el
reparto pudiera descuadrar la cantidad de una línea, descuadraría el inventario ya descontado al
confirmar el pedido (la cantidad no cambia, solo se redistribuye entre filas).

**Independent Test**: se puede probar invocando `set_assignments` con distintas combinaciones de
reparto sobre una línea de `quantity=2` y verificando tanto el 422 de suma incorrecta como el
resultado exitoso, sin pasar por cocina ni por el cobro final.

**Acceptance Scenarios**:

1. **Given** una cuenta de $10.00 entre 3 comensales, **When** se busca un botón de "dividir entre
   3", **Then** no existe — hay que asignar ítem por ítem o unidad por unidad; 3 conos idénticos
   de $3.335 c/u repartidos 1 a cada uno cobran exactamente $3.335 sin redondeo (`RN-MESA-05`).
2. **Given** una línea de `quantity=2`, **When** se reparte A=1, B=2 (suma 3≠2), **Then** el
   sistema rechaza con `422` "El reparto de este producto suma 3 unidad(es) y la línea tiene 2" —
   ninguna asignación se aplica, la cantidad total no se toca (`RN-MESA-06`).
3. **Given** un ítem `en_preparacion`, **When** se intenta repartirlo por unidades entre dos
   comensales, **Then** el sistema rechaza con `422` "No se puede repartir por unidades un
   producto que cocina aún no ha terminado" — pero asignarlo **entero** a un solo comensal sí se
   permite en cualquier estado (`RN-MESA-07`).
4. **Given** un ítem `anulado`, **When** se intenta asignarlo a cualquier comensal, **Then** el
   sistema rechaza con `422` "Un producto anulado no se cobra, así que no se puede asignar"
   (`RN-MESA-08`).
5. **Given** un ítem de otra mesa, o de un pedido ya `cancelada`/`pagada`, **When** se intenta
   incluirlo en el reparto, **Then** el sistema rechaza con `422` "El producto ... no pertenece a
   la cuenta de esta mesa" — el universo asignable excluye explícitamente ambos casos
   (`RN-MESA-09`).
6. **Given** un reparto válido de una línea de 2 unidades en 1+1, **When** se aplica, **Then** la
   línea original queda con 1 unidad y nace una fila nueva con la otra, copiando precio, opciones
   y estado de cocina — la cantidad total sigue siendo 2 y el stock no se toca de nuevo
   (`RN-MESA-06`).

---

### User Story 3 - A-15 [PROTEGIDA]: cuatro huecos de seguridad del cobro dividido, cerrados antes de dar a los cajeros la capacidad de armarlo a mano (Priority: P1)

Antes de que los cajeros pudieran construir bloques de pago dividido manualmente desde el POS, se
cerraron cuatro huecos de seguridad que existían "desde que se implementó el split" pero eran
difíciles de alcanzar hasta entonces: comensales repetidos en `splits` causaban doble cobro y
doble factura; montos de nivel raíz (`tip`, `discount`, `tax`, `payments`) con
`billing_mode='split'` se ignoraban en silencio y el cajero perdía la propina sin enterarse;
importes negativos no se validaban; y el bloque sin comensal asignado (el que teclea el mesero)
salía sin nombre en la factura.

**Why this priority**: es la regla `[PROTEGIDA]` más importante de esta spec — dos testigos
(código + `memoria-historica.md` entrada #11, 2026-08-04, commit `42b5dec3`, Deimer Hernandez).
**Confirmado en la entrevista de negocio (P12, cajero jefe)**: sin ventana de exposición real —
la capacidad de armar bloques manuales se empezó a usar en producción **después** del blindaje,
no antes. Reintroducir cualquiera de los cuatro huecos durante la modernización reproduciría
exactamente los bugs ya corregidos, con el agravante de que ahora sí hay un caller real que los
puede disparar.

**Independent Test**: se puede probar completamente ejecutando
`python -m app.scripts.test_split_blindaje` contra un `pos-backend` en ejecución — el script
cubre las cuatro defensas con datos desechables propios y es, según el propio registro de
anomalías, "la base más sólida de todo el reconocimiento para proteger una regla `PROTEGIDA`".

**Acceptance Scenarios**:

1. **Given** un cierre `split` con el mismo `participant_id` repetido en dos bloques de `splits`,
   **When** se invoca `close_session`, **Then** el sistema rechaza con `422` "Hay comensales
   repetidos en el split: cada uno paga una sola vez", **no se crea ninguna venta** y la sesión
   sigue `active` (`RN-MESA-12`). **Verificado por** `test_split_blindaje.py`, blindaje 1.
2. **Given** un cierre `split` con `tip="5.00"` puesto en la raíz del payload (fuera de los
   bloques), **When** se invoca `close_session`, **Then** el sistema rechaza con `422` — la
   propina nunca se cobra ni se reparte en silencio (`RN-MESA-11`). **Verificado por**
   `test_split_blindaje.py`, blindaje 2. Lo mismo aplica a `discount`, `tax` y `payments` de
   raíz.
3. **Given** un payload de cierre con `discount="-5.00"`, **When** se valida el esquema
   `CloseSessionIn`, **Then** la validación de Pydantic lo rechaza antes de llegar al servicio —
   los importes negativos no pasan (`RN-MESA-11`, campo `ge=0`). **Verificado por**
   `test_split_blindaje.py`, blindaje 3.
4. **Given** un `split` con un bloque de `participant_id=None` (lo que teclea el mesero sin
   asignar a nadie), **When** se cobra exitosamente, **Then** la venta de ese bloque se factura a
   nombre de "Mesa N" o "Sin asignar" — nunca sale sin nombre (`RN-MESA-18`). **Verificado por**
   `test_split_blindaje.py`, blindaje 4.
5. **Given** un `split` que cubre exactamente el conjunto de comensales con consumo (ni falta, ni
   sobra, ni se repite ninguno), **When** se cierra la sesión, **Then** el cobro procede y emite
   una venta por cada bloque (`RN-MESA-12`). **Verificado por** `test_split_blindaje.py`, caso
   exitoso final ("el cierre válido emite 3 ventas").

**Especificación de esta spec**: los cuatro chequeos (RN-MESA-11, RN-MESA-12, RN-MESA-18) se fijan
como invariantes de test obligatorios en cualquier reimplementación — **especificar tal cual, no
tocar**, precisamente porque ya fueron huecos reales con impacto económico directo (doble cobro,
propina perdida).

---

### User Story 4 - El dinero: el vuelto solo sale de efectivo, y cerrar una mesa es todo-o-nada (Priority: P1)

El total de cualquier venta (mostrador, unificada o cada bloque de dividida) nunca puede ser
negativo. El pago debe cubrir el total exacto o más — no se permite cobro parcial. Si el pago
incluye efectivo de sobra, el sistema calcula vuelto; pero si lo que sobra viene de un medio
electrónico (tarjeta, transferencia), se rechaza — el vuelto nunca sale de un pago "de más" que no
sea efectivo. Cerrar la sesión completa (ventas, pedidos a `pagada`, cierre de comensales,
liberación de mesa) ocurre en una única transacción: si cualquier paso falla, se revierte todo.

**Why this priority**: son las reglas con impacto económico más directo e inmediato de todo el
módulo — un fallo aquí es dinero real mal calculado o mal repartido en el mismo cobro.

**Independent Test**: se puede probar invocando `close_session` con distintas combinaciones de
pago (insuficiente, con sobra en efectivo, con sobra electrónica) sobre una cuenta de total
conocido, y verificando el resultado o el rechazo sin depender de cocina ni de otros módulos.

**Acceptance Scenarios**:

1. **Given** una cuenta de $15.000 con un descuento de $20.000 sin impuesto ni propina, **When**
   se intenta cobrar, **Then** el sistema rechaza con `422` "El total no puede ser negativo" —
   mismo mecanismo que rige mostrador (`RN-MESA-19`).
2. **Given** una cuenta de $50.000, **When** se paga con un único pago de $45.000, **Then** el
   sistema rechaza con `422` "El pago (45000) no cubre el total (50000)"; no se comitea nada — ni
   venta, ni cambio de estado de pedidos, ni liberación de mesa (`RN-MESA-20`).
3. **Given** una cuenta de $30.000 pagada con tarjeta por $35.000, **When** se cobra, **Then** el
   sistema rechaza con `422` "Los pagos que no son en efectivo (35000) no pueden superar el total
   (30000): el vuelto solo sale del efectivo" (`RN-MESA-21`).
4. **Given** la misma cuenta de $30.000 pagada con $35.000 en efectivo, **When** se cobra,
   **Then** el sistema acepta el pago y calcula $5.000 de vuelto sin rechazo (`RN-MESA-21`,
   contraste positivo).
5. **Given** un cierre `split` de 3 comensales donde el bloque de Ana y de Luis se arman sin
   problema pero el de Marta trae un pago insuficiente, **When** `build_sale` lanza `422` a mitad
   del bucle sobre el bloque de Marta, **Then** toda la transacción se revierte — no quedan
   ventas parciales de Ana ni de Luis, la mesa no cambia de estado (`RN-MESA-22`).
6. **Given** cualquier cierre exitoso de sesión, **When** se inspecciona `cash_movements` del
   turno, **Then** no se escribió ningún movimiento manual — el dinero del cierre lo deriva
   `reconcile` desde `Payment`, y escribir además un movimiento lo contaría dos veces
   (`RN-MESA-23`).

---

### User Story 5 - `unified` exige pagos en la raíz; el nombre de la factura sigue un orden de prioridad (Priority: P2)

`billing_mode=unified` exige que el payload traiga `payments` (rechaza `422` si viene vacío). El
nombre a quien se factura la cuenta unificada sigue un orden de prioridad explícito: lo que
escriba el cajero manda primero; si no escribe nada, se usan los nombres de los comensales con
consumo real, ordenados alfabéticamente; si tampoco hay comensales identificados (mesa atendida
solo por el mesero, sin nadie que escaneó QR), se usa el número de la mesa. Cada bloque de un
cierre dividido calcula sus propias promociones y descuentos de combo de forma independiente,
sin mezclarse con los de otro comensal.

**Why this priority**: son reglas de presentación y de encaje de `unified`/`split` que completan
el contrato de `close_session`, pero de menor riesgo económico directo que las de User Story 1-4.

**Independent Test**: se puede probar invocando `close_session` con `billing_mode=unified` sin
`payments` (verificar el `422`), y por separado variando `customer_name` y los comensales con
consumo para verificar el orden de prioridad del nombre de factura.

**Acceptance Scenarios**:

1. **Given** un cierre con `billing_mode="unified"` y `payments=[]`, **When** se invoca
   `close_session`, **Then** el sistema rechaza con `422` "billing_mode='unified' requiere
   'payments'" (`RN-MESA-10`).
2. **Given** una sesión sin nombre escrito por el cajero, con Ana y Luis como únicos comensales
   con consumo, **When** se calcula el nombre de la factura, **Then** resulta "Ana, Luis" (orden
   alfabético) (`RN-MESA-17`).
3. **Given** la misma sesión pero con el cajero escribiendo "Constructora XYZ S.A." como nombre,
   **When** se calcula el nombre de la factura, **Then** resulta exactamente ese texto, aunque
   Ana y Luis tengan consumo — el cajero siempre tiene prioridad (`RN-MESA-17`).
4. **Given** una mesa atendida solo por el mesero, sin ningún comensal identificado con consumo y
   sin nombre escrito por el cajero, **When** se calcula el nombre, **Then** resulta "Mesa N"
   (`RN-MESA-17`).
5. **Given** un comensal que agregó un combo distinto al de otro comensal en el mismo cierre
   `split`, **When** se calculan los descuentos de cada bloque, **Then** el combo de uno no se
   mezcla con las líneas del otro — cada bloque evalúa promociones y combos únicamente sobre sus
   propias líneas (`RN-MESA-14`).

---

### User Story 6 - A-11: la prohibición total de descuento manual del cajero también rige aquí (Priority: P2) — regla compartida

El campo de descuento (`discount`) que acepta el cierre de sesión (tanto `unified` como cada
bloque de `split`) no tiene, en el esquema, ningún tope superior propio de este módulo — el único
freno hoy es que el total resultante no quede negativo (User Story 4, `RN-MESA-19`). Esta misma
carencia existe en los otros caminos de cobro del sistema (mostrador, `pay_order` legado), y el
negocio ya decidió, para los tres por igual, que el cajero **no debería poder aplicar descuento
manual en absoluto**.

**Why this priority**: la regla en sí (el mecanismo de tope o prohibición) se especifica
formalmente en la **spec 011** (constructor de venta compartido), que es donde vive el mecanismo
común a los tres caminos. Esta spec documenta únicamente que el cierre unificado y dividido de
mesa comparte exactamente la misma carencia y queda alcanzado por la misma decisión de negocio —
no la respecifica de cero.

**Independent Test**: se puede verificar inspeccionando `discount` en `CloseSessionIn` y en el
esquema de cada bloque de `splits`, comparando que el patrón (sin `le=`) es idéntico al que
motivó A-11 en `sales/schemas.py`.

**Acceptance Scenarios**:

1. **Given** el esquema de cierre de sesión, **When** se inspecciona el campo `discount` (tanto
   de la raíz `unified` como de cada bloque `split`), **Then** solo exige `ge=0` — ningún límite
   superior propio, mismo patrón que motivó A-11 en `sales/schemas.py`.
2. **Given** la pregunta de alcance repreguntada al negocio en la ronda 3 (simulada, P30): "¿la
   prohibición de descuento manual aplica a los tres caminos de cobro?", **When** el negocio
   responde, **Then** confirma explícitamente que sí, a los tres por igual (mostrador, unificado,
   dividido) — sin excepción para el cierre de mesa.

**Especificación de esta spec**: la regla de negocio en sí (tope o prohibición del descuento
manual, y su mecanismo de aplicación) **no se especifica aquí** — es responsabilidad de la spec
011. Esta spec únicamente deja registrado que el alcance decidido por el negocio incluye
explícitamente el cierre unificado y dividido de mesa.

---

### User Story 7 - A-17 (porción mesa): el reparto de comensales no toma lock de fila, a diferencia del cierre (Priority: P2) — `[DUDOSA]`, decisión abierta

`add_participant`, `remove_participant` y `set_assignments` cargan la sesión sin `lock=True`
(`with_for_update`); solo comprueban `status != "active"` sobre una lectura no bloqueada. Un
cierre en curso (lock tomado, sin commit aún) no bloquea una petición concurrente de reparto: por
las garantías MVCC de PostgreSQL, esa lectura no bloqueada puede ver el estado previo al commit
del cierre y reasignar un ítem cuyas líneas de venta ya fueron calculadas.

**Why this priority**: es una condición de carrera real y verificable por código, pero de
ocurrencia rara en la operación diaria (exige que dos miembros del staff operen la misma mesa casi
simultáneamente). Prioridad P2, no P1, porque a diferencia de User Story 1 (que sí tiene lock), no
hay evidencia de que se haya materializado en producción.

**Independent Test**: se puede verificar por inspección de código —comparando `_load(db, id,
lock=True)` en `close_session` (línea 228) contra `_load(db, id)` sin `lock=True` en
`add_participant` (línea 335), `remove_participant` (línea 370) y `set_assignments` (línea 403)—
sin necesidad de reproducir la condición de carrera con datos reales.

**Acceptance Scenarios**:

1. **Given** el código de `close_session`, **When** se inspecciona la carga de la sesión,
   **Then** usa `_load(db, table_session_id, lock=True)`, que aplica `SELECT...FOR UPDATE`
   (`RN-MESA-01`, User Story 1).
2. **Given** el código de `add_participant`, `remove_participant` y `set_assignments`, **When**
   se inspecciona la carga de la sesión en cada uno, **Then** ninguno pasa `lock=True` — la
   comprobación de `status != "active"` se hace sobre una lectura no bloqueada
   (`RN-MESA-02 [DUDOSA]`, `table_sessions/service.py:38-55,335,370,403`).
3. **Given** un cierre de mesa con el lock tomado pero sin commit todavía, **When** llega una
   petición concurrente de `PUT /assignments` sobre la misma mesa, **Then** su `SELECT` no se
   bloquea (MVCC no bloquea lectores) y puede ver `status="active"` (el commit del cierre aún no
   es visible), permitiendo reasignar un ítem cuyas líneas de venta el cierre ya calculó
   (`RN-MESA-02`).

**Nota — pregunta de negocio sin cerrar**: a diferencia de A-15 (User Story 3, cerrada en P12) y
de A-11 (User Story 6, cerrada en ronda 3), la pregunta que desbloquearía esta anomalía —¿existe
algún mecanismo externo (por ejemplo, un cliente restringido a una sola pantalla por mesa) que en
la práctica serialice el reparto de cuenta con el cierre de sesión?— **sigue sin decisión
concluyente**. No formó parte de las preguntas cerradas en la ronda 1-2 ni en la ronda 3
(simulada) de entrevista de negocio; queda pendiente de incluirse en la próxima ronda real de
negocio.

**Tratamiento acordado** (`registro-de-anomalias.md`, A-17): esta spec **documenta sin
especificar como contrato deseado**. La corrección propuesta si el negocio la autoriza —añadir
`with_for_update()` a `add_participant`, `remove_participant` y `set_assignments`, consistente con
el patrón ya usado por `close_session`— se registra aquí como la dirección de corrección conocida,
no como comportamiento a implementar por esta spec.

---

### User Story 8 - Liberar una mesa exige dos condiciones simultáneas: nadie activo Y nada que cobrar (Priority: P2)

Una sesión de mesa vuelve a `libre` automáticamente solo cuando se cumplen **ambas** condiciones a
la vez: ningún comensal sigue `open`, y no queda ningún pedido en un estado distinto de
`pagada`/`cancelada`. Si falta cualquiera de las dos, la mesa sigue `ocupada` — es la única
función que decide esto, y la comparten el "salir" del comensal, la expiración de su token, la
cancelación de su último pedido, y el barrido periódico.

**Why this priority**: es el mecanismo de liberación que citan tanto el cierre manual como el
barrido automático (User Story 9); un fallo aquí libera una mesa con dinero pendiente de cobro, o
mantiene ocupada indefinidamente una mesa realmente vacía.

**Independent Test**: se puede probar invocando `try_release_if_empty(db, table_session_id)`
directamente sobre sesiones en distintas combinaciones de comensales activos y pedidos vivos, sin
pasar por el barrido ni por el frontend — que es exactamente lo que hace
`test_table_release.py`.

**Acceptance Scenarios**:

1. **Given** una mesa con dos comensales, **When** uno de los dos sale (`leave`) mientras el otro
   sigue `open`, **Then** la mesa sigue `ocupada` — falta la condición de "nadie activo"
   (`RN-CART-15`, evidencia: `table_sessions/service.py:74-116`, función
   `try_release_if_empty`). **Verificado por** `test_table_release.py`, caso 2: "sale uno de dos
   → la mesa SIGUE ocupada".
2. **Given** la misma mesa, **When** también sale el segundo comensal, **Then** la mesa se libera
   de inmediato (`RN-CART-15`). **Verificado por** `test_table_release.py`, caso 2 continuación:
   "sale el segundo → mesa libre".
3. **Given** un comensal que se va dejando un pedido `abierta` sin cobrar, **When** sale de la
   sesión, **Then** la mesa **sigue ocupada** aunque ya no quede nadie — es un descuadre real que
   debe ver el personal, no algo que se tape automáticamente (`RN-CART-15`). **Verificado por**
   `test_table_release.py`, caso 3.
4. **Given** un comensal que se va dejando un pedido ya `cancelada`, **When** sale de la sesión,
   **Then** la mesa se libera — un pedido cancelado no cuenta como "algo que cobrar"
   (`RN-CART-15`, contraste con el caso 3). **Verificado por** `test_table_release.py`, caso 4.
5. **Given** una sesión ya `closed`, **When** se invoca `try_release_if_empty` de nuevo sobre
   ella, **Then** no hace nada y devuelve `False` — es idempotente (evidencia:
   `table_sessions/service.py:91-93`). **Verificado por** `test_table_release.py`, caso 8.

---

### User Story 9 - RN-SCHED: el barrido cierra mesas abandonadas sin pedir a los 30 minutos exactos, y por tope duro a las 6 horas — pero nunca si queda algo por cobrar (Priority: P1)

Un job periódico recorre las sesiones activas de todos los tenants y cierra dos tipos de mesa
vencida: la que nadie tocó y nadie pidió nada en los últimos 30 minutos exactos de inactividad de
**todos** sus comensales, y la que lleva abierta más de 6 horas con o sin actividad (tope duro,
por si hay consumo que alguien olvidó cobrar). Si una sesión vencida por cualquiera de los dos
criterios todavía tiene pedidos facturables pendientes de cobro, el sistema **no** cierra la
sesión ni libera la mesa — solo expulsa a los comensales, dejando la cuenta abierta para que el
cajero la cobre.

**Why this priority**: es el mecanismo que evita que una mesa quede ocupada indefinidamente cuando
nadie pulsa "Salir", pero también el que, si falla en la dirección contraria, puede cerrar una
mesa con actividad real (ver User Story 10, A-28). Prioridad P1 porque protege dinero real: cerrar
de más pierde la cuenta a cobrar; no cerrar nunca bloquea mesas para nuevos clientes.

**Independent Test**: se puede probar invocando `sweep_orphan_sessions()` directamente sobre
sesiones envejecidas artificialmente (ajustando `expires_at` de los comensales o `opened_at` de
la sesión) — que es exactamente el mecanismo de `test_table_release.py`, casos 5 a 7.

**Acceptance Scenarios**:

1. **Given** un comensal que escanea el QR a las 10:00:00 sin pedir nada, **When** el barrido
   corre a las 10:30:00 exactas, **Then** la sesión se considera abandonada — la comparación
   `última_actividad > límite` ya es falsa en el instante exacto, así que el corte real es a los
   30:00 minutos, no después (`RN-SCHED-01`).
2. **Given** una sesión envejecida a `EMPTY_SESSION_TTL_MINUTES - 10` minutos de inactividad (aún
   dentro de la ventana), **When** corre el barrido, **Then** no la toca — la mesa sigue
   `ocupada` (`RN-SCHED-01`). **Verificado por** `test_table_release.py`, caso 6.
3. **Given** esa misma sesión pero envejecida a `EMPTY_SESSION_TTL_MINUTES + 1` minutos, **When**
   corre el barrido, **Then** la suelta — mesa `libre` (`RN-SCHED-01`). **Verificado por**
   `test_table_release.py`, caso 5.
4. **Given** una sesión abierta a las 09:00:00 con consumo, **When** el barrido corre a las
   15:00:01 (más de 6 horas después), **Then** entra en la lista de vencidas por tope duro **sin
   importar** si el comensal sigue interactuando activamente en ese momento (`RN-SCHED-02`,
   operador `<` estricto sobre `opened_at`).
5. **Given** una sesión vencida (por cualquiera de los dos criterios) con un pedido `abierta` de
   $20.000 sin pagar, **When** corre el barrido, **Then** el sistema **no** cierra la sesión ni
   libera la mesa — solo cierra a los comensales (`close_participants`), dejando la sesión
   abierta y la mesa ocupada para que el cajero la cobre desde la terminal (`RN-SCHED-03`).
   **Verificado por** `test_table_release.py`, caso 7: "con un pedido vivo el barrido NO la
   suelta".
6. **Given** una sesión vencida y sin nada que cobrar, cuya mesa además tiene un `CustomerOrder`
   no terminal huérfano (sin `table_session_id`, remanente histórico) asociado a la misma mesa
   física, **When** el barrido la cierra, **Then** la sesión se cierra pero la mesa **no** vuelve
   a `libre` — la segunda red de seguridad evita que quede incobrable (`RN-SCHED-04`).
7. **Given** cualquier cierre disparado por el barrido, **When** se inspecciona quién lo cerró,
   **Then** queda `closed_by=None` explícitamente — para distinguirlo en auditoría de un cierre
   manual hecho por un cajero (`RN-SCHED-05`).

---

### User Story 10 - El barrido es resiliente: multi-tenant, un solo worker por ciclo, y degrada con elegancia si faltan sus dependencias (Priority: P2)

El barrido recorre todos los tenants del sistema en el mismo ciclo; si uno falla (excepción no
controlada), los demás igual se procesan. Solo un worker ejecuta el barrido por ciclo gracias a un
lock distribuido en Redis, cuyo TTL es la mitad del intervalo configurado — así, si el proceso que
tomó el lock muere a medias, el siguiente ciclo puede retomarlo sin quedar bloqueado
indefinidamente. Si Redis no está disponible, el barrido de ese ciclo simplemente se omite (con
log de advertencia); si `APScheduler` no está instalado, la aplicación arranca igual, pero ninguna
sesión huérfana se cierra sola.

**Why this priority**: son garantías operativas de robustez del propio job, no del negocio directo
de la mesa — de ahí P2, un escalón por debajo de las reglas de qué mesa se cierra y cuándo (User
Story 9).

**Independent Test**: se puede verificar por inspección de código el manejo de cada excepción
(`try/except` alrededor de cada tenant, alrededor de la conexión a Redis, y alrededor del
`import` de `apscheduler`), sin necesidad de derribar Redis o desinstalar la dependencia en un
entorno real para confirmarlo.

**Acceptance Scenarios**:

1. **Given** varios tenants, uno de los cuales lanza una excepción no controlada durante su propio
   barrido, **When** `sweep_orphan_sessions()` recorre la lista, **Then** el error de ese tenant
   se registra en log y el recorrido continúa con los demás — un tenant roto no detiene el
   barrido de los demás (`RN-SCHED-06`).
2. **Given** `SESSION_SWEEP_INTERVAL_MINUTES=15` (valor por defecto), **When** se calcula el TTL
   del lock distribuido, **Then** resulta `max(15*60/2, 30)=450` segundos (7.5 minutos) — si el
   worker que tomó el lock muere a los 2 minutos, el lock expira a los 7.5 minutos, permitiendo
   que el siguiente ciclo (a los 15 minutos) lo retome sin bloqueo indefinido (`RN-SCHED-07`).
3. **Given** Redis inalcanzable en el momento de intentar tomar el lock, **When** se ejecuta el
   ciclo del barrido, **Then** el sistema registra un warning y omite el barrido de ese ciclo por
   completo, sin propagar la excepción (`RN-SCHED-08`). Aplica igual a la expiración de
   promociones de medianoche, que comparte el mismo patrón de lock.
4. **Given** un entorno donde el paquete `apscheduler` no está instalado, **When** arranca la
   aplicación, **Then** arranca igual (captura explícita de `ImportError`, con warning de log),
   pero ninguna sesión huérfana se cerrará automáticamente hasta que se instale la dependencia o
   se corra el barrido manualmente por cron (`RN-SCHED-09`).

**Nota — riesgo de configuración documentado (A-28)**: el comentario de `app/core/config.py`
declara explícitamente el invariante `SESSION_TTL_REFRESH_SLACK_MINUTES <
EMPTY_SESSION_TTL_MINUTES`, necesario para que el barrido de esta spec no cierre mesas
**activas**. No existe ningún `assert` ni validador de arranque que lo garantice — el único
chequeo vive en `test_session_ttl.py`, que no corre en CI. Esta spec documenta el riesgo tal como
existe hoy; **no lo corrige** (ver Assumptions).

---

### User Story 11 - A-29 (porción mesa): con dos o más combos distintos en un mismo cierre, se pierde la trazabilidad del combo específico (Priority: P3) — `PENDIENTE`, sin especificar

Si las líneas cobradas en un cierre `unified` (o en un bloque de `split`) usan exactamente un
combo distinto, `promotion_id` de la venta registra ese combo. Con cero combos, o con dos o más
combos distintos, el sistema no registra ningún combo específico en `promotion_id` — cae al
resultado de la evaluación general de promociones, que puede ser `None`. El descuento monetario de
todos los combos se suma correctamente al total en cualquier caso; solo se pierde la trazabilidad
en reportes agrupados por promoción/combo específico.

**Why this priority**: mismo mecanismo documentado también en las specs 008 y 011 para sus
respectivos caminos de cobro — sin impacto práctico confirmado (P21, entrevista de negocio: el
negocio no usa hoy el reporte de ventas por combo/promoción). Prioridad P3, la más baja de esta
spec.

**Independent Test**: se puede probar cobrando una venta unificada o un bloque de split cuyas
líneas incluyan dos `combo_id` distintos, e inspeccionando que `promotion_id` de la `Sale`
resultante queda `None` (o el resultado de `promotions.evaluate`) pese a que el descuento
monetario de ambos combos sí se sumó al total.

**Acceptance Scenarios**:

1. **Given** un cierre unificado cuyas líneas usan exactamente un `combo_id`, **When** se calcula
   `final_promotion_id`, **Then** registra ese combo específico (`RN-MESA-15`).
2. **Given** el mismo cierre pero con líneas que usan dos `combo_id` distintos, **When** se
   calcula `final_promotion_id`, **Then** no queda ligado a ninguno de los dos — cae al resultado
   de `promotions.evaluate` sobre las líneas sin combo, que puede ser `None`, aunque el
   descuento monetario de ambos combos sí se sumó correctamente al total (`RN-MESA-15
   [DUDOSA]`).

**Tratamiento acordado** (`registro-de-anomalias.md`, A-29): documentar sin especificar hasta
respuesta de negocio. Queda como decisión de negocio abierta si la pérdida de trazabilidad por
promoción en reportes es aceptable; esta spec no la fija como comportamiento deseado ni
obligatorio para la modernización.

---

### User Story 12 - A-38 (cluster, porción mesa): mesa de un solo comensal en modo `split`, y comensal con productos anulados que bloquea su propia salida (Priority: P3) — `PENDIENTE`, sin especificar

No existe ninguna validación de cardinalidad mínima para `billing_mode=split`: una mesa con un
solo comensal puede cerrarse por esa vía, lo que en la práctica equivale a un `unified` disfrazado
por el camino de `split`. Por separado, `remove_participant` cuenta cualquier `OrderItem` con el
`participant_id` del comensal sin filtrar por `estado_cocina` ni por el estado del pedido — puede
bloquear la eliminación de un comensal cuyo único "consumo" ya no es cobrable en absoluto (un
producto que cocina anuló después de asignárselo).

**Why this priority**: son hallazgos de bajo impacto individual, casos límite de uso operativo
diario más que riesgos activos — de ahí la prioridad P3, la más baja de esta spec, compartida con
User Story 11.

**Independent Test**: se puede probar invocando `close_session` con `billing_mode=split` y un
solo bloque para verificar la ausencia de rechazo por cardinalidad, y por separado invocando
`remove_participant` sobre un comensal cuyo único producto asignado está `anulado` para observar
el `409`.

**Acceptance Scenarios**:

1. **Given** una mesa con un único comensal con consumo, **When** se cierra con
   `billing_mode="split"` y un solo bloque cubriendo a ese comensal, **Then** el sistema no
   rechaza por cardinalidad mínima — el resultado es indistinguible en la práctica de un cierre
   `unified` (`RN-MESA-13 [DUDOSA]`).
2. **Given** un comensal ("Marta") al que se le asignó un helado que luego cocina anuló, **When**
   se intenta `DELETE participants/{marta_id}`, **Then** el sistema rechaza con `409` "«Marta»
   tiene 1 producto(s) asignado(s)" — el conteo real no excluye ítems anulados, aunque el
   mensaje hable de "producto(s) asignado(s)" en términos de algo cobrable, y ese producto en
   realidad nunca se cobrará (`RN-MESA-24 [DUDOSA]`).

**Tratamiento acordado** (`registro-de-anomalias.md`, A-38): documentar sin especificar. Ambas
quedan `PENDIENTE` de decisión de negocio (¿debería exigirse un mínimo de comensales para
`split`?; ¿debería `remove_participant` excluir ítems ya no cobrables al contar "productos
asignados"?) y esta spec no las fija como comportamiento deseado ni obligatorio para la
modernización.

---

### Edge Cases

- **Dos cierres concurrentes sobre la misma mesa**: resuelto por el lock optimista de
  `close_session` — el segundo espera y ve `closed` (`RN-MESA-01`, User Story 1).
- **Reparto de cuenta concurrente con un cierre en curso (lock tomado, sin commit)**: no
  protegido — condición de carrera real y documentada, pero sin decisión de negocio que cierre si
  es aceptable (`RN-MESA-02`, User Story 7).
- **Sesión vencida por el barrido con pedidos facturables**: nunca se cierra la sesión ni se
  libera la mesa; solo se expulsa a los comensales (`RN-SCHED-03`, User Story 9).
- **Sesión vencida y sin nada que cobrar, pero con un pedido huérfano histórico fuera de la
  sesión**: la sesión se cierra, la mesa no vuelve a `libre` (`RN-SCHED-04`, User Story 9).
- **Redis caído en el momento del barrido**: el ciclo completo se omite silenciosamente (con
  warning de log), no se reintenta dentro del mismo ciclo (`RN-SCHED-08`, User Story 10).
- **`APScheduler` no instalado**: la aplicación no se cae; simplemente ninguna sesión se cierra
  sola hasta correr el barrido manualmente (`app/scripts/sweep_sessions`) o instalar la
  dependencia (`RN-SCHED-09`, User Story 10).
- **Bloque de `split` sin comensal asignado (`participant_id=None`)**: no se prorratea ni se
  excluye — forma su propio grupo, con nombre de factura ("Mesa N"/"Sin asignar") garantizado por
  A-15 (`RN-MESA-18`, User Story 3).
- **Dos o más combos distintos en el mismo cierre**: el descuento monetario de ambos se suma
  correctamente, pero `promotion_id` no queda ligado a ninguno — ver User Story 11 (A-29,
  `PENDIENTE`).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Regla crítica, User Story 1]**: `close_session` DEBE tomar un lock `SELECT...FOR
  UPDATE` sobre la fila de la sesión antes de comprobar su estado, para impedir que dos cierres
  concurrentes cobren la misma mesa dos veces (`RN-MESA-01`).
- **FR-002 [`[DUDOSA]`, anomalía A-17, User Story 7 — documentada sin especificar como contrato]**:
  `add_participant`, `remove_participant` y `set_assignments` NO toman lock de fila al cargar la
  sesión, a diferencia de `close_session`; la pregunta de si algún mecanismo externo serializa
  esto en la práctica sigue sin decisión de negocio (`RN-MESA-02`).
- **FR-003**: `close_session` DEBE rechazar con `409` si la sesión no tiene al menos un pedido
  cobrable (`RN-MESA-03`).
- **FR-004**: `close_session` DEBE rechazar con `409` si algún pedido de la sesión sigue
  `recibida` (sin confirmar), o si algún ítem sigue `pendiente`/`en_preparacion` en cocina
  (`RN-MESA-04`).
- **FR-005**: El reparto de cuenta DEBE ser exclusivamente por asignación explícita de ítem o de
  unidad a un `participant_id`; el sistema NO DEBE ofrecer ninguna división porcentual o "a
  partes iguales" automática (`RN-MESA-05`).
- **FR-006**: Repartir una línea entre varios comensales DEBE exigir que la suma de las unidades
  pedidas sea exactamente igual a la cantidad original de la línea; si no cuadra, DEBE rechazarse
  con `422` sin aplicar ninguna asignación (`RN-MESA-06`).
- **FR-007**: Repartir una línea **por unidades** entre varios comensales NO DEBE permitirse si el
  ítem está `pendiente`/`en_preparacion`; reasignar el ítem **completo** a otro comensal SÍ DEBE
  permitirse en cualquier estado de cocina (`RN-MESA-07`).
- **FR-008**: Un ítem `anulado` NO DEBE poder asignarse a ningún comensal (`RN-MESA-08`).
- **FR-009**: El universo de ítems asignables en el reparto DEBE excluir los de otras mesas y los
  de pedidos ya `cancelada`/`pagada` (`RN-MESA-09`).
- **FR-010**: `billing_mode=unified` DEBE exigir `payments` no vacío; `billing_mode=split` DEBE
  exigir `splits` no vacío; ambos DEBEN rechazarse con `422` si faltan (`RN-MESA-10`).
- **FR-011 [Regla protegida A-15, invariante de test obligatorio]**: En `billing_mode=split`, los
  campos de nivel raíz (`discount`, `tax`, `tip`, `payments`) DEBEN estar prohibidos — el sistema
  DEBE rechazar con `422` si vienen poblados, sin ignorarlos en silencio. Cada bloque de `splits`
  lleva sus propios importes (`RN-MESA-11`).
- **FR-012 [Regla protegida A-15, invariante de test obligatorio]**: En `billing_mode=split`, cada
  comensal con consumo (ítem no anulado asignado, incluido `None` para "agregado por el mesero")
  DEBE aparecer **exactamente una vez** en `splits` — ni faltar, ni sobrar, ni repetirse. Un
  comensal repetido DEBE rechazarse con `422` antes de cobrar ninguna línea, para no cobrar sus
  mismas líneas dos veces (`RN-MESA-12`).
- **FR-013**: Cada bloque de un cierre `split` DEBE calcular sus propias promociones y descuentos
  de combo de forma independiente, sin mezclar líneas de comensales distintos (`RN-MESA-14`).
- **FR-014 [`[DUDOSA]`, anomalía A-29, `PENDIENTE` — documentada sin especificar como contrato]**:
  si las líneas cobradas en un cierre `unified` (o en un bloque `split`) usan exactamente un
  combo distinto, `promotion_id` DEBE registrar ese combo; con cero o con dos o más combos
  distintos, el sistema usa el resultado de la evaluación general de promociones (que puede ser
  `None`) — con dos o más combos, ninguno queda registrado individualmente, aunque el descuento
  monetario de todos se sume correctamente al total (`RN-MESA-15`).
- **FR-015**: El cálculo de la cuenta consolidada (`compute_bill` de este módulo, usado por el
  preview `GET /table-sessions/{id}/bill`) DEBE usar exactamente el mismo cálculo por comensal
  que el cierre `split` real, de forma que el preview coincida siempre con lo que se cobra
  (`RN-MESA-16`).
- **FR-016**: El nombre de la factura en un cierre `unified` DEBE resolverse en este orden de
  prioridad: (1) lo que escriba el cajero, si no está vacío; (2) los nombres de los comensales
  con consumo, unidos y ordenados alfabéticamente; (3) el número de la mesa, si no hay comensales
  identificados (`RN-MESA-17`).
- **FR-017 [Regla protegida A-15]**: En un cierre `split`, la venta de un bloque sin comensal
  asignado (`participant_id=None`) DEBE facturarse a nombre de "Mesa N" o "Sin asignar" — nunca
  sin nombre (`RN-MESA-18`).
- **FR-018**: El total de cualquier venta producida por el cierre de sesión (unificada o cada
  bloque dividido) NO DEBE poder ser negativo (`RN-MESA-19`).
- **FR-019**: El pago recibido DEBE cubrir el total exacto o más; el cobro parcial DEBE
  rechazarse (`RN-MESA-20`).
- **FR-020 [Regla crítica]**: El vuelto SOLO DEBE poder salir de un exceso pagado en efectivo;
  un exceso pagado por un medio electrónico DEBE rechazarse (`RN-MESA-21`).
- **FR-021 [Regla crítica]**: Cerrar una sesión (ventas, pedidos a `pagada`, cierre de comensales,
  liberación de mesa) DEBE ejecutarse como una única transacción todo-o-nada: si cualquier paso
  falla, DEBE revertirse por completo, incluidas las ventas ya armadas para otros comensales en
  la misma llamada (`RN-MESA-22`).
- **FR-022**: Cerrar una sesión de mesa NO DEBE escribir ningún movimiento de caja manual — el
  dinero cobrado lo deriva la reconciliación de turno desde los pagos, y un movimiento adicional
  lo contaría dos veces (`RN-MESA-23`).
- **FR-023 [`[DUDOSA]`, anomalía A-38, `PENDIENTE` — documentada sin especificar como contrato]**:
  `remove_participant` cuenta cualquier `OrderItem` con el `participant_id` del comensal, sin
  filtrar por `estado_cocina` ni por el estado del pedido; puede rechazar con `409` la
  eliminación de un comensal cuyo único "consumo" ya no es cobrable (producto anulado o pedido ya
  no vivo) (`RN-MESA-24`).
- **FR-024**: Un comensal agregado desde el POS por staff (sin QR) DEBE crearse sin `expires_at`
  y sin carrito propio — es una etiqueta de cobro, no puede loguearse ni pedir por su cuenta
  (`RN-MESA-25`).
- **FR-025**: Agregar o repartir comensales DEBE rechazarse si la sesión ya no está `active`
  (`RN-MESA-26`).
- **FR-026**: El nombre de un comensal agregado por staff DEBE rechazarse si queda vacío tras
  recortar espacios (`RN-MESA-27`).
- **FR-027 [`[DUDOSA]`, anomalía A-38, `PENDIENTE` — documentada sin especificar como contrato]**:
  `billing_mode=split` NO DEBE tener ninguna restricción de cardinalidad mínima de comensales; una
  mesa de un solo comensal puede cerrarse por esa vía, equivalente en la práctica a un `unified`
  disfrazado (`RN-MESA-13`).
- **FR-028 [Regla crítica]**: Una sesión de mesa DEBE volver a `libre` automáticamente solo cuando
  se cumplen simultáneamente dos condiciones: ningún comensal sigue `open`, y no queda ningún
  pedido en estado distinto de `pagada`/`cancelada`. Si falta cualquiera, la mesa DEBE permanecer
  ocupada — es la única función que decide esto, compartida por el "salir" del comensal, la
  expiración de token, la cancelación del último pedido y el barrido (`RN-CART-15`, evidencia en
  `table_sessions/service.py:74-116`, función `try_release_if_empty`).
- **FR-029 [Regla crítica]**: Una `TableSession` activa sin ningún pedido facturable DEBE
  considerarse abandonada cuando ningún comensal tiene actividad dentro de los
  `EMPTY_SESSION_TTL_MINUTES` (30) minutos previos; el corte real es a los 30:00 minutos exactos
  de inactividad, no después, por el uso del operador `>` estricto (`RN-SCHED-01`).
- **FR-030 [Regla crítica]**: Cualquier `TableSession` activa cuyo `opened_at` sea anterior al
  corte de `TABLE_SESSION_MAX_HOURS` (6) horas DEBE considerarse vencida por tope duro,
  independientemente de si hay actividad de comensales (`RN-SCHED-02`).
- **FR-031 [Regla crítica]**: Una sesión vencida (por `RN-SCHED-01` o `RN-SCHED-02`) que todavía
  tenga pedidos facturables pendientes de cobro NO DEBE cerrarse ni liberar la mesa — el sistema
  DEBE cerrar únicamente a los comensales, dejando la sesión abierta para que el cajero pueda
  cobrarla (`RN-SCHED-03`).
- **FR-032**: Al cerrar una sesión por barrido, la mesa DEBE volver a `libre` solo si, además,
  no existen `CustomerOrder` no terminales asociados a la misma mesa física que hayan quedado sin
  `table_session_id` (huérfanos históricos) — segunda red de seguridad (`RN-SCHED-04`).
- **FR-033**: El cierre de sesión disparado por el barrido DEBE atribuirse al sistema
  (`closed_by=None`), distinguible en auditoría de un cierre manual por un cajero (`RN-SCHED-05`).
- **FR-034**: El barrido DEBE ejecutarse sobre todos los tenants en el mismo ciclo; una excepción
  no controlada en el barrido de un tenant NO DEBE impedir que se procesen los demás
  (`RN-SCHED-06`).
- **FR-035**: Solo un worker DEBE ejecutar el barrido por ciclo, mediante un lock distribuido en
  Redis cuyo TTL DEBE ser la mitad del intervalo configurado del barrido, con un mínimo de 30
  segundos (`RN-SCHED-07`).
- **FR-036**: Si Redis no está disponible al intentar tomar el lock, el barrido (y la expiración
  de promociones de medianoche, que comparte el mismo patrón) DEBEN omitirse por completo en ese
  ciclo, registrando un warning, sin propagar la excepción (`RN-SCHED-08`).
- **FR-037**: Si `APScheduler` no está instalado, la aplicación DEBE arrancar igual (capturando
  `ImportError` explícitamente), sin que ninguna sesión huérfana se cierre automáticamente hasta
  instalar la dependencia o correr el barrido manual (`RN-SCHED-09`).
- **FR-038 [`[ACCIDENTAL]`, anomalía A-28, riesgo de configuración documentado — no corregido por
  esta spec]**: el invariante `SESSION_TTL_REFRESH_SLACK_MINUTES < EMPTY_SESSION_TTL_MINUTES`,
  necesario para que el barrido de `RN-SCHED-01` no cierre mesas activas, NO tiene ningún
  `assert` ni validador de arranque que lo garantice hoy. Esta spec documenta el riesgo tal como
  existe; su corrección (validación en el arranque) queda fuera del alcance de esta spec.

### Key Entities *(include if feature involves data)*

- **TableSession**: sesión de una mesa. Atributos relevantes: `status` (`active`/`closed`),
  `billing_mode` (`unified`/`split`, fijado al cerrar), `opened_at` (para el tope duro de
  `RN-SCHED-02`), `dining_table_id`. Su cierre exige lock de fila (`RN-MESA-01`).
- **SessionParticipant**: comensal de una sesión. Atributos relevantes: `status` (`open`/cerrado),
  `expires_at` (`None` si lo agregó staff sin QR, `RN-MESA-25`), `display_name`/`display_label`
  (desambiguado, no vacío, `RN-MESA-27`). Su presencia `open` es una de las dos condiciones de
  liberación de mesa (`RN-CART-15`).
- **OrderItem**: línea de un pedido dentro de una sesión de mesa. Atributos relevantes:
  `participant_id` (gobierna el reparto, `RN-MESA-05` a `RN-MESA-09`), `quantity` (debe cuadrar
  exacto al repartir, `RN-MESA-06`), `estado_cocina` (anulado excluye de la cuenta y del reparto,
  `RN-MESA-08`, `RN-MESA-16`).
- **CustomerOrder**: pedido asociado a la sesión. Su `status` determina si entra en la cuenta
  (`_billable_orders` excluye `cancelada`/`pagada`) y si bloquea el cierre (`recibida` bloquea,
  `RN-MESA-04`) o la liberación de mesa (`RN-CART-15`).
- **Sale**: venta creada al cerrar la sesión (vía `build_sale`, spec 011). Una por cierre
  `unified`, una por cada bloque de `split`. Atributos relevantes a esta spec: `table_session_id`,
  `participant_id` (solo en `split`), `customer_name` (orden de prioridad, `RN-MESA-17`;
  garantía de nombre en bloques sin comensal, `RN-MESA-18`), `promotion_id` (regla de combo
  único, `RN-MESA-15`).
- **DiningTable**: mesa física. Su `status` (`ocupada`/`libre`) lo gobierna
  `try_release_if_empty` (`RN-CART-15`) y el barrido (`RN-SCHED-01` a `RN-SCHED-04`).
- **CashShift**: turno de caja abierto, requerido para cerrar cualquier sesión de mesa
  (`ensure_open_shift`). El cierre nunca escribe movimientos manuales sobre él (`RN-MESA-23`).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-MESA-01` a `RN-MESA-27` y `RN-SCHED-01` a `RN-SCHED-09`
  puede verificarse ejecutando los pasos descritos en esta spec contra un `pos-backend` en
  ejecución, sin necesitar leer `table_sessions/service.py` ni `core/scheduler.py` para entender
  el comportamiento esperado.
- **SC-002**: Los cuatro blindajes de la regla protegida A-15 (comensales repetidos, montos de
  raíz en `split`, importes negativos, bloque sin nombre) quedan cubiertos punto por punto por
  `app/scripts/test_split_blindaje.py`, la base de test más sólida de todo el reconocimiento para
  proteger una regla `PROTEGIDA` — ningún cambio futuro en `close_session`/`_close_split` puede
  reintroducir uno de los cuatro huecos sin que este script lo detecte.
- **SC-003**: La condición de liberación de mesa (`RN-CART-15`, "nadie activo Y nada que
  cobrar", ambas necesarias) y el comportamiento del barrido sobre sesiones envejecidas quedan
  cubiertos punto por punto por `app/scripts/test_table_release.py`, cuyos ocho casos son la base
  directa de los escenarios de aceptación de User Story 8 y User Story 9.
- **SC-004**: Ningún cierre de sesión puede dejar una venta parcial de mesa tras un fallo a mitad
  de un cierre `split` de varios comensales — verificable inspeccionando que toda la cascada
  (ventas, estado de pedidos, cierre de sesión/comensales, liberación de mesa) vive dentro de la
  misma transacción de base de datos (`RN-MESA-22`).
- **SC-005**: El barrido nunca cierra una sesión con pedidos facturables pendientes de cobro —
  verificable con el 100% de los casos de `RN-SCHED-03` reproducidos contra un `pos-backend` en
  ejecución.
- **SC-006**: Las anomalías `PENDIENTE` de esta spec (A-29/`RN-MESA-15`, A-38/`RN-MESA-13` y
  `RN-MESA-24`) quedan documentadas con su comportamiento observado, su evidencia de código y su
  tratamiento acordado (documentar sin especificar), de forma que el equipo de modernización no
  las reintroduzca por accidente ni las trate como si ya tuvieran una decisión de negocio cuando
  no la tienen.
- **SC-007**: La anomalía `[DUDOSA]` A-17 (porción de mesa, `RN-MESA-02`) queda señalada
  explícitamente como pregunta de negocio sin cerrar, distinta de A-15 y A-11 (ambas cerradas en
  esta spec), para que la próxima ronda real de entrevista de negocio la incluya sin necesidad de
  releer el código.

## Out of Scope

- **La confirmación de pedidos y su reversa** (el punto real de descuento de inventario, el ciclo
  de cobro legado `block`/`pay` por pedido individual, la cancelación asimétrica) — cubierto por
  la spec 008.
- **La operación de cocina y la administración de mesas físicas** (transición de
  `estado_cocina`, consolidación de carritos, mover/fusionar mesas, el camino `group_bill` de
  mesas fusionadas citado en A-01) — cubierto por la spec 009.
- **El constructor de venta compartido y la emisión de factura** (`build_sale`, la especificación
  completa del tope o prohibición de descuento manual del cajero que motiva A-11, el punto único
  de emisión de factura) — cubierto por la spec 011; esta spec solo documenta que el cierre de
  mesa comparte esas reglas, sin respecificarlas.
- **El motor de evaluación de promociones y combos** (`promotions.evaluate`,
  `combo_discount_for_lines`, prioridad entre promociones aplicables) — cubierto por las specs
  012 y 013, aún no escritas en este reconocimiento; esta spec solo documenta **cuándo** y
  **cómo** el cierre de mesa invoca ese motor y registra su resultado (`RN-MESA-14`,
  `RN-MESA-15`), no cómo decide el motor mismo.
- **La expiración de promociones de medianoche** (`expire_promotions`, RN-SCHED-10, RN-SCHED-11)
  — comparte archivo (`core/scheduler.py`) y patrón de lock distribuido con el barrido de
  sesiones (`RN-SCHED-08`), pero es un dominio de negocio distinto (promociones, no mesas); fuera
  del alcance declarado de esta spec (RN-MESA/RN-SCHED-01 a 09).
- **La apertura de sesión de mesa vía QR y el ciclo de vida del carrito del comensal**
  (`cart/service.py`, incluida la cancelación del propio comensal que dispara
  `try_release_if_empty`) — el mecanismo de liberación en sí (`RN-CART-15`) se documenta aquí por
  vivir en `table_sessions/service.py`, pero sus llamadores desde `cart` no son objeto de esta
  spec.

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia del resto de
  las guías de este template ("evitar detalles de implementación"), aquí los endpoints, códigos
  de estado HTTP, nombres de campo, constantes internas (`EMPTY_SESSION_TTL_MINUTES`,
  `TABLE_SESSION_MAX_HOURS`) y valores literales **son** el contrato observable que se está
  documentando — se citan explícitamente porque los criterios de aceptación deben ser
  verificables directamente contra `pos-backend` en ejecución o contra los scripts de
  characterization citados.
- **A-15 (`RN-MESA-11`, `RN-MESA-12`, `RN-MESA-18`) se especifica tal cual, sin tocar**: es una
  regla `[PROTEGIDA]` con dos testigos (CÓDIGO + `memoria-historica.md` #11) y decisión de
  negocio cerrada (P12: sin ventana de exposición). Los cuatro blindajes se fijan como invariantes
  de test obligatorios de máxima prioridad en cualquier reimplementación.
- **RN-MESA-01, RN-MESA-21, RN-MESA-22 y RN-MESA-23 se especifican como invariantes obligatorios,
  no como comportamiento opcional**: son las reglas con mayor impacto económico directo de la
  spec (doble cobro, vuelto mal calculado, cierre parcial, doble conteo de caja); cualquier
  reimplementación debe tratarlas como tests de regresión de máxima prioridad, junto con
  `RN-SCHED-03` (nunca cerrar una sesión con algo pendiente de cobrar).
- **A-17 (porción de mesa, `RN-MESA-02`) se documenta pero NO se especifica como contrato**: a
  diferencia de A-15 y A-11 (ambas cerradas), la pregunta de negocio que la desbloquearía sigue
  sin decisión — no se cerró en la ronda 1-2 ni en la ronda 3 (simulada). Queda explícitamente
  pendiente de incluirse en la próxima ronda real de negocio; esta spec no asume una respuesta
  en ningún sentido.
- **A-29 (`RN-MESA-15`) y A-38 (`RN-MESA-13`, `RN-MESA-24`) se documentan pero NO se especifican
  como contrato**: siguiendo instrucción explícita de alcance, ambas anomalías quedan con
  clasificación `PENDIENTE` en `registro-de-anomalias.md` — se describe el comportamiento
  observado hoy (porque esta spec documenta "lo que el sistema YA hace"), pero no se fija como
  comportamiento correcto ni obligatorio para la modernización.
- **A-28 se documenta como riesgo de configuración, no se corrige en esta spec**: el invariante
  `SESSION_TTL_REFRESH_SLACK_MINUTES < EMPTY_SESSION_TTL_MINUTES` sigue sin validación de
  arranque; esta spec fija el comportamiento actual del barrido (`RN-SCHED-01`) asumiendo que el
  invariante se respeta en la configuración vigente, y señala explícitamente que violarlo puede
  cerrar mesas activas.
- **A-11 se referencia, no se respecifica**: esta spec confirma que el alcance decidido por el
  negocio (ronda 3, P30) incluye el cierre unificado y dividido de mesa, pero delega la
  especificación completa del mecanismo de tope/prohibición de descuento manual a la spec 011,
  para no duplicar una regla compartida entre los tres caminos de cobro del sistema.
- **`RN-CART-14`/`RN-CART-15` se citan pese al prefijo `RN-CART`**: la función que gobiernan
  (`try_release_if_empty`) vive físicamente en `table_sessions/service.py:74-116`, el archivo en
  alcance de esta spec; se documentan aquí porque el mecanismo de liberación de mesa es parte
  directa del ciclo de vida de la sesión, aunque su numeración de regla de negocio quedó asignada
  al módulo `cart` en el reconocimiento original.
