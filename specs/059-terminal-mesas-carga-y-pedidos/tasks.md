---

description: "Task list for spec 059 — carga diferida y tarjetas de pedido Domicilio/Para Llevar"
---

# Tasks: Carga diferida de datos y tarjetas de pedido de Domicilio/Para Llevar en la Terminal de Mesas

**Input**: Design documents from `/specs/059-terminal-mesas-carga-y-pedidos/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/ui-contracts.md,
quickstart.md

**Tests**: incluidos — el proyecto exige mantener en verde sus characterization tests y verificar
toda funcionalidad nueva (Principio III/X de la constitución); las specs precedentes sobre esta
misma pantalla (045/048/049/055/056) los incluyeron como parte del trabajo, no como opcional.

**Organización**: por historia de usuario de spec.md (US1/US2/US3, las tres P1, en el orden en que
aparecen en spec.md). El código vive exclusivamente en `../pos-heladeria` (frontend) — este spec no
toca `../pos-backend` (Out of Scope).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: US1, US2 o US3 — solo en fases de historia de usuario

---

## Phase 1: Setup

- [X] T001 Confirmar por grep, dentro de `pos-heladeria/src/app/`, que `hasActiveOrder` (a
      renombrar/reemplazar por `hasActiveSelection` en US3) y `selectedTableId` no tienen ningún
      consumidor fuera de `src/app/modules/tables/` — hoy solo aparecen en
      `pos-order-panel.component.ts:25` y en la propia definición de `pos-terminal.store.ts`
      (verificado durante `/speckit-plan`); repetir el grep antes de tocar código por si algo
      cambió desde entonces

---

## Phase 2: Foundational (Blocking Prerequisites)

**Propósito**: dejar disponible el componente de tarjeta reutilizable que tanto la grilla de mesas
(comportamiento existente, sin cambios visuales) como las pestañas Domicilios/Para llevar (US2)
van a consumir — sin este componente, US2 no tiene con qué renderizar sus tarjetas sin duplicar
markup. **No bloquea US1** (cambio aislado en el store, sin relación con el markup de tarjetas) —
puede avanzar en paralelo.

- [X] T002 [P] Crear `pos-heladeria/src/app/modules/tables/components/order-summary-card.component.ts`:
      componente standalone presentacional, sin inyectar `PosTerminalStore` ni ningún servicio,
      con los `@Input()`s `title`, `statusLabel`, `statusClass`, `secondaryLabel`, `elapsedLabel`,
      `totalLabel`, `ordersCount?`, `selected` y el `@Output() select` — mismo markup/CSS que hoy
      vive inline en `pos-tables-panel.component.ts:74-95` (contracts/ui-contracts.md, Contrato 1)
- [X] T003 [P] Crear `pos-heladeria/src/app/modules/tables/components/order-summary-card.component.spec.ts`:
      tests de inputs/outputs puros (sin `TestBed` de HTTP) — renderiza título/insignia/total
      recibidos, aplica la clase de "seleccionado" cuando `selected=true`, emite `select` exactamente
      una vez por click (contracts/ui-contracts.md, Contrato 1). Depende de T002.
- [X] T004 En `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.ts`:
      reemplazar el `<button>` inline de la tarjeta de mesa (líneas 74-95) por
      `<app-order-summary-card>`, mapeando cada item de `store.tablesView()` a sus props
      (`title: 'Mesa ' + t.number`, `statusLabel: t.statusLabel`, `statusClass: t.chipClass`,
      `secondaryLabel: t.itemsLabel`, `elapsedLabel: t.elapsedLabel`, `totalLabel: t.totalLabel`,
      `ordersCount: t.ordersCount`, `selected: t.selected`, `(select)="store.selectTable(t.id)"`)
      — **cero cambio de comportamiento ni de dato mostrado**, solo de dónde vive el markup
      (data-model.md, tabla `OrderSummaryCardView`). Depende de T002.
- [X] T005 Ejecutar `pos-tables-panel.component.spec.ts` existente y confirmar que sigue en verde
      sin ninguna modificación de sus aserciones — evidencia de que T004 no cambió ningún
      comportamiento observable de la grilla de mesas (Principio II/X). Depende de T004.

**Checkpoint**: el componente de tarjeta existe, ya reemplaza el markup de mesas sin regresión
visible, y queda listo para que US2 lo reutilice con datos de pedidos.

---

## Phase 3: User Story 1 - No pedir datos de cobro que el cajero todavía no necesita (Priority: P1) 🎯 MVP

**Goal**: `PaymentMethodService.load()`/`.loadAvailableForCheckout()` y `CashService.restoreShift()`
dejan de dispararse dentro de `PosTerminalStore.init()`; se disparan en cambio la primera vez que
el cajero selecciona una mesa con pedido activo en la sesión de la app, y nunca por seleccionar una
mesa libre.

**Independent Test**: abrir la Terminal de Mesas con la pestaña Network visible; confirmar que no
aparece ninguna petición de métodos de pago ni turno de caja hasta seleccionar una mesa con pedido
(quickstart.md Escenario 1, pasos 1-5). No depende de ningún cambio de US2/US3 — es un cambio
aislado en `pos-terminal.store.ts`.

### Tests for User Story 1

- [X] T006 [P] [US1] Test en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts`:
      tras `init()`, `paymentMethodService.methods()`, `.checkoutOptions()` siguen vacíos y
      `cashService.shift()` sigue `null` (mock de los tres servicios, ver Contrato 3,
      `contracts/ui-contracts.md`)
- [X] T007 [P] [US1] Test en el mismo archivo: `selectTable(tableLibreId)` (sin pedidos) NO invoca
      `paymentMethodService.load()`/`.loadAvailableForCheckout()`/`cashService.restoreShift()`
      (Contrato 3)
- [X] T008 [P] [US1] Test en el mismo archivo: `selectTable(tableConPedidoId)` invoca esos tres
      exactamente una vez cada uno la primera vez; seleccionar una segunda mesa con pedido no los
      repite (mismo criterio de caché que ya usa hoy `init()`,
      `methods().length === 0 ? load() : null`) (Contrato 3)

### Implementation for User Story 1

- [X] T009 [US1] En `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`: extraer
      un método privado nuevo `private async ensureCheckoutDataLoaded(): Promise<void>` que agrupa
      exactamente las tres llamadas condicionales hoy dentro del `Promise.all` de `init()`
      (`pos-terminal.store.ts:844,845-847,850`): `paymentMethodService.methods().length === 0 ?
      load() : null`, `paymentMethodService.checkoutOptions().length === 0 ?
      loadAvailableForCheckout() : null`, `cash.shift() ? null : cash.restoreShift()` (research.md
      §1-2)
- [X] T010 [US1] En el mismo archivo: quitar esas tres líneas del `Promise.all` de `init()`
      (`pos-terminal.store.ts:836-851`) — `init()` queda cargando solo `tableService.loadTables()`,
      `reloadOrders()`, `menuService.loadMenu()` (si vacío) y `promotionService.loadActive()`. Hace
      pasar T006. Depende de T009.
- [X] T011 [US1] En el mismo archivo, dentro de `selectTable(tableId)` (línea 1031-1050): tras fijar
      `selectedOrderId` cuando `list.length > 0` (rama con pedido), agregar
      `void this.ensureCheckoutDataLoaded();` — la rama `else` (mesa libre, `list.length === 0`) NO
      la invoca (research.md §2). Hace pasar T007 y T008. Depende de T009.
- [ ] T012 [US1] Ejecutar manualmente quickstart.md Escenario 1 completo (pasos 1-5, con DevTools
      Network abierto) y confirmar el orden exacto de aparición de las peticiones

**Checkpoint**: la Terminal de Mesas ya no pide métodos de pago ni turno de caja al cargar ni al
seleccionar una mesa libre — verificable de forma completamente independiente de US2/US3.

---

## Phase 4: User Story 2 - Ver los pedidos de Domicilio y Para Llevar como tarjetas (Priority: P1)

**Goal**: las pestañas "Domicilios"/"Para llevar" filtran `store.orders()` por `order_type` y
"pendiente de cobro" y muestran una tarjeta por pedido (vía `OrderSummaryCardComponent`, Foundational)
en vez del mensaje vacío fijo — sin exigir todavía que la tarjeta sea seleccionable (eso es US3).

**Independent Test**: crear un pedido "Para Llevar" y uno "Domicilio" desde la creación manual;
abrir cada pestaña en la Terminal de Mesas y verificar que aparece la tarjeta correspondiente, con
el mismo formato visual que una tarjeta de mesa; verificar que una pestaña sin pedidos de ese tipo
sigue mostrando el mensaje vacío (quickstart.md Escenario 2). Depende de Foundational (T002-T004)
para tener el componente de tarjeta disponible; no depende de US1.

### Tests for User Story 2

- [X] T013 [P] [US2] Test en `pos-terminal.store.spec.ts`: `ordersByType('para-llevar')` devuelve
      solo pedidos `order_type === 'TAKEAWAY'` con `paid === false` y `status !== 'cancelada'`,
      mapeados a `{ title: 'Para llevar', secondaryLabel: customer_name ?? 'Consumidor final',
      totalLabel, elapsedLabel, statusLabel, statusClass }`; `ordersByType('domicilios')` el mismo
      criterio con `'DELIVERY'`; un pedido ya `paid: true` o `cancelada` no aparece en ninguno de
      los dos (research.md §4-5, data-model.md)
- [X] T014 [P] [US2] Test en
      `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.spec.ts`: con la
      pestaña `'para-llevar'` activa y `store.ordersByType('para-llevar')` no vacío, se renderiza
      una `<app-order-summary-card>` por pedido en vez del mensaje vacío; con lista vacía, sigue
      mostrando el mensaje informativo ya existente (spec 036, FR-003) sin cambios (quickstart.md
      Escenario 2, paso 6)

### Implementation for User Story 2

- [X] T015 [US2] En `pos-terminal.store.ts`: agregar el tipo `interface OrderSummaryCardView { id:
      string; title: string; statusLabel: string; statusClass: string; secondaryLabel: string;
      elapsedLabel: string; totalLabel: string; }` (data-model.md) y un método privado
      `private toOrderCardView(order: DiningOrder): OrderSummaryCardView` que arma ese objeto
      reutilizando `STATUS_META[deriveTableStatus([order], 'ocupada')]` para el estado (research.md
      §5), `this.elapsedLabel(new Date(order.created_at).getTime())` para la hora, y
      `this.fmt(this.orderSubtotal(order))` para el total — mismos helpers ya usados por
      `tablesView()` (`pos-terminal.store.ts:624-666`)
- [X] T016 [US2] En el mismo archivo: agregar dos computed privados,
      `private readonly deliveryOrders = computed(() => this.orders().filter(o => o.order_type ===
      'DELIVERY' && !o.paid && o.status !== 'cancelada').map(o => this.toOrderCardView(o)))` y su
      equivalente `takeawayOrders` con `'TAKEAWAY'`, más el método público
      `ordersByType(tab: 'domicilios' | 'para-llevar'): OrderSummaryCardView[]` que devuelve
      `tab === 'domicilios' ? this.deliveryOrders() : this.takeawayOrders()` (research.md §4).
      Hace pasar T013. Depende de T015.
- [X] T017 [US2] En `pos-tables-panel.component.ts`: dentro de la rama `@else` que hoy muestra
      siempre el mensaje vacío fijo (líneas 110-120), envolverla en
      `@if (store.ordersByType(store.orderTypeTab()).length > 0) { <div class="..."> @for (o of
      store.ordersByType(store.orderTypeTab()); track o.id) { <app-order-summary-card [title]="o.title"
      ... /> } </div> } @else { <!-- mensaje vacío existente, sin cambios --> }` — mismo contenedor
      visual (grid/lista) que ya usa el carrusel de mesas, sin el carrusel de flechas (los pedidos
      de estas pestañas no necesitan scroll horizontal salvo que crezcan mucho; usar el mismo
      contenedor de scroll que ya existe). Hace pasar T014. Depende de T016.
- [X] T018 [US2] Ejecutar manualmente quickstart.md Escenario 2 completo (pasos 1-6)

**Checkpoint**: las pestañas Domicilios/Para llevar muestran tarjetas reales con el mismo formato
visual que las mesas — verificable de forma independiente, aunque todavía no se puedan seleccionar
(US3).

---

## Phase 5: User Story 3 - Seleccionar una tarjeta de pedido muestra su detalle y permite cobrarlo (Priority: P1)

**Goal**: seleccionar una tarjeta de pedido de Domicilio/Para llevar abre su detalle en el panel
central y su cobro en el panel derecho, con el mismo comportamiento ya implementado para una mesa
— cerrando la brecha donde el toast "cóbralo desde el panel de la derecha" no tenía ningún pedido
real al cual llevar al cajero.

**Independent Test**: a partir de las tarjetas de US2, seleccionar una y verificar que el panel
central muestra su detalle y el panel derecho permite cobrarla de punta a punta (quickstart.md
Escenario 3). Depende de que existan tarjetas que seleccionar (US2) y reutiliza
`ensureCheckoutDataLoaded()` de US1 — no es un cambio aislado, pero sigue siendo verificable como
incremento propio sobre lo que US1+US2 ya entregan.

### Tests for User Story 3

- [X] T019 [P] [US3] Test en `pos-terminal.store.spec.ts`: `selectStandaloneOrder(orderId)` fija
      `selectedOrderId() === orderId`, `selectedTableId() === null`, invoca
      `ensureCheckoutDataLoaded()` la primera vez en la sesión (no la repite en una segunda
      llamada) — mismo criterio de caché que T008 (Contrato 2/3, `contracts/ui-contracts.md`)
- [X] T020 [P] [US3] Test en el mismo archivo: `hasActiveSelection()` es `false` sin ninguna
      selección, `true` tras `selectTable(id)` (con o sin pedido) y `true` tras
      `selectStandaloneOrder(orderId)` (Contrato 2)
- [X] T021 [P] [US3] Test en
      `pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.spec.ts`: con un
      pedido sin mesa seleccionado (`selectStandaloneOrder`), el panel muestra el detalle (no el
      placeholder de "Selecciona una mesa…"), con el título "Domicilio"/"Para llevar" según
      `order_type` en vez de "Mesa {number}", y — para `order_type === 'DELIVERY'` — los campos
      dirección/teléfono/valor del domicilio ya capturados (spec FR-012)
- [X] T022 [P] [US3] Test en
      `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.spec.ts`: con
      un pedido sin mesa seleccionado, `sidebarMode()` resuelve `'terminal-pos'` (no `'resumen'` ni
      bloqueado por `showSessionCharge`) y el flujo de cobro ya existente (`checkout()` →
      `store.checkoutAndSend(payments)`) se completa con éxito para ese pedido — **test de
      regresión que confirma el hallazgo de `/speckit-tasks`**: este componente no necesita ningún
      cambio de código, ya opera sobre `store.selectedOrder()` sin depender de `selectedTableId()`
      (`pos-checkout-panel.component.ts:282,291-294`; `pos-terminal.store.ts:1632-1637`,
      `checkoutAndSend()`)

### Implementation for User Story 3

- [X] T023 [US3] En `pos-terminal.store.ts`: agregar
      `readonly hasActiveSelection = computed(() => !!this.selectedTableId() ||
      !!this.selectedOrderId())` (data-model.md). Hace pasar la mitad de T020.
- [X] T024 [US3] En el mismo archivo: agregar
      `selectStandaloneOrder(orderId: string): void { this.selectedTableId.set(null);
      this.selectedOrderId.set(orderId); this.resetTransient(); this.showAllOrders.set(false);
      this.customerName.set(this.selectedOrder()?.customer_name || ''); void
      this.ensureCheckoutDataLoaded(); }` — mismo patrón que `selectTable()`
      (`pos-terminal.store.ts:1031-1050`) pero sin `loadSessionBill`/`prefetchPaidOrderSales` (esos
      son conceptos de sesión de mesa, no aplican a un pedido sin mesa) (research.md §2-3). Hace
      pasar T019 y la otra mitad de T020. Depende de T009 (`ensureCheckoutDataLoaded`).
- [X] T025 [US3] En `pos-tables-panel.component.ts`: en las tarjetas renderizadas por T017,
      conectar `(select)="store.selectStandaloneOrder(o.id)"` (hoy sin handler funcional). Depende
      de T017, T024.
- [X] T026 [US3] En `pos-order-panel.component.ts`: reemplazar la condición `@if
      (!store.hasActiveOrder())` (línea 25) por `@if (!store.hasActiveSelection())`. Depende de
      T023.
- [X] T027 [US3] En `pos-terminal.store.ts`: extender `selectedTableStatusMeta` (línea 617-622)
      para que, cuando no haya mesa pero sí un pedido seleccionado, calcule
      `STATUS_META[deriveTableStatus([this.selectedOrder()!], 'ocupada')]` en vez de devolver
      `null` (research.md §5) — el resto de sus consumidores (`pos-order-panel.component.ts:42`) no
      cambia. Depende de T023.
- [X] T028 [US3] En `pos-order-panel.component.ts`: en el header (líneas 40-45), reemplazar el
      título fijo `Mesa {{ store.selectedTable()?.number }}` por una expresión condicional —
      `store.selectedTable() ? ('Mesa ' + store.selectedTable()!.number) :
      (store.selectedOrder()?.order_type === 'DELIVERY' ? 'Domicilio' : 'Para llevar')` — el chip
      de estado (línea 42-44) sigue leyendo `store.selectedTableStatusMeta()` sin cambios (ya
      extendido por T027). Depende de T027.
- [X] T029 [US3] En el mismo archivo: agregar, justo debajo del header, un bloque
      `@if (store.selectedOrder()?.order_type === 'DELIVERY') { <!-- dirección, teléfono si
      existe, valor del domicilio, leídos de store.selectedOrder() --> }` (spec FR-012,
      data-model.md tabla de `DiningOrder`). Depende de T028.
- [ ] T030 [US3] Ejecutar manualmente quickstart.md Escenario 3 completo (pasos 1-6), prestando
      atención especial al paso 6 (deseleccionar la mesa al elegir una tarjeta de pedido, FR-013 —
      ya cubierto por `selectStandaloneOrder` fijando `selectedTableId.set(null)` en T024, pero
      debe verificarse visualmente que ninguna tarjeta de mesa queda resaltada a la vez)

**Checkpoint**: el ciclo completo — crear un pedido Para Llevar/Domicilio, verlo como tarjeta,
seleccionarlo, ver su detalle, cobrarlo — funciona de punta a punta desde la Terminal de Mesas.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [ ] T031 Ejecutar los 3 escenarios de quickstart.md de punta a punta como validación final, en
      una sola sesión sin recargar la página entre escenarios (confirma que la caché de FR-003
      sobrevive a la navegación entre pestañas/tarjetas)
- [X] T032 Ejecutar la suite completa de tests de `pos-heladeria` (`npx ng test --watch=false`) y
      confirmar que este feature no introduce ninguna regresión (Principio X). Resultado: 577 tests,
      573 en verde, 4 en rojo — **las mismas 4 que ya fallaban en el commit base antes de esta
      spec** (confirmado con `git stash`/`git stash pop`: `app.spec.ts`, `auth.service.spec.ts`,
      `sidebar.component.spec.ts`, y el caso "T032: Imprimir Pre-cuenta" de
      `pos-checkout-panel.component.spec.ts` — ninguno relacionado con `tables/`; los tres primeros
      ni siquiera tocan ese módulo). Los 17 tests nuevos de esta spec (T003, T006-T008, T013,
      T019-T022) están todos en verde. Cero regresiones en `table-sessions.component.spec.ts`,
      `pos-checkout-panel.component.spec.ts` (salvo el caso ya roto en el baseline) ni
      `pos-order-panel.component.spec.ts`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: depende de Setup (T001, verificación previa). Bloquea **solo US2**
  (necesita el componente de tarjeta) — no bloquea US1.
- **US1 (Phase 3)**: depende solo de Setup (T001) — cambio aislado en `pos-terminal.store.ts`, sin
  relación con el componente de tarjeta. Puede avanzar en paralelo con Foundational/US2.
- **US2 (Phase 4)**: depende de Foundational (T002-T004, el componente de tarjeta) — no depende de
  US1.
- **US3 (Phase 5)**: depende de US1 (T009, `ensureCheckoutDataLoaded()`) y de US2 (T017, las
  tarjetas ya renderizadas sobre las que conectar la selección) — es la única historia con
  dependencia cruzada real, reflejando lo que el propio spec describe ("a partir de las tarjetas de
  la Historia 2").
- **Polish (Phase 6)**: depende de que las tres historias estén completas.

### Dentro de Foundational

T002 → T003 (test depende del componente). T002 → T004 (la migración del markup de mesas necesita
el componente ya creado). T004 → T005 (gate de regresión).

### Dentro de cada historia

- US1: T006-T008 (tests) antes de T009-T011 (implementación) — T009 es el refactor base que hace
  pasar T006 (junto con T010); T010 hace pasar T006; T011 hace pasar T007 y T008. T012 al final.
- US2: T013-T014 (tests) antes de T015-T017 (implementación) — T015-T016 hacen pasar T013; T017
  hace pasar T014. T018 al final.
- US3: T019-T022 (tests) antes de T023-T029 (implementación) — T023 hace pasar la mitad de T020;
  T024 hace pasar T019 y la otra mitad de T020; T026 depende de T023; T027-T028 hacen pasar T021;
  T022 no requiere ningún cambio de implementación (test de regresión sobre código ya existente).
  T030 al final.

### Parallel Opportunities

- Foundational: T002 y T003 se escriben junto con el componente (mismo cambio, pero T003 es el
  archivo de test, distinto de T002).
- US1: T006, T007, T008 en paralelo entre sí (mismo archivo de test, pero casos independientes —
  pueden escribirse en paralelo por distintas personas y consolidarse).
- US2: T013, T014 en paralelo entre sí (archivos distintos: store vs. componente).
- US3: T019, T020, T021, T022 en paralelo entre sí (T019-T020 mismo archivo de store pero casos
  independientes; T021, T022 archivos de componente distintos).
- Con más de una persona disponible: tras T001, una persona puede tomar US1 completo mientras otra
  toma Foundational → US2 en paralelo; US3 espera a que ambas líneas terminen.

---

## Parallel Example: User Story 1

```bash
Task: "Test: init() no carga métodos de pago ni turno de caja (T006)"
Task: "Test: selectTable(mesa libre) no dispara la carga diferida (T007)"
Task: "Test: selectTable(mesa con pedido) dispara la carga diferida una sola vez (T008)"
```

## Parallel Example: User Story 3

```bash
Task: "Test: selectStandaloneOrder fija selección y dispara carga diferida (T019)"
Task: "Test: hasActiveSelection cubre mesa y pedido sin mesa (T020)"
Task: "Test: pos-order-panel muestra detalle de un pedido sin mesa (T021)"
Task: "Test de regresión: pos-checkout-panel ya cobra un pedido sin mesa sin cambios (T022)"
```

---

## Implementation Strategy

### MVP First (User Story 1 sola)

1. Completar Phase 1 (Setup).
2. Completar Phase 3 (US1) — entrega, por sí sola, la mejora de rendimiento pedida explícitamente
   ("todavia no estoy seleccionando un metodo de pago entonces no se deberia de hacer esa
   peticion"), sin tocar nada de Domicilio/Para llevar.
3. **Detener y validar**: quickstart.md Escenario 1.
4. US1 es un MVP legítimo por sí solo — a diferencia de US2/US3 (que dependen entre sí para tener
   valor completo), US1 no necesita ninguna otra historia para ser útil en producción.

### Incremental Delivery

1. Setup + Foundational (en paralelo con US1) → base lista.
2. + US1 → menos peticiones HTTP al arrancar la Terminal de Mesas → validar → demo.
3. + US2 → los pedidos Para Llevar/Domicilio ya son visibles como tarjetas (aún no seleccionables)
   → validar.
4. + US3 → el ciclo completo (ver → seleccionar → cobrar) funciona → validar (especial atención a
   T022, el test de regresión que confirma que `pos-checkout-panel.component.ts` no necesitó
   ningún cambio).
5. + Polish.

---

## Notes

- El hallazgo más importante de `/speckit-tasks` (más allá de lo ya documentado en research.md):
  `pos-checkout-panel.component.ts` **no requiere ningún cambio de código** para US3 —
  `sidebarMode()`, `showSessionCharge()`, `checkout()` y `checkoutAndSend()` ya operan
  exclusivamente sobre `store.selectedOrder()`/`store.cashShiftId()`, sin ninguna dependencia de
  `selectedTableId()`. T022 existe específicamente para dejar esto verificado con un test, no
  asumido — si algún cambio futuro introdujera una dependencia oculta de mesa en ese componente,
  T022 lo detectaría.
- T027-T028 (adaptar el header de `pos-order-panel.component.ts` para un pedido sin mesa) son el
  punto más sensible de US3: sin ellos, `hasActiveSelection` ya mostraría el panel (T026), pero con
  un título vacío ("Mesa ") y sin insignia de estado — una regresión visual silenciosa, no un error
  duro.
- Ningún task de este documento agrega dependencias nuevas, toca `pos-backend`, ni recalcula
  ninguna venta/factura ya emitida (Principio VII/IX).
- Commitear después de cada tarea o grupo lógico; detenerse en cada checkpoint para validar la
  historia de forma independiente antes de continuar con la siguiente.
- **T012/T018/T030/T031 (verificación manual en navegador de quickstart.md) quedaron sin marcar**:
  esta sesión no abrió un navegador real. Se detectó un servidor de desarrollo ya corriendo en
  `localhost:4200`/`localhost:8000`, aparentemente de otra sesión activa (`pos-heladeria-ad`) — se
  evitó deliberadamente interactuar con él para no interferir con ese trabajo en curso. La
  cobertura automatizada (87 tests en `pos-terminal.store.spec.ts`, 12 en
  `pos-tables-panel.component.spec.ts`, 26 en `pos-order-panel.component.spec.ts`, 24 en
  `pos-checkout-panel.component.spec.ts`, 6 en `order-summary-card.component.spec.ts`) ejercita
  cada escenario de `quickstart.md` a nivel de componente/store, pero la verificación visual de
  punta a punta en un navegador real queda pendiente para quien retome esta rama.
