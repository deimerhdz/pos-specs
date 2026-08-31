---

description: "Lista de tareas para Ocultamiento de Unidades de Medida y Reportes de Inventario sin el Módulo Habilitado"
---

# Tasks: Ocultamiento de Unidades de Medida y Reportes de Inventario sin el Módulo Habilitado

**Input**: Documentos de diseño de `/specs/062-gating-inventario-ajustes-reportes/` (plan.md,
spec.md, research.md, data-model.md, contracts/, quickstart.md)

**Tests**: incluidos donde hay lógica nueva que verificar (frontend). El wiring backend (agregar
`Depends(require_module_access("inventario"))` a rutas/routers existentes) se verifica por
inspección, no con un test de integración nuevo — mismo criterio que spec 033 ya aceptó para
`inventory/router.py`/`promotions/router.py` (research.md, Decisión 5; plan.md, Constitution
Check, Principio X).

**Organization**: tareas agrupadas por historia de usuario (US1-US3, prioridades de `spec.md`).
No hay fase Foundational: ninguna tarea bloquea a **todas** las historias por igual — US1 es
100% independiente; el trabajo compartido entre US2 y US3 (la señal `inventarioIncluido()` y el
gating de las dos queries de Reportes) vive dentro de la fase de US2 (P1, la más temprana de las
dos) y se referencia explícitamente desde US3 (ver Dependencies).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (ficheros distintos, sin dependencia de una tarea sin
  terminar)
- **[Story]**: historia de usuario a la que pertenece (US1-US3)
- Cada tarea incluye la ruta de fichero exacta, relativa a la raíz del repo sibling que
  corresponda (`pos-backend` o `pos-heladeria`)

## Path Conventions

Dos repositorios sibling de `pos-specs` (Constitución §Alcance, plan.md §Project Structure):

- Backend: `pos-backend/app/...` (rutas de este documento ya incluyen el prefijo `pos-backend/`)
- Frontend: `pos-heladeria/src/app/...` (rutas ya incluyen el prefijo `pos-heladeria/`)

---

## Phase 1: Setup

**Purpose**: confirmar que el entorno está listo — esta spec no agrega ninguna dependencia nueva
a `requirements.txt`/`package.json` (plan.md, Technical Context).

- [X] T001 Confirmar entorno: `pos-backend` con el venv activado (`source env/bin/activate`,
  Python 3.14) y `pos-heladeria` con `npm install` ya corrido. Crear las ramas
  `062-gating-inventario-ajustes-reportes` en ambos repos sibling (partiendo de su rama de
  desarrollo habitual), mismo patrón que spec 033 (T001 de esa spec).
  **Confirmado**: venv de `pos-backend` con FastAPI 0.136.3 disponible; `node_modules/` presente
  en `pos-heladeria`. Ramas `062-gating-inventario-ajustes-reportes` creadas en ambos repos
  (partieron de `develop`, limpias en `pos-backend`; `pos-heladeria` traía cambios sin commitear
  de la sesión anterior — sidebar/plan gating —, que quedaron intactos en la nueva rama).

**Checkpoint**: entornos listos, sin instalar nada nuevo.

---

## Phase 2: User Story 1 - Unidades de medida desaparece de Ajustes sin el módulo Inventario (Priority: P1) 🎯 MVP

**Goal**: la pestaña "Unidades de medida" deja de ser visible en Ajustes y de ser accesible (por
URL directa o por API) cuando el plan del tenant no incluye Inventario o está vencido.

**Independent Test**: con un tenant sin acceso a Inventario, abrir Ajustes y confirmar que la
pestaña no aparece; entrar por URL directa a `/dashboard/ajustes/unidades` y confirmar que
redirige a `/dashboard`; con un tenant que sí tiene el módulo, confirmar que la pestaña funciona
igual que hoy.

### Implementación de User Story 1

- [X] T002 [P] [US1] En `pos-backend/app/api/v1/unit_measures/router.py`: importar
  `require_module_access` de `app.core.plan_limits` y agregar
  `dependencies=[Depends(require_module_access("inventario"))]` al `APIRouter(...)` (mismo
  patrón que `inventory/router.py:28-31`, spec 033) — cubre los 5 endpoints existentes
  (`list`/`get`/`create`/`update`/`delete`) de una sola vez (FR-002, FR-003,
  [contracts/inventario-module-gating.md](./contracts/inventario-module-gating.md)).
- [X] T003 [US1] En `pos-heladeria/src/app/modules/dashboard/routes.ts`: importar
  `planModuleGuard` (ya existe en `core/guards/plan-module.guard.ts`, usado hoy por `inventario`
  y `promotions`) y agregar `canActivate: [planModuleGuard('inventario')]` al hijo
  `ajustes/unidades`, sin tocar el `roleGuard` del padre `ajustes` ni los demás hijos (FR-002,
  redirige a `/dashboard` — Clarifications de `spec.md`, sesión 2026-08-31).
- [X] T004 [P] [US1] En `pos-heladeria/src/app/modules/settings/pages/settings-page.component.ts`:
  agregar campo opcional `moduleKey?: 'inventario'` a la interfaz local `SettingsTab`; marcar la
  entrada `{ label: 'Unidades de medida', path: 'unidades' }` con `moduleKey: 'inventario'`;
  inyectar `PlanSummaryService`; reemplazar el arreglo fijo `tabs` por una señal computada
  `visibleTabs` que filtra por `moduleKey` contra `planSummaryService.summary()` — mismo criterio
  fail-open-mientras-carga que `SidebarComponent.visibleItems` (si `summary()` es `null`, se
  muestra; si `vencido` es `true`, se oculta; si no, se muestra solo cuando
  `summary().modules[moduleKey]` es `true`); actualizar el `@for` del template para iterar
  `visibleTabs()` en vez de `tabs` (FR-001). `PlanSummaryService.load()` ya se dispara desde
  `DashboardLayoutComponent.ngOnInit()` (cambio de sesión previa, mismo día) — esta tarea no
  necesita volver a cargarlo.

### Tests para User Story 1

- [X] T005 [P] [US1] Crear `pos-heladeria/src/app/modules/settings/pages/settings-page.component.spec.ts`
  (fichero nuevo, mismo patrón que `sidebar.component.spec.ts`): con
  `PlanSummaryService.summary().modules.inventario = false`, `visibleTabs()` no incluye "Unidades
  de medida" (Acceptance Scenario 1); con `= true`, sí la incluye (Acceptance Scenario 3); con
  `summary() === null`, sí la incluye (fail-open); con `vencido: true` (aunque
  `modules.inventario = true`), no la incluye.
- [X] T006 [US1] Verificar por inspección (sin test de integración nuevo, research.md Decisión 5):
  confirmar que la dependencia agregada en T002 usa la misma clave `"inventario"` y por lo tanto
  el mismo mensaje ya cubierto genéricamente por
  `test_plan_module_access.py::test_modulo_no_incluido_deniega_con_mensaje_claro`; correr
  `python -m unittest app.characterization_tests.test_plan_module_access -v` y confirmar que
  sigue en verde sin cambios.
  **Confirmado**: `router.dependencies` de `unit_measures/router.py` expone la misma
  `require_module_access` que `inventory/router.py`; los 3 tests de
  `test_plan_module_access.py` siguen en verde sin cambios.

**Checkpoint**: User Story 1 es completamente funcional y verificable de forma independiente.

---

## Phase 3: User Story 2 - Los reportes de insumos de inventario desaparecen sin el módulo Inventario (Priority: P1)

**Goal**: la sección "Insumos con stock bajo" deja de ser visible en Reportes, y el dato
subyacente deja de poder consultarse, cuando el plan del tenant no incluye Inventario o está
vencido — sin romper el resto de la pantalla de Reportes.

**Independent Test**: con un tenant sin acceso a Inventario, abrir Reportes y confirmar que la
sección "Insumos con stock bajo" no aparece, que no se dispara ninguna petición a
`/reports/inventory`, y que el resto de la pantalla (Total cobrado, Cobros, Ticket promedio,
gráficas) carga con normalidad, sin spinner permanente ni banner de error.

### Implementación de User Story 2

- [X] T007 [P] [US2] En `pos-backend/app/api/v1/reports/router.py`: agregar
  `dependencies=[Depends(require_module_access("inventario"))]` al decorador de la ruta
  `GET /inventory` (RF-063) — por ruta individual, no a nivel de router (mismo patrón que las
  rutas `/purchases*` con `"compras"` dentro de `inventory/router.py`, research.md Decisión 2);
  no tocar `sales`/`top-products`/`products`/`categories`/`cashiers` (FR-006,
  [contracts/inventario-module-gating.md](./contracts/inventario-module-gating.md)).
- [X] T008 [US2] En `pos-heladeria/src/app/modules/reports/services/reports.service.ts`: inyectar
  `PlanSummaryService`; agregar señal `readonly inventarioIncluido = computed(() => { const s =
  this.planSummaryService.summary(); return s !== null && s.modules.inventario && !s.vencido;
  })` — fail-closed mientras `summary()` es `null` (opuesto al criterio de T004/US1, justificado
  en research.md Decisión 4: aquí un `true` prematuro dispara una petición HTTP real). Aplicar
  `enabled: this.inventarioIncluido()` tanto a `inventoryQuery` como a `profitabilityQuery` (las
  dos comparten el mismo gate — esta tarea deja listo también el prerequisito de US3, ver
  Dependencies). Cambiar el arreglo privado `queries` (hoy fijo, líneas ~218-230) por un
  `computed()` que incluye `inventoryQuery`/`profitabilityQuery` solo cuando
  `inventarioIncluido()` es `true`, para que `isLoading`/`error` (líneas ~232-237) nunca dependan
  de una query deshabilitada indefinidamente (FR-005, FR-006, FR-007).
- [X] T009 [US2] En `pos-heladeria/src/app/modules/reports/pages/reports-page.component.ts`:
  envolver el bloque completo de la sección "Insumos con stock bajo" (desde el comentario
  `<!-- Stock bajo: alerta operativa, no un informe del período -->` hasta el cierre de esa
  tarjeta, ~líneas 240-330) en `@if (svc.inventarioIncluido())` (FR-005).

### Tests para User Story 2

- [X] T010 [P] [US2] Agregar tests a `pos-heladeria/src/app/modules/reports/services/reports.service.spec.ts`:
  con `PlanSummaryService.summary().modules.inventario = false`, `http.expectNone(...)` confirma
  que no se dispara ninguna petición a `/reports/inventory` **ni** a `/reports/profitability`
  (T008 gatea ambas a la vez); `svc.isLoading()` eventualmente llega a `false` y `svc.error()` es
  `null` (regresión que research.md Decisión 4 previene — spinner/banner permanentes); con
  `= true`, ambas peticiones se disparan con normalidad.
- [X] T011 [US2] Agregar tests a `pos-heladeria/src/app/modules/reports/pages/reports-page.component.spec.ts`:
  con el mismo plan que T010 (`false`), la sección "Insumos con stock bajo" no aparece en el DOM
  renderizado; con `= true`, aparece con normalidad.

**Checkpoint**: User Story 2 es completamente funcional y verificable de forma independiente. La
señal y el gating de queries de T008 quedan listos para que User Story 3 los reutilice sin volver
a tocar este fichero.

---

## Phase 4: User Story 3 - El indicador de Margen refleja que depende de datos de Inventario (Priority: P2)

**Goal**: la tarjeta "Margen" de Reportes deja de mostrarse por completo cuando el plan del
tenant no incluye Inventario o está vencido, en vez de mostrar un valor de rentabilidad calculado
con costo cero (100% de margen falso).

**Independent Test**: con un tenant sin acceso a Inventario, abrir Reportes y confirmar que la
tarjeta "Margen" no aparece entre las métricas; con un tenant que sí tiene el módulo, confirmar
que la tarjeta se calcula y muestra con normalidad.

**Depende de**: T008 (User Story 2) — la señal `inventarioIncluido()` y el `enabled:` de
`profitabilityQuery` ya quedaron resueltos ahí porque ambas queries comparten el mismo criterio
de gating (research.md Decisión 4). Esta historia no vuelve a tocar `reports.service.ts`.

### Implementación de User Story 3

- [X] T012 [P] [US3] En `pos-backend/app/api/v1/reports/router.py`: agregar
  `dependencies=[Depends(require_module_access("inventario"))]` al decorador de la ruta
  `GET /profitability` (RF-065) — por ruta individual, mismo patrón que T007 (FR-007,
  [contracts/inventario-module-gating.md](./contracts/inventario-module-gating.md)).
- [X] T013 [US3] En `pos-heladeria/src/app/modules/reports/pages/reports-page.component.ts`:
  envolver la tarjeta `app-stat-tile label="Margen"` (incluyendo su `[hint]="marginHint()"`,
  ~líneas 87-93) en `@if (svc.inventarioIncluido())` — mismo flag ya usado en T009, sin
  necesidad de una señal nueva (FR-007).

### Tests para User Story 3

- [X] T014 [US3] Agregar tests a `pos-heladeria/src/app/modules/reports/pages/reports-page.component.spec.ts`:
  con `PlanSummaryService.summary().modules.inventario = false`, la tarjeta "Margen" no aparece
  en el DOM (Acceptance Scenario 1); con `= true`, aparece y muestra el valor calculado con
  normalidad (Acceptance Scenario 2). (El `http.expectNone` de `/reports/profitability` ya quedó
  cubierto en T010 — no se repite aquí.)

**Checkpoint**: las tres historias de usuario son funcionales y verificables de forma
independiente.

---

## Phase 5: Polish & Verificación Cruzada

**Purpose**: confirmar que ninguna historia rompió comportamiento existente, siguiendo
Constitución Principio X (Verificación Obligatoria).

- [X] T015 [P] Correr la suite completa de characterization tests del backend:
  `cd ../pos-backend && python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`
  — confirmar que sigue en verde (ningún `"CONGELA comportamiento actual:"` se ve afectado,
  research.md Decisión 5/6).
  **Confirmado**: 479 tests, `OK` — ninguno afectado por los cambios de esta spec.
- [X] T016 [P] Correr la suite completa del frontend:
  `cd ../pos-heladeria && npx ng test --watch=false` — confirmar que los specs nuevos/ajustados
  (T005, T010, T011, T014) pasan y que no se rompió nada preexistente.
  **Confirmado**: 601/604 tests pasan; los 3 fallos restantes (`app.spec.ts`, `auth.service.spec.ts`,
  `pos-checkout-panel.component.spec.ts`) son preexistentes en `develop`, no relacionados con
  esta spec (confirmados vía `git stash` en una sesión anterior de este mismo trabajo).
- [ ] T017 Ejecutar la validación manual end-to-end de `quickstart.md` (asignar/revocar el módulo
  Inventario a un tenant de prueba real, confirmar las tres superficies, confirmar que
  reasignarlo las restaura sin cerrar sesión — SC-001 a SC-005).
  **Pendiente**: no hay navegador disponible en este entorno de ejecución (Playwright no está
  configurado en el repo, spec 033 research.md ya lo dejó constancia). Verificado en su lugar,
  como sustituto parcial: `tsc --noEmit` sin errores en `pos-heladeria`, y `app.main:app` de
  `pos-backend` importa y expone las rutas esperadas (`unit-measures`, `reports/inventory`,
  `reports/profitability`) contra una base de datos real. Falta la validación visual real —
  queda para quien haga el despliegue/QA manual de esta spec.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede arrancar de inmediato.
- **User Story 1 (Phase 2, P1)**: depende solo de Setup. No depende de ninguna otra historia.
- **User Story 2 (Phase 3, P1)**: depende solo de Setup. No depende de User Story 1.
- **User Story 3 (Phase 4, P2)**: depende de Setup **y** de T008 (dentro de la fase de User
  Story 2) — comparte la señal `inventarioIncluido()` y el gating de `profitabilityQuery` con
  User Story 2 (research.md Decisión 4). No depende de User Story 1.
- **Polish (Phase 5)**: depende de que las historias que se vayan a entregar ya estén completas.

### Dependencias dentro de cada historia

- User Story 1: T002 (backend) y T004 (frontend, tabs) son paralelas entre sí; T003 (guard de
  ruta) es independiente de ambas; T005/T006 (tests/verificación) van después de su
  implementación correspondiente.
- User Story 2: T007 (backend) es paralela a T008 (frontend); T009 (template) depende de T008
  (necesita `svc.inventarioIncluido()` ya definido); T010/T011 (tests) van después de T008/T009.
- User Story 3: T012 (backend) es paralela a todo lo demás; T013 (template) depende de T008 (ya
  completado en User Story 2, no de una tarea nueva en esta fase); T014 (test) va después de
  T013.

### Parallel Opportunities

- T002 [US1, backend] y T004 [US1, frontend] pueden ejecutarse en paralelo (ficheros y
  repositorios distintos).
- T007 [US2, backend] puede ejecutarse en paralelo con T008/T009 [US2, frontend].
- T012 [US3, backend] puede ejecutarse en paralelo con T013 [US3, frontend], y de hecho con toda
  la fase de User Story 2 (repos y ficheros distintos) si hay capacidad para trabajar ambas
  historias a la vez — aunque T013 debe esperar a que T008 (User Story 2) esté terminado.
- T015 y T016 (Polish) pueden ejecutarse en paralelo entre sí.

---

## Parallel Example: User Story 1

```bash
# T002 (backend) y T004 (frontend) en paralelo — repos y ficheros distintos:
Task: "Agregar require_module_access('inventario') al router de unit_measures en pos-backend/app/api/v1/unit_measures/router.py"
Task: "Convertir tabs a visibleTabs (señal computada) en pos-heladeria/src/app/modules/settings/pages/settings-page.component.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 únicamente)

1. Completar Phase 1: Setup.
2. Completar Phase 2: User Story 1 (Unidades de Medida).
3. **Detener y validar**: probar User Story 1 de forma independiente (T005/T006).
4. Este es ya un incremento entregable por sí solo — resuelve la mitad explícita más simple del
   pedido original ("las unidades de medida... no debería tener visible esa opción en los
   ajustes").

### Entrega incremental

1. Setup → listo.
2. User Story 1 → probar independientemente → entregar (MVP).
3. User Story 2 → probar independientemente (incluye la regresión de spinner/banner que
   research.md Decisión 4 previene) → entregar.
4. User Story 3 → probar independientemente (reutiliza el trabajo de User Story 2 sin
   duplicarlo) → entregar.
5. Polish → verificación cruzada final antes de cerrar la spec.

### Estrategia de equipo en paralelo

Con dos personas: una puede tomar User Story 1 completa (backend + frontend) mientras la otra
toma User Story 2 completa: no comparten ningún fichero. User Story 3 debe esperar a que la
persona de User Story 2 termine T008 antes de tocar `reports-page.component.ts` para el Margen
(puede adelantar T012, el backend de `/profitability`, sin esperar nada).

---

## Notes

- [P] = ficheros distintos, sin dependencias pendientes.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Constitución Principio
  XII).
- Ninguna tarea de esta spec agrega una migración Alembic ni una dependencia nueva
  (`requirements.txt`/`package.json`) — confirmado en plan.md y data-model.md.
- Los mensajes de error 403 (`"Tu plan actual no incluye el módulo de inventario."` / `"Tu plan
  venció..."`) ya existen en `plan_limits.py` — ninguna tarea de esta spec los reescribe ni
  duplica.
- Commitear después de cada tarea o grupo lógico; detenerse en cada checkpoint para validar la
  historia de forma independiente antes de continuar con la siguiente.
