# Feature Specification: Confirmación de pedido, cobro legado y cancelación

**Feature Branch**: `008-confirmacion-cobro-legado-y-cancelacion-de-pedidos`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/orders/checkout.py` y `consumption.py`, para que sirva de contrato
formal de cara a la modernización (Principio I y Principio III de la
[Constitución](../../.specify/memory/constitution.md)). Es el dominio de "la verdad del dinero y
del inventario de un pedido": el momento exacto en que un pedido QR pasa de `recibida` a
`abierta` (el único punto de descuento real de inventario de ese camino), el ciclo de cobro
legado por pedido individual (`block`→`pay`, sin caller confirmado en la UI hoy), y la
cancelación asimétrica según qué tanto avanzó cocina. Incluye una regla protegida que corrige un
bug histórico real (A-25) y tres anomalías que se documentan **sin especificar como contrato**
porque su clasificación en `registro-de-anomalias.md` sigue `PENDIENTE` (A-29, A-38, A-42), más
una que se documenta como código muerto candidato a retirar (A-01, camino B) y una regla
compartida cuya especificación completa vive en la spec 011 (A-11).

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE
de la confirmación de pedidos QR (el punto real de descuento de inventario), el ciclo de cobro
legado por pedido individual, y la cancelación asimétrica del sistema POS Heladería, tomado de
`reglas-de-negocio.md` (RN-ORD-01 a RN-ORD-35, RN-ORD-65) y de `registro-de-anomalias.md` (A-01,
A-11, A-25, A-29, A-38, A-42), para que sirva de contrato en la modernización."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `orders/checkout.py` y
  `orders/consumption.py`, no uno deseado. Las anomalías conocidas se marcan inline con su
  tratamiento acordado (registro-de-anomalias.md). A-25 es la regla protegida más importante de
  esta spec: corrige un bug real de inventario sobrestimado y se especifica como invariante de
  diseño explícito, no solo como ausencia de funcionalidad.
-->

### User Story 1 - Confirmar un pedido es el único punto real de descuento de inventario (Priority: P1)

Un miembro del staff acepta un pedido QR que llegó en estado `recibida`. En ese instante —y solo
en ese instante— el sistema descuenta físicamente del inventario todo lo que ese pedido consume,
antes de marcarlo `abierta`. Si falta stock de un solo insumo, nada se descuenta: la transacción
entera se revierte y el pedido sigue `recibida`, listo para reintentar o cancelar.

**Why this priority**: es el corazón de la spec — el docstring de `confirm_order` lo declara
explícitamente ("es el único punto de descuento de los pedidos por QR"). Cualquier regresión
aquí, o cualquier vía alterna que la esquive, reproduce exactamente el bug histórico que motivó
A-25 (User Story 3): inventario sobrestimado sin que nadie se entere.

**Independent Test**: se puede probar invocando `confirm_order(db, order_id, user)` sobre un
pedido `recibida` con ítems conocidos y comparando el stock antes/después, sin depender de
cocina, cobro ni cancelación.

**Acceptance Scenarios**:

1. **Given** un pedido `recibida` con 2 unidades de "Helado de vainilla" (receta 200g base por
   unidad), **When** el staff confirma el pedido, **Then** el sistema descuenta 400g del insumo
   **antes** del commit que pone `status=abierta` — la mutación de inventario y el cambio de
   estado ocurren en la misma transacción (`RN-ORD-10`).
2. **Given** un pedido con ítems cuyo `estado_cocina` es `anulado`, **When** se confirma,
   **Then** esos ítems se excluyen del descuento — solo los ítems consumibles participan
   (`RN-ORD-11`).
3. **Given** un pedido `recibida` cuyo único ítem está `anulado` (lista de entradas consumibles
   vacía), **When** se intenta confirmar, **Then** el sistema responde `409` "El pedido no tiene
   ítems", sin tocar inventario ni cambiar el estado (`RN-ORD-12`).
4. **Given** un pedido en cualquier estado distinto de `recibida` (`abierta`, `bloqueada`,
   `pagada`, `cancelada`), **When** se intenta confirmar (incluido un reintento sobre un pedido
   ya confirmado), **Then** el sistema rechaza con `409` "Solo se confirman pedidos en
   'recibida'" — esta guarda es lo que impide el doble descuento (`RN-ORD-13`).
5. **Given** un pedido con 2 ítems, donde el primero descuenta bien y el segundo requiere 500g de
   un insumo con solo 100g disponibles, **When** se confirma, **Then** `InsufficientStockError`
   provoca rollback completo — **incluido** el descuento ya aplicado al primer ítem — y el
   pedido permanece `recibida` (`RN-ORD-14`).
6. **Given** que el carrito del comensal ya pasó su propia validación de disponibilidad antes de
   enviarse, **When** ese mismo pedido llega a `confirm_order`, **Then** el sistema vuelve a
   validar el stock con lock real de fila — la validación del carrito fue preventiva y pudo
   quedar obsoleta; `confirm_order` es la única fuente de verdad (`RN-ORD-15`).
7. **Given** un pedido con varias líneas que tocan varios insumos, **When** se confirma,
   **Then** los locks de fila se adquieren en orden canónico de id de insumo, no en el orden de
   las líneas del pedido — evita deadlocks entre confirmaciones concurrentes de pedidos distintos
   que comparten insumos (`RN-ORD-16`).
8. **Given** un lote de líneas a descontar donde una o más no consumirían **ningún** insumo,
   **When** se procesa el lote, **Then** el sistema rechaza el lote **completo** antes de
   descontar ninguna línea — "mejor parar y configurar la receta que emitir una venta que
   descuadra el inventario" (`RN-ORD-17`).
9. **Given** cualquier movimiento de inventario que resulte de una confirmación, **When** se
   inspecciona su cantidad, **Then** es siempre estrictamente positiva — invariante de base de
   datos sobre `order_items.quantity` (`RN-ORD-26`).
10. **Given** el motivo registrado en el movimiento de salida por confirmación, **When** se
    compara con el de una reversa de cancelación, **Then** son distintos aunque ambos sean
    contablemente "ajuste": la confirmación usa `reason='venta', reference_type='order'`
    (`RN-ORD-29`).

---

### User Story 2 - Cancelar un pedido revierte el inventario de forma asimétrica, no simétrica (Priority: P1)

Un pedido se cancela en algún punto de su ciclo de vida. El sistema **no** devuelve al stock
todo lo que se había descontado por igual: solo devuelve lo que cocina todavía no había tocado.
Lo que ya se combinó físicamente en la preparación se registra como pérdida en auditoría, nunca
vuelve al kardex, y un pedido que nunca llegó a confirmarse no genera ningún movimiento porque
nunca descontó nada.

**Why this priority**: corrige directamente el bug que motivó este trabajo — "cancelar un pedido
ya en preparación reingresaba insumos que físicamente ya se habían usado y sobrestimaba el
inventario en silencio hasta el conteo físico" (docstring de `test_cancel_inventory.py`). Es,
junto con User Story 1, el invariante de inventario más crítico de todo el módulo.

**Independent Test**: se puede probar completamente ejecutando
`python -m app.scripts.test_cancel_inventory` contra un `pos-backend` en ejecución — el script
cubre exactamente las cuatro ramas de esta historia (recibida, pendiente, en curso, mixto) con
datos desechables propios.

**Acceptance Scenarios**:

1. **Given** un pedido `recibida` (nunca confirmado) con un ítem `pendiente` cantidad 3, **When**
   se cancela, **Then** el sistema no genera **ningún** movimiento de inventario, sin importar el
   `estado_cocina` de sus ítems — nunca se descontó, así que no hay nada que revertir
   (`RN-ORD-20`). **Verificado por** `test_cancel_inventory.py`, caso 1: "cancelar 'recibida' no
   genera movimientos" / "no cambia el stock".
2. **Given** un pedido ya confirmado (`abierta`) con un ítem `pendiente` (`quantity=2`, receta
   150g/unidad), **When** se cancela, **Then** el sistema registra un movimiento
   `type='in', quantity=300g, reason='ajuste', reference_type='order_void'` y el stock sube 300g
   — cocina aún no lo había tocado (`RN-ORD-21`). **Verificado por** `test_cancel_inventory.py`,
   caso 2: "confirmar descuenta 2 líneas × 2 ud" y "cancelar con todo 'pendiente' devuelve el
   stock" + "escribe 2 entradas".
3. **Given** un pedido `abierta` con un ítem `en_preparacion` o `listo` (p. ej. una unidad de
   "Malteada de fresa"), **When** se cancela, **Then** el insumo **no** vuelve al stock — se
   agrega a la lista de pérdidos con su `estado_cocina`, no a la de reversión; el 'out' de la
   confirmación ya representa esa pérdida, escribir otro movimiento la descontaría dos veces
   (`RN-ORD-22`). **Verificado por** `test_cancel_inventory.py`, caso 3: "cancelar en cocina NO
   devuelve el stock (es pérdida)" / "no escribe ninguna entrada 'in'".
4. **Given** un pedido con ítems mixtos (uno `pendiente`, otro `en_preparacion`), **When** se
   cancela, **Then** solo la línea `pendiente` revierte al stock; la línea en curso no
   (`RN-ORD-35`, cualquier estado no listado en `_CONSUMED_KITCHEN`/`_NOT_DEDUCTED` cae por
   omisión en "sí se revierte"). **Verificado por** `test_cancel_inventory.py`, caso 4: "mixto:
   vuelve solo la línea pendiente".
5. **Given** un ítem ya `anulado` individualmente antes de la cancelación de la orden completa,
   **When** se cancela la orden, **Then** ese ítem se ignora por completo en la reversa de la
   orden — su reversa (si procedía) ya ocurrió al anularlo con `void_item` (`RN-ORD-23`).
6. **Given** una cancelación con al menos un ítem "perdido" (rama `en_preparacion`/`listo`),
   **When** se completa la cancelación, **Then** el sistema escribe un `warning` en el log y un
   registro de auditoría (`action='order.cancel.loss'`) con el detalle de ítems perdidos y
   revertidos — **sin** crear ningún `InventoryMovement` para la pérdida; la pérdida se traza en
   auditoría, no en kardex (`RN-ORD-24`).
7. **Given** cualquier pedido en estado terminal (`pagada` o `cancelada`), **When** se intenta
   cancelar de nuevo, **Then** el sistema rechaza con `409` "La orden ya es terminal", sin
   reversa duplicada (`RN-ORD-18`).
8. **Given** un pedido en `recibida`, `abierta` o `bloqueada` (cualquier estado no terminal),
   **When** el staff lo cancela, **Then** no hay ninguna restricción adicional de estado más
   allá de esa lista terminal — el staff puede cancelar en cualquier momento no terminal, a
   diferencia de la confirmación, que exige `recibida` exclusivamente (`RN-ORD-19`).
9. **Given** cualquier cancelación, **When** se completa, **Then** el sistema exige un `motivo`
   (mínimo 1 carácter) y lo registra en `OrderCancelLog`, junto con el actor real: `user_id` si
   cancela el staff, o `participant_id` si cancela el propio comensal desde el QR — nunca ambos
   obligatorios a la vez (`RN-ORD-25`). **Ejemplo**: comensal cancela vía QR sin `user` →
   `actor_id=None`, `participant_id=<id comensal>`; staff cancela → `participant_id=None`,
   `user_id`/`user_name` presentes.
10. **Given** una reversa de inventario que sí procede (rama `pendiente`), **When** se calcula la
    cantidad a devolver, **Then** se usa exactamente la misma función (`plan_line_consumption`)
    que se usó para descontarla — garantiza simetría numérica; "una reversa que calcule distinto
    descuadra el inventario para siempre" (`RN-ORD-27`).
11. **Given** cualquier movimiento `type='in'` de reversa, **When** se intenta aplicar, **Then**
    nunca falla por falta de stock disponible — a diferencia de los `out`, los `in` siempre suman
    (`RN-ORD-28`).

**Nota — anomalía A-42 (porción de esta spec, `PENDIENTE`)**: el registro de auditoría que
escribe `RN-ORD-24` (pérdidas por cancelación) sigue funcionando exactamente como se documenta
arriba, pero la pestaña de Ajustes → Auditoría que antes lo mostraba fue retirada del producto.
Confirmado en la segunda ronda de entrevista de negocio (P25): **nadie consulta `audit_logs` hoy,
ni por interfaz ni por otra vía**. El comportamiento de escritura (`record_audit`) se especifica
tal cual arriba porque es lo que el sistema hace; la decisión de si mantener el registro sin
lector, construir una interfaz nueva, o retirar la escritura, queda `PENDIENTE` de decisión de
negocio y no se especifica en esta spec.

---

### User Story 3 - No existe ninguna transición libre de estado de pedido (Priority: P1) — regla protegida A-25

El sistema no ofrece, en ningún endpoint, la posibilidad de asignar directamente cualquier
estado a un pedido. Cada transición legítima (`recibida→abierta`, `abierta→bloqueada`,
`→pagada`, `→cancelada`) tiene su propio endpoint dedicado, con sus propias reglas y efectos
sobre inventario. Esta ausencia es deliberada, no un vacío de funcionalidad por completar.

**Why this priority**: `PATCH /{order_id}/status` existió y se retiró **a propósito** porque
permitía saltarse `confirm_order` (User Story 1) y pasar un pedido de `recibida` a `abierta` sin
tocar inventario — sobrestimándolo en silencio (`memoria-historica.md` entrada #3, 2026-07-28,
commit `5c1db9ed`, Deimer Hernandez). Reintroducir una vía de transición libre de estado
reproduciría exactamente el bug ya corregido. Es la regla con más riesgo de reintroducción
silenciosa de todo el módulo si alguien "simplifica" el router en la modernización.

**Independent Test**: se puede verificar negativamente — no existe ninguna ruta `PATCH`
genérica de status en `app/api/v1/orders/router.py`, y un `grep` del código y del cliente
frontend no encuentra ningún caller de una transición de estado que no pase por uno de los
cuatro endpoints dedicados.

**Acceptance Scenarios**:

1. **Given** el router de pedidos, **When** se enumeran sus endpoints, **Then** no existe ningún
   `PATCH /{order_id}/status` ni equivalente que reciba un `status` arbitrario como parámetro
   (`RN-ORD-65`).
2. **Given** las cuatro transiciones legítimas de un pedido, **When** se busca su endpoint,
   **Then** cada una tiene una ruta dedicada y distinta: `recibida→abierta` es
   `POST /orders/{id}/confirm` (descuenta inventario, User Story 1); `abierta→bloqueada` es
   `POST /orders/{id}/block` (valida preparación); `→pagada` se resuelve vía cobro (legado:
   `POST /orders/{id}/pay`, User Story 4; o mesa: spec 010); `→cancelada` es
   `POST /orders/{id}/cancel` (revierte lo no preparado, User Story 2) (`RN-ORD-65`).
3. **Given** el comentario que documenta la remoción en `router.py`, **When** se lee, **Then**
   declara explícitamente la razón: el antiguo `PATCH` "permitía pasar un pedido de 'recibida' a
   'abierta' esquivando `confirm_order` —el único punto que descuenta stock— y dejaba el
   inventario sobrestimado sin que nadie se enterara" (`RN-ORD-65`,
   `memoria-historica.md` #3).

**Especificación explícita de esta spec**: "no existe transición libre de status" se fija aquí
como **invariante de diseño**, no como ausencia temporal de funcionalidad. Cualquier
reimplementación de este módulo en la fase de modernización debe preservar que cada transición
de estado de pedido tenga su propio endpoint con sus propias reglas — nunca un endpoint genérico
que acepte cualquier status como parámetro libre.

---

### User Story 4 - El ciclo de cobro legado exige bloqueo previo, y hoy no tiene caller confirmado en la UI (Priority: P2)

Antes de poder cobrar un pedido individual por el ciclo legado (`pay_order`), el pedido debe
pasar primero por un bloqueo explícito (`block_order`) que exige que esté `abierta`, que la
versión enviada coincida con la actual (lock optimista), y que no queden ítems sin terminar en
cocina. Solo entonces el pedido puede pasar a `pagada`. Hoy, ningún flujo de la interfaz llama a
ninguno de los dos endpoints (`block`/`pay`) — es un ciclo de cobro completo, funcional, pero sin
uso real confirmado.

**Why this priority**: aunque no tenga uso confirmado hoy, el código sigue activo, expuesto por
API y ejecutable; documentarlo con precisión es indispensable antes de decidir en modernización
si se retira o se conserva. Prioridad P2 (no P1) precisamente porque no está en el camino real
de uso diario — a diferencia de User Story 1 y 2, que sí lo están.

**Independent Test**: se puede probar invocando `block_order` y luego `pay_order` directamente
sobre un pedido de prueba, sin pasar por ninguna pantalla del frontend — que es, de hecho, la
única forma de ejercitarlo hoy, porque no existe ningún botón ni llamada de servicio en
`pos-heladeria` que invoque `POST /orders/{id}/block` ni `POST /orders/{id}/pay` (confirmado con
`grep` cruzado en ambos repositorios: 0 resultados de uso real, solo la interfaz TypeScript que
documenta la forma del payload).

**Acceptance Scenarios**:

1. **Given** un pedido `abierta`, versión `3`, con un ítem `pendiente` en cocina, **When** se
   invoca `block_order(version=3)`, **Then** el sistema rechaza con `409` "Hay ítems sin
   terminar en cocina; anúlalos o espera a que estén listos" (`RN-ORD-01`).
2. **Given** el mismo pedido con el ítem ya `listo`, **When** se invoca `block_order(version=3)`,
   **Then** el pedido pasa a `bloqueada` y su versión sube a `4` — lock optimista más chequeo de
   cocina, ambos deliberados (`RN-ORD-01`).
3. **Given** un pedido en cualquier estado distinto de `bloqueada`, **When** se invoca
   `pay_order`, **Then** el sistema rechaza con `409` "La orden debe estar bloqueada para cobrar
   (bloquea primero)" (`RN-ORD-05`).
4. **Given** un pedido `bloqueada`, **When** se invoca `pay_order` sin un turno de caja abierto
   (`cash_shift_id` inválido o cerrado), **Then** el sistema rechaza antes de tocar ninguna línea
   o inventario — el chequeo de turno abierto es explícito y temprano (`RN-ORD-06`).
5. **Given** un pedido `bloqueada` con turno de caja abierto, **When** se cobra exitosamente,
   **Then** el pedido pasa a `pagada` y **no** se ejecuta ningún movimiento de inventario
   adicional — el descuento real ya ocurrió al confirmar el pedido (User Story 1); el cobro
   legado nunca vuelve a tocar stock (`RN-ORD-09`).
6. **Given** líneas cobradas sin combo y líneas con combo, **When** se cobra vía `pay_order`,
   **Then** el sistema evalúa promociones percent/fixed sobre las líneas sin combo, suma el
   ahorro propio de las líneas con combo, y agrega ambos al descuento manual capturado — mismo
   motor que usa el mostrador (`RN-ORD-07`).
7. **Given** el reparto de la cuenta de una mesa por comensal, **When** se calcula el `split`,
   **Then** las líneas se agrupan por `participant_id`; las líneas sin comensal asignado (las que
   agregó el mesero) forman su propio grupo — no se prorratean ni se excluyen (`RN-ORD-04`).

**Nota documental — anomalía A-01, camino B (código muerto, candidato a retiro)**: `compute_bill`
de este módulo (`checkout.py:127-180`) calcula "cuánto debe una mesa" filtrando únicamente
órdenes con `status != 'cancelada'` — **incluye las ya `pagada`** en el total (`RN-ORD-03`,
`[DUDOSA]`). No aplica ningún descuento de promociones. Es el **camino B** de la anomalía A-01:
tres implementaciones distintas de la misma pregunta de negocio en todo el sistema. La
convención vigente y correcta (`table_sessions.compute_bill`, que sí excluye pagadas y sí aplica
promociones) vive en la **spec 010**. Este camino B no tiene caller confirmado hoy (mismo `grep`
cruzado de la nota anterior) — es código muerto, pero **peligroso si se reactiva**: mostraría a
un cajero una cuenta que incluye órdenes que el cliente ya pagó, sin ningún descuento aplicado.
**Tratamiento acordado** (`registro-de-anomalias.md`, A-01): documentar como candidato a retirar
o unificar con el camino de spec 010 en la fase de modernización; no se especifica como
comportamiento deseado.

---

### User Story 5 - La prohibición total de descuento manual del cajero también rige en el camino legado (Priority: P2) — regla compartida A-11

El campo de descuento (`discount`) que acepta `pay_order` no tiene, en el propio esquema, ningún
tope superior — el único freno hoy es que el total resultante no quede negativo. Esta misma
carencia existe en los otros dos caminos de cobro del sistema (mostrador y mesa), y el negocio ya
decidió, para los tres por igual, que el cajero **no debería poder aplicar descuento manual en
absoluto**.

**Why this priority**: la regla en sí (el tope/prohibición de descuento manual) se especifica
formalmente en la **spec 011** (constructor de venta compartido), que es donde vive el mecanismo
común a los tres caminos. Esta spec documenta únicamente que el camino legado de esta spec
comparte exactamente la misma carencia y queda alcanzado por la misma decisión de negocio — no
la respecifica de cero.

**Independent Test**: se puede verificar inspeccionando el esquema `PayIn` de este módulo
(`discount: Decimal = Field(0, ge=0, ...)`, sin `le=`) y comparando que el patrón es idéntico al
de `sales/schemas.py` que motivó A-11 en la spec 011.

**Acceptance Scenarios**:

1. **Given** el esquema `PayIn` de `orders/schemas.py`, **When** se inspecciona el campo
   `discount`, **Then** solo tiene `ge=0` (no negativo) — ningún límite superior, mismo patrón
   verificado en `sales/schemas.py` para A-11.
2. **Given** la pregunta de alcance repreguntada al negocio en la ronda 3 (simulada, P30): "¿la
   prohibición de descuento manual aplica a los tres caminos de cobro?", **When** el negocio
   responde, **Then** confirma explícitamente que sí, a los tres por igual — "no tendría sentido
   prohibirlo en uno y dejarlo abierto en otro, el cajero simplemente usaría el camino que se lo
   permite" — alcanzando también a este camino legado, aunque no tenga uso confirmado hoy (User
   Story 4).

**Especificación de esta spec**: la regla de negocio en sí (tope o prohibición del descuento
manual, y su mecanismo de aplicación) **no se especifica aquí** — es responsabilidad de la spec
011. Esta spec únicamente deja registrado que el alcance decidido por el negocio incluye
explícitamente el camino legado de `pay_order`, para que la spec 011 (o su implementación) no lo
omita por no ser el camino de uso diario.

---

### User Story 6 - Liberar una mesa exige que todas sus órdenes terminen, con reglas menores documentadas sin especificar (Priority: P3) — cluster A-38

Para liberar una mesa (dejarla disponible de nuevo), todas sus órdenes deben estar en un estado
terminal (`pagada` o `cancelada`). Al liberar, el sistema cierra en cascada la sesión de mesa,
sus comensales y los carritos que quedaran abiertos — pero esa cascada no vuelve a validar por sí
misma que no queden pendientes: confía en que quien la invoca ya lo hizo.

**Why this priority**: son hallazgos de bajo impacto individual, casos límite de uso operativo
diario más que riesgos activos — de ahí la prioridad P3, la más baja de esta spec.

**Independent Test**: se puede probar invocando `release_table` sobre una mesa con una orden
`bloqueada` y verificando el rechazo, y por separado invocando `close_table_sessions` de forma
aislada (sin pasar por `release_table`) para observar que no valida pendientes por sí sola.

**Acceptance Scenarios**:

1. **Given** una mesa con una orden `bloqueada` (no terminal), **When** se intenta liberar la
   mesa, **Then** el sistema rechaza con `409` "La mesa tiene órdenes sin cerrar (paga o cancela
   primero)", listando cada orden bloqueante con su cantidad de ítems no anulados (`RN-ORD-30`).
2. **Given** líneas de venta cobradas que incluyen líneas con `combo_id` y líneas sin él,
   **When** se calculan los descuentos percent/fixed, **Then** las líneas de combo se excluyen
   de ese cálculo porque ya tienen su propio ahorro — no se acumulan (`RN-ORD-33`).
3. **Given** `close_table_sessions`, invocada directamente sin pasar por `release_table`,
   **When** se ejecuta, **Then** cierra la sesión de mesa, sus comensales y sus carritos abiertos
   sin verificar por sí misma si quedan órdenes pendientes — esa responsabilidad recae en quien
   la invoca (`release_table`, el cierre con `billing_mode` de spec 010, o el job de sesiones
   huérfanas) (`RN-ORD-31`, `[DUDOSA]`, `PENDIENTE`).
4. **Given** una línea de venta cuyo producto o variante fueron borrados físicamente después de
   la venta, **When** se reconstruye su descripción ("Producto - Variante"), **Then** el sistema
   cae a solo el nombre de la variante, o a cadena vacía si tampoco hay variante — sin marcar la
   línea como huérfana (`RN-ORD-32`, `[DUDOSA]`, `PENDIENTE`).

**Nota documental — precisión de comentario, no anomalía de comportamiento (RN-ORD-34)**: el
docstring de `confirm_order` (User Story 1) se autodescribe como "el único punto de descuento de
los pedidos por QR". Es preciso solo si se lee como "único punto de **salida** ('out')" — el
mismo módulo también escribe movimientos `'in'` en la reversa de `cancel_order` (User Story 2),
un segundo punto de escritura en el kardex de la misma orden, aunque no sea técnicamente un
"descuento". No es una contradicción de comportamiento real; es una imprecisión de redacción que
puede inducir a error si se lee de forma aislada.

**Tratamiento acordado para RN-ORD-31 y RN-ORD-32** (`registro-de-anomalias.md`, A-38):
documentar sin especificar. Ambas permanecen `PENDIENTE` de decisión de negocio (¿existe hoy
algún caller que no valide antes de invocar el cierre en cascada?; ¿puede un `OrderItem`
referenciar hoy una variante ya eliminada físicamente?) y esta spec no las fija como
comportamiento deseado ni obligatorio para la modernización.

---

### Edge Cases

- **Reintento de confirmación sobre un pedido ya `abierta`**: rechazado por `RN-ORD-13`
  (User Story 1, escenario 4) — la guarda de estado es lo único que impide el doble descuento,
  no una verificación de "¿ya se descontó esto?" a nivel de línea.
- **Cancelación de un pedido con ítems en los cuatro estados posibles a la vez** (`pendiente`,
  `en_preparacion`, `listo`, `anulado`): cada ítem sigue su propia rama de forma independiente
  dentro de la misma cancelación — reversión solo para `pendiente`, pérdida para
  `en_preparacion`/`listo`, ignorado si `anulado` (`RN-ORD-20` a `RN-ORD-23`, `RN-ORD-35`).
- **Falla de stock al aplicar un movimiento `'in'` de reversa**: no puede ocurrir por diseño —
  los movimientos de entrada nunca fallan por falta de stock disponible (`RN-ORD-28`).
- **Dos combos distintos en la misma venta cobrada por el ciclo legado**: el descuento monetario
  de ambos se suma correctamente al total, pero `promotion_id` no queda con ninguno de los dos
  registrado — ver anomalía A-29 en Requirements (`RN-ORD-08`).
- **`block_order`/`pay_order` invocados hoy**: solo es posible llamándolos directamente contra la
  API (curl, script, cliente HTTP genérico) — no existe ningún flujo del producto que los
  ejercite en el uso normal (User Story 4).
- **Bloqueo con versión desactualizada**: `block_order` con un `version` que no coincide con el
  actual de la orden se rechaza con `409` "Conflicto de versión (la orden cambió)" antes de
  evaluar el chequeo de cocina — lock optimista explícito (`RN-ORD-01`).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: `block_order` DEBE exigir que la orden esté `abierta`, que la versión enviada
  coincida con la versión actual (lock optimista), y que no existan ítems en cocina sin terminar,
  antes de pasarla a `bloqueada` (`RN-ORD-01`).
- **FR-002**: Los ítems `anulado` NO DEBEN contarse en `compute_bill` (de este módulo) ni en el
  split de cuenta (`RN-ORD-02`).
- **FR-003 [`[DUDOSA]`, anomalía A-01 camino B — documentado como código muerto candidato a
  retirar, no especificado como comportamiento deseado]**: `compute_bill` de este módulo excluye
  únicamente órdenes `cancelada`; **incluye** órdenes ya `pagada` en el total. No tiene caller
  confirmado hoy en ningún flujo del producto. La convención correcta y vigente
  (`table_sessions.compute_bill`, que sí excluye pagadas y sí aplica promociones) vive en la
  spec 010 (`RN-ORD-03`).
- **FR-004**: El split de cuenta DEBE agruparse por `participant_id`; las líneas sin comensal
  asignado (`participant_id=None`, agregadas por el mesero) DEBEN formar su propio grupo, sin
  prorratearse ni excluirse (`RN-ORD-04`).
- **FR-005**: `pay_order` DEBE proceder únicamente si la orden está en estado `bloqueada`
  (`RN-ORD-05`).
- **FR-006**: `pay_order` DEBE exigir un turno de caja abierto, verificado antes de tocar
  cualquier línea o inventario (`RN-ORD-06`).
- **FR-007**: El cobro legado DEBE aplicar descuento automático de promociones: percent/fixed
  sobre líneas sin combo, más el ahorro propio de las líneas con combo, sumados al descuento
  manual capturado — mismo motor que usa el mostrador (`RN-ORD-07`).
- **FR-008 [`[DUDOSA]`, anomalía A-29, `PENDIENTE` — documentada sin especificar como contrato]**:
  si las líneas cobradas usan exactamente un combo distinto, `promotion_id` DEBE registrar ese
  combo; con cero o con dos o más combos distintos, el sistema usa el resultado de la evaluación
  general de promociones (que puede ser `None`) — con dos o más combos, no queda registrado
  ningún combo específico, aunque el descuento monetario de todos se sume correctamente al total.
  Confirmado en entrevista de negocio (P21): sin impacto práctico porque el negocio no usa el
  reporte de ventas por combo/promoción hoy (`RN-ORD-08`).
- **FR-009**: `pay_order` NO DEBE ejecutar ningún movimiento de inventario adicional — el
  descuento real ya ocurrió al confirmar el pedido (`RN-ORD-09`).
- **FR-010 [Regla crítica, User Story 1]**: La transición `recibida → abierta` (`confirm_order`)
  DEBE ser el único punto donde se descuenta físicamente el inventario de un pedido del camino
  QR, ejecutado antes del commit que marca `status=abierta` (`RN-ORD-10`).
- **FR-011**: La confirmación DEBE excluir del descuento los ítems con `estado_cocina='anulado'`
  (`RN-ORD-11`).
- **FR-012**: Confirmar un pedido cuya lista de ítems consumibles resulte vacía DEBE rechazarse
  con `409`, sin tocar inventario ni cambiar el estado (`RN-ORD-12`).
- **FR-013**: La confirmación DEBE ser válida únicamente desde el estado `recibida`; cualquier
  otro estado (incluidos reintentos) DEBE rechazarse con `409` (`RN-ORD-13`).
- **FR-014**: Un fallo de stock durante la confirmación DEBE revertir la transacción completa
  —incluidos los descuentos ya aplicados a líneas anteriores del mismo lote— dejando el pedido en
  `recibida` (`RN-ORD-14`).
- **FR-015**: La validación de disponibilidad hecha al armar el carrito es preventiva, no
  autoritativa; `confirm_order` DEBE revalidar el stock real con lock de fila en el momento de
  confirmar, sin confiar en el chequeo previo (`RN-ORD-15`).
- **FR-016**: Los locks de inventario durante la confirmación DEBEN adquirirse en orden canónico
  de id de insumo (no en el orden de las líneas del pedido), para evitar deadlocks entre
  confirmaciones concurrentes (`RN-ORD-16`).
- **FR-017**: Si alguna línea de un lote de confirmación no consumiría ningún insumo, el sistema
  DEBE rechazar el lote completo antes de descontar cualquier línea (`RN-ORD-17`).
- **FR-018**: La cancelación DEBE rechazarse con `409` si la orden ya está en un estado terminal
  (`pagada` o `cancelada`), sin generar reversa duplicada (`RN-ORD-18`).
- **FR-019**: Fuera de los estados terminales, la cancelación NO DEBE tener ninguna restricción
  adicional de estado — el staff puede cancelar un pedido `recibida`, `abierta` o `bloqueada` en
  cualquier momento (`RN-ORD-19`).
- **FR-020 [Regla crítica, User Story 2]**: Cancelar un pedido en `recibida` (nunca confirmado)
  NO DEBE generar ningún movimiento de inventario, sin importar el `estado_cocina` de sus ítems
  (`RN-ORD-20`).
- **FR-021 [Regla crítica]**: Cancelar un ítem `pendiente` de un pedido ya confirmado DEBE
  devolver su insumo al stock con un movimiento `type='in'` (`RN-ORD-21`).
- **FR-022 [Regla crítica]**: Cancelar un ítem `en_preparacion` o `listo` NO DEBE devolver
  inventario al stock — el insumo ya se considera físicamente consumido; el 'out' de la
  confirmación ya representa esa pérdida (`RN-ORD-22`).
- **FR-023**: Un ítem ya `anulado` individualmente DEBE ignorarse por completo en la cancelación
  de la orden — su reversa, si procedía, ya ocurrió al anularlo (`RN-ORD-23`).
- **FR-024 [Regla crítica; nota de alcance A-42]**: Cuando la cancelación produce al menos un
  ítem "perdido", el sistema DEBE registrar un warning de log y un registro de auditoría
  (`action='order.cancel.loss'`) con el detalle — sin crear ningún `InventoryMovement` para esa
  pérdida. Esta escritura de auditoría sigue activa aunque la interfaz que antes la mostraba
  (Ajustes → Auditoría) fue retirada; confirmado en entrevista de negocio (P25) que nadie
  consulta `audit_logs` hoy — la decisión de mantener, exponer o retirar ese registro queda
  `PENDIENTE`, sin afectar la obligación de seguir escribiéndolo tal como se especifica aquí
  (`RN-ORD-24`).
- **FR-025**: Toda cancelación DEBE exigir y registrar un `motivo` (mínimo 1 carácter) en
  `OrderCancelLog`, y DEBE atarse a un actor real: `user_id`/`user_name` si cancela el staff, o
  `participant_id` si cancela el comensal desde el QR — nunca ambos obligatorios simultáneamente
  (`RN-ORD-25`).
- **FR-026**: La cantidad de cualquier ítem de orden DEBE ser estrictamente positiva —
  garantizado por una constraint de base de datos (`RN-ORD-26`).
- **FR-027**: La cantidad que se devuelve al revertir un ítem DEBE calcularse con exactamente la
  misma función usada para descontarlo (`plan_line_consumption`), garantizando simetría numérica
  (`RN-ORD-27`).
- **FR-028**: Los movimientos `type='in'` producidos por una reversa (anulación de ítem
  `pendiente` o cancelación de orden) NUNCA DEBEN fallar por falta de stock disponible
  (`RN-ORD-28`).
- **FR-029**: El motivo del movimiento de inventario DEBE distinguir venta de reversa aunque
  ambos sean "ajuste" contablemente: la confirmación usa `reason='venta', reference_type='order'`;
  la reversa usa `reason='ajuste', reference_type='order_void'` (`RN-ORD-29`).
- **FR-030**: Liberar una mesa (`release_table`) DEBE exigir que todas sus órdenes estén en
  estado terminal; si alguna no lo está, DEBE rechazarse con `409` listando las órdenes
  bloqueantes (`RN-ORD-30`).
- **FR-031 [`[DUDOSA]`, anomalía A-38, `PENDIENTE` — documentada sin especificar como contrato]**:
  el cierre en cascada de sesiones de mesa (`close_table_sessions`) no valida por sí mismo que no
  queden órdenes pendientes; esa responsabilidad recae en quien lo invoca (`release_table`, el
  cierre con `billing_mode` de spec 010, o el job de sesiones huérfanas) (`RN-ORD-31`).
- **FR-032 [`[DUDOSA]`, anomalía A-38, `PENDIENTE` — documentada sin especificar como contrato]**:
  la descripción de una línea de venta se construye como snapshot `"Producto - Variante"`; si el
  producto referenciado ya no existe, cae al solo nombre de la variante, o a cadena vacía si
  tampoco hay variante — sin marcar la línea como huérfana (`RN-ORD-32`).
- **FR-033**: Las líneas de combo NO DEBEN acumularse con descuentos percent/fixed al calcular el
  pago de una orden de mesa — ya tienen su propio ahorro (`RN-ORD-33`).
- **FR-034 [Precisión documental, no anomalía de comportamiento]**: la afirmación de "único punto
  de descuento" del docstring de `confirm_order` DEBE entenderse como "único punto de **salida**
  ('out')"; la reversa de cancelación también escribe en el kardex (movimientos `'in'`), un
  segundo punto de escritura legítimo para la misma orden (`RN-ORD-34`).
- **FR-035**: La decisión de qué rama de la reversa asimétrica aplica a cada ítem DEBE gobernarse
  por dos únicas constantes (`_CONSUMED_KITCHEN`, `_NOT_DEDUCTED`); cualquier `estado_cocina` no
  incluido explícitamente en ninguna de las dos DEBE caer, por omisión, en la rama "sí se
  revierte" (`RN-ORD-35`).
- **FR-036 [Regla protegida A-25, invariante de diseño explícito]**: El sistema NO DEBE ofrecer
  ningún endpoint que permita asignar libremente cualquier `status` a un pedido. Cada transición
  legítima (`recibida→abierta`, `abierta→bloqueada`, `→pagada`, `→cancelada`) DEBE tener su
  propio endpoint dedicado, con sus propias reglas y efectos sobre inventario. Esta ausencia es
  una decisión de diseño deliberada — corrige un bug histórico real de inventario sobrestimado
  (`memoria-historica.md` #3) — y DEBE tratarse como invariante de test obligatorio en cualquier
  reimplementación, no como funcionalidad pendiente de agregar (`RN-ORD-65`).
- **FR-037 [Regla compartida A-11, referenciada — especificación completa en spec 011]**: el
  campo `discount` de `pay_order` comparte, sin corrección propia de esta spec, la misma carencia
  de tope superior que motivó A-11 en el resto de los caminos de cobro. El alcance decidido por
  el negocio (ronda 3, P30) incluye explícitamente este camino legado dentro de la prohibición
  total de descuento manual del cajero — la regla y su mecanismo se especifican en la spec 011,
  no aquí.

### Key Entities *(include if feature involves data)*

- **CustomerOrder**: pedido de un comensal (camino QR) o abierto por staff. Atributos relevantes
  a esta spec: `status` (`recibida`, `abierta`, `bloqueada`, `pagada`, `cancelada`), `version`
  (lock optimista, `RN-ORD-01`), `dining_table_id`, `participant_id`. Su transición de estado
  nunca es libre (`RN-ORD-65`).
- **OrderItem**: línea de un pedido. Atributos relevantes: `estado_cocina` (`pendiente`,
  `en_preparacion`, `listo`, `anulado` — gobierna la rama de reversa asimétrica, `RN-ORD-35`),
  `quantity` (siempre positiva, `RN-ORD-26`), `participant_id` (para el split, `RN-ORD-04`).
- **OrderCancelLog**: registro de cada cancelación. Atributos relevantes: `motivo` (obligatorio,
  `RN-ORD-25`), `user_id`/`user_name` (actor staff) o `participant_id` (actor comensal), nunca
  ambos obligatorios a la vez.
- **InventoryMovement**: movimiento de kardex. Atributos relevantes a esta spec: `type` (`'out'`
  al confirmar, `'in'` al revertir), `reason` (`'venta'` vs `'ajuste'`, `RN-ORD-29`),
  `reference_type` (`'order'` vs `'order_void'`), `quantity` (siempre positiva). La pérdida por
  cancelación de un ítem consumido **no** genera ninguna fila aquí (`RN-ORD-22`, `RN-ORD-24`).
- **AuditLog** (vía `record_audit`): registro de la pérdida por cancelación
  (`action='order.cancel.loss'`). Sigue escribiéndose (`RN-ORD-24`) aunque la interfaz que lo
  mostraba fue retirada (nota A-42 en User Story 2).
- **CashShift**: turno de caja. Su apertura es condición obligatoria para `pay_order`
  (`RN-ORD-06`), verificada antes de tocar cualquier línea del cobro legado.
- **Sale**: venta creada por `pay_order` al cobrar (vía `build_sale`, spec 011). No genera
  movimiento de inventario adicional (`RN-ORD-09`); su `promotion_id` sigue la regla de
  prioridad de combo único (`RN-ORD-08`, A-29).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-ORD-01` a `RN-ORD-35` y `RN-ORD-65` puede verificarse
  ejecutando los pasos descritos en esta spec contra un `pos-backend` en ejecución, sin necesitar
  leer `checkout.py` ni `consumption.py` para entender el comportamiento esperado.
- **SC-002**: La reversa asimétrica de cancelación (`RN-ORD-20` a `RN-ORD-24`, la regla más
  crítica de esta spec junto con `RN-ORD-10`) queda cubierta punto por punto por
  `app/scripts/test_cancel_inventory.py`, cuyos cuatro casos (recibida→cero movimientos,
  pendiente→entrada real, en curso→sin movimiento/pérdida, mixto→solo lo pendiente) son la base
  directa de los escenarios de aceptación de User Story 2. El script no corre en CI hoy (brecha
  general documentada en A-27, fuera del alcance de esta spec).
- **SC-003**: Ningún pedido puede terminar `abierta` sin haber pasado por `confirm_order` — el
  100% de los intentos de alcanzar ese estado por una vía distinta (incluida la ya retirada
  `PATCH /status`) queda bloqueado, verificable inspeccionando que el único endpoint que muta
  `status` hacia `abierta` es `POST /orders/{id}/confirm` (`RN-ORD-65`, User Story 3).
- **SC-004**: El camino de cobro legado (`block`→`pay`) queda documentado con precisión suficiente
  para decidir en modernización, sin ambigüedad, si se retira o se conserva — incluida la
  confirmación explícita (`grep` cruzado en ambos repositorios, 0 resultados) de que no tiene
  caller real en la interfaz hoy (User Story 4).
- **SC-005**: Las anomalías `PENDIENTE` de esta spec (A-29/`RN-ORD-08`, A-38/`RN-ORD-31`,
  `RN-ORD-32`, A-42/`RN-ORD-24`) quedan documentadas con su comportamiento observado, su
  evidencia de código, y su tratamiento acordado (documentar sin especificar), de forma que el
  equipo de modernización no las reintroduzca por accidente ni las trate como si ya tuvieran una
  decisión de negocio cuando no la tienen.
- **SC-006**: La anomalía A-01 (camino B, código muerto de `compute_bill` en este módulo) queda
  señalada explícitamente como candidata a retiro o unificación con la convención correcta de la
  spec 010, evitando que se reactive sin corregirla durante la modernización.

## Out of Scope

- **La operación de cocina y la administración de mesas físicas** (transición de
  `estado_cocina`, consolidación de carritos, mover/fusionar mesas) — cubierto por la spec 009,
  aún no escrita en este reconocimiento (`RN-ORD-36` a `RN-ORD-64`, `RN-ORD-66`).
- **El cierre real de la sesión de mesa** (reparto entre comensales, cobro unificado o dividido,
  barrido de sesiones abandonadas, y la convención correcta de `compute_bill` que resuelve
  A-01) — cubierto por la spec 010, aún no escrita en este reconocimiento.
- **El constructor de venta compartido** (`build_sale`, la especificación completa del tope o
  prohibición de descuento manual del cajero que motiva A-11, la emisión de factura en punto
  único) — cubierto por la spec 011, aún no escrita en este reconocimiento.
- **El motor de evaluación de promociones y combos** (`promotions.evaluate`,
  `combo_discount_for_lines`, prioridad entre promociones aplicables) — cubierto por las specs
  012 y 013, aún no escritas en este reconocimiento; esta spec solo documenta **dónde** y
  **cuándo** el cobro legado invoca ese motor (`RN-ORD-07`, `RN-ORD-08`), no cómo decide el
  motor mismo.
- **El cálculo de qué consume cada línea** (receta fija, opciones, chequeo de disponibilidad) —
  cubierto por la spec 003 (`consumption_plan.py`); esta spec asume que
  `plan_line_consumption`/`required_consumption` ya calculan correctamente el consumo por línea y
  se limita a documentar cuándo se invoca (`out` al confirmar, `in` al revertir) y con qué
  garantías de simetría (`RN-ORD-27`).

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia del resto de
  las guías de este template ("evitar detalles de implementación"), aquí los endpoints, códigos
  de estado HTTP, nombres de campo, constantes internas (`_CONSUMED_KITCHEN`, `_NOT_DEDUCTED`) y
  valores literales **son** el contrato observable que se está documentando — se citan
  explícitamente porque los criterios de aceptación deben ser verificables directamente contra
  `pos-backend` en ejecución o contra el script de characterization citado.
- **RN-ORD-10 y RN-ORD-20 a RN-ORD-24 se especifican como invariantes obligatorios, no como
  comportamiento opcional**: junto con RN-ORD-65 (A-25), son las reglas con más evidencia de daño
  operativo histórico real si se rompen; cualquier reimplementación de este módulo debe tratarlas
  como tests de regresión de máxima prioridad.
- **A-25/RN-ORD-65 se especifica tal cual, sin tocar**: es una regla `[PROTEGIDA]` con dos
  testigos (CÓDIGO + `memoria-historica.md`) — la especificación formal declara "no existe
  transición libre de status" como invariante de diseño explícito.
- **A-29 (`RN-ORD-08`), A-38 (`RN-ORD-31`, `RN-ORD-32`) y la porción de interfaz de A-42
  (`RN-ORD-24`) se documentan pero NO se especifican como contrato**: siguiendo instrucción
  explícita de alcance, estas anomalías quedan con clasificación `PENDIENTE` en
  `registro-de-anomalias.md` — se describe el comportamiento observado hoy (porque esta spec
  documenta "lo que el sistema YA hace"), pero no se fija como el comportamiento correcto ni
  obligatorio para la modernización. Quedan como decisiones de negocio abiertas.
- **RN-ORD-34 se documenta como precisión documental, no como anomalía de comportamiento**: el
  propio registro de anomalías la trata como observación autoreconocida ("no es una
  contradicción real"), así que esta spec la especifica como corrección de redacción esperada en
  la modernización, no como comportamiento a decidir.
- **A-01 (camino B de esta spec) se documenta como código muerto candidato a retiro, no como
  comportamiento a preservar**: a diferencia de las reglas protegidas o intencionales de esta
  spec, `compute_bill` de `checkout.py` no se especifica como el cálculo correcto de cuenta —
  esa convención vive en la spec 010. Esta spec únicamente fija que, mientras el código exista,
  su comportamiento observado es el descrito en FR-003/RN-ORD-03.
- **A-11 se referencia, no se respecifica**: esta spec confirma que el alcance decidido por el
  negocio (ronda 3, P30) incluye el camino legado, pero delega la especificación completa del
  mecanismo de tope/prohibición a la spec 011, para no duplicar una regla compartida entre los
  tres caminos de cobro del sistema.
- **La ausencia de caller confirmado para `block_order`/`pay_order` (User Story 4) se verificó
  con `grep` cruzado en ambos repositorios (`pos-backend`, `pos-heladeria`) al momento de esta
  spec (2026-08-16), con 0 resultados de uso real** — solo se encontraron referencias de tipo
  TypeScript que documentan la forma del payload (`dining.interface.ts`), sin ninguna llamada de
  servicio que las invoque. No se puede descartar un caller externo al frontend conocido (un
  script, una integración no documentada), pero no hay evidencia de ninguno en este
  reconocimiento.
- **El script de characterization citado (`test_cancel_inventory.py`) no corre en CI
  actualmente**: vive en `app/scripts/` como script autoejecutable contra una base de datos real,
  no como suite de `pytest` automatizada (brecha general documentada en A-27, fuera del alcance
  de esta spec).
