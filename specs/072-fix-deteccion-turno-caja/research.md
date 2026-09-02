# Phase 0 — Research

**Spec**: [spec.md](./spec.md) · **Fecha**: 2026-09-02

No hay ningún `NEEDS CLARIFICATION` pendiente en el Technical Context — todo se resolvió leyendo
el código real de `pos-heladeria` (dos veces: una revisión propia y una investigación
independiente de un subagente, ambas coincidentes en la mecánica). Este documento registra las
decisiones técnicas del fix, no una elección de tecnología (no hay ninguna nueva).

---

## D1 — Dónde enganchar el disparador reactivo, sin repetir el error ya documentado de un `effect()` global

**Decisión**: el chequeo de turno de caja se dispara desde dentro de
`PosTerminalStore.reloadOrders()`
([pos-terminal.store.ts:1074-1081](../../../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts#L1074-L1081))
— el único punto por el que pasan **las tres** fuentes de actualización de pedidos: la carga
inicial (`init()`, línea 936), el sondeo (`startPolling()`) y los eventos en tiempo real
(`scheduleReload()` → `connectRealtime()`, líneas 1026-1067). Se agrega, justo después de
`this.announcePending(orders)`, una llamada a `ensureCheckoutDataLoaded()` (ya existente,
[líneas 963-971](../../../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts#L963-L971))
condicionada a que ahora exista algo real para cobrar (ver D3).

**Rationale**: el propio archivo documenta, en el método vecino `prefetchPaidOrderSales`
([líneas 267-280](../../../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts#L267-L280)),
que un `effect()` de Angular reaccionando sobre `orders()`/`selectedTableId()` **ya rompió "decenas
de specs ajenos a esta pantalla"** la primera vez que se intentó ese patrón en este store, porque
cualquier test que arma un `PosTerminalStore` y llama `store.orders.set([...])` directamente
—sin pasar por `selectTable()`— dispara el efecto sin que el test lo espere ni lo mockee. Se
verificó que ese riesgo es real hoy también: `pos-terminal.store.spec.ts` tiene **35 usos** de
`store.orders.set([...])` fuera de `reloadOrders()`. Enganchar el disparador **dentro del método**
en vez de en un `effect()` reactivo separado hace que esos 35 usos existentes queden intactos —
ninguno pasa por `reloadOrders()`, así que ninguno dispara el nuevo chequeo, cero riesgo de
regresión en esa suite por este cambio.

**Alternativas consideradas**:
- Un `effect()` sobre `pendingOfSelectedTable()`/`centralState()` — la opción "natural" que
  también propuso la investigación inicial, descartada por la razón anterior (mismo error que ya
  se documentó y evitó una vez en este archivo).
- Enganchar en `announcePending()` en vez de en `reloadOrders()` — equivalente en efecto, pero
  `announcePending` es sobre la campana/sonido, no sobre cobro; mezclarlo ahí sería menos legible
  que un paso explícito adicional en `reloadOrders()`.

---

## D2 — Qué condición dispara el chequeo dentro de `reloadOrders()`

**Decisión**: la misma condición que ya usa `selectTable()` para decidir si llama
`ensureCheckoutDataLoaded()` — "hay algo real que cobrar" — extraída a un método privado
compartido en vez de duplicar la expresión. Concretamente: `pendingOrders().length > 0` (algún QR
por confirmar en cualquier mesa, spec 059 lo ya usa como universo del panel "Pagos por confirmar",
[línea 459](../../../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts#L459))
**o** la mesa/pedido actualmente seleccionado ya tiene algo cobrable (`ordersOfTable(tableId)` no
vacío para `selectedTableId()`, o `selectedOrder()` no nulo para un pedido de Domicilio/Para
llevar seleccionado vía `selectStandaloneOrder()`).

**Rationale**: cubre en una sola condición los dos disparadores imperativos que ya existían
(`selectTable()` línea 1153, `selectStandaloneOrder()` línea 1182) **más** el caso reactivo del
defecto reportado (US1: mesa vacía ya seleccionada que recibe su primer pedido). No dispara
mientras solo haya mesas libres sin ningún pedido en ningún lado (preserva la spec 059 FR-001).

**Alternativas consideradas**: disparar siempre que `orders()` cambie, sin condición — descartada
porque violaría la spec 059 FR-001 (pediría datos de cobro con solo mesas libres, la razón de ser
de la "carga diferida").

---

## D3 — Cómo descubrir el turno abierto sin depender de `localStorage`

**Decisión**: `CashService.restoreShift()`
([cash.service.ts:50-63](../../../pos-heladeria/src/app/modules/cash-register/services/cash.service.ts#L50-L63))
se reemplaza por `discoverOpenShift()`, con esta lógica:

1. **Camino rápido** (sin cambio de comportamiento cuando ya funciona hoy): si
   `localStorage['cash.register']` apunta a una caja con turno abierto, usarlo — evita listar
   todas las cajas en el caso ya cubierto.
2. **Descubrimiento** (nuevo, cuando (1) no resuelve nada — clave ausente o esa caja puntual sin
   turno): pedir `listRegisters()` y, para cada una, `getCurrentShift(id)` en paralelo,
   exactamente con el mismo patrón `Promise.all(...map(async r => try/catch))` que ya usa
   `CashSessionStore.loadOverview()`
   ([cash-session.store.ts:281-297](../../../pos-heladeria/src/app/modules/cash-register/services/cash-session.store.ts#L281-L297)) —
   se reutiliza el patrón, no se inventa uno nuevo.
3. **Regla de resolución** (FR-004/User Story 3): si aparece **exactamente un** turno abierto
   entre todas las cajas, se adopta (`this.shift.set(...)` y se guarda su `cash_register_id` en
   `localStorage` para que la próxima vez tome el camino rápido). Si aparecen **cero o más de
   uno**, `this.shift` queda en `null` — cero turnos sigue bloqueando (FR-003); más de uno exige
   seguir usando "Operar" para elegir, sin adivinar (FR-004).

**Rationale**: el backend ya expone todo lo necesario
(`GET /cash/registers`, `GET /cash/shifts/current?cash_register_id=`, sin cambios — confirmado
que `current_shift` en
[router.py:89-94](../../../pos-backend/app/api/v1/cash/router.py#L89-L94) hace un lookup simple
por caja, sin ningún filtro por usuario/sesión que pudiera explicar un falso negativo). No hace
falta ningún endpoint nuevo ni cambio de esquema — el fix es 100% frontend.

**Alternativas consideradas**:
- Backend nuevo `GET /cash/shifts/current` sin `cash_register_id` (turno abierto del tenant,
  cualquiera que sea la caja) — descartada: mezclaría un cambio de contrato de API con un fix de
  frontend (Principio VI, no mezclar clases de cambio), y el frontend ya puede resolverlo sin
  tocar el backend reutilizando dos endpoints existentes.
- No perseverar en localStorage en absoluto y siempre listar todas las cajas — descartada: pierde
  la optimización de la spec 059 (una sola llamada en el caso común, en vez de N+1) sin necesidad,
  ya que el camino rápido sigue siendo válido cuando `localStorage` está actualizado.

---

## D4 — Por qué el retraso acotado (~2s, Clarifications) no exige rediseñar la caché de spec 059

**Decisión**: el guard existente de `ensureCheckoutDataLoaded()`
(`this.cash.shift() ? null : this.cash.discoverOpenShift()`) se conserva tal cual. Como el nuevo
disparador de D1 se re-evalúa en **cada** `reloadOrders()` (cada sondeo, cada evento en tiempo
real) mientras haya algo pendiente de cobrar, un primer intento que resuelva `null` (turno
realmente cerrado, o temporalmente sin respuesta) **se reintenta solo** en el siguiente ciclo,
sin necesitar ninguna invalidación especial — apenas el turno se abra o dos cajas dejen de estar
ambiguas, el siguiente `reloadOrders()` lo recoge. Una vez que `cash.shift()` resuelve un valor no
nulo, el guard evita pedirlo de nuevo (FR-003 de la spec 059, sin cambio).

**Rationale**: satisface FR-005 (no quedarse pegado en un resultado que no reflejaba la realidad)
sin ningún mecanismo nuevo de caducidad de caché — el propio ciclo de sondeo/tiempo real ya
reintenta lo suficiente. El costo (una llamada a `listRegisters` + N `getCurrentShift` en paralelo
por ciclo, **solo** mientras algo esté pendiente de cobrar y el turno siga sin resolverse) es
despreciable para el tamaño típico de un tenant (una a tres cajas) y acotado en el tiempo (el
turno se abre o dos cajas se resuelven en minutos, no indefinidamente).

**Alternativas consideradas**: invalidar `cash.shift()` con un TTL explícito — descartada por
sobre-ingeniería; el reintento ya ocurre gratis por el disparador de D1.

---

## D5 — Decisión de negocio y characterization tests

**Decisión**: no aplica ninguna entrada nueva en `registro-de-anomalias.md` (ya lo declara
spec.md, "Autorización de negocio") — es una corrección, no una regla de negocio nueva. Se
verificó que ningún test backend ni frontend relacionado (`test_cash_timezone.py`,
`cash.service.spec.ts`, `cash-session.store.spec.ts`,
`pos-terminal.store.spec.ts` describe "carga diferida... spec 059, Historia 1") lleva el prefijo
`"CONGELA comportamiento actual:"`, y que ninguno de los 35 usos de `store.orders.set(...)` fuera
de `reloadOrders()` se ve afectado (D1). El único test que ejercita `restoreShift()` indirectamente
— ninguno, no existe ese test hoy — así que renombrarlo/reescribirlo como `discoverOpenShift()` no
rompe ninguna prueba existente; solo hay que actualizar los dos call sites reales
(`pos-terminal.store.ts:969`, `cash-session.store.ts:251`) y los dos *stubs* de mock que hoy
exponen `restoreShift` (`pos-terminal.store.spec.ts:52`, `pos-tables-panel.component.spec.ts:65`).
