# Feature Specification: Estado "Pagada" Correcto y Formato de Moneda Reutilizable

**Feature Branch**: `035-estado-pagado-formato-moneda`

**Created**: 2026-08-25

**Status**: Draft

**Naturaleza de esta spec**: agrupa dos correcciones/mejoras independientes que el negocio
reportó juntas en la misma solicitud (Historia 1 y Historia 2 de abajo son verificables y
desplegables por separado, Principio VI de la
[Constitución](../../.specify/memory/constitution.md)):

- **Historia 1** cambia un comportamiento ya documentado en
  [spec 029](../029-correccion-cobro-cierre-mesa/spec.md) (el campo `status` de una orden
  nunca llegaba a `'pagada'` en el camino de cobro de Terminal de Mesas, a propósito — el
  frontend usa un campo calculado `paid` en su lugar). Esta spec autoriza explícitamente
  revertir esa decisión **solo** para el camino de cobro identificado abajo (Principio II:
  comportamiento existente protegido salvo decisión de negocio explícita, registrada aquí).
- **Historia 2** es una capacidad nueva (Principio IV) que no cambia ninguna regla de
  negocio de precios existente — solo cómo se capturan en pantalla.

**Input**: User description: "necesito solucionar dos problemas, en la terminal de mesas
cuando se confirma el pago de una orden, esta queda en estado abierta, pero en realidad
debería de quedar en estado pagada, en base de datos y en el frontend en la lista de
órdenes se está mostrando como si estuviera pagada; y la otra tarea es que quiero que hagas
reutilizable cualquier input que se use para ingresar precios o montos, para que se auto
formate y aparezca en formato de moneda de Colombia, por ejemplo 50,000 o 8,000"

## Hallazgo relevante (antes de leer los requisitos)

Se investigó el comportamiento actual antes de escribir esta spec, porque lo reportado no
coincidía exactamente con el código:

- La lista "Órdenes" (pantalla "Comandas de la operación") **ya muestra "Pagada"
  correctamente** para la orden del ejemplo — lo hace a propósito, calculando si ya existe
  una Venta (`Sale`) asociada al pedido (campo `paid`), precisamente porque spec 029 ya
  demostró que confiar en el campo `status` crudo era insuficiente. La pestaña "Pagadas" del
  mismo listado también filtra correctamente por esa misma señal. Esa parte de lo reportado
  **no está rota** — es el comportamiento ya vigente y correcto.
- Lo que sí es cierto: el campo `status` en base de datos se queda en `'abierta'` cuando el
  cobro se hace desde Terminal de Mesas ("Cobrar y enviar" sobre una comanda), aunque la
  Venta ya exista. Esto expone datos inconsistentes a cualquier consumidor que lea `status`
  directamente (reportes, exportaciones, integraciones futuras) en vez de pasar por el campo
  calculado `paid`.
- El motivo por el que spec 029 evitó tocar `status` en ese punto no fue arbitrario: la
  misma función interna que deja la orden en `'abierta'` (`_deduct_and_open`, en
  `pos-backend/app/api/v1/orders/checkout.py`) también la usan otros dos caminos donde
  **todavía no existe ninguna Venta** en ese instante (el comensal por QR, cuyo pago se
  confirma antes de que exista Venta — la Venta se crea después, al cerrar la sesión de
  mesa). Cambiar esa función de forma genérica marcaría como "pagada" una orden que en
  realidad no tiene ninguna Venta todavía.
- Además, varias operaciones de gestión de mesa (liberar, mover una orden a otra mesa,
  fusionar mesas — `pos-backend/app/api/v1/orders/tables_advanced.py`) usan `status` para
  decidir si una mesa "todavía tiene trabajo pendiente". Bajo el flujo de "Cobrar y enviar"
  (spec 028), el cobro ocurre **antes** de que cocina prepare el pedido — así que una orden
  recién cobrada por ese camino puede tener ítems todavía sin preparar. Marcarla `'pagada'`
  sin más podría permitir liberar o reasignar esa mesa mientras cocina sigue trabajando en
  ella.

Estos hallazgos se reflejan en el alcance de la Historia 1 abajo, incluyendo un punto que
necesita una decisión de negocio explícita (ver Clarifications).

## Corrección 2026-08-25 (durante la implementación)

Tras implementar la primera versión de Historia 1 (acotada a `checkout_and_send`), el negocio
reportó que un pedido QR seguía en `status: "abierta"` con `paid: true` **después** de que el
cajero aprobara su comprobante desde la Terminal de Mesas — el mismo síntoma reportado
originalmente, en un camino que el hallazgo de arriba había asumido, de forma incorrecta, que
todavía no generaba ninguna Venta.

**Lo que el hallazgo original tenía mal**: `approve_payment_attempt` y
`confirm_cash_payment_attempt` (`pos-backend/app/api/v1/orders/checkout.py`) — las funciones
que el cajero dispara al aprobar un comprobante o confirmar efectivo de un pedido QR desde la
Terminal de Mesas — **ya generan la Venta en esa misma llamada** desde spec 028 (antes de esa
spec, la Venta solo se generaba al cerrar la sesión de mesa; spec 028 lo adelantó para que
"Reimprimir Factura" y "Liberar Mesa" funcionaran sobre un pedido QR individual, ver el
docstring de `approve_payment_attempt`). El código de `pos-backend/app/api/v1/table_sessions/
service.py:_billable_orders` ya documentaba este mismo hecho en su propio comentario ("evita
facturar dos veces un pedido QR que ya se cobró al aprobar/confirmar su pago, aunque su
`status` siga en `'abierta'`") — la spec original no lo tuvo en cuenta al investigar.

**Lo que sigue siendo cierto**: el camino de recuperación manual `confirm_order` (llamado
directamente, sin pasar por aprobar/confirmar un pago) no genera ninguna Venta — solo
funciona sobre una orden cuyo intento de pago ya está `'confirmado'` por otro medio, y en ese
caso no debe marcarse `'pagada'` (Acceptance Scenario 4a).

**Impacto adicional descubierto**: la Terminal de Mesas (frontend,
`pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`) tiene su propio
criterio independiente de "esta mesa todavía tiene consumo activo" (`activeOrders`/
`tableOrders`), que también excluía `status === 'pagada'` sin mirar cocina — el mismo patrón
que FR-003 ya corrigió en el backend (`tables_advanced.py`), pero en un lugar distinto que la
spec original no había revisado. Sin corregirlo también ahí, un pedido QR pagado mientras
cocina lo sigue preparando habría hecho ver la mesa como libre en el tablero del cajero. Ver
FR-003a (nueva).

Ver FR-002 (corregida) y FR-003a (nueva) abajo.

## Clarifications

### Session 2026-08-25

- Q: Al corregir el `status` de una orden cobrada por Terminal de Mesas para que sí llegue a
  `'pagada'` en base de datos, ¿el sistema debe además dejar de usar `status` como señal de
  "esta mesa todavía tiene trabajo pendiente de cocina" en las operaciones de liberar/mover/
  fusionar mesa, para no perder la protección que existe hoy? → A: Sí — corregir `status`
  **y** actualizar esas operaciones para que la protección dependa de si quedan ítems sin
  terminar de preparar en cocina, no de `status`.
- Q: El separador de miles a usar en el formato de moneda — ¿el mismo que ya usa el resto de
  la aplicación (punto, ej. `$ 50.000`, formato colombiano real usado hoy en
  `shared/money.ts`), o el que aparece en el ejemplo literal del usuario (coma, ej.
  `50,000`, estilo estadounidense)? → A: Punto — igual que `shared/money.ts` ya usa en toda
  la aplicación, para mantener consistencia entre lo que se escribe y lo que se muestra
  después.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El estado de un pedido cobrado queda "pagada" de verdad (Priority: P1)

Un cajero cobra una comanda desde la Terminal de Mesas ("Cobrar y enviar"), o aprueba el
comprobante/confirma el efectivo de un pedido enviado por un comensal vía QR. Hoy, aunque la
venta se registra correctamente en los tres casos y el listado de Órdenes ya la muestra como
"Pagada" (ver Hallazgo relevante), el campo `status` del pedido en base de datos se queda en
`'abierta'`. El negocio necesita que, en el momento en que se confirma cualquiera de esos
cobros, el `status` real del pedido pase a `'pagada'` — para que cualquier reporte,
exportación o integración que consulte ese campo directamente vea un dato correcto, sin
depender de que cada consumidor sepa que debe calcular el estado real a partir de si existe
una venta.

**Why this priority**: es el defecto de datos que originó el reporte — aunque la pantalla
principal ya compensa el problema, el dato de origen sigue siendo incorrecto y cualquier
otro punto del sistema que lo use directamente hereda ese error.

**Independent Test**: cobrar una comanda desde Terminal de Mesas, o aprobar/confirmar el pago
de un pedido QR, y consultar el pedido directamente (por API o en base de datos): su `status`
debe ser `'pagada'`, no `'abierta'`.

**Acceptance Scenarios**:

1. **Given** una comanda con ítems pendientes de cobro en Terminal de Mesas, **When** el
   cajero confirma "Cobrar y enviar", **Then** el pedido queda con `status = 'pagada'` en
   base de datos, en la misma operación en la que se registra la venta — no en un paso
   posterior ni con retraso.
2. **Given** un pedido recién cobrado por ese camino, **When** se consulta el listado de
   Órdenes, **Then** sigue mostrando "Pagada" exactamente igual que hoy (este comportamiento
   ya es correcto y no debe cambiar).
3. **Given** un pedido recién cobrado por ese camino cuyos ítems todavía no terminan de
   prepararse en cocina, **When** el personal intenta liberar la mesa, moverla o fusionarla
   con otra, **Then** el sistema sigue impidiéndolo mientras quede trabajo de cocina
   pendiente — el cambio de `status` no debe abrir una forma nueva de perder de vista una
   mesa con comida sin terminar.
4. **Given** un comprobante de transferencia que el cajero aprueba, o un pago en efectivo que
   el cajero confirma, para un pedido enviado por el comensal vía QR, **When** eso ocurre,
   **Then** el pedido queda con `status = 'pagada'` en la misma operación en la que se
   registra la venta — igual que en el Escenario 1, sin esperar a que se cierre la sesión de
   mesa *(corregido 2026-08-25: el redactado original de este escenario asumía, de forma
   incorrecta, que este camino no genera venta hasta cerrar la mesa — ver Corrección abajo)*.
4a. **Given** un intento de pago que ya quedó `'confirmado'` sin que exista todavía una venta
   para su pedido (vía de recuperación manual, `confirm_order`, sin pasar por
   aprobar/confirmar un pago) **When** el personal lo confirma para que pase a cocina,
   **Then** su `status` avanza únicamente a `'abierta'` — este camino de recuperación no crea
   ninguna venta, así que no debe marcarse `'pagada'`.
5. **Given** una comanda cuyo cobro falla por falta de stock al momento de enviarla a
   cocina, **When** eso ocurre, **Then** ni la venta ni el cambio de `status` quedan
   aplicados — se revierte todo junto, igual que hoy.

---

### User Story 2 - Cualquier campo de precio o monto se auto-formatea como moneda colombiana (Priority: P2)

Cualquier persona del staff que escriba un precio o un monto en cualquier pantalla del
sistema (precio de un producto o presentación, costo de un insumo, fondo inicial de caja,
un movimiento de caja, un descuento, el precio de un plan, etc.) ve el número formateado con
separador de miles mientras escribe, en vez de una cadena de dígitos sin separar — igual que
ya se ve el mismo número una vez guardado en el resto de la aplicación.

**Why this priority**: es una mejora de calidad percibida y de prevención de errores (un
cajero puede confundir `50000` con `500000` a simple vista mientras escribe), pero ninguna
regla de negocio de precios cambia — no bloquea ninguna operación existente.

**Independent Test**: abrir cualquier pantalla con un campo de precio o monto (por ejemplo,
"Fondo inicial" al abrir caja) y escribir un número; el campo debe mostrar el separador de
miles en vivo, y el valor que finalmente se guarda/envía debe seguir siendo el número
correcto, sin el separador.

**Acceptance Scenarios**:

1. **Given** un campo de precio o monto vacío, **When** el usuario escribe `500000`,
   **Then** el campo muestra el número ya formateado con separador de miles mientras se
   escribe (no solo después de guardar).
2. **Given** un campo con un valor ya formateado, **When** el usuario lo borra y escribe un
   número nuevo, o pega un valor desde el portapapeles, **Then** el campo se reformatea
   correctamente y el valor numérico subyacente (el que se guarda o se envía al backend)
   sigue siendo un número limpio, sin el separador ni ningún otro carácter.
3. **Given** un campo de precio o monto que hoy permite dejarse vacío con un significado
   propio (por ejemplo, "hereda el precio por defecto del paquete" en la configuración de
   promociones), **When** el usuario lo deja vacío, **Then** el campo sigue aceptando ese
   vacío como válido — no se fuerza a mostrar `0`.
4. **Given** un campo que alterna entre "porcentaje" y "monto en pesos" según otra opción
   elegida en el mismo formulario (por ejemplo, el descuento de una promoción), **When** el
   modo activo es "monto en pesos", **Then** se aplica el formato de moneda; **When** el
   modo activo es "porcentaje", **Then** no se aplica (el campo se comporta como un número
   simple).
5. **Given** un campo de precio o monto que admite centavos (por ejemplo, precio de un plan
   o costo de un insumo), **When** el usuario escribe decimales, **Then** el formato de
   miles no le impide escribir ni ver los centavos.

---

### Edge Cases

- Un usuario borra todo el contenido de un campo de moneda y lo deja vacío: no debe quedar
  mostrando `$ 0` ni un separador suelto — el campo queda genuinamente vacío.
- Un usuario pega un texto con caracteres no numéricos (por ejemplo, copia un precio desde
  un documento con el símbolo de moneda incluido): el campo debe quedarse solo con los
  dígitos válidos, sin romper el formato ni bloquear la escritura.
- Doble clic o reenvío accidental de "Cobrar y enviar" sobre la misma comanda: el chequeo de
  versión ya existente sigue evitando una segunda venta o un segundo cambio de estado (sin
  cambios respecto a hoy).
- Una integración o reporte externo que ya lee el campo `status` crudo de pedidos cobrados
  **antes** de este cambio seguirá viendo esos pedidos históricos en `'abierta'` — esta spec
  no reescribe datos ya existentes, solo corrige el camino hacia adelante.

## Requirements *(mandatory)*

### Functional Requirements

**Estado "pagada" (Historia 1)**

- **FR-001**: Cuando se confirma el cobro de una comanda desde Terminal de Mesas ("Cobrar y
  enviar"), el sistema DEBE dejar el `status` del pedido en `'pagada'` en la misma operación
  en la que registra la venta.
- **FR-002** *(corregida 2026-08-25 — ver Corrección abajo)*: El sistema DEBE dejar el
  `status` de un pedido QR en `'pagada'` en la misma operación en la que el cajero aprueba su
  comprobante de transferencia o confirma su pago en efectivo (`approve_payment_attempt`/
  `confirm_cash_payment_attempt`, spec 026/028) — ambas ya registran la venta en ese mismo
  instante, no al cerrar la sesión de mesa. El sistema NO DEBE marcar `'pagada'` una orden en
  ningún punto del ciclo de vida en el que todavía no exista una venta registrada para ella
  (en particular, `confirm_order` como vía de recuperación manual — spec 026, sin comprobante
  ni pago de por medio — sigue avanzando solo a `'abierta'`, sin crear ninguna venta).
- **FR-003**: Las operaciones de liberar una mesa, mover una orden a otra mesa, y fusionar
  mesas (que hoy usan `status` para decidir si una mesa "todavía tiene trabajo pendiente")
  DEBEN dejar de asumir que una orden con `status = 'pagada'` ya no tiene nada pendiente en
  cocina. En su lugar, DEBEN bloquear esas operaciones mientras la orden tenga algún ítem
  todavía sin terminar de preparar (`estado_cocina` distinto de `listo`/`anulado`),
  independientemente de si su `status` ya es `'pagada'` — así se conserva la protección
  actual (Acceptance Scenario 3) sin depender de que `status` siga siendo no-terminal.
- **FR-003a** *(nueva 2026-08-25)*: La Terminal de Mesas (frontend) DEBE aplicar el mismo
  criterio de FR-003 al decidir si una mesa "todavía tiene consumo activo" para pintar su
  tablero (`PosTerminalStore.activeOrders`/`tableOrders`) — una orden `'pagada'` con ítems
  sin terminar de preparar sigue contando como consumo vivo de la mesa, para que la mesa no
  se vea libre mientras cocina sigue trabajando en un pedido ya cobrado.
- **FR-004**: El sistema DEBE seguir mostrando "Pagada" en el listado de Órdenes exactamente
  igual que hoy, para cualquier pedido con una venta registrada — este cambio no debe
  alterar ese comportamiento ya correcto.
- **FR-005**: El sistema DEBE seguir aplicando, sin cambios, la garantía ya vigente de que un
  cobro fallido (por ejemplo, por falta de stock al enviar a cocina) no deja ni la venta ni
  el cambio de estado aplicados parcialmente.

**Formato de moneda reutilizable (Historia 2)**

- **FR-006**: El sistema DEBE ofrecer un único control de entrada reutilizable para capturar
  precios y montos, usado consistentemente en todas las pantallas donde hoy se captura un
  precio o un monto en pesos (ver Assumptions para el inventario identificado).
- **FR-007**: Ese control DEBE mostrar el número con separador de miles mientras el usuario
  escribe, no solo después de guardar, usando el mismo formato colombiano (punto como
  separador de miles, ej. `$ 50.000`) que ya usa `shared/money.ts` para mostrar precios
  guardados en el resto de la aplicación — sin introducir un segundo formato distinto.
- **FR-008**: El valor que el control entrega al resto del formulario (el que se guarda o se
  envía al backend) DEBE ser siempre un número limpio, sin el separador de miles ni ningún
  otro carácter de formato.
- **FR-009**: El control DEBE permitir que el campo quede vacío cuando el campo lo permite
  hoy (por ejemplo, un valor de paquete que hereda un valor por defecto) — un campo vacío no
  se convierte en `0`.
- **FR-010**: El control DEBE admitir campos que hoy permiten decimales (centavos) sin que
  el formato de miles interfiera con escribirlos o verlos.
- **FR-011**: Un campo que hoy alterna entre "porcentaje" y "monto en pesos" según otra
  selección del mismo formulario DEBE aplicar el formato de moneda únicamente cuando
  representa un monto en pesos.

### Key Entities

- No se introducen entidades ni campos de datos nuevos. Historia 1 cambia el valor que ya
  toma un campo existente (`CustomerOrder.status`) en un punto adicional de su ciclo de
  vida ya definido (spec 029, `data-model.md`). Historia 2 es puramente de interfaz —no
  cambia cómo se almacena ningún precio o monto.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los pedidos cobrados por cualquiera de los tres caminos que generan
  una venta y no dejaban `status='pagada'` (`checkout_and_send`, `approve_payment_attempt`,
  `confirm_cash_payment_attempt`) quedan con `status = 'pagada'` en base de datos
  inmediatamente al confirmarse el cobro.
- **SC-002**: El 0% de los pedidos confirmados por la vía de recuperación manual
  (`confirm_order`, sin venta todavía) queda marcado `'pagada'` antes de que exista una venta
  real para ellos.
- **SC-003**: El listado de Órdenes sigue mostrando "Pagada" para el 100% de los pedidos con
  venta registrada, sin regresión respecto al comportamiento actual.
- **SC-004**: El 100% de los campos de precio/monto identificados en esta spec muestran el
  número con separador de miles mientras se escribe, y el valor guardado coincide
  exactamente con el número que el usuario quiso ingresar (sin errores de redondeo ni de
  formato) en el 100% de los casos probados.
- **SC-005**: El 0% de las mesas con una orden `'pagada'` por cualquiera de los tres caminos
  de SC-001 y comida todavía en preparación puede liberarse, moverse o fusionarse — ni en las
  operaciones del backend (`tables_advanced.py`) ni en el tablero de la Terminal de Mesas
  (frontend, FR-003a) — mismo criterio de protección que
  existe hoy, verificado de nuevo tras el cambio de estado.

## Assumptions

- **Historia 1 se acota a los tres caminos que registran una Venta sin dejar el pedido en
  `'pagada'`** *(ampliado 2026-08-25 — ver Corrección arriba)*: `checkout_and_send` (Terminal
  de Mesas, "Cobrar y enviar"), `approve_payment_attempt` (aprobar comprobante de un pedido
  QR) y `confirm_cash_payment_attempt` (confirmar efectivo de un pedido QR) — los tres, spec
  028. Los otros dos caminos que también generan una venta (`pay_order`, el camino legado de
  cobro, y el cierre de sesión de mesa, `close_session`) **ya** dejan el pedido en `'pagada'`
  hoy — no requieren cambio. `confirm_order` (vía de recuperación manual, sin generar venta)
  tampoco cambia — sigue avanzando solo a `'abierta'`.
- **El campo calculado `paid` (spec 029) no se retira** — sigue siendo la señal que ya usan
  correctamente el listado de Órdenes y otras protecciones del sistema (por ejemplo, contra
  anular un ítem de un pedido ya pagado). Esta spec solo agrega que, además, `status`
  también quede correcto para el camino identificado.
- **Esta spec revierte, para un caso puntual, una decisión de negocio ya registrada en
  spec 029** — corresponde documentar esta decisión en
  `specs/000-reconocimiento/registro-de-anomalias.md` (Principio II de la Constitución)
  antes de implementarla, señalando qué cambia, por qué, y qué funcionalidades quedan
  afectadas (las identificadas en FR-003).
- **Inventario de campos de precio/monto a migrar a Historia 2** (relevado antes de escribir
  esta spec, en `pos-heladeria`): fondo inicial de caja y movimientos de caja (módulo caja),
  efectivo contado en arqueo parcial, precio por presentación de producto, precio extra de
  una opción, costo unitario de insumo (alta y por línea de compra), precio de combo y monto
  de descuento de promociones, precio de paquete en el selector de alcance de promociones,
  monto que paga el comensal y monto recibido en confirmación de pago en efectivo (módulo
  mesas), y precio mensual/anual de un plan (Super Admin). Cualquier campo de precio o monto
  que se agregue después de esta spec debe usar el mismo control reutilizable.
- **Los campos que representan cantidades, no dinero** (unidades de un insumo, cantidad
  mínima de una promoción, número de billetes/monedas en un arqueo, longitud de un campo de
  catálogo de métodos de pago) quedan fuera de alcance — no se les aplica formato de moneda.
- **El control reutilizable se construye sin depender de una librería externa nueva** — no
  existe hoy ninguna ya instalada en `pos-heladeria` para esto, y el formato de miles se
  puede resolver con las utilidades nativas del navegador, igual que ya lo hace
  `shared/money.ts` para la visualización. Si el trabajo posterior de diseño encuentra que
  hace falta una dependencia nueva, debe justificarse aparte (Principio IX).
