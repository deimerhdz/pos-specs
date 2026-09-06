---

description: "Task list for spec 076 — rediseño responsive de la Terminal de Mesas"
---

# Tasks: Rediseño responsive de la Terminal de Mesas (escritorio/tablet/móvil)

**Input**: Design documents from `/specs/076-terminal-mesas-rediseno-responsive/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/order-response.md,
quickstart.md

**Tests**: incluidos — el proyecto exige mantener en verde sus characterization tests y verificar
toda funcionalidad nueva (Principio III/X de la constitución); el plan (Constitution Check) los deja
pendientes explícitamente para esta fase, no como opcionales.

**Organización**: por historia de usuario de spec.md (US1-US5, en su orden de prioridad P1→P3). El
código vive en dos repositorios: `../pos-heladeria` (frontend, todas las historias) y
`../pos-backend` (backend, solo US4 — exposición de un campo ya existente, sin migraciones).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: US1 a US5 — solo en fases de historia de usuario

---

## Phase 1: Setup

- [X] T001 [P] Registrar la línea base de tests antes de tocar código: correr
      `npx ng test --watch=false` en `pos-heladeria` (módulo `tables/`) y
      `python -m app.scripts.test_table_release` en `pos-backend`, y anotar el resultado (número de
      tests en verde/rojo) para poder distinguir después cualquier regresión introducida por esta
      spec de fallos preexistentes (Principio III/X). **Resultado**: frontend 687 en verde / 7 en
      rojo (694 totales) — los 7 rojos ya fallaban antes de tocar código: 4 en
      `tenant.service.spec.ts` (conflicto de `TestBed` no relacionado), 1 en
      `pos-checkout-panel.component.spec.ts` ("T032: Imprimir Pre-cuenta"), 1 en
      `transfer-details-step.component.spec.ts`, y 1 adicional no relacionado con `tables/`. Backend:
      **no se pudo ejecutar** — el entorno sandbox de esta sesión no tiene el schema del tenant de
      prueba provisionado (`relation "heladeria.dining_tables" does not exist`); es una limitación
      del entorno local, no del código. Se documenta como riesgo conocido para T024/T039 (backend).

---

## Phase 2: Foundational (Blocking Prerequisites)

**Nota**: a diferencia de spec 059 (que necesitaba un componente de tarjeta reutilizable nuevo),
esta spec reutiliza `order-summary-card.component.ts` ya existente — no hay ninguna pieza de
infraestructura que bloquee a la vez a **todas** las historias. Las dependencias reales son
punto-a-punto entre historias específicas (mismo archivo compartido) y se documentan en
"Dependencies & Execution Order" al final. No hay tareas en esta fase.

---

## Phase 3: User Story 1 - Ver y elegir cualquier mesa sin importar el dispositivo (Priority: P1) 🎯 MVP

**Goal**: reemplazar el carrusel horizontal de una sola fila (con flechas) por una cuadrícula CSS
que envuelve en varias filas — 4 columnas en escritorio (≥1024px), 3 en tablet (768-1023px), 2 en
móvil (<768px) — reutilizando los breakpoints `lg`/`md` ya usados en `table-sessions.component.ts`.

**Independent Test**: abrir la Terminal de Mesas en los 3 anchos de referencia y verificar que se
ven varias filas de tarjetas con scroll vertical, sin flechas de desplazamiento horizontal, con la
cantidad de columnas que corresponde a cada ancho (quickstart.md, Historia 1). No depende de
ninguna otra historia — es un cambio aislado de contenedor visual.

### Tests for User Story 1

- [X] T002 [P] [US1] Test en
      `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.spec.ts`: el
      contenedor de tarjetas de mesa ya NO renderiza los botones "‹"/"›" de desplazamiento (buscar
      por `aria-label="Ver mesas anteriores"`/`"Ver más mesas"` — deben estar ausentes) y expone las
      clases de grilla responsive (`grid`, `grid-cols-2`, `md:grid-cols-3`, `lg:grid-cols-4`, o los
      tokens equivalentes elegidos en T005) en el contenedor que envuelve el `@for` de
      `store.tablesView()` (FR-001 a FR-004)
- [X] T003 [P] [US1] Test en el mismo archivo: la rama que renderiza `store.ordersByType(...)`
      (pestañas Domicilios/Para llevar) usa el mismo contenedor de grilla responsive que T002, en
      vez de su `flex flex-wrap` actual (FR-005)
- [X] T004 [P] [US1] Test en el mismo archivo: tras el cambio de contenedor, cada
      `<app-order-summary-card>` sigue recibiendo exactamente los mismos inputs que hoy
      (`title`/`statusLabel`/`statusClass`/`secondaryLabel`/`elapsedLabel`/`totalLabel`/
      `ordersCount`/`selected`) — ninguno se pierde ni cambia de valor (FR-006, no regresión)

### Implementation for User Story 1

- [X] T005 [US1] En `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.ts`:
      reemplazar el bloque del carrusel (líneas 65-102: botones ‹/›, `div #carousel` con
      `overflow-x-auto`) por un contenedor `<div class="grid grid-cols-2 md:grid-cols-3
      lg:grid-cols-4 gap-3 ...">` que envuelve el mismo `@for (t of store.tablesView(); track
      t.id)` con scroll vertical (`overflow-y-auto` en el ancestro que ya lo tenga, o en este mismo
      contenedor si hace falta) en vez de horizontal. Hace pasar T002. Depende de T001. **Hecho**:
      se usó `max-h-[45vh] overflow-y-auto` (altura acotada para no empujar el panel central que
      sigue debajo, ver research.md). Fix adicional necesario y no anticipado en el task original:
      `order-summary-card.component.ts` fijaba el ancho de cada tarjeta a `w-[calc((100%-2.25rem)/4)]`
      (asumiendo 4 visibles del carrusel) — se cambió a `w-full` porque ahora es la grilla CSS del
      padre la que controla el ancho de cada celda; con el cálculo viejo, 2/3 columnas en
      tablet/móvil se habrían roto (tarjetas del ancho de 1/4 dentro de una celda más ancha).
- [X] T006 [US1] En el mismo archivo: aplicar el mismo contenedor de grilla (mismas clases que T005)
      a la rama `@else if (store.ordersByType(...).length > 0)` (líneas 103-122), reemplazando su
      `flex flex-wrap` actual. Hace pasar T003. Depende de T005.
- [X] T007 [US1] En el mismo archivo: eliminar `scrollCarousel(direction)` (líneas 160-168) y el
      `viewChild<ElementRef<HTMLDivElement>>('carousel')` (línea 141), sin uso tras T005/T006. Hace
      pasar T004 (nada roto por código muerto residual). Depende de T005, T006.
- [ ] T008 [US1] Ejecutar manualmente quickstart.md Historia 1 completa (pasos 1-6) en los 3 anchos
      de referencia (~1280px, ~900px, ~390px). **Pendiente**: este entorno de implementación no
      tiene navegador/herramienta visual disponible — verificado por tests automatizados (T002-T004)
      que confirman las clases de breakpoint correctas y la ausencia del carrusel, pero la
      verificación visual real en las 3 anchuras queda pendiente para quien despliegue/revise en un
      navegador.

**Checkpoint**: la grilla de mesas y de pedidos Domicilio/Para llevar ya es responsive — verificable
de forma completamente independiente del resto de historias.

---

## Phase 4: User Story 2 - Ver de un vistazo cuántas mesas están libres, ocupadas o pendientes (Priority: P1)

**Goal**: un resumen de ocupación (total/ocupadas/libres/pendientes) en la parte superior de la
pestaña "Mesas", más el mismo conteo junto a cada filtro de ocupación — ambos derivados del mismo
origen de datos para que nunca puedan discreparse entre sí (FR-011).

**Independent Test**: contar manualmente las tarjetas visibles por estado y verificar que el
resumen superior y los contadores de los filtros coinciden exactamente con esa cuenta
(quickstart.md, Historia 2). Depende de US1 solo por compartir archivo (`pos-tables-panel.component.ts`)
— su lógica es independiente y podría implementarse antes si se prefiere.

### Tests for User Story 2

- [X] T009 [P] [US2] Test en
      `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts`: con un fixture de
      mesas en distintos estados, `occupancySummary()` devuelve `{ total, ocupadas, libres,
      pendientes }` consistente con `tablesView()` — `ocupadas` cuenta los estados
      `ocupada`/`en_preparacion`/`listo`, `pendientes` cuenta `por_confirmar`/`pago_pendiente`,
      `libres` cuenta `libre` (data-model.md, `OccupancySummary`)
- [X] T010 [P] [US2] Test en `pos-tables-panel.component.spec.ts`: el resumen superior renderiza los
      4 valores de `store.occupancySummary()`; cada botón de filtro
      ("Todas"/"Libres"/"Ocupadas"/"Pendientes") muestra, junto a su etiqueta, el conteo que le
      corresponde de ese mismo `occupancySummary()` — nunca un número calculado aparte (FR-010,
      FR-011)
- [X] T011 [P] [US2] Test en `pos-terminal.store.spec.ts`: al cambiar el estado de una mesa en el
      fixture (por ejemplo, de `ocupada` a `libre`), `occupancySummary()` se recalcula solo, sin
      ninguna llamada manual — mismo mecanismo reactivo (`computed`) que ya usa `tablesView()`
      (FR-012)

### Implementation for User Story 2

- [X] T012 [US2] En `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`: agregar
      `readonly occupancySummary = computed(() => { const list = this.tablesView(); return { total:
      list.length, ocupadas: list.filter(t => ['ocupada','en_preparacion','listo'].includes(t.status)).length,
      libres: list.filter(t => t.status === 'libre').length, pendientes: list.filter(t =>
      ['por_confirmar','pago_pendiente'].includes(t.status)).length }; })` — ajustar el nombre exacto
      del campo de estado crudo dentro de cada item de `tablesView()` al que realmente expone hoy
      `deriveTableStatus()` (verificar contra `pos-terminal.store.ts:168-205,624-666` antes de
      escribir el filtro). Hace pasar T009. Depende de T001. **Hecho** con un ajuste de diseño: en
      vez de derivar de `tablesView()` (que ya aplica `search()`/`filter()`, lo que haría que el
      resumen cambiara al buscar/filtrar), se agregó un computed privado `tableStatuses` sobre
      `this.tables()` sin filtrar — ver nota nueva en data-model.md/research.md §2.
- [X] T013 [US2] En `pos-tables-panel.component.ts`: agregar el resumen superior (nuevo bloque junto
      a la franja de pestañas/buscador) leyendo `store.occupancySummary()` — 4 valores: total,
      ocupadas, libres, pendientes (FR-009). Depende de T012.
- [X] T014 [US2] En el mismo archivo: junto a cada botón del array `filters` (líneas 149-154:
      `'todas'`/`'libres'`/`'ocupadas'`/`'pendientes'`), mostrar el conteo correspondiente de
      `store.occupancySummary()` (`'todas'→total`, `'libres'→libres`, `'ocupadas'→ocupadas`,
      `'pendientes'→pendientes`). Hace pasar T010. Depende de T012, T006 (mismo archivo que T006 ya
      modificó — aplicar sobre esa versión para evitar conflicto). **Hecho** vía `store.filterCount(f.key)`
      (método público agregado junto a `occupancySummary`) en vez de un mapeo inline en el template,
      para no duplicar el switch de claves en la plantilla. Efecto colateral corregido en el mismo
      cambio: el helper de test `tabButton()` (usado por specs anteriores a esta historia) hacía
      match exacto de texto — se actualizó a comparación por prefijo con espacio normalizado para
      seguir encontrando "Libres"/"Ocupadas"/etc. ahora que llevan un número al lado, sin romper
      ningún test ya existente.
- [ ] T015 [US2] Ejecutar manualmente quickstart.md Historia 2 completa (pasos 1-4). **Pendiente**
      (misma limitación de entorno que T008) — cubierto por tests automatizados equivalentes.

**Checkpoint**: los contadores de ocupación son visibles y siempre consistentes entre sí —
verificable de forma independiente.

---

## Phase 5: User Story 3 - Ver y abrir el turno de caja sin salir de la Terminal de Mesas (Priority: P2)

**Goal**: una barra superior nueva con turno de caja actual (o botón de abrirlo con atajo `F1`,
reutilizando `openShift()` ya existente), reloj en vivo, indicador de sincronización y etiqueta fija
"POS" — sin ningún ícono de bloqueo.

**Independent Test**: abrir la Terminal de Mesas sin turno de caja abierto, verificar que aparece el
botón de abrir turno, activarlo (clic o `F1`) y confirmar que dispara el mismo flujo ya
implementado en el módulo de Caja; repetir con un turno ya abierto y verificar que en su lugar se
muestra el cajero de turno (quickstart.md, Historia 3). Archivo distinto de US1/US2/US4/US5 — puede
implementarse en paralelo con cualquiera de ellas.

### Tests for User Story 3

- [X] T016 [P] [US3] Test nuevo en
      `pos-heladeria/src/app/modules/tables/components/pos-terminal-topbar.component.spec.ts`: sin
      turno abierto (`status !== 'open'` o ausencia de turno), se muestra el botón de abrir turno;
      con turno abierto, se muestra el nombre del cajero de turno y NO se muestra el botón (FR-015,
      FR-017)
- [X] T017 [P] [US3] Test en el mismo archivo (fake timers): el reloj muestra hora y fecha actuales
      y se actualiza con el tiempo (FR-013); con `navigator.onLine`/el estado de error del store
      mockeados, el indicador de sincronización refleja "en línea" o "sin conexión" según
      corresponda (FR-014, research.md §3)
- [X] T018 [P] [US3] Test en el mismo archivo: activar el botón de abrir turno invoca el mismo flujo
      ya existente (`CashSessionStore`/`CashService.openShift()` o la navegación equivalente al
      módulo de Caja) — sin ninguna lógica de apertura de turno duplicada (FR-016); también
      verifica que la etiqueta fija "POS" no dispara ninguna acción al pulsarla y que no existe
      ningún ícono de bloqueo/candado (FR-018, FR-019). **Hallazgo adicional no anticipado**: se
      agregó una prueba extra (no listada originalmente) de que un usuario con rol Mesero no ve el
      botón de abrir turno — spec 075 excluye Caja del alcance de ese rol; navegar ahí lo habría
      rebotado de inmediato por el `roleGuard` ya existente. Documentado como nuevo Edge Case en
      spec.md.
- [X] T019 [P] [US3] Test en
      `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.spec.ts`: presionar `F1`
      invoca la apertura de turno solo cuando no hay uno abierto y el foco no está en un campo de
      texto (mismo chequeo `typing` ya usado por los demás atajos); no hace nada si ya hay turno
      abierto o si se está escribiendo (Edge Cases de spec.md)

### Implementation for User Story 3

- [X] T020 [US3] Crear
      `pos-heladeria/src/app/modules/tables/components/pos-terminal-topbar.component.ts`:
      componente standalone que consume `CashSessionStore`/`CashService` (turno actual, apertura de
      turno) y muestra identificador de terminal/caja, turno de caja actual o botón de abrirlo,
      reloj en vivo, indicador de sincronización (`navigator.onLine` + `store.error()`, research.md
      §3) y etiqueta fija "POS" — sin ícono de candado (FR-013 a FR-019); en móvil, condensa estos
      elementos (hora sin segundos, fecha abreviada, etiqueta de turno corta, omite "POS") sin
      ocultar ninguna acción funcional (FR-020). Depende de T001. **Hecho**: además inyecta
      `AuthService` para condicionar el botón de abrir turno a roles Admin/Cajero (ver hallazgo de
      T018) y `PosTerminalStore` (ya provisto por el componente padre) solo para leer `error()` del
      indicador de sincronización.
- [X] T021 [US3] En
      `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts`: reemplazar la franja
      superior actual (líneas 56-77: botón de sidebar + título + hints F2/F3/ESC) insertando
      `<app-pos-terminal-topbar>`, conservando el toggle de sidebar y el título "Terminal de mesas"
      integrados en ella. Depende de T020. **Hecho**: el hint F2/F3/ESC que vivía aquí se retira sin
      reemplazo en este mismo lugar — Historia 5 (T035) lo reemplaza por una lista más completa de
      atajos dentro del panel derecho.
- [X] T022 [US3] En el mismo archivo, dentro de `onKey()` (líneas 270-297): agregar la rama `else if
      (e.key === 'F1') { ... }` que solo actúa si `!typing` y no hay turno de caja abierto,
      invocando la misma acción de abrir turno que el botón de T020 (FR-015, FR-016). Hace pasar
      T019. Depende de T020, T021. **Hecho** vía `this.topbar()?.openShift()` (nuevo `viewChild`,
      mismo patrón que `tablesPanel`/`focusSearch()`) — `openShift()` ya trae su propio chequeo de
      rol/turno abierto, sin duplicarlo aquí.
- [ ] T023 [US3] Ejecutar manualmente quickstart.md Historia 3 completa (pasos 1-5, incluida la
      simulación de desconexión de red en DevTools). **Pendiente** (misma limitación de entorno que
      T008/T015) — cubierto por los 7 tests de `pos-terminal-topbar.component.spec.ts` y los 3 de
      `table-sessions.component.spec.ts` (atajo F1).

**Checkpoint**: la barra superior queda completamente funcional — verificable de forma
independiente de las demás historias.

---

## Phase 6: User Story 4 - Ver quién atiende cada pedido de mesa (Priority: P2)

**Goal**: exponer `DiningOrder.user_id` (ya capturado hoy, sin migración) como el nombre del usuario
de staff (Cajero o Mesero) que creó el pedido, y mostrarlo como "Atendido por" en la tarjeta de
mesa.

**Independent Test**: crear un pedido de mesa con un usuario Cajero y otro con un usuario Mesero;
verificar que ambas tarjetas muestran "Atendido por" con el nombre correspondiente, sin distinción
de rol; verificar que un pedido de cliente por QR o una mesa libre no muestran esa referencia
(quickstart.md, Historia 4). Depende de US1 (la tarjeta debe seguir viviendo dentro de la grilla ya
migrada) y comparte archivos con US2 (`pos-terminal.store.ts`, `pos-tables-panel.component.ts`) —
se implementa después de esa para evitar conflictos de archivo; su parte de backend es
independiente y puede adelantarse en cualquier momento.

### Tests for User Story 4

- [X] T024 [P] [US4] Backend: crear
      `pos-backend/app/scripts/test_order_staff_user.py` (script autoejecutable, mismo patrón que
      `test_table_release.py` — sin pytest): crea una orden de mesa con `user_id` de un usuario de
      staff conocido y verifica que `OrderResponse.staff_user_name` devuelve el nombre esperado;
      crea otra orden sin `user_id` (equivalente a un pedido QR) y verifica que el campo es `None`;
      borra los datos de prueba que crea al terminar (mismo criterio que `test_table_release.py`).
      **Escrito y verificado parcialmente**: sintaxis válida, imports correctos, y ejecutado contra
      la base de este entorno — llega hasta `shared.tenants` correctamente (existe) pero se detiene
      con el mensaje de guardia esperado porque el tenant de prueba de este sandbox no tiene ni el
      schema `heladeria.*` provisionado (mismo hallazgo que T001) ni ningún usuario de staff — no es
      una falla del código, es que este entorno no tiene un tenant de prueba completo. Queda
      pendiente de correr de punta a punta en un entorno con esa base ya provisionada.
- [X] T025 [P] [US4] Frontend: test en `pos-terminal.store.spec.ts`: el view-model de tarjeta de una
      mesa con un pedido cuyo `staff_user_name` está presente expone `atendidoPor` con ese valor;
      una mesa libre o con un pedido sin `staff_user_name` (QR) expone `atendidoPor: null` (FR-022,
      FR-023); con más de un pedido activo en la mesa, `atendidoPor` corresponde al mismo pedido
      principal que ya determina el resto de la tarjeta (FR-024)
- [X] T026 [P] [US4] Frontend: test en `order-summary-card.component.spec.ts`: el nuevo input
      opcional `atendidoPor` se renderiza como referencia "Atendido por" cuando tiene valor, y no
      renderiza ningún elemento (sin espacio vacío) cuando es `null`/`undefined`

### Implementation for User Story 4

- [X] T027 [US4] Backend: en `pos-backend/app/api/v1/orders/schemas.py`, agregar `staff_user_name:
      str | None = None` a `OrderResponse` (líneas 197-224), como campo computado (mismo patrón ya
      usado por `paid`, contracts/order-response.md). Depende de T001.
- [X] T028 [US4] Backend: en `pos-backend/app/api/v1/orders/service.py`, en el mismo punto donde hoy
      se calcula/asigna `paid` antes de serializar cada orden, agregar el lookup de `user_id →
      nombre para mostrar` en `shared.users` (mismo criterio de resolución de nombre ya usado por
      `CashShift.user_name`) y asignarlo a `staff_user_name`; `None` si `user_id` es `None`. Hace
      pasar T024. Depende de T027. **Hecho**: se agregó `service.staff_user_names(db, user_ids)`
      (resolución en bloque, mismo patrón que `paid_order_ids`) y se conectó tanto en
      `router.list_orders()` (endpoint que consume `DiningSessionService.listOrders()` del frontend)
      como en `router._load_order()` (GET de una sola orden), para no dejar ese segundo camino
      inconsistente. Verificado por import (`python -c "from app.api.v1.orders import
      service,schemas,router"`) — no se pudo ejecutar contra una base de datos real en este entorno
      (ver T001, T024).
- [X] T029 [US4] Frontend: en
      `pos-heladeria/src/app/modules/tables/interfaces/dining.interface.ts`, agregar
      `staff_user_name?: string | null` a la interfaz `DiningOrder`.
- [X] T030 [US4] Frontend: en `pos-terminal.store.ts`, en la construcción del view-model de tarjeta
      (`tablesView()`/`toOrderCardView()`), agregar `atendidoPor: order.staff_user_name ?? null`
      derivado del mismo pedido principal ya usado para el resto de la tarjeta (FR-024). Hace pasar
      T025. Depende de T029, T012 (mismo archivo que T012 ya modificó). **Hecho**: en `tablesView()`
      se identifica el "pedido principal" (el más antiguo, mismo criterio que ya usaba `oldest` para
      `elapsedLabel`) y se toma su `staff_user_name`; en `toOrderCardView()` (pedidos sin mesa) se usa
      directamente el único pedido del que ya parte esa función.
- [X] T031 [US4] Frontend: en `order-summary-card.component.ts`, agregar `@Input() atendidoPor?:
      string | null` y renderizarlo condicionalmente (`@if (atendidoPor) { ... }`) junto al resto
      del contenido de la tarjeta. Hace pasar T026.
- [X] T032 [US4] Frontend: en `pos-tables-panel.component.ts`, pasar `[atendidoPor]="t.atendidoPor"`
      (y su equivalente en la rama de `store.ordersByType()`) a `<app-order-summary-card>`. Depende
      de T030, T031, T014 (mismo archivo que T014 ya modificó).
- [ ] T033 [US4] Ejecutar manualmente quickstart.md Historia 4 completa (pasos 1-4), con un usuario
      Cajero y uno Mesero. **Pendiente** (misma limitación de entorno que T008/T015/T023/T036 —
      sin navegador ni base de datos con tenant provisionado en este sandbox) — cubierto por los
      tests automatizados de T025/T026 (frontend) y el script de T024 (backend, verificado hasta
      donde el entorno lo permite).

**Checkpoint**: "Atendido por" visible de punta a punta (backend + frontend), sin ninguna migración
— verificable de forma independiente.

---

## Phase 7: User Story 5 - Saber qué hacer cuando no hay ninguna mesa seleccionada (Priority: P3)

**Goal**: el panel derecho, sin ninguna mesa/pedido seleccionado, muestra un mensaje de bienvenida,
la lista de atajos de teclado disponibles (F2/F3/F1) y el botón fijo "+ Crear pedido nuevo".

**Independent Test**: abrir la Terminal de Mesas sin seleccionar nada y verificar que el panel
derecho muestra el mensaje de bienvenida, los atajos y el botón fijo; al seleccionar una mesa o
pedido, verificar que ese contenido desaparece sin afectar el comportamiento ya existente
(quickstart.md, Historia 5). Archivo independiente (`pos-checkout-panel.component.ts`) — puede
implementarse en paralelo con cualquier otra historia.

### Tests for User Story 5

- [X] T034 [P] [US5] Test en
      `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.spec.ts`: sin
      ninguna mesa/pedido seleccionado, el placeholder de modo `'terminal-pos'` muestra el mensaje
      de bienvenida, la lista de atajos (buscar mesa `F2`, crear pedido `F3`, abrir turno de caja
      `F1`) y el botón fijo "+ Crear pedido nuevo" (FR-025 a FR-027); al seleccionar una mesa/pedido,
      ese contenido deja de mostrarse y el comportamiento ya existente de detalle/cobro no cambia
      (FR-028)

### Implementation for User Story 5

- [X] T035 [US5] En `pos-checkout-panel.component.ts`, en el bloque de placeholder de modo
      `'terminal-pos'` sin selección (líneas 126-146), agregar el mensaje de bienvenida y la lista
      de atajos de teclado (F2/F3/F1) junto al botón "+ Crear pedido nuevo" ya existente, fijándolo
      en la parte inferior del panel. Hace pasar T034. Depende de T022 (el atajo `F1` ya debe
      existir para poder documentarlo aquí con su comportamiento real). **Hecho** con un ajuste de
      condición no anticipado por el task original: el `@if (!store.selectedOrder())` existente
      englobaba tanto "nada seleccionado" como "mesa libre ya seleccionada" (ambos casos sin pedido).
      Se dividió usando `store.hasActiveSelection()` (ya existente, spec 059) para que el mensaje de
      bienvenida + atajos solo aparezca cuando de verdad no hay ninguna mesa/pedido elegido — una
      mesa libre seleccionada sigue mostrando solo el botón de crear pedido, sin regresión (FR-025
      exige "sin ninguna mesa ni pedido seleccionado", no solo "sin pedido"). También se ajustó el
      título del panel para mostrar "Detalle de mesa / pedido — sin selección" en ese caso vacío
      (antes decía "Pedido de mostrador" incluso sin nada seleccionado).
- [ ] T036 [US5] Ejecutar manualmente quickstart.md Historia 5 completa (pasos 1-2). **Pendiente**
      (misma limitación de entorno que T008/T015/T023) — cubierto por los 3 tests nuevos en
      `pos-checkout-panel.component.spec.ts`.

**Checkpoint**: todas las historias son ahora independientemente funcionales.

---

## Phase 8: Polish & Cross-Cutting Concerns

- [ ] T037 [P] Ejecutar quickstart.md completo (las 5 historias + la sección de "Regresión rápida")
      de punta a punta, en una sola sesión sin recargar la página entre historias. **Pendiente**
      (requiere navegador — ver T008/T015/T023/T033/T036) — la "Regresión rápida" (vocabulario y
      colores sin cambios, atajos F2/F3/ESC/Ctrl+P intactos) sí queda cubierta indirectamente por
      T038 (0 regresiones en la suite completa).
- [X] T038 Ejecutar la suite completa de tests de `pos-heladeria` (`npx ng test --watch=false`) y
      comparar contra la línea base de T001 — confirmar que cualquier test en rojo ya lo estaba
      antes de esta spec (Principio X). **Resultado**: 714 en verde / 7 en rojo (721 totales) — las
      mismas 7 fallas y los mismos 5 archivos de la línea base de T001, ninguna nueva. 27 tests
      nuevos agregados por esta spec (694→721), todos en verde: 9 (US1/US2/US4 en
      `pos-tables-panel.component.spec.ts` + `pos-terminal.store.spec.ts`), 2
      (`order-summary-card.component.spec.ts`), 7 (`pos-terminal-topbar.component.spec.ts`), 3
      (atajo F1 en `table-sessions.component.spec.ts`), 3 (`pos-checkout-panel.component.spec.ts`,
      Historia 5), y el resto ya contado en ejecuciones parciales previas. Cero regresiones.
- [ ] T039 Ejecutar `python -m app.scripts.test_order_staff_user` y
      `python -m app.scripts.test_table_release` en `pos-backend` y confirmar que ambos pasan
      (Principio III/X). **Parcial**: `test_table_release.py` falla exactamente igual que en la
      línea base de T001 (mismo error de schema `heladeria.*` sin provisionar en este sandbox) —
      confirma que los cambios de esta spec no le introdujeron ninguna regresión propia.
      `test_order_staff_user.py` (nuevo) llega hasta el mismo punto de guardia esperado (T024). El
      import completo de `app.api.v1.orders` (`router`+`service`+`schemas`) se verificó sin errores.
      **Pendiente**: correr ambos de punta a punta contra una base con el tenant de prueba
      completamente provisionado, fuera del alcance de este entorno.
- [ ] T040 [P] Comparar visualmente los 3 anchos de referencia contra las imágenes originales en
      `pos-specs/terminal-mesas/screen.png`, `screen-tablet.png`, `screen-mobile.png` — confirmar
      fidelidad de estructura/disposición sin haber copiado los colores exactos del mockup (FR-029,
      FR-030). **Pendiente** (requiere navegador) — la fidelidad estructural (columnas por
      breakpoint, ausencia de carrusel) ya la verifican T002/T003; la preservación de colores/
      vocabulario ya existentes (FR-029/FR-030) no se tocó en ningún archivo de esta spec —
      `STATUS_META` (`pos-terminal.store.ts:115-123`) sigue exactamente igual.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: vacía — ver nota en esa fase.
- **US1 (Phase 3)**: depende solo de Setup (T001). Puede empezar de inmediato.
- **US2 (Phase 4)**: depende de Setup (T001); comparte archivo con US1
  (`pos-tables-panel.component.ts`) — se implementa después de US1 para evitar conflictos de merge,
  aunque su lógica (`occupancySummary`) es independiente.
- **US3 (Phase 5)**: depende solo de Setup (T001) — archivo propio
  (`pos-terminal-topbar.component.ts` + `table-sessions.component.ts`), sin relación con US1/US2/US4.
  Puede avanzar en paralelo con cualquiera de ellas.
- **US4 (Phase 6)**: su parte de backend (T024, T027, T028) depende solo de Setup y puede
  adelantarse en paralelo con cualquier historia de frontend. Su parte de frontend comparte archivo
  con US1/US2 (`pos-terminal.store.ts`, `pos-tables-panel.component.ts`) — se implementa después de
  US2 para evitar conflictos.
- **US5 (Phase 7)**: archivo propio (`pos-checkout-panel.component.ts`), pero su tarea de
  implementación (T035) depende de que el atajo `F1` de US3 (T022) ya exista, para documentarlo
  correctamente en la lista de atajos.
- **Polish (Phase 8)**: depende de que las 5 historias estén completas.

### Dentro de cada historia

- US1: T002-T004 (tests) antes de T005-T007 (implementación) — T005 hace pasar T002, T006 hace
  pasar T003, T007 hace pasar T004 (no-regresión). T008 al final.
- US2: T009-T011 (tests) antes de T012-T014 (implementación) — T012 hace pasar T009 y T011; T013-T014
  hacen pasar T010. T015 al final.
- US3: T016-T019 (tests) antes de T020-T022 (implementación) — T020 hace pasar T016-T018; T022 hace
  pasar T019. T023 al final.
- US4: T024-T026 (tests) antes de T027-T032 (implementación) — T027-T028 hacen pasar T024 (backend);
  T029-T030 hacen pasar T025; T031 hace pasar T026; T032 conecta todo en la UI. T033 al final.
- US5: T034 (test) antes de T035 (implementación). T036 al final.

### Parallel Opportunities

- US1: T002, T003, T004 en paralelo entre sí (mismo archivo de test, casos independientes).
- US2: T009, T010, T011 en paralelo entre sí (T009/T011 mismo archivo de store, casos
  independientes; T010 archivo de componente distinto).
- US3: T016, T017, T018, T019 en paralelo entre sí (T016-T018 mismo archivo de test del componente
  nuevo, casos independientes; T019 archivo distinto).
- US4: T024 (backend) en paralelo con T025-T026 (frontend) — repositorios y archivos distintos.
- US5: T034 puede escribirse en paralelo con cualquier test de otra historia (archivo propio).
- Con más de una persona disponible: una persona puede tomar US1→US2→US4(frontend) en secuencia
  (mismos archivos) mientras otra toma US3 en paralelo completo, y una tercera adelanta US4(backend)
  y US5 en paralelo con ambas líneas.

---

## Parallel Example: User Story 1

```bash
Task: "Test: la grilla ya no tiene flechas de desplazamiento y expone clases responsive (T002)"
Task: "Test: la rama de Domicilio/Para llevar usa el mismo contenedor de grilla (T003)"
Task: "Test: ningún input de order-summary-card se pierde tras el cambio de contenedor (T004)"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Phase 1: Setup (T001).
2. Completar Phase 3: User Story 1 (T002-T008).
3. **Detener y validar**: probar US1 de forma independiente (quickstart.md Historia 1).
4. Este MVP ya resuelve el pedido central de la spec: la grilla responsive reemplazando el carrusel.

### Entrega incremental

1. Setup → US1 (MVP: grilla responsive).
2. Añadir US2 (contadores de ocupación) → validar → entregar.
3. Añadir US3 (barra superior operativa) → validar → entregar — puede intercalarse en cualquier
   momento después de Setup, en paralelo con US1/US2.
4. Añadir US4 (Atendido por) → validar → entregar.
5. Añadir US5 (panel sin selección) → validar → entregar.

### Estrategia con equipo paralelo

Con más de una persona disponible tras Setup:

- Persona A: US1 → US2 → US4 (frontend) en secuencia (mismos archivos).
- Persona B: US3 completo, en paralelo con la línea de A.
- Persona C: US4 (backend) y US5, en paralelo con ambas líneas — US5 solo espera a que T022 (atajo
  `F1` de US3) exista antes de documentarlo en su lista de atajos.

---

## Notes

- [P] tasks = archivo distinto, sin dependencias pendientes.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad.
- Escribir los tests de cada historia primero y confirmar que fallan antes de implementar.
- Commitear después de cada tarea o grupo lógico de tareas.
- Detenerse en cualquier checkpoint para validar una historia de forma independiente.
- Evitar: tareas vagas, conflictos de archivo simultáneos entre historias que comparten archivo
  (US1/US2/US4 en frontend — respetar el orden de Dependencies arriba), y dependencias cruzadas que
  rompan la independencia de una historia sin que spec.md la haya declarado.
