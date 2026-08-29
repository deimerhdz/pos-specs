---

description: "Task list template for feature implementation"
---

# Tasks: Rediseño del panel de pedido de mesa — cliente, pedidos y cuenta

**Input**: Design documents from `/specs/049-rediseno-panel-pedido-mesa/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/ui-store-contract.md](./contracts/ui-store-contract.md), [quickstart.md](./quickstart.md)

**Tests**: Se incluyen tareas de test. La Constitución (Principio X, Verificación Obligatoria) exige
verificar toda corrección de comportamiento, y research.md ya identificó: (a) un test existente
(`pos-order-panel.component.spec.ts`, caso "el descuento mostrado en el total es siempre $0 sin
promociones activas") que debe moverse/adaptarse porque la fila que verifica desaparece de ese
componente (US1); (b) cobertura nueva que falta para `showAllOrders`/`ordersView`/
`selectedTableStatusMeta` y la generalización de `marcarListo`/`voidPersistedCombo`/`avanzarItem`
(US2).

**Organization**: Tres historias de usuario (spec.md: US1 y US2 en P1, US3 en P2). Todas las rutas
de archivo son relativas al repositorio de la aplicación `../pos-heladeria` (el código no vive en
este repositorio de specs). No hay ninguna tarea sobre `pos-backend` — esta spec no cambia backend.

**Orden de implementación elegido (no solo por prioridad)**: US1 y US2 tocan regiones distintas del
mismo archivo `pos-order-panel.component.ts` (US1: bloque de totales, líneas ~143-170; US2:
cabecera y pestañas, líneas ~36-70), así que se implementan en secuencia, no en paralelo, aunque
ambas sean P1. US3 (P2) queda deliberadamente al final: la reescritura de pestañas de US2
(contracts/ui-store-contract.md) ya construye el nuevo bloque **sin** ningún control "+ Nuevo
pedido" — para cuando arranca la Fase 5, esa parte de FR-001 ya está satisfecha por construcción, y
lo único que queda pendiente de US3 es el cleanup del método `newOrderOnTable()`, ya sin ningún
llamador, en el store (research.md, D2).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de tareas incompletas)
- **[Story]**: Historia de usuario a la que pertenece la tarea (US1, US2, US3)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

Todas las rutas usan como raíz `pos-heladeria/` (repositorio hermano de este `pos-specs`), según la
`Project Structure` de [plan.md](./plan.md).

---

## Phase 1: Setup

**Purpose**: Confirmar el estado base del entorno antes de tocar cualquier archivo

- [X] T001 Ejecutar la suite de tests existente en `pos-heladeria` (`ng test`) y registrar el
  estado real como línea base de regresión (Principio X)
- [X] T002 [P] Confirmar que `ng build` compila sin errores en `pos-heladeria`, como referencia
  antes del cambio

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Prerrequisitos bloqueantes compartidos por todas las historias

**Nota**: esta feature no tiene prerrequisitos fundacionales bloqueantes ni cambios de backend — el
único acoplamiento entre historias es el orden de implementación explicado arriba (mismo archivo,
regiones distintas), no un prerrequisito técnico compartido.

**Checkpoint**: Fase 1 completa — la primera historia de usuario puede comenzar.

---

## Phase 3: User Story 1 - La cuenta de la mesa concentra toda la información de cobro (Priority: P1) 🎯 MVP

**Goal**: El panel de pedido deja de mostrar Subtotal/Descuento/Total; el panel de cuenta de la
mesa (`session-bill-panel.component.ts`) muestra esa misma información agregada, sumada a partir
del desglose por comensal que ya recibe hoy.

**Independent Test**: Seleccionar una mesa con un pedido con descuento por promoción activo;
verificar que el panel de pedido ya no muestra ninguna fila Subtotal/Descuento/Total, y que el
panel de cuenta de esa misma mesa sí las muestra (además del desglose por comensal y el "Total" que
ya tenía), con los mismos importes.

### Tests for User Story 1 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T003 [P] [US1] En `pos-heladeria/src/app/modules/tables/components/session-bill-panel.component.spec.ts`, agregar tests que confirmen: (a) con un `bill.split` de dos líneas, una con `discount > 0` y otra sin descuento, aparecen filas "Subtotal" (suma de `subtotal` de ambas líneas) y "Descuento" (suma de `discount`, solo la línea que sí tiene); (b) con `discount = 0` en todas las líneas, la fila "Descuento" no aparece (mismo criterio de ocultar-en-cero que hoy usa `pos-order-panel.component.ts`); mover aquí, adaptado a esta nueva ubicación, el caso "el descuento mostrado en el total es siempre $0 sin promociones activas" hoy en `pos-order-panel.component.spec.ts` (~línea 121, `describe('PosOrderPanelComponent — sin descuento manual')`) (research.md, "Resumen de impacto en tests existentes"; data-model.md, `billSummary`)
- [X] T004 [P] [US1] En `pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.spec.ts`, agregar un test que confirme que, con un pedido seleccionado (con o sin descuento), el panel ya **no** renderiza ninguna fila "Subtotal", "Descuento" ni "Total"; retirar del `describe('PosOrderPanelComponent — sin descuento manual')` (~línea 114-127) cualquier aserción que dependiera de esa fila existiendo ahí, dejando solo la aserción de que no existe ningún control de descuento manual (FR-002)

### Implementation for User Story 1

- [X] T005 [US1] En `pos-heladeria/src/app/modules/tables/components/session-bill-panel.component.ts`, agregar el computed `billSummary` (`{ subtotal, discount }`, sumando `Number(line.subtotal)`/`Number(line.discount)` sobre `currentBill()?.split`, `null` sin cuenta) y renderizar dos filas nuevas, "Subtotal" y "Descuento" (esta última solo si `billSummary()!.discount > 0`), entre el desglose por comensal (líneas 60-89) y la fila "Total" ya existente (líneas 83-88), con el mismo formato `number: '1.2-2'` que ya usa el resto del panel (data-model.md, "Vistas nuevas en `session-bill-panel.component.ts`")
- [X] T006 [US1] En `pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.ts`, eliminar el bloque "Totales" (líneas ~143-170: filas Subtotal/Descuento/Total) reubicando los botones "Guardar pedido" y "Marcar pedido listo" que hoy viven dentro de ese mismo bloque justo debajo de la lista de ítems/"+ Agregar producto" (dentro del `@else` del carrito, ~línea 141), sin cambiar a qué pedido apuntan (`store.hasDraft()`/`store.saveOrder()`/`store.selectedOrder()`/`store.marcarListo()` sin argumento, comportamiento idéntico al actual)

**Checkpoint**: En este punto, la Historia 1 debe funcionar y probarse de forma completa e
independiente (aunque el resto del panel de pedido siga con su cabecera y pestañas actuales).

---

## Phase 4: User Story 2 - Ver el cliente y sus pedidos en el nuevo formato de pestañas (Priority: P1)

**Goal**: Cabecera de una sola fila (mesa + estado + cliente de solo lectura); pestaña "Todos los
pedidos (N)" que muestra todas las tarjetas de pedido a la vez, más una pestaña "Pedido N" por
cada pedido; cada tarjeta con su hora, su estado agregado y sus ítems con cantidad/nombre/
variante/precio, conservando el pill de estado y la acción "✓ Listo" por ítem.

**Independent Test**: Seleccionar una mesa con dos pedidos activos; verificar la cabecera de una
sola fila, las tres pestañas ("Todos los pedidos (2)", "Pedido 1", "Pedido 2"), que "Todos los
pedidos" muestra ambas tarjetas a la vez, que cada pestaña individual muestra solo su tarjeta, y
que marcar un ítem o el pedido completo como listo desde una tarjeta que no es la seleccionada por
defecto afecta al pedido correcto. Verificar además que una mesa con un único pedido no muestra
ningún selector de pestañas.

### Tests for User Story 2 ⚠️

> **NOTE: Escribir estos tests primero y confirmar que fallan antes de implementar**

- [X] T007 [P] [US2] En `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts`, agregar tests que confirmen: (a) `showAllOrders()` inicia en `true` al seleccionar una mesa con >1 pedido activo, y no aplica (la UI no la consulta) con un único pedido; (b) `orderTabs()` rotula "Pedido 1"/"Pedido 2" en el mismo orden que ya devuelve `ordersOfTable`, sin usar `customer_name`; (c) `ordersView()` devuelve una entrada por pedido de la mesa seleccionada, cada una con sus `items` (mismas líneas que hoy produce `cartView()` para ese pedido) y `pending` igual a `hasPendingKitchenWork(order)`; (d) `selectedTableStatusMeta()` devuelve `null` sin mesa seleccionada y el mismo `{label, chipClass}` que ya usa `tablesView()` para esa mesa, incluso si un filtro activo la excluye de `tablesView()`; (e) `marcarListo(orderId)` con un id que no es `selectedOrder()` marca listo ese pedido, no el seleccionado; `marcarListo()` sin argumento preserva el comportamiento actual; (f) `voidPersistedCombo(comboId)` anula el combo correcto aunque su pedido no sea el seleccionado; (g) `avanzarItem(key)` encuentra y avanza un ítem de un pedido no seleccionado (research.md, D4/D5/D6/D7)
- [X] T008 [P] [US2] En `pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.spec.ts`, agregar tests que confirmen: (a) la cabecera muestra el número de mesa, un chip de estado y el nombre del cliente como texto, sin ningún `<input>` editable; (b) con dos pedidos activos aparecen las pestañas "Todos los pedidos (2)", "Pedido 1", "Pedido 2", con "Todos los pedidos" activa por defecto; (c) en "Todos los pedidos" se ven ambas tarjetas a la vez, cada una con su propia hora y su pastilla de estado; (d) elegir "Pedido 1" muestra solo esa tarjeta y oculta la de "Pedido 2"; (e) "+ Agregar producto" no aparece en "Todos los pedidos" pero sí dentro de una pestaña individual; (f) con un único pedido activo no aparece ningún selector de pestañas (FR-010)

### Implementation for User Story 2

- [X] T009 [US2] En `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`, extraer la construcción de líneas persistidas que hoy vive inline en `cartView()` (líneas ~566-617: filtro de anulados, `plainItems`, `comboGroups`, `persistedCombos`) a un método privado `persistedItemsView(order: DiningOrder): CartLine[]`; `cartView()` pasa a ser `persistedItemsView(this.selectedOrder()) + draft`, sin cambiar su resultado observable para el pedido seleccionado (research.md, D4)
- [X] T010 [US2] En el mismo archivo, agregar el signal `showAllOrders = signal(false)` (se pone en `true` dentro de `selectTable()`/`resetTransient()` cuando `ordersOfTable(tableId).length > 1`, y se reinicia junto al resto del estado transitorio de la selección), el computed `ordersView` (mapea `ordersOfTable(selectedTableId())` a `{order, items: persistedItemsView(order), createdAtLabel, pending: hasPendingKitchenWork(order)}`), el computed `tableItemsView` (todas las líneas de `ordersView()` en una sola lista, con su `orderId` de origen) y el computed `selectedTableStatusMeta` (`STATUS_META[deriveTableStatus(tableOrders(id), table.status)]` para `selectedTable()`, `null` si no hay mesa); cambiar la etiqueta de `orderTabs()` (línea ~505) de `o.customer_name || 'Pedido'` a `'Pedido ' + (i + 1)` (depende de T009 para `persistedItemsView`)
- [X] T011 [US2] En el mismo archivo, generalizar `marcarListo` (línea ~1245, agregar parámetro opcional `orderId?: string`, usar `this.orders().find(o => o.id === orderId)` cuando se pase, `this.selectedOrder()` en caso contrario), `voidPersistedCombo` (línea ~1195, buscar el pedido dueño del combo recorriendo `this.orders()` en vez de `this.selectedOrder()`) y `avanzarItem` (línea ~1267, buscar la línea en `tableItemsView` en vez de en `cartView()`) — ninguno cambia su comportamiento cuando se invoca igual que hoy (sin `orderId`, sobre el pedido seleccionado) (research.md, D6; depende de T010 para `tableItemsView`)
- [X] T012 [US2] En `pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.ts`, reemplazar el bloque de cabecera (líneas ~36-46, tal como quedó tras T006) por una sola fila con el número de mesa, el chip de `store.selectedTableStatusMeta()` y `store.customerName() || store.customerPlaceholder()` como texto (nunca un `<input>`), conservando el botón "← Volver" (depende de T010 para `selectedTableStatusMeta`)
- [X] T013 [US2] En el mismo archivo, reemplazar el bloque "Cliente" + pestañas + "+ Nuevo pedido" (líneas ~47-69, tal como quedó tras T006/T012) por: el nombre de cliente ya movido a la cabecera en T012 (se retira el `<label>`/`<input>` "Cliente" de aquí), y una fila de pestañas "Todos los pedidos (N)" (activa cuando `store.showAllOrders()`) seguida de una pestaña por `store.orderTabs()` (click → `store.showAllOrders.set(false); store.selectOrder(tab.id)`) — sin ningún control "+ Nuevo pedido" (depende de T010 para `showAllOrders`/`orderTabs`)
- [X] T014 [US2] En el mismo archivo, hacer que el contenido del panel alterne: `store.showAllOrders()` verdadero → iterar `store.ordersView()`, una tarjeta por pedido (hora, pastilla "Pendiente"/"Listo" según `pending`, sus `items` con el mismo pill/✓Listo/Anular de siempre, y un botón "Marcar pedido listo" al pie que llama `store.marcarListo(card.order.id)` cuando `pending`, sin "+ Agregar producto" en este modo); `store.showAllOrders()` falso → la tarjeta única de `store.selectedOrder()` que ya quedó de T006 (catálogo embebido, "+ Agregar producto", "Guardar pedido", "Marcar pedido listo" sin argumento) (depende de T011 para las acciones generalizadas y de T013 para las pestañas)

**Checkpoint**: En este punto, las Historias 1 y 2 deben funcionar juntas de forma independiente.

---

## Phase 5: User Story 3 - El panel de pedido ya no ofrece iniciar una segunda ronda (Priority: P2)

**Goal**: Ningún control del panel de pedido permite iniciar una ronda de pedido adicional para una
mesa ya ocupada.

**Independent Test**: Seleccionar una mesa ocupada con uno o más pedidos activos y confirmar que en
ningún estado de las pestañas aparece un control "+ Nuevo pedido" ni equivalente.

**Nota**: la reescritura de pestañas de T013 (US2) ya construye ese bloque sin ningún control
"+ Nuevo pedido" — el criterio de aceptación de esta historia ya queda satisfecho por construcción
al llegar aquí. Lo único pendiente es el cleanup del método ya sin uso en el store.

### Tests for User Story 3 ⚠️

- [X] T015 [P] [US3] En `pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.spec.ts`, agregar un test de regresión que confirme, sobre una mesa con múltiples pedidos activos, que no existe ningún botón/enlace "+ Nuevo pedido" en el DOM renderizado (blindaje explícito de FR-001, aunque ya deba pasar tras T013)

### Implementation for User Story 3

- [X] T016 [US3] En `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`, eliminar el método `newOrderOnTable()` (línea ~918) — sin otros llamadores en `pos-heladeria/src` tras T013 (research.md, D2)

**Checkpoint**: Las tres historias deben funcionar juntas de forma independiente.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final

- [X] T017 [P] Ejecutar manualmente los tres escenarios de [quickstart.md](./quickstart.md) contra un entorno local con `pos-heladeria` y `pos-backend` corriendo — **parcial**: sin navegador disponible en este entorno de implementación no se pudo ejecutar la QA manual completa (clics reales sobre las pestañas y las tarjetas). Se verificó en su lugar: `ng build` sin errores, `ng serve` sirviendo `HTTP 200` sin errores de consola en el arranque, `pos-backend` corriendo y alcanzable, y la cobertura automatizada de T004/T007/T008/T015 que ejercita exactamente los mismos tres escenarios (cabecera de solo lectura, pestañas "Todos los pedidos"/"Pedido N", ausencia de "+ Nuevo pedido", resumen migrado a la cuenta). **Queda pendiente que el usuario/QA ejecute el recorrido manual real** antes de dar la spec por verificada en producción (Principio X)
- [X] T018 Ejecutar la suite completa de tests de `pos-heladeria` (`ng test`) y confirmar que no hay regresiones más allá de los tests nuevos/adaptados de T003-T004, T007-T008 y T015, comparando contra la línea base de T001 — **478/489 tests pasan** (53/58 archivos); los 11 que fallan son exactamente los mismos preexistentes ya documentados en specs 046/047/048 (`app.spec.ts`, `auth.service.spec.ts`, `sidebar.component.spec.ts`, regresión de `MoneyInputComponent`); cero regresiones nuevas — 478 = 459 (línea base T001) + 19 tests nuevos (5 en session-bill-panel/pos-order-panel para US1, 6 en pos-terminal.store + 7 en pos-order-panel para US2, 1 en pos-order-panel para US3)
- [X] T019 [P] Ejecutar `ng build` y confirmar que compila sin errores nuevos respecto a la línea base de T002 — confirmado, mismos dos warnings preexistentes (budget de bundle, `qrcode` CommonJS), sin errores nuevos

---

## Fase 7: Correcciones tras QA manual del usuario (post-T017)

**Contexto**: el usuario ejecutó el recorrido manual pendiente de T017 y reportó dos defectos sobre
la misma pantalla, ambos con el mismo pedido de fondo: un pedido de mostrador cobrado por
adelantado (`hold_for_payment`, spec 028) cuya cocina seguía sin terminar.

- [X] T020 **Bugfix**: "Marcar pedido listo" (`marcarListo()`) llamaba a `POST /orders/{id}/ready`, que el backend rechaza con `409` en cuanto `order.status === 'pagada'` (`registro-de-anomalias.md`, A-16) — exactamente el pedido de mostrador ya cobrado por adelantado. El botón quedaba sin efecto mientras que el link "✓ Listo" por ítem (`avanzarItem()`, PATCH por ítem, que no mira el status del pedido) sí funcionaba. Se cambió `marcarListo()` para hacer lo mismo que el link — PATCH `estado_cocina: 'listo'` por cada ítem pendiente del pedido — en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`. Tests actualizados/agregados en `pos-terminal.store.spec.ts` y `pos-order-panel.component.spec.ts` (ya no esperan `/orders/{id}/ready`, esperan PATCH por ítem)
- [X] T021 **Bugfix**: "Cuenta de la mesa" mostraba Subtotal/Total en $0 y sin fila de Descuento para una mesa cuyo único pedido ya estaba pagado — comportamiento correcto del backend (`compute_bill` excluye a propósito los pedidos `'pagada'` para no facturarlos dos veces al cerrar sesión, `pos-backend/app/api/v1/table_sessions/service.py`), pero confuso para el cajero, que sí ve items con precio. Se agregó un resumen aparte "Ya pagado" (Subtotal/Descuento/Total de las ventas reales de los pedidos ya cobrados de la mesa, vía `resolveSaleForOrder`/`findSaleForOrder`, T033 — no un recálculo propio de descuento) en `session-bill-panel.component.ts`, alimentado por un nuevo computed `PosTerminalStore.selectedTablePaidSummary` y precargado desde `selectTable()` (no desde un `effect()` global — el primer intento rompió ~10 tests ajenos que arman el store con pedidos `paid` sin esperar esta llamada). Sin cambios de backend, sin cambios en el cálculo de lo pendiente por cobrar ni en `total()` (el que valida el pago al cerrar sesión)
- [X] T022 Ejecutar la suite completa de tests de `pos-heladeria` (`ng test`) tras T020/T021 y `ng build` — **486/497 tests pasan**, mismos 11 fallos preexistentes, cero regresiones (486 = 479 + 7 tests nuevos de T020/T021); build sin errores nuevos
- [X] T023 **Ajuste visual**: el usuario reportó, con una captura de "Mesa 1" en la vista "Todos los
  pedidos", que el nombre del comensal en la cabecera y la nota/variante de cada ítem (p. ej.
  "Con sabor a vainilla") eran difíciles de leer — ambos usaban `text-gray-500` de bajo contraste, y
  la nota además `text-xs` (más pequeño que el resto del texto de la tarjeta). No es un defecto
  funcional (FR-006/FR-008/FR-013 ya se cumplían), es legibilidad. Se cambió, en
  `pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.ts`: el nombre del
  cliente en la cabecera de `text-sm text-gray-500` a `text-sm font-semibold text-gray-700`; y las
  tres líneas de nota por ítem (bullets, una en la tarjeta de "Todos los pedidos", una en la vista
  de pedido individual) de `text-xs text-gray-500 pl-1` a `text-sm font-medium text-gray-700 pl-1`.
  Sin cambios de comportamiento ni de datos — `pos-order-panel.component.spec.ts` no hace
  aserciones sobre clases CSS; 23/23 tests de este componente siguen pasando tras el cambio

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: Sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: Sin tareas bloqueantes.
- **User Story 1 (Phase 3)**: Puede empezar tras la Fase 1.
- **User Story 2 (Phase 4)**: Puede empezar tras la Fase 1; en la práctica se implementa después de
  la Historia 1 porque ambas tocan regiones distintas del mismo archivo
  `pos-order-panel.component.ts` (ver "Orden de implementación elegido" arriba) — T012/T013
  reemplazan bloques que T006 ya dejó en su forma intermedia.
- **User Story 3 (Phase 5)**: Depende de que T013 (US2) ya haya reescrito el bloque de pestañas sin
  "+ Nuevo pedido" — de lo contrario T016 dejaría un botón en el template llamando a un método que
  ya no existe.
- **Polish (Phase 6)**: Depende de que las tres historias estén completas.

### Within Each User Story

- Los tests (T003-T004, T007-T008, T015) se escriben y deben fallar antes de su implementación
  correspondiente — excepto T015, que ya pasa al llegar a la Fase 5 por construcción (ver Nota de
  esa fase), y aun así se agrega como blindaje de regresión.
- Dentro de la Historia 2: T009 (extracción de `persistedItemsView`) antes de T010 (que la usa);
  T010 antes de T011 (que usa `tableItemsView`); T010 antes de T012/T013 (que usan
  `selectedTableStatusMeta`/`showAllOrders`/`orderTabs`); T011 y T013 antes de T014 (que conecta
  las acciones generalizadas con las pestañas).

### Parallel Opportunities

- T001/T002 (Setup) en paralelo.
- T003/T004 (tests US1) en paralelo entre sí — archivos distintos.
- T007/T008 (tests US2) en paralelo entre sí — archivos distintos.
- T017/T019 (Polish) pueden iniciarse en paralelo; T018 es la validación final tras ambos.

---

## Parallel Example: User Story 1

```bash
# Lanzar juntos los dos tests de la Historia 1 (archivos distintos):
Task: "Agregar tests de billSummary/filas Subtotal-Descuento en session-bill-panel.component.spec.ts"
Task: "Agregar test de ausencia de filas de totales en pos-order-panel.component.spec.ts"
```

## Parallel Example: User Story 2

```bash
# Lanzar juntos los dos tests de la Historia 2 (archivos distintos):
Task: "Agregar tests de showAllOrders/ordersView/orderTabs/selectedTableStatusMeta/acciones generalizadas en pos-terminal.store.spec.ts"
Task: "Agregar tests de cabecera/pestañas/tarjetas en pos-order-panel.component.spec.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (sin tareas).
3. Completar Fase 3: User Story 1 (T003-T006).
4. **DETENERSE Y VALIDAR**: probar la Historia 1 de forma independiente (totales fuera del panel
   de pedido, presentes en la cuenta).
5. Desplegar/demostrar si está listo.

### Incremental Delivery

1. Completar Setup + Foundational → base lista.
2. Agregar Historia 1 → probar de forma independiente → Desplegar/Demo (MVP).
3. Agregar Historia 2 → probar de forma independiente → Desplegar/Demo.
4. Agregar Historia 3 → probar de forma independiente → Desplegar/Demo.
5. Completar Fase 6: Polish (T017-T019).

---

## Notes

- [P] = archivos distintos o bloques independientes sin dependencias entre sí.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- No hay ninguna tarea de backend — esta spec no cambia `pos-backend`.
- Verificar que los tests fallan antes de implementar (salvo T015, ver Nota de la Fase 5).
- Commit tras cada tarea o grupo lógico.
- Evitar: tareas vagas, conflictos de mismo archivo sin orden declarado, dependencias cruzadas entre
  historias que rompan su independencia más allá de lo ya documentado arriba.
