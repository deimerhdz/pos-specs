# Phase 0 Research: Carga diferida de datos y tarjetas de pedido de Domicilio/Para Llevar

Todas las incógnitas técnicas de este plan se resolvieron por investigación directa del código
actual de `pos-heladeria` (citado en `spec.md` y en `plan.md`), no quedó ningún
`NEEDS CLARIFICATION` pendiente en el Technical Context. Este documento consolida esas decisiones.

## 1. Qué peticiones diferir y cuáles mantener eager

**Decision**: Diferir únicamente `PaymentMethodService.load()`,
`PaymentMethodService.loadAvailableForCheckout()` y `CashService.restoreShift()`. Mantener eager
`TableService.loadTables()`, `reloadOrders()` (`DiningSessionService.listOrders()`),
`MenuService.loadMenu()` y `PromotionService.loadActive()`.

**Rationale**: `orderSubtotal()` (`pos-terminal.store.ts:1788`) — que alimenta el total con
descuento ya visible en cada tarjeta de `tablesView()` desde el primer render — depende de
`menuService.categories()` (vía `lookup()`, línea 413-415) y de `promotionService.activePromotions()`
para calcular el precio con descuento por producto/categoría. Diferir cualquiera de esos dos
mostraría un total incorrecto (sin descuento) en el primer render, una regresión de exactitud que
el spec descarta explícitamente (FR-004, Out of Scope). En cambio, `paymentMethodsAvailable()` y
`cashShiftId` no tienen ningún consumidor fuera de `pos-checkout-panel.component.ts`, confirmado
por grep sobre todo el módulo `tables/`.

**Alternatives considered**:
- *Diferir también menú/promociones*: descartada — rompería SC-004 (el tiempo hasta que la grilla
  es operable no debe verse afectado) al mostrar totales incorrectos que luego "saltan" cuando la
  petición diferida resuelve; el spec la descarta explícitamente en Out of Scope.
- *Diferir todo detrás de un `IntersectionObserver`/lazy-render genérico*: descartada por
  sobre-ingeniería — el spec pide un disparador concreto y observable (selección de un pedido), no
  un mecanismo de virtualización de UI.

## 2. Punto exacto de disparo de la carga diferida

**Decision**: Disparar `PaymentMethodService.load()` + `.loadAvailableForCheckout()` +
`CashService.restoreShift()` (cuando aplique, ver §4) en el mismo punto donde hoy se establece
`selectedOrderId` con un valor real — es decir, dentro de `selectTable()` cuando la mesa tiene al
menos un pedido, y dentro del nuevo `selectOrder()` para pedidos sin mesa (§3) — nunca al solo
seleccionar una mesa libre.

**Rationale**: confirmado por clarificación con el dueño de negocio (`spec.md`, sección
Clarifications) y por lectura directa de `pos-checkout-panel.component.ts:131-144`: el selector de
método de pago (`app-payment-input`, que consume `paymentMethodsAvailable()`) y el turno de caja
usado en `checkout()` (`pos-terminal.store.ts:1636`) solo se renderizan/usan dentro del `@else`
que exige `store.selectedOrder()` con valor — la rama de mesa libre solo muestra
"+ Crear pedido nuevo", sin ninguno de esos dos datos.

**Alternatives considered**:
- *Disparar al seleccionar cualquier mesa (incluso libre)*: descartada explícitamente durante
  `/speckit-clarify` — sobre-carga innecesaria, ya que una mesa libre no renderiza ningún control
  que necesite esos datos.
- *Disparar solo al pulsar el selector de método de pago dentro del panel de cobro*: descartada —
  introduciría una espera visible justo en el momento más sensible (el cajero ya está listo para
  cobrar); cargar en cuanto el pedido se selecciona da tiempo a que la petición resuelva antes de
  que el cajero llegue a necesitarla.

## 3. Modelo de selección para un pedido sin mesa

**Decision**: Introducir un signal derivado `hasActiveSelection = computed(() =>
!!selectedTableId() || !!selectedOrderId())` y usarlo (en vez de `hasActiveOrder`, hoy
`!!selectedTableId()`) como condición de la que dependen `centralState()`/`effectiveCentralView()`
y el gate de placeholder en `pos-order-panel.component.ts`. Un nuevo método público
`selectStandaloneOrder(orderId: string)` (o una sobrecarga de `selectOrder()`) limpia
`selectedTableId` a `null`, fija `selectedOrderId` y reutiliza el resto de la lógica de
`resetTransient()` ya existente.

**Rationale**: hoy `centralState()` (`pos-terminal.store.ts:511-520`) y `hasActiveOrder`
(línea 561) están gateados exclusivamente en `selectedTableId()` — un pedido sin mesa seleccionado
solo con `selectOrder()` (que ya existe, línea 1052, pero se usa hoy únicamente para alternar entre
pedidos de una misma mesa ya seleccionada) deja `hasActiveOrder()` en `false`, así que
`pos-order-panel.component.ts` seguiría mostrando su placeholder de "nada seleccionado" en vez del
detalle del pedido — exactamente el bug que el spec (Historia 3) pide cerrar. Introducir un signal
derivado que combina ambas fuentes de selección es el cambio de menor superficie: no reemplaza
`selectedTableId` (that still means "hay una mesa de por medio", usado en otros puntos como
`orderTabs()`/`prefetchPaidOrderSales(tableId)` que no aplican a un pedido sin mesa), solo amplía
qué cuenta como "hay algo seleccionado".

**Alternatives considered**:
- *Modelar un `selectedTableId` sintético (id ficticio) para pedidos sin mesa*: descartada —
  contaminaría cualquier lógica que asuma que `selectedTableId` es un id real de
  `tables()` (p. ej. `selectedTable()`, `prefetchPaidOrderSales`), multiplicando casos borde en vez
  de reducirlos.
- *Reescribir `centralState()`/`hasActiveOrder` para que dependan solo de `selectedOrderId()`*:
  descartada — `selectTable()` selecciona una mesa **antes** de saber si tiene pedidos (mesa
  libre), así que `selectedTableId` sigue siendo necesario como señal independiente para ese caso
  (Historia 1, mesa libre).

## 4. Origen de los datos de las tarjetas Domicilio/Para llevar (sin backend nuevo)

**Decision**: Filtrar `store.orders()` (ya poblado por `reloadOrders()` →
`DiningSessionService.listOrders(undefined, true)`, que trae todos los pedidos de sesiones activas
sin filtrar por tipo) por `order.order_type === 'DELIVERY' | 'TAKEAWAY'` y por "pendiente de
cobro" (`!order.paid && order.status !== 'cancelada'`), en un computed nuevo del store
(`ordersByType('domicilios' | 'para-llevar')`).

**Rationale**: `DiningOrder.order_type`, `customer_name`, `delivery_address`, `delivery_phone` y
`delivery_fee` ya existen en la respuesta de la API desde spec 055/056 — no hace falta ningún
endpoint ni parámetro de filtro nuevo en el backend. El filtro "pendiente de cobro" reproduce la
regla ya definida en spec 036 para estos tipos de orden ("desaparece automáticamente del listado en
cuanto se cobra/factura, sin paso de cierre") — a diferencia de una mesa (que usa `deriveTableStatus`,
con reglas de kitchen-ready más elaboradas porque una mesa puede tener un pedido pagado que sigue
"ocupando" la mesa hasta que se libera, spec 047/048), un pedido de Domicilio/Para llevar no tiene
ningún paso de "liberar" — su único criterio de visibilidad es si ya se cobró o no.

**Alternatives considered**:
- *Pedir `GET /orders?order_type=DELIVERY`*: descartada — cambio de backend fuera de alcance
  (spec, Out of Scope); el filtro por tipo ya es trivial en el cliente sobre datos que ya se
  cargan.
- *Reutilizar `deriveTableStatus`/`tableOrders()` tal cual para pedidos sin mesa*: descartada —
  esa función asume una lista de pedidos de la MISMA mesa con reglas de agregación (varias
  rondas); un pedido de Domicilio/Para llevar es siempre un pedido individual, así que se adapta
  pasándole un arreglo de un solo elemento (§5), no reutilizando la función de "pertenencia a
  mesa".

## 5. Insignia de estado y referencia visual de la tarjeta de pedido

**Decision**: Reutilizar `deriveTableStatus([order], 'ocupada')` (pasando un arreglo de un solo
pedido) + `STATUS_META` para la insignia de estado de cada tarjeta — mismo resultado visual que ya
usan las tarjetas de mesa para un pedido en ese mismo estado de cocina/pago. Como referencia
identificable de la tarjeta, reutilizar el mismo patrón `elapsedLabel`/"🕐" ya usado por
`tablesView()` (hora relativa de creación), no un número de orden nuevo.

**Rationale**: ya documentado en `spec.md` (Assumptions) — `DiningOrder` no tiene ningún
consecutivo corto antes de facturar; inventar uno sería un dato nuevo no solicitado. Pasar
`'ocupada'` como segundo argumento (`tableStatus`) a `deriveTableStatus` es seguro porque esa rama
solo se usa como fallback cuando `orders.length === 0` (línea 186-190) — un pedido de
Domicilio/Para llevar por definición siempre tiene al menos un pedido (el que se está mostrando),
así que ese fallback nunca se alcanza en este caso.

**Alternatives considered**:
- *Duplicar la lógica de `STATUS_META` con un vocabulario propio ("Preparando", etc.)*: descartada
  — ya evaluada y descartada en `spec.md` (Assumptions): ese texto no existe hoy en el sistema y
  divergiría del vocabulario ya usado por las tarjetas de mesa que esta misma tarjeta debe imitar
  visualmente (FR-005).

## 6. Extracción de la tarjeta a un componente reutilizable

**Decision**: Crear `order-summary-card.component.ts` (standalone, presentacional, sin inyectar
`PosTerminalStore` directamente) con `@Input()`s primitivos (título, insignia+clase, línea
secundaria, total, seleccionado) y un `@Output() select`. `pos-tables-panel.component.ts` lo usa
tanto para tarjetas de mesa (mapeando `tablesView()` a esas props) como para tarjetas de pedido
Domicilio/Para llevar (mapeando el nuevo `ordersByType(...)`).

**Rationale**: hoy no existe ningún componente de tarjeta — el markup vive inline
(`pos-tables-panel.component.ts:74-95`). El spec exige explícitamente "reutilizar el mismo formato
card" (FR-005) para dos fuentes de datos distintas (mesas vs. pedidos sin mesa); un componente
presentacional con props primitivos evita duplicar el markup/CSS dos veces y evita que el
componente de tarjeta necesite conocer la diferencia entre "mesa" y "pedido" — esa diferencia la
resuelve el `computed()` de mapeo en `pos-tables-panel.component.ts`, no el componente de tarjeta.

**Alternatives considered**:
- *Un solo `@Input() item: TableCardView | OrderCardView` con un union type*: descartada —
  obligaría al componente de tarjeta a hacer *type narrowing* interno, acoplándolo a los dos
  modelos de origen en vez de a un contrato visual plano; los props primitivos mantienen el
  componente ciego a de dónde viene el dato (Principio de mínima superficie, alineado con
  Principio V de la Constitución: no mezclar responsabilidades no relacionadas).
