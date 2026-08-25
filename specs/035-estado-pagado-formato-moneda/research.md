# Research: Estado "Pagada" Correcto y Formato de Moneda Reutilizable

## Decisión 1 — Dónde fijar `status = 'pagada'`

**Decisión** *(ampliada 2026-08-25 — ver más abajo "Corrección tras implementar")*: agregar
`order.status = "pagada"` en los tres puntos donde se construye una `Sale` sin dejar el
pedido en `'pagada'`, todos en `pos-backend/app/api/v1/orders/checkout.py`:
`checkout_and_send` (línea ~490, justo después de `_deduct_and_open`), y
`approve_payment_attempt`/`confirm_cash_payment_attempt` (justo después de su respectivo
`build_sale(...)`, antes del `db.commit()`). **No** se modifica `_deduct_and_open` ni
`_confirm_order_impl` en sí — siguen fijando únicamente `'abierta'`, porque los reusa también
`confirm_order` (vía de recuperación manual), que no construye ninguna `Sale`.

**Justificación**: `_deduct_and_open` es una función compartida por tres llamadores
(`checkout_and_send`, y — vía `_confirm_order_impl` — `confirm_order`,
`approve_payment_attempt` y `confirm_cash_payment_attempt`). De esos, `confirm_order` es el
único que **no** construye ninguna `Sale` en la misma llamada — los otros tres sí (spec 028).
Fijar `status = 'pagada'` en cada uno de esos tres puntos, después de su propio `build_sale`,
evita marcar como pagado un pedido que todavía no tiene ninguna venta (el único caso real:
`confirm_order`).

Los otros dos caminos que también crean una `Sale` (`checkout.pay_order:295` y
`table_sessions.service.close_session:~284`) **ya** fijan `status = 'pagada'` hoy — no
requieren cambio.

**Corrección tras implementar (2026-08-25)**: la primera versión de esta decisión solo cubría
`checkout_and_send`, asumiendo — incorrectamente — que el camino QR
(`approve_payment_attempt`/`confirm_cash_payment_attempt`) no generaba ninguna `Sale` hasta
cerrar la sesión de mesa. El negocio reportó el mismo síntoma para un pedido QR ya aprobado
por el cajero, lo que llevó a releer `checkout.py` con más cuidado: ambas funciones **ya**
construyen la `Sale` en su propia llamada desde spec 028 (ver su docstring, que lo explica
explícitamente) — el hueco real estaba ahí también, no solo en `checkout_and_send`. El propio
`table_sessions/service.py:_billable_orders` ya documentaba este hecho en su comentario
("evita facturar dos veces un pedido QR que ya se cobró al aprobar/confirmar su pago, aunque
su `status` siga en `'abierta'`") — una señal que la investigación original pasó por alto.

**Alternativas consideradas**:
- **Modificar `_deduct_and_open` para que reciba un parámetro `mark_paid: bool`**: técnicamente
  equivalente, pero cambia la firma de una función compartida por tres llamadores para un caso
  que solo aplica a uno — más superficie de cambio sin beneficio adicional sobre fijar el
  `status` directamente en `checkout_and_send`, después de la llamada.
- **Calcular `status = 'pagada'` de forma perezosa (como ya hace el campo `paid`)**: se
  descarta porque el pedido explícito del negocio (spec 035) es que el dato quede correcto
  **en base de datos**, no solo calculado en la respuesta — es exactamente lo que el campo
  `paid` (spec 029) ya resuelve para el frontend; esta spec corrige el dato de origen.

## Decisión 2 — Cómo proteger una mesa con comida sin terminar tras el cobro

**Decisión**: en `orders/tables_advanced.py`, la condición de "esta orden todavía bloquea"
**no** es una sola — son dos predicados distintos, porque el `TERMINAL = ('pagada',
'cancelada')` original se usaba con dos sentidos opuestos que solo coincidían por
casualidad (ambos ceden en `'pagada'`/`'cancelada'`, pero por razones distintas):

- **`_table_occupied_by_order(order)`** — ¿esta orden hace que su mesa cuente como "con
  trabajo pendiente"? Usado por `_active_orders_on_table` (línea 23-29, que a su vez
  alimenta `set_table_status` y los chequeos de mesa destino/origen de `move_order`).
  `'cancelada'` nunca ocupa la mesa; `'pagada'` deja de ocupar la mesa solo si no le quedan
  ítems `EN_CURSO`; cualquier otro estado siempre la ocupa.
- **`_order_locked_for_move_or_merge(order)`** — ¿esta orden **específica** no se puede
  mover/fusionar todavía? Usado por el chequeo propio de `move_order` (línea 49, sobre la
  orden que se mueve) y de `merge_orders` (línea 83, sobre las órdenes a fusionar).
  `'cancelada'` **siempre** lo impide (a diferencia del predicado anterior — no tiene sentido
  administrativo mover/fusionar una orden ya cancelada, aunque no "ocupe" la mesa); un estado
  no-terminal **nunca** lo impide (a diferencia del predicado anterior — siempre fue movible
  antes de esta spec); `'pagada'` lo impide solo si le quedan ítems `EN_CURSO`.

Ambos predicados comparten la misma pregunta de fondo para el caso `'pagada'` (¿le quedan
ítems `EN_CURSO`, `models/order_item.py`?, factorizada en `_has_pending_kitchen_work`), pero
difieren en cómo tratan `'cancelada'` y los estados no-terminales — de ahí que **no** se
pudieran unificar en un único booleano sin invertir alguno de los dos usos (se intentó primero
con un solo predicado reutilizado con `not` en `move_order`/`merge_orders`; el test nuevo de
T008 lo detectó de inmediato: una orden `'pagada'` con un ítem `'pendiente'` dejaba mover la
orden en vez de bloquearlo, porque el predicado único devolvía `True` para "ocupa la mesa" y
`not True` = `False` no bloqueaba el movimiento — la corrección fue separar los dos
predicados).

`_active_orders_on_table` pasó de una consulta SQL con `.notin_(TERMINAL)` a cargar las
órdenes de la mesa con sus ítems (`selectinload(CustomerOrder.items)`) y filtrar en Python con
`_table_occupied_by_order` — se prefirió sobre una `EXISTS` correlacionada en SQL porque el
resto del archivo y del módulo (`checkout.py:97-102`, `kitchen.py:83`,
`table_sessions/service.py:232,566`) ya resuelve "¿tiene ítems `EN_CURSO`?" en Python sobre la
colección `order.items`, nunca con una subconsulta SQL — consistente con la convención ya
establecida en vez de introducir una forma nueva.

`group_bill` (línea 94-125) **no** se toca — su exclusión de órdenes `pagada`/`cancelada` del
total es una decisión distinta y ya correcta (spec 019/017): una orden pagada no debe
recobrarse aunque cocina siga trabajando en ella, así que seguir excluyéndola del total es lo
correcto independientemente de este cambio.

**Justificación**: bajo el flujo "Cobrar y enviar" (spec 028), el cobro ocurre **antes** de
que cocina prepare el pedido — así que, a partir de esta spec, una orden puede estar
`'pagada'` y con ítems todavía `pendiente`/`en_preparacion`. Sin este ajuste, liberar la mesa,
moverla a otra, o fusionarla con otra dejaría de estar protegido en ese estado intermedio,
justo lo que el Acceptance Scenario 3 de Historia 1 prohíbe.

**Verificación de no-regresión sobre characterization tests**: se leyó
`test_orders_tables_advanced.py` completo. El único test `"CONGELA comportamiento actual:"`
que ejercita esta guarda (`test_set_table_status_409_con_ordenes_activas_y_ok_sin_ellas`,
líneas 54-69) usa una orden creada sin ningún `OrderItem`. Bajo la nueva condición, una orden
sin ítems no tiene ningún ítem "no listo" — el `EXISTS` es vacíamente falso — así que sigue
sin bloquear una vez `status = 'pagada'`, exactamente el resultado que el test ya espera. Los
CONGELA de `move_order`/`merge_orders` (líneas 73-125) tampoco usan `status = 'pagada'` en
ningún punto. **Ningún test CONGELA requiere modificación** — el caso nuevo (orden `'pagada'`
con ítems pendientes) se cubre con un test adicional, no con la edición de uno protegido.

**Alternativas consideradas**:
- **Revisar el `status` del pedido en el frontend (Terminal de Mesas) antes de permitir
  liberar/mover/fusionar, sin tocar el backend**: se descarta — el backend es la única
  frontera que puede garantizar la regla de forma consistente para cualquier cliente
  (dashboard, futuras integraciones), y ya es donde vive la guarda hoy.
- **No proteger este caso y aceptar el riesgo**: era la Opción B presentada al negocio en la
  fase de `/speckit-specify`; se descartó explícitamente en Clarifications (sesión
  2026-08-25, Q1: A).

## Decisión 2a — La misma protección, también en el frontend (descubierta al ampliar Decisión 1)

**Decisión**: en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`, los
`computed`/métodos `activeOrders` y `tableOrders` dejan de excluir toda orden
`status === 'pagada'` sin condición — igual que en el backend (Decisión 2), una orden
`'pagada'` con ítems `estado_cocina` en `EN_CURSO`/`KITCHEN_NOT_READY` (`'pendiente'` |
`'en_preparacion'`) sigue contando como consumo vivo de la mesa. Se agrega
`hasPendingKitchenWork` en `src/app/modules/orders/order-status.util.ts` (junto a
`KITCHEN_NOT_READY`, que ya vivía ahí) para no duplicar el criterio en el store.

**Justificación**: al ampliar la Decisión 1 para cubrir también `approve_payment_attempt`/
`confirm_cash_payment_attempt` (ver Decisión 1, "Corrección tras implementar"), se releyó
`pos-terminal.store.ts` y se encontró que tiene su **propio** criterio independiente de "esta
mesa todavía tiene consumo activo" — no llama a ningún endpoint del backend para decidirlo,
lo calcula localmente sobre los pedidos ya cargados. Ese criterio tenía exactamente el mismo
patrón que `_active_orders_on_table` tenía antes de la Decisión 2 (`status !== 'pagada'`, sin
mirar cocina). Sin corregirlo también aquí, una orden que pasara a `'pagada'` por el fix de
Decisión 1 mientras cocina la sigue preparando habría hecho que `tableOrders(tableId)`
quedara vacío para esa mesa, y `centralState`/`deriveTableStatus` la habrían mostrado como
`'mesa-libre'`/sin consumo en el tablero del cajero — el mismo problema de Decisión 2, pero
del lado del frontend y sin ningún endpoint de por medio que lo hubiera evitado.

`deriveTableStatus` (mismo archivo) no necesitó cambios de lógica — ya usa `order.paid`, no
`status`, para decidir `'listo'` vs `'pago_pendiente'` (comentario propio, citando D2 de spec
029) — pero su comentario sí se corrigió: afirmaba que "los caminos QR/mostrador nunca llegan
a `status === 'pagada'`", ya no es cierto tras esta spec.

**Alternativas consideradas**:
- **No corregir el frontend, confiar en que `paid` ya es la señal correcta ahí también**: se
  descarta — es precisamente el error que llevó a este hallazgo: `paid` es la señal correcta
  para *mostrar* "Pagada" (`displayOrderStatus`), pero `activeOrders`/`tableOrders` no
  preguntan "¿está pagado?", preguntan "¿la mesa todavía tiene algo pendiente?" — dos
  preguntas distintas que antes coincidían por accidente (ninguna orden llegaba a `'pagada'`
  mientras cocinaba) y dejaron de coincidir en cuanto Decisión 1 corrigió `status`.

## Decisión 3 — Diseño del componente de moneda reutilizable

**Decisión**: nuevo componente standalone `MoneyInputComponent`
(`pos-heladeria/src/app/shared/money-input/money-input.component.ts`), que implementa
`ControlValueAccessor` auto-inyectando `NgControl` en el constructor (mismo patrón exacto que
`shared/password-input/password-input.component.ts` y
`shared/searchable-select/searchable-select.component.ts` — sin proveedor
`NG_VALUE_ACCESSOR`, por el mismo motivo de DI circular ya documentado en esos archivos).
Formatea con `Intl.NumberFormat('es-CO')` (mismo locale que `shared/money.ts:formatMoney`) en
cada evento `input`, preservando la posición del cursor. El valor que expone vía
`writeValue`/`onChange` al `FormControl`/`ngModel` que lo use es siempre el número limpio
(sin separador), nunca la cadena formateada.

**Justificación**: reutiliza un patrón CVA ya probado en el proyecto en vez de inventar uno
nuevo, y reutiliza el mismo locale/formato que ya usa `formatMoney` para la visualización —
así lo que el usuario escribe y lo que ve después coinciden exactamente (Clarification Q2:
A). Al ser un `ControlValueAccessor`, funciona indistintamente con `formControlName`
(reactive forms, la mitad de los ~12 puntos de uso) y con `[(ngModel)]`/`[ngModel]` (la otra
mitad) sin que cada punto de uso tenga que adaptar su forma de enlazar el valor.

**Casos especiales que el diseño debe cubrir** (de `spec.md`, FR-009/FR-010/FR-011):
- **Vacío se queda vacío** (`scope-picker.component.ts`, valor de paquete que hereda el
  default): el componente no debe convertir `''`/`null` en `0` — `writeValue(null)` limpia el
  campo, y un campo vacío emite `null` (no `0`) por `onChange`.
- **Decimales** (`option-form.component.ts:extra_price`, `inventory-item-form.component.ts:
  unit_cost`, `purchase-form.component.ts`, `plan-form.component.ts:precio_mensual/anual`,
  todos con `step="0.01"` hoy): el componente acepta un `@Input() decimals` opcional
  (`0` por defecto, igual que `formatMoney` que redondea a peso entero); los campos que hoy
  permiten centavos lo activan explícitamente.
- **Campo dual porcentaje/pesos** (`promotions-page.component.ts`, líneas 463/757): el
  componente en sí siempre formatea como moneda — es el propio formulario de promociones
  quien decide, con la misma lógica condicional que ya tiene hoy (`form.type === 'percent'`),
  si renderiza el `MoneyInputComponent` o un `<input type="number">` simple según el modo
  activo. No se le pide al componente nuevo que sepa de porcentajes.

**Alternativas consideradas**:
- **Directiva en vez de componente** (aplicada sobre el `<input>` existente en cada punto de
  uso): se descarta — una directiva no puede encapsular igual de bien el `ControlValueAccessor`
  completo (necesitaría seguir coexistiendo con el `type="number"` nativo del navegador, que
  no admite bien un valor con separadores mientras se escribe); un componente propio con su
  propio `<input type="text" inputmode="decimal">` interno da control total sobre el
  formateo en cada pulsación.
- **Librería externa** (`ngx-mask`, `imask`, etc.): se descarta — no hay ninguna instalada hoy,
  y el formateo de miles se resuelve por completo con `Intl.NumberFormat` nativo; agregar una
  dependencia solo para esto no se justifica (Principio IX).
- **Formatear en el propio `formatMoney`/`money` pipe existentes**: se descarta — esas
  utilidades son de solo-visualización (transforman un número ya conocido para interpolarlo
  en una plantilla); no manejan eventos de teclado, posición del cursor, ni la entrada activa
  de un `<input>`. Se reutiliza su mismo locale (`es-CO`), no su implementación.

## Decisión 4 — Registro de la decisión de negocio (Principio II)

**Decisión**: antes de implementar Historia 1, se agrega una entrada a
`specs/000-reconocimiento/registro-de-anomalias.md` documentando: qué cambia (`status` de una
orden cobrada por Terminal de Mesas pasa a `'pagada'`, revirtiendo para ese caso puntual la
decisión de spec 029), por qué (el dato crudo debe ser correcto para cualquier consumidor que
no pase por el campo calculado `paid`), quién y cuándo autoriza el cambio, y qué
funcionalidades quedan afectadas (las tres funciones de `tables_advanced.py` identificadas en
Decisión 2).

**Justificación**: exigencia directa del Principio II de la Constitución — un cambio de
comportamiento existente documentado en un spec anterior no se revierte sin esa entrada,
incluso cuando el propio spec nuevo ya lo autoriza explícitamente en su texto.

**Alternativas consideradas**: ninguna — es un requisito de proceso, no una decisión técnica
con alternativas.
