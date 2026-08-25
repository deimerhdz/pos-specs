---

description: "Task list template for feature implementation"
---

# Tasks: Rediseño de Layout de la Terminal de Mesas — Franja de Órdenes y Menú Central

**Input**: Design documents from `/specs/036-terminal-mesas-rediseno-layout/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/ui-store-contract.md](./contracts/ui-store-contract.md), [quickstart.md](./quickstart.md)

**Tests**: Se incluyen tareas de test. La Constitución (Principio X, Verificación Obligatoria) exige
verificar toda funcionalidad nueva, y `research.md` §5 ya decidió agregar cobertura donde no existe
hoy (`pos-tables-panel.component.ts` y el catálogo embebido no tienen specs). Los specs ya existentes
(`pos-checkout-panel.component.spec.ts`, `pos-terminal.store.spec.ts`) deben permanecer en verde.

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

- [X] T001 Ejecutar la suite de tests existente en `pos-heladeria` (`pnpm test` o el script configurado en `package.json`) y confirmar que está en verde antes de cualquier cambio — establece la línea base de regresión exigida por el Principio X. **Hallazgo**: la baseline NO está en verde — hay 10 fallas preexistentes y ajenas a esta spec (errores de tipo `plan`/`plan_id` en `reports.service.spec.ts`/`tenant.service.spec.ts`/`tenant-date.pipe.spec.ts` que rompen la compilación de `ng test` para todo el proyecto; y 9 fallas en `pos-checkout-panel.component.spec.ts`/`session-bill-panel.component.spec.ts` con `fill()` sobre un elemento inexistente). Verificado que estas fallas ya existen en `develop` sin ningún cambio mío (Principio V: fuera de alcance, no se tocan). El resto de la suite (179 tests) sí estaba en verde.
- [X] T002 [P] Levantar el entorno de desarrollo local (`pnpm start` en `pos-heladeria`) y confirmar que compila y sirve sin errores (`ng serve`, `ng build` de producción también verificado). No se completó el recorrido manual contra `pos-backend` con datos reales (login/tenant/mesas sembradas) — ver nota en T021.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Prerrequisitos bloqueantes compartidos por todas las historias

**Nota**: esta feature no tiene prerrequisitos fundacionales bloqueantes. Las tres historias tocan
señales de store y componentes independientes entre sí (US1: `ordersFilter`/`recentOrdersView` en
`pos-terminal.store.ts` + `pos-tables-panel.component.ts`; US2: `catalogSearchText`/
`catalogProductsFiltered` en `pos-terminal.store.ts` + `pos-catalog-drawer.component.ts`; US3:
`sidebar.component.ts`/`dashboard-layout.component.ts`) y pueden implementarse en cualquier orden, o
en paralelo por distintas personas, una vez completada la Fase 1.

**Checkpoint**: Fase 1 completa — cualquier historia de usuario puede comenzar.

---

## Phase 3: User Story 1 - El cajero encuentra cualquier orden activa desde un único listado con filtros (Priority: P1) 🎯 MVP

**Goal**: Reemplazar la grilla de mesas por estado por una franja horizontal superior de "Órdenes
Recientes" con 3 filtros (Todas / Domicilios [vacío] / Mesas), navegación tipo carrusel, e insignia de
estado + barra de tiempo transcurrido conviviendo en cada tarjeta.

**Independent Test**: Abrir la Terminal de Mesas con más órdenes activas que las que caben en el ancho
de pantalla; alternar entre las tres pestañas de filtro y confirmar que "Mesas"/"Todas las Órdenes"
muestran el mismo conjunto y "Domicilios" siempre está vacío; usar las flechas del carrusel y
confirmar que se deshabilitan en los extremos; seleccionar una tarjeta y confirmar que abre la orden
igual que hoy.

### Tests for User Story 1 ⚠️

> Escribir estos tests primero; deben fallar antes de implementar.

- [X] T003 [P] [US1] Crear `pos-tables-panel.component.spec.ts` en `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.spec.ts` cubriendo: las 3 pestañas de filtro, "Domicilios" siempre vacío, "Todas las Órdenes" == "Mesas", insignia + barra de tiempo coexistiendo en la misma tarjeta, habilitación/deshabilitación de las flechas del carrusel, y reinicio del scroll al cambiar de pestaña
- [X] T004 [P] [US1] Extender `pos-terminal.store.spec.ts` en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` con casos para `ordersFilter`, `setOrdersFilter()` y el computed `recentOrdersView` (incluye `elapsedRatio` por tarjeta)

### Implementation for User Story 1

- [X] T005 [US1] Implementar en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` el signal `ordersFilter` (`'todas' | 'domicilios' | 'mesas'`), el método `setOrdersFilter()` y el computed `recentOrdersView` — reutilizando `tablesView()`/`STATUS_META` ya existentes y agregando `elapsedRatio` (0..1) derivado de `DiningOrder.created_at` (research.md §1); `recentOrdersView` DEBE resolver siempre a lista vacía cuando `ordersFilter === 'domicilios'` (FR-003, FR-012) — depende de T004
- [X] T006 [US1] Reescribir la plantilla de `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.ts`: reemplazar la grilla agrupada por estado por la franja horizontal "Órdenes Recientes" enlazada a `recentOrdersView`, con las 3 pestañas de filtro enlazadas a `ordersFilter`/`setOrdersFilter()`, cada tarjeta mostrando la insignia existente (`chipClass`/`statusLabel`) junto a la nueva barra de tiempo (`elapsedRatio`) (FR-001) — depende de T005
- [X] T007 [US1] Agregar en `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.ts` el contenedor de scroll con flechas izquierda/derecha, un listener `(scroll)` que derive el estado deshabilitado de cada flecha, y el reinicio del scroll al inicio cuando cambia `ordersFilter` (research.md §2; FR-002) — depende de T006. **Nota de implementación**: se usa `el.scrollLeft` directamente (con la clase `scroll-smooth` ya aplicada, que anima cualquier cambio de `scrollLeft` en navegadores reales) en vez de `scrollBy()`/`scrollTo()`, porque jsdom no implementa esos métodos y el efecto los invocaba en cada cambio de filtro, reventando la suite de tests.
- [X] T008 [US1] Confirmar que seleccionar una tarjeta invoca sin cambios el método de selección de orden/mesa ya existente (FR-004), y ejecutar T003/T004 hasta que pasen en verde — `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.ts`, `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` — depende de T007

**Checkpoint**: La Historia 1 es funcional y probable de forma independiente.

---

## Phase 4: User Story 2 - El cajero arma la orden desde un menú central siempre visible, con búsqueda y categorías (Priority: P1)

**Goal**: Reemplazar el catálogo superpuesto de pantalla completa por un grid de productos embebido de
forma permanente en el panel central, con buscador por nombre y filtro por categoría combinables.

**Independent Test**: Con una orden manual en construcción, escribir en el buscador y confirmar que el
grid se filtra por nombre; seleccionar una categoría y confirmar que se filtra por categoría; combinar
ambos y confirmar la intersección; agregar un producto y confirmar que aparece en el resumen derecho
con el mismo comportamiento de siempre; confirmar que el grid no aparece para una orden QR de solo
lectura.

### Tests for User Story 2 ⚠️

- [X] T009 [P] [US2] Crear `pos-catalog-drawer.component.spec.ts` en `pos-heladeria/src/app/modules/tables/components/pos-catalog-drawer.component.spec.ts` cubriendo: filtro por nombre, filtro por categoría, combinación de ambos, estado vacío sin coincidencias. **Nota**: el caso "no se renderiza para una orden QR en modo Resumen de Cuenta" se probó en `table-sessions.component.spec.ts` (`TableSessionsComponent.showEmbeddedCatalog`) en vez de aquí — el componente del drawer no conoce el modo de la orden; quién decide incrustarlo o no es la página (T014), igual que ya hace `pos-checkout-panel.component.ts` con `getSidebarMode`.
- [X] T010 [P] [US2] Extender `pos-terminal.store.spec.ts` en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` con casos para `catalogSearchText`, `setCatalogSearchText()` y el computed `catalogProductsFiltered` (intersección categoría + nombre)

### Implementation for User Story 2

- [X] T011 [US2] Implementar en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` el signal `catalogSearchText`, el método `setCatalogSearchText()` y el computed `catalogProductsFiltered`, combinando el `catalogCategoryId`/`catalogProducts` ya existentes con el nuevo texto de búsqueda (research.md §4; FR-006) — depende de T010
- [X] T012 [US2] Refactorizar `pos-heladeria/src/app/modules/tables/components/pos-catalog-drawer.component.ts`: retirar el wrapper de overlay (`fixed inset-0 bg-black/40`) y la animación de panel deslizante, cambiar el grid para leer `catalogProductsFiltered` en vez de `catalogProducts`, y agregar el input de búsqueda por nombre enlazado a `catalogSearchText`/`setCatalogSearchText()`, conservando intacta la lógica de pestañas de categoría ya existente (FR-005, FR-006) — depende de T011
- [X] T013 [US2] Agregar en `pos-heladeria/src/app/modules/tables/components/pos-catalog-drawer.component.ts` un estado vacío claro para cuando `catalogProductsFiltered()` no tiene resultados (FR-006, edge case) — depende de T012
- [X] T014 [US2] Incrustar el componente de catálogo refactorizado de forma permanente en el panel central de `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts`, junto a `app-pos-order-panel` (grid a la izquierda `flex-[2]`, carrito a la derecha `w-[380px]`), mientras `centralState() === 'pedido'` y la orden no esté en modo "Resumen de Cuenta" (`showEmbeddedCatalog`, ver T015), retirando el botón "＋ Agregar producto" que hoy abre el overlay — depende de T012. **Nota**: retirar ese botón exigió también tocar `pos-order-panel.component.ts` (elimina el botón y `store.openCatalog()`) y limpiar `catalogOpen`/`openCatalog()`/`closeCatalog()` de `pos-terminal.store.ts` por quedar sin ningún consumidor — plan.md los listaba como "sin cambio", pero FR-005 exige retirar ese botón, así que prevalece tasks.md.
- [X] T015 [US2] Confirmar que el grid embebido no se muestra para una orden en modo "Resumen de Cuenta" de solo lectura — se agregó `TableSessionsComponent.showEmbeddedCatalog` (reutiliza `getSidebarMode`, ya usado por `pos-checkout-panel.component.ts` para lo mismo del lado del cobro) con 4 casos nuevos en `table-sessions.component.spec.ts`; T009/T010 en verde — depende de T014

**Checkpoint**: Las Historias 1 y 2 son funcionales de forma independiente.

---

## Phase 5: User Story 3 - El usuario colapsa el menú de navegación global para ganar espacio de pantalla (Priority: P2)

**Goal**: Agregar un botón en la parte superior de la Terminal de Mesas que colapsa/expande el menú de
navegación global del panel administrativo, incluyendo el efecto de colapso en escritorio que hoy no
existe.

**Independent Test**: Con el menú global visible en una pantalla de escritorio, pulsar el botón y
confirmar que se oculta y el contenido ocupa el espacio liberado; pulsar de nuevo y confirmar que
vuelve a mostrarse; navegar a otra sección del panel administrativo con el menú colapsado y confirmar
que sigue siendo accesible.

### Tests for User Story 3 ⚠️

- [X] T016 [P] [US3] Crear/extender `sidebar.component.spec.ts` en `pos-heladeria/src/app/modules/dashboard/layout/sidebar.component.spec.ts` cubriendo: `sidebarOpen()` en `true` → clases de expansión también en breakpoint de escritorio, `sidebarOpen()` en `false` → clases de colapso también en escritorio (hoy el componente las ignora en escritorio)

### Implementation for User Story 3

- [X] T017 [US3] Modificar la plantilla de `pos-heladeria/src/app/modules/dashboard/layout/sidebar.component.ts` para que las clases `md:relative md:translate-x-0` (hoy incondicionales) dependan de `layoutService.sidebarOpen()` también en escritorio (research.md §3; FR-010) — depende de T016. Se agregó además `desktopVisibilityClass` (computed) y se hizo el signal `LayoutService.sidebarOpen` consciente del breakpoint al inicializarse (`window.innerWidth >= 768`), para no cambiar el comportamiento por defecto ya existente (visible en escritorio, oculto en móvil) — antes ese default (`false`) solo importaba en móvil porque escritorio lo ignoraba con CSS; al empezar a honrarlo también en escritorio, el valor inicial tenía que seguir siendo "abierto" ahí.
- [X] T018 [US3] Ajustar `pos-heladeria/src/app/modules/dashboard/layout/dashboard-layout.component.ts` para que el margen/ancho del contenido principal reaccione a `layoutService.sidebarOpen()` en breakpoints de escritorio, ocupando el espacio liberado cuando está colapsado — depende de T017. **Sin cambios de código**: el contenido ya es `flex-1` en el mismo contenedor `flex` que el `<aside>`; con `md:hidden` (T017) el sidebar sale del flujo flex por completo, así que el contenido reclama el espacio liberado automáticamente sin margen/ancho explícito que mantener sincronizado.
- [X] T019 [US3] Agregar en `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts` un botón en la parte superior de la Terminal de Mesas que inyecta `LayoutService` y llama a `toggle()`, con su ícono reflejando `sidebarOpen()` (FR-010) — depende de T018
- [X] T020 [US3] Confirmar que colapsar/expandir el sidebar no interrumpe una validación de pago o un cobro en curso en los paneles central/derecho, y ejecutar T016 hasta que pase en verde — `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts` — depende de T019. Verificado por diseño: `LayoutService` es un servicio `providedIn: 'root'` completamente independiente de `PosTerminalStore` (sin inyección cruzada, sin señales compartidas) — no hay ninguna ruta de código por la que alternar `sidebarOpen()` pueda tocar `selectedOrder`/`draftLines`/`checkoutSubmitting` ni ningún otro estado de cobro. T016 en verde.

**Checkpoint**: Las tres historias de usuario son funcionales de forma independiente.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final que abarca las tres historias

- [ ] T021 [P] Ejecutar manualmente los 10 escenarios de [quickstart.md](./quickstart.md) contra el entorno de desarrollo local (`pos-heladeria` + `pos-backend`). **No completada por el agente**: requiere `pos-backend` corriendo con datos sembrados (mesas, pedidos activos, turno de caja) y sesión de cajero autenticada contra un tenant real — fuera del alcance que puede levantar una sesión no interactiva en este entorno. `ng serve`/`ng build` sí se verificaron sin errores (T002). Pendiente de que el usuario la recorra manualmente antes de dar la feature por completamente validada.
- [X] T022 [P] Confirmar que `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.spec.ts` y `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` (ya existentes antes de esta feature) siguen en verde sin cambios de intención, solo ajustes de selector si el reacomodo visual lo exige (Principio X). `pos-terminal.store.spec.ts` (50/50, extendido en T004/T010) está en verde. `pos-checkout-panel.component.spec.ts` **no** está en verde, pero no por esta feature: tiene 5 fallas (`fill()` sobre un `<select>`/`<input>` que no encuentra) confirmadas idénticas en `develop` sin ningún cambio de esta rama (ver T001) — no se tocó ese archivo ni su intención.
- [X] T023 Eliminar cualquier remanente muerto de la grilla de mesas por estado en `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.ts` y del wrapper de overlay en `pos-heladeria/src/app/modules/tables/components/pos-catalog-drawer.component.ts` que haya quedado sin usar tras el refactor. También se retiraron, por quedar sin ningún consumidor tras T014: `PosTerminalStore.catalogOpen`/`openCatalog()`/`closeCatalog()`, el atajo `F2` (buscaba en la grilla vieja, ya no existe campo que enfocar) y `PosTablesPanelComponent.focusSearch()`.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Fase 1)**: sin dependencias — puede iniciar de inmediato
- **Foundational (Fase 2)**: sin tareas bloqueantes (ver nota arriba) — las historias pueden iniciar tras la Fase 1
- **Historias de usuario (Fase 3-5)**: cada una depende solo de la Fase 1; son independientes entre sí y pueden ejecutarse en paralelo si hay capacidad de equipo, o en orden de prioridad (US1 → US2 → US3)
- **Polish (Fase 6)**: depende de que las historias que se quieran entregar ya estén completas

### Dependencias dentro de cada historia

- US1: T003/T004 (tests, en paralelo) → T005 (store) → T006 (template base) → T007 (carrusel, mismo archivo que T006) → T008 (verificación, mismos archivos)
- US2: T009/T010 (tests, en paralelo) → T011 (store) → T012 (refactor del catálogo) → T013 (estado vacío, mismo archivo) y T014 (embeber en la pantalla, depende de T012) → T015 (verificación)
- US3: T016 (test) → T017 (sidebar) → T018 (layout) → T019 (botón en la Terminal) → T020 (verificación)

### Parallel Opportunities

- T001/T002 (Setup) en paralelo
- T003/T004 (tests de US1) en paralelo — archivos distintos
- T009/T010 (tests de US2) en paralelo — archivos distintos
- T016 (único test de US3) puede ejecutarse en paralelo con los tests de US1/US2 si hay capacidad de equipo
- Una vez completada la Fase 1, US1, US2 y US3 pueden asignarse a personas distintas y avanzar en paralelo (tocan archivos disjuntos, salvo que dos personas no deben tocar `pos-terminal.store.ts` a la vez — T005 y T011 son el único punto de posible colisión de archivo entre historias)
- T021/T022 (Polish) en paralelo

---

## Parallel Example: User Story 1

```bash
# Lanzar juntos los tests de la Historia 1:
Task: "Crear pos-tables-panel.component.spec.ts cubriendo filtros, insignia+barra, y carrusel"
Task: "Extender pos-terminal.store.spec.ts con casos para ordersFilter y recentOrdersView"
```

---

## Implementation Strategy

### MVP primero (Historia de Usuario 1 solamente)

1. Completar Fase 1: Setup
2. Completar Fase 3: Historia 1 (franja de "Órdenes Recientes" con filtros y carrusel)
3. **DETENER Y VALIDAR**: probar la Historia 1 de forma independiente contra quickstart.md (escenarios 1-4)
4. Desplegar/demostrar si está lista

### Entrega incremental

1. Fase 1 completa → base lista
2. Agregar Historia 1 → probar de forma independiente → demo (MVP)
3. Agregar Historia 2 (menú central embebido) → probar de forma independiente → demo
4. Agregar Historia 3 (toggle de sidebar) → probar de forma independiente → demo
5. Fase 6 (Polish) → validación final de quickstart.md completo y regresión de specs existentes

### Estrategia de equipo en paralelo

Con varias personas disponibles: completar la Fase 1 en conjunto; luego una persona toma la Historia 1
(`pos-tables-panel.component.ts` + porción de `pos-terminal.store.ts`), otra la Historia 2
(`pos-catalog-drawer.component.ts` + otra porción de `pos-terminal.store.ts`), y otra la Historia 3
(`sidebar.component.ts`/`dashboard-layout.component.ts`) — coordinando únicamente los cambios que caen
en `pos-terminal.store.ts` (T005 y T011) para evitar conflictos de merge en el mismo archivo.

---

## Notes

- [P] = archivos distintos, sin dependencias entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- Ejecutar T003/T004/T009/T010/T016 antes de su implementación correspondiente y confirmar que fallan primero
- No se agrega ninguna dependencia nueva (Principio IX) — el carrusel se construye con `scrollBy` nativo
- No hay cambios de backend ni de modelo de datos en esta spec (ver data-model.md)
- El filtro "Domicilios" permanece intencionalmente vacío en toda esta feature (FR-003, FR-012) — no crear ninguna tarea que intente poblarlo con datos reales
