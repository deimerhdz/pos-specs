---

description: "Task list template for feature implementation"
---

# Tasks: Rediseño de Layout de la Terminal de Mesas — Grilla de Mesas, Pagos por Confirmar y Menú Central

**Input**: Design documents from `/specs/036-terminal-mesas-rediseno-layout/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/ui-store-contract.md](./contracts/ui-store-contract.md), [quickstart.md](./quickstart.md)

**Nota de regeneración**: esta versión de `tasks.md` reemplaza por completo la anterior (2026-08-25),
generada antes de que la sesión de clarificación del 2026-08-26 reemplazara la imagen de referencia
original por dos prototipos nuevos y corrigiera el diseño objetivo (franja de "órdenes" con carrusel →
grilla de mesas + "Pagos por confirmar"; catálogo permanente → lista de ítems + "+ Agregar producto").
Se verificó contra el repositorio real (`pos-heladeria`, rama `develop`, árbol de trabajo limpio) que
**ninguna de las tareas de la versión anterior llegó a implementarse ni a comitearse** — sus marcas
`[X]` no reflejan el estado real del código. Esta versión parte del código base real y actual.

**Tests**: Se incluyen tareas de test. La Constitución (Principio X, Verificación Obligatoria) exige
verificar toda funcionalidad nueva, y `research.md` §6 ya decidió agregar cobertura donde no existe hoy
(`pos-tables-panel.component.ts` no tiene spec; el componente nuevo `pending-payments-panel` tampoco).
Los specs ya existentes (`pos-checkout-panel.component.spec.ts`, `pos-order-panel.component.spec.ts`,
`split-bill-panel.component.spec.ts`, `session-bill-panel.component.spec.ts`,
`pos-terminal.store.spec.ts`) deben permanecer en verde.

**Organization**: Las tareas están agrupadas por historia de usuario (spec.md) para que cada una se
pueda implementar y probar de forma independiente. Todas las rutas de archivo son relativas al
repositorio de la aplicación `../pos-heladeria` (el código no vive en este repositorio de specs).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de tareas incompletas)
- **[Story]**: Historia de usuario a la que pertenece la tarea (US1, US2, US3)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

Todas las rutas usan como raíz `pos-heladeria/` (repositorio hermano de este `pos-specs`), según la
`Project Structure` de [plan.md](./plan.md). No hay cambios en `pos-backend` en esta spec.

---

## Phase 1: Setup

**Purpose**: Confirmar el estado base del entorno antes de tocar cualquier archivo

- [X] T001 Ejecutar la suite de tests existente en `pos-heladeria` (`pnpm test` o el script configurado en `package.json`) y confirmar/registrar el estado real (verde, o fallas preexistentes ajenas a esta spec) como línea base de regresión (Principio X) — el build de tests no compilaba en `develop` por 3 fixtures desactualizados (spec 033: campo `plan` retirado de `Tenant`/`TenantInfo`); corregidos como parte de esta tarea (fuera de alcance de negocio, solo tipos de fixture) para poder registrar línea base real: **364/376 tests pasan**, 12 fallas preexistentes y ajenas a esta spec (`app.spec.ts`, `auth.service.spec.ts`, `sidebar.component.spec.ts` — nav de super-admin —, y una regresión de `MoneyInputComponent` en `pos-checkout-panel.component.spec.ts`/`session-bill-panel.component.spec.ts`)
- [X] T002 [P] Levantar el entorno de desarrollo local (`pnpm start` en `pos-heladeria`) y confirmar que compila y sirve sin errores — `ng build` y `ng serve` verificados sin errores (solo warnings preexistentes de budget/CommonJS)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Prerrequisitos bloqueantes compartidos por todas las historias

**Nota**: esta feature no tiene prerrequisitos fundacionales bloqueantes. Las tres historias tocan
señales de store y componentes mayormente independientes entre sí (US1: `orderTypeTab`/
`pendingPaymentsView` en `pos-terminal.store.ts` + `pos-tables-panel.component.ts` + componente nuevo
`pending-payments-panel.component.ts`; US2: `catalogSearchText`/`catalogProductsFiltered` en
`pos-terminal.store.ts` + `pos-catalog-drawer.component.ts` + `pos-order-panel.component.ts`; US3:
`sidebar.component.ts`/`dashboard-layout.component.ts`) y pueden implementarse en cualquier orden, o en
paralelo por distintas personas, una vez completada la Fase 1. El único punto de posible colisión de
archivo es `pos-terminal.store.ts`, compartido por US1 y US2.

**Checkpoint**: Fase 1 completa — cualquier historia de usuario puede comenzar.

---

## Phase 3: User Story 1 - El cajero encuentra cualquier mesa y revisa los pagos pendientes de confirmar desde la parte superior de la pantalla (Priority: P1) 🎯 MVP

**Goal**: Agregar pestañas de tipo de orden ("Mesas"/"Domicilios"/"Para llevar") por encima de la
grilla de mesas ya existente (sin tocar su filtro de ocupación), y una sección nueva "Pagos por
confirmar" debajo de la grilla que agrupa todos los pagos pendientes de revisión ya existentes,
reutilizando la misma lógica de confirmación de hoy.

**Independent Test**: Abrir la Terminal de Mesas con mesas en distintos estados de ocupación y con
pagos pendientes de revisión; alternar entre las pestañas "Mesas"/"Domicilios"/"Para llevar" y los
filtros de ocupación ya existentes; confirmar un pago desde "Pagos por confirmar" y verificar que
produce el mismo efecto que confirmarlo desde el panel de la mesa; seleccionar una tarjeta (desde la
grilla o desde "Pagos por confirmar") y confirmar que abre la misma orden que hoy.

### Tests for User Story 1 ⚠️

> Escribir estos tests primero; deben fallar antes de implementar.

- [X] T003 [P] [US1] Crear `pos-tables-panel.component.spec.ts` en `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.spec.ts` cubriendo: las 3 pestañas de tipo de orden, "Domicilios"/"Para llevar" siempre vacíos con mensaje claro, y que el filtro de ocupación ya existente (Todas/Libres/Ocupadas/Pendientes) sigue funcionando sin cambios cuando la pestaña activa es "Mesas"
- [X] T004 [P] [US1] Extender `pos-terminal.store.spec.ts` en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` con casos para `orderTypeTab`, `setOrderTypeTab()` y el computed `pendingPaymentsView` (une `pendingOrders()` con `tables()`; vacío cuando `orderTypeTab() !== 'mesas'`)
- [X] T005 [P] [US1] Crear `pending-payments-panel.component.spec.ts` en `pos-heladeria/src/app/modules/tables/components/pending-payments-panel.component.spec.ts` cubriendo: renderizado de cada pago pendiente (cliente/mesa, método, estado, total), confirmar efectivo, aprobar/rechazar transferencia, y que seleccionar una tarjeta invoca `selectTable()`

### Implementation for User Story 1

- [X] T006 [US1] Implementar en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` el signal `orderTypeTab` (`'mesas' | 'domicilios' | 'para-llevar'`, por defecto `'mesas'`) y el método `setOrderTypeTab()`, sin tocar el `filter` de ocupación ya existente (FR-001, FR-003) — depende de T004
- [X] T007 [US1] Implementar en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` el computed `pendingPaymentsView`, uniendo `pendingOrders()` (ya existente) con `tables()` para exponer número/nombre de mesa, método de pago, estado de revisión y total; DEBE resolver a lista vacía cuando `orderTypeTab() !== 'mesas'` (research.md §2; FR-004) — depende de T004
- [X] T008 [US1] Agregar en `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.ts` las pestañas de tipo de orden enlazadas a `orderTypeTab()`/`setOrderTypeTab()` por encima del filtro de ocupación ya existente; cuando la pestaña activa no es "Mesas", la grilla muestra un estado vacío claro en vez de la lista de mesas (FR-001, FR-003) — depende de T006, T003
- [X] T009 [US1] Crear `pos-heladeria/src/app/modules/tables/components/pending-payments-panel.component.ts`, renderizando `pendingPaymentsView` como una tarjeta por pago pendiente; evaluar reutilizar `payment-attempt-review-panel.component.ts` en modo compacto o construir un componente delgado que delegue la confirmación/aprobación/rechazo a los mismos métodos del store que ya usa ese componente (research.md §2; FR-004) — depende de T007, T005 — se optó por embeber `<app-payment-attempt-review-panel>` (tal cual, sin modo compacto nuevo) por cada pago, ya que ese componente ya es una tarjeta angosta; cero lógica de confirmación duplicada
- [X] T010 [US1] Montar `<app-pending-payments-panel>` debajo de `<app-pos-tables-panel>` en `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts`, envolviendo ambos en un contenedor `flex flex-col` (research.md §2; FR-004) — depende de T009
- [X] T011 [US1] Confirmar que seleccionar una tarjeta (desde la grilla o desde "Pagos por confirmar") invoca `store.selectTable()` sin cambios y abre el mismo panel central/derecho ya implementado (FR-005); ajustar únicamente la disposición visual de `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.ts` a los prototipos, sin tocar su lógica ni la de `split-bill-panel.component.ts`/`session-bill-panel.component.ts` (FR-009, FR-010, FR-011) — depende de T010 — FR-005 verificado y probado (ambas vistas llaman a `selectTable()`, sin lógica nueva); el reacomodo visual del panel derecho a los prototipos **no se aplicó**: esta sesión no tuvo acceso a las imágenes de referencia, y `pos-checkout-panel.component.ts` ya tenía fallas preexistentes de otra causa (ver T001) — modificar su layout a ciegas arriesgaba una regresión real sin poder contrastarla contra el prototipo
- [X] T012 [US1] Ejecutar T003/T004/T005 hasta que pasen en verde — depende de T011

**Checkpoint**: La Historia 1 es funcional y probable de forma independiente.

---

## Phase 4: User Story 2 - El cajero arma la orden desde la lista de ítems del pedido, agregando productos desde una grilla con búsqueda y categorías (Priority: P1)

**Goal**: El panel central muestra por defecto la lista de ítems del pedido en construcción (patrón ya
existente en `pos-order-panel.component.ts`); el botón "+ Agregar producto" abre, embebida en el mismo
panel (sin overlay de pantalla completa), la grilla de catálogo con buscador por nombre nuevo combinado
con el filtro de categoría ya existente.

**Independent Test**: Con una orden en construcción, pulsar "+ Agregar producto", escribir en el
buscador y confirmar que la grilla se filtra por nombre; seleccionar una categoría y confirmar que se
filtra por categoría; agregar un producto y confirmar que el panel central vuelve a la lista de ítems
con el producto agregado, y que el resumen derecho refleja el mismo comportamiento ya implementado.

### Tests for User Story 2 ⚠️

- [X] T013 [P] [US2] Extender `pos-order-panel.component.spec.ts` en `pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.spec.ts` cubriendo: pulsar "+ Agregar producto" embebe el catálogo en el mismo panel (no navega ni abre overlay), y seleccionar un producto regresa a la lista de ítems con el producto agregado
- [X] T014 [P] [US2] Crear o extender `pos-catalog-drawer.component.spec.ts` en `pos-heladeria/src/app/modules/tables/components/pos-catalog-drawer.component.spec.ts` cubriendo: filtro por nombre nuevo, combinación con filtro de categoría ya existente, estado vacío sin coincidencias, y ausencia del wrapper de overlay de pantalla completa
- [X] T015 [P] [US2] Extender `pos-terminal.store.spec.ts` en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` con casos para `catalogSearchText`, `setCatalogSearchText()` y el computed `catalogProductsFiltered` (intersección categoría + nombre)

### Implementation for User Story 2

- [X] T016 [US2] Implementar en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` el signal `catalogSearchText`, el método `setCatalogSearchText()` y el computed `catalogProductsFiltered`, combinando el `catalogCategoryId`/`catalogProducts` ya existentes con el nuevo texto de búsqueda (research.md §3; FR-007) — depende de T015
- [X] T017 [US2] Retirar en `pos-heladeria/src/app/modules/tables/components/pos-catalog-drawer.component.ts` el wrapper de overlay (`fixed inset-0 bg-black/40`) y su animación de panel deslizante; agregar el input de búsqueda por nombre enlazado a `catalogSearchText`/`setCatalogSearchText()`; cambiar el grid para leer `catalogProductsFiltered` en vez de `catalogProducts`, conservando intacta la lógica de pestañas de categoría ya existente (FR-006, FR-007) — depende de T016, T014
- [X] T018 [US2] Agregar en `pos-heladeria/src/app/modules/tables/components/pos-catalog-drawer.component.ts` un estado vacío claro para cuando `catalogProductsFiltered()` no tiene resultados (FR-007, edge case) — depende de T017
- [X] T019 [US2] En `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts` (o dentro de `pos-order-panel.component.ts`), alternar dentro del mismo panel central entre la lista de ítems del pedido (`cartView`, cuando `catalogOpen()` es falso) y el catálogo embebido refactorizado (cuando `catalogOpen()` es verdadero); seleccionar un producto desde la grilla DEBE volver `catalogOpen` a falso automáticamente, reutilizando `openCatalog()`/`addToOrder()` ya existentes (FR-006, FR-007) — depende de T017 — implementado dentro de `pos-order-panel.component.ts` (computed `showCatalog`); ya era el comportamiento existente de `addDraftFromSelection()`/`addComboDraft()`, sin cambios de lógica
- [X] T020 [US2] Confirmar que el catálogo embebido no se ofrece para una orden QR en modo "Resumen de Cuenta" de solo lectura, con el mismo criterio ya usado por `pos-checkout-panel.component.ts` para su propio modo de solo lectura (US2, escenario 5) — depende de T019
- [X] T021 [US2] Ejecutar T013/T014/T015 hasta que pasen en verde — depende de T020

**Checkpoint**: Las Historias 1 y 2 son funcionales de forma independiente.

---

## Phase 5: User Story 3 - El usuario colapsa el menú de navegación global para ganar espacio de pantalla (Priority: P2)

**Goal**: Agregar un botón en la parte superior de la Terminal de Mesas que colapsa/expande el menú de
navegación global del panel administrativo, incluyendo el efecto de colapso en escritorio que hoy no
existe (`sidebarOpen()` hoy solo controla el slide-over móvil).

**Independent Test**: Con el menú global visible en una pantalla de escritorio, pulsar el botón y
confirmar que se oculta y el contenido ocupa el espacio liberado; pulsar de nuevo y confirmar que
vuelve a mostrarse; navegar a otra sección del panel administrativo con el menú colapsado y confirmar
que sigue siendo accesible.

### Tests for User Story 3 ⚠️

- [X] T022 [P] [US3] Crear/extender `sidebar.component.spec.ts` en `pos-heladeria/src/app/modules/dashboard/layout/sidebar.component.spec.ts` cubriendo: `sidebarOpen()` en `true`/`false` reflejándose también en clases de escritorio (hoy el componente las ignora ahí)

### Implementation for User Story 3

- [X] T023 [US3] Modificar la plantilla de `pos-heladeria/src/app/modules/dashboard/layout/sidebar.component.ts` para que las clases `md:relative md:translate-x-0` (hoy incondicionales) dependan de `layoutService.sidebarOpen()` también en escritorio (research.md §4; FR-012) — depende de T022 — se retiraron por completo (el `<aside>` ya es `fixed` en todos los breakpoints y las clases reactivas `-translate-x-full`/`translate-x-0` bastan solas); también se ajustó el valor inicial de `sidebarOpen()` en `layout.service.ts` (según `window.innerWidth` al arrancar) para no cambiar el comportamiento por defecto ya existente (visible en escritorio, oculto en móvil) — ver nota de alcance en Completion Report
- [X] T024 [US3] Ajustar `pos-heladeria/src/app/modules/dashboard/layout/dashboard-layout.component.ts` para que el contenido principal ocupe el espacio liberado cuando el sidebar está colapsado en escritorio — depende de T023
- [X] T025 [US3] Agregar en `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts` un botón en la parte superior de la Terminal de Mesas que inyecta `LayoutService` y llama a `toggle()`, con su ícono reflejando `sidebarOpen()` (FR-012) — depende de T024
- [X] T026 [US3] Confirmar que colapsar/expandir el sidebar no interrumpe una validación de pago o un cobro en curso en los paneles central/derecho (edge case); ejecutar T022 hasta que pase en verde — depende de T025

**Checkpoint**: Las tres historias de usuario son funcionales de forma independiente.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final que abarca las tres historias

- [ ] T027 [P] Ejecutar manualmente los 11 escenarios de [quickstart.md](./quickstart.md) contra el entorno de desarrollo local (`pos-heladeria` + `pos-backend`) — **no ejecutado en esta sesión**: requiere `pos-backend` corriendo y un navegador interactivo, no disponibles en este entorno de agente; queda pendiente de validación manual por el equipo antes de dar por cerrado el MVP (ver Completion Report)
- [X] T028 [P] Confirmar que `pos-checkout-panel.component.spec.ts`, `pos-order-panel.component.spec.ts` (previo a los cambios de US2), `split-bill-panel.component.spec.ts`, `session-bill-panel.component.spec.ts` y `pos-terminal.store.spec.ts` (ya existentes antes de esta feature) siguen en verde sin cambios de intención, solo ajustes de selector si el reacomodo visual lo exige (Principio X) — `pos-order-panel`, `split-bill-panel` y `pos-terminal.store` en verde; `pos-checkout-panel`/`session-bill-panel` tienen 9 fallas **preexistentes en `develop`, ajenas a esta spec** (regresión de `MoneyInputComponent`, ver T001) que ni se introdujeron ni se agravaron aquí — línea base idéntica antes/después, confirmada con la suite completa
- [X] T029 Eliminar cualquier remanente muerto del wrapper de overlay retirado de `pos-catalog-drawer.component.ts` y de cualquier estilo o marcado sin usar tras el refactor de `pos-tables-panel.component.ts`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Fase 1)**: sin dependencias — puede iniciar de inmediato
- **Foundational (Fase 2)**: sin tareas bloqueantes (ver nota arriba) — las historias pueden iniciar tras la Fase 1
- **Historias de usuario (Fase 3-5)**: cada una depende solo de la Fase 1; son independientes entre sí y pueden ejecutarse en paralelo si hay capacidad de equipo, o en orden de prioridad (US1 → US2 → US3)
- **Polish (Fase 6)**: depende de que las historias que se quieran entregar ya estén completas

### Dependencias dentro de cada historia

- US1: T003/T004/T005 (tests, en paralelo) → T006/T007 (store, mismo archivo, pueden hacerse en secuencia o por la misma persona) → T008 (pestañas en la grilla) y T009 (componente nuevo, en paralelo entre sí tras T006/T007) → T010 (montar en la página) → T011 (verificación panel derecho) → T012 (verificación tests)
- US2: T013/T014/T015 (tests, en paralelo) → T016 (store) → T017 (refactor del catálogo, depende de T016 y T014) → T018 (estado vacío, mismo archivo) → T019 (alternar lista/catálogo en el panel central) → T020 (modo solo lectura) → T021 (verificación tests)
- US3: T022 (test) → T023 (sidebar) → T024 (layout) → T025 (botón en la Terminal) → T026 (verificación)

### Parallel Opportunities

- T001/T002 (Setup) en paralelo
- T003/T004/T005 (tests de US1) en paralelo — archivos distintos
- T013/T014/T015 (tests de US2) en paralelo — archivos distintos
- T022 (único test de US3) puede ejecutarse en paralelo con los tests de US1/US2 si hay capacidad de equipo
- Una vez completada la Fase 1, US1, US2 y US3 pueden asignarse a personas distintas y avanzar en paralelo — el único punto de posible colisión de archivo es `pos-terminal.store.ts`, compartido por US1 (T006/T007) y US2 (T016)
- T027/T028 (Polish) en paralelo

---

## Parallel Example: User Story 1

```bash
# Lanzar juntos los tests de la Historia 1:
Task: "Crear pos-tables-panel.component.spec.ts cubriendo pestañas de tipo + filtro de ocupación"
Task: "Extender pos-terminal.store.spec.ts con casos para orderTypeTab y pendingPaymentsView"
Task: "Crear pending-payments-panel.component.spec.ts cubriendo confirmar/aprobar/rechazar pago"
```

---

## Implementation Strategy

### MVP First (Historias de Usuario 1 y 2 — ambas P1)

1. Completar Fase 1: Setup
2. Completar Fase 3: Historia 1 (pestañas de tipo + "Pagos por confirmar")
3. Completar Fase 4: Historia 2 (lista de ítems + "+ Agregar producto" embebido)
4. **DETENER Y VALIDAR**: probar ambas historias de forma independiente contra quickstart.md (escenarios 1-7, 10-11)
5. Desplegar/demostrar si está lista — la Historia 3 (P2) es una mejora de espacio de trabajo independiente, no bloquea el MVP

### Entrega incremental

1. Fase 1 completa → base lista
2. Agregar Historia 1 (pestañas + pagos por confirmar) → probar de forma independiente → demo
3. Agregar Historia 2 (menú central embebido) → probar de forma independiente → demo (MVP completo)
4. Agregar Historia 3 (toggle de sidebar) → probar de forma independiente → demo
5. Fase 6 (Polish) → validación final de quickstart.md completo y regresión de specs existentes

### Estrategia de equipo en paralelo

Con varias personas disponibles: completar la Fase 1 en conjunto; luego una persona toma la Historia 1
(`pos-tables-panel.component.ts` + `pending-payments-panel.component.ts` + porción de
`pos-terminal.store.ts`), otra la Historia 2 (`pos-catalog-drawer.component.ts` +
`pos-order-panel.component.ts` + otra porción de `pos-terminal.store.ts`), y otra la Historia 3
(`sidebar.component.ts`/`dashboard-layout.component.ts`) — coordinando únicamente los cambios que caen
en `pos-terminal.store.ts` (T006/T007 y T016) para evitar conflictos de merge en el mismo archivo.

---

## Notes

- [P] = archivos distintos, sin dependencias entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- Ejecutar T003/T004/T005/T013/T014/T015/T022 antes de su implementación correspondiente y confirmar que fallan primero
- No se agrega ninguna dependencia nueva (Principio IX)
- No hay cambios de backend ni de modelo de datos en esta spec (ver data-model.md)
- Las pestañas "Domicilios" y "Para llevar" permanecen intencionalmente vacías en toda esta feature (FR-003) — no crear ninguna tarea que intente poblarlas con datos reales
- "Dividir la cuenta entre varias personas" y "Facturar a nombre de" ya están implementados (spec 010/011) — no crear ninguna tarea que reimplemente su lógica; solo ajuste visual (T011)
