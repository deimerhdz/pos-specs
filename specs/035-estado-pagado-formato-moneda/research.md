# Research: Estado "Pagada" Correcto y Formato de Moneda Reutilizable

## Decisión 1 — Dónde fijar `status = 'pagada'`

**Decisión**: agregar `order.status = "pagada"` dentro de
`checkout.checkout_and_send` (`pos-backend/app/api/v1/orders/checkout.py:421-505`), justo
después de `_deduct_and_open(db, order, cashier)` (línea 490), en la misma transacción que ya
crea la `Sale` vía `build_sale` (línea 471). **No** se modifica `_deduct_and_open` en sí
(sigue fijando `'abierta'` para sus otros dos llamadores).

**Justificación**: `_deduct_and_open` es una función compartida por tres caminos
(`checkout_and_send`, y — vía `_confirm_order_impl` — `confirm_order` y las confirmaciones de
pago QR, `approve_payment_attempt`/`confirm_cash_payment_attempt`). Solo en
`checkout_and_send` existe ya una `Sale` en el momento en que se llama a
`_deduct_and_open`: los otros dos caminos confirman el pedido hacia cocina **antes** de que
exista cualquier venta (el comensal QR paga o transfiere, pero la `Sale` real se crea después,
al cerrar la sesión de mesa — `close_session`). Si `_deduct_and_open` fijara `'pagada'` de
forma genérica, marcaría como pagados pedidos QR que todavía no tienen ninguna venta.

Los otros dos caminos que sí crean una `Sale` (`checkout.pay_order:295` y
`table_sessions.service.close_session:~284`) **ya** fijan `status = 'pagada'` hoy — no
requieren cambio. El único hueco real es `checkout_and_send`.

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

**Decisión**: en `orders/tables_advanced.py`, la condición de "esta orden todavía bloquea la
mesa" deja de ser `status not in ('pagada', 'cancelada')` y pasa a ser: `status != 'cancelada'
AND (status != 'pagada' OR existe algún OrderItem de la orden con estado_cocina NOT IN
('listo', 'anulado'))`. Se aplica en los tres puntos que hoy comparan contra `TERMINAL`:
- `_active_orders_on_table` (línea 23-29): se agrega un `EXISTS` correlacionado contra
  `order_items` para el caso `status = 'pagada'`.
- `move_order`, chequeo de la orden que se mueve (línea 49): mismo predicado, evaluado en
  Python sobre la orden ya cargada (con sus `items`).
- `merge_orders`, chequeo de las órdenes a fusionar (línea 83): igual, en Python sobre las
  órdenes ya cargadas.

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
