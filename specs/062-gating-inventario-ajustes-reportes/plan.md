# Implementation Plan: Ocultamiento de Unidades de Medida y Reportes de Inventario sin el Módulo Habilitado

**Branch**: `062-gating-inventario-ajustes-reportes` | **Date**: 2026-08-31 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/062-gating-inventario-ajustes-reportes/spec.md`

## Summary

Esta spec extiende el gating de módulo por plan de la spec 033 —hoy aplicado a las pantallas
de nivel superior Inventario y Promociones (`app/core/plan_limits.py::require_module_access`,
`core/guards/plan-module.guard.ts`)— a tres superficies que ese trabajo no cubrió: la pestaña
"Unidades de medida" dentro de Ajustes, la sección "Insumos con stock bajo" de Reportes, y la
tarjeta "Margen" de Reportes (esta última, por decisión explícita tras el análisis de la spec,
se oculta por completo en vez de mostrar un valor calculado sin datos de inventario).

No se agrega ninguna entidad, columna ni endpoint nuevo: las tres superficies reutilizan
exactamente `require_module_access("inventario")` (backend) y `PlanSummaryService`/
`planModuleGuard('inventario')` (frontend), ya existentes. El trabajo es 100% wiring —conectar
mecanismos ya probados a rutas y componentes que hoy no los usan— con una excepción de diseño
no trivial: `ReportsService` hoy agrega el estado de `isLoading`/`error` de sus seis queries de
forma incondicional (`reports.service.ts:216-237`); si el backend empieza a devolver 403 en
`/reports/inventory` y `/reports/profitability` para un tenant sin Inventario, sin tocar el
frontend, esas dos queries entrarían en error permanentemente y el error se filtraría a **todo**
el panel de Reportes (o, si en cambio se deshabilitan sin más, quedarían `isPending` para
siempre y el panel completo no saldría nunca del estado de carga). La Fase 0 (research.md,
Decisión 4) documenta la solución: una señal `inventarioIncluido()` que decide a la vez si esas
dos queries se habilitan y si participan en el agregado de `isLoading`/`error`.

## Technical Context

**Language/Version**: Backend — Python 3.14.4 (venv `pos-backend/env`). Frontend — TypeScript
5.9.2 (Angular 21.1.x, standalone components + signals).

**Primary Dependencies**: Ninguna dependencia nueva en ningún repositorio.
- Backend: reutiliza `app.core.plan_limits.require_module_access` (FastAPI `Depends`, spec 033)
  tal cual existe hoy — no se le agrega ningún parámetro ni variante.
- Frontend: reutiliza `PlanSummaryService` (`modules/plan/services/plan-summary.service.ts`) y
  `planModuleGuard` (`core/guards/plan-module.guard.ts`), y `@tanstack/angular-query-experimental`
  (ya en uso en `reports.service.ts`) para el `enabled:` reactivo de las dos queries afectadas.

**Storage**: PostgreSQL 16, schema-per-tenant. **Sin migración**: no se agrega ninguna tabla ni
columna. `UnitMeasure`, `InventoryItem`, `RecipeItem` y el cálculo de `cogs`/`margin` en
`reports/service.py` no cambian de forma; solo ganan una comprobación de acceso antes de
ejecutarse.

**Testing**: Backend — `unittest` vía `python -m unittest` (sin pytest en el repo), mismo patrón
que `app/characterization_tests/test_plan_module_access.py` (spec 033). El comportamiento de
`require_module_access` en sí ya está cubierto genéricamente por ese archivo y no cambia — esta
spec no agrega un test nuevo de esa función, solo confirma (research.md Decisión 5) que su
wiring en los routers/rutas afectados sigue el mismo patrón no cubierto por test dedicado que ya
aceptaron `inventory/router.py`/`promotions/router.py` en spec 033. Frontend — Vitest vía
`ng test` (`@angular/build:unit-test`), specs colocados (`*.spec.ts`), mismo patrón
`TestBed` + `HttpClientTesting` + `QueryClient` de prueba que `reports.service.spec.ts`.

**Target Platform**: Linux server (API `pos-backend`) + navegador (SPA `pos-heladeria`).

**Project Type**: Web application (backend FastAPI + frontend Angular, repos sibling de
`pos-specs`).

**Performance Goals**: Ninguno nuevo. `require_module_access` ya es un `SELECT` de una fila por
PK (`Plan` por `tenant.plan_id`) sin lock — el mismo costo que ya paga cada request a
Inventario/Promociones, ahora también pagado por `unit-measures/*` y por dos de los seis
`GET /reports/*`.

**Constraints**:
- `ReportsService` no debe disparar `GET /reports/inventory` ni `GET /reports/profitability`
  para un tenant sin el módulo Inventario (o vencido) — dispararlas y descartar el resultado en
  el template no basta, porque ambas alimentan el agregado compartido `isLoading`/`error` que
  gobierna el spinner y el banner de error de **todo** Reportes, no solo de sus propias tarjetas
  (research.md Decisión 4).
- La pestaña "Unidades de medida" es un hijo anidado dentro de la ruta `ajustes` (que ya aplica
  `roleGuard([UserRole.ADMIN])` a nivel de padre); el guard de módulo se agrega solo a ese hijo,
  sin tocar el guard del padre ni el de las demás pestañas de Ajustes.
- Ningún dato existente se pierde ni se recalcula: ocultar/denegar una superficie no borra
  unidades de medida, insumos ni ventas históricas — mismo criterio que spec 033 (FR-004 de esta
  spec).
- El mensaje de denegación backend (`"Tu plan actual no incluye el módulo de inventario."` /
  `"Tu plan venció. Debe renovarse para seguir usando el sistema."`, ambos ya definidos en
  `plan_limits.py`) no cambia ni se duplica — las tres superficies nuevas devuelven exactamente
  esos mismos textos, no una variante propia.

**Scale/Scope**: 0 tablas/columnas nuevas, 0 endpoints nuevos. 1 router backend gana un
`dependencies=[...]` a nivel de router (`unit_measures/router.py`, 5 endpoints), 2 rutas backend
dentro de un router ya gateado parcialmente ganan la misma dependencia a nivel de ruta
(`reports/router.py::inventory`, `::profitability`). 1 ruta frontend gana `canActivate` nuevo
(`dashboard/routes.ts::ajustes/unidades`), 1 componente pasa de lista estática a señal filtrada
(`settings-page.component.ts`), 1 servicio gana una señal de gating que además reconfigura su
agregado de estado (`reports.service.ts`), 1 componente oculta dos bloques de template
(`reports-page.component.ts`).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 3 historias priorizadas, 8 FRs, y una sesión de clarificación (2026-08-31) que resolvió el único punto ambiguo (destino de la redirección de Unidades de Medida). | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El comportamiento que cambia (Unidades de Medida, insumos con stock bajo y Margen dejan de ser accesibles sin el módulo Inventario) es exactamente el comportamiento nuevo que `spec.md` define, no la corrección de una anomalía heredada — no requiere entrada en `registro-de-anomalias.md` (mismo razonamiento que spec 033 aplicó a Inventario/Promociones). Para un tenant **con** el módulo Inventario incluido (el caso de todos los tenants reales hoy, research.md Decisión 6), las tres superficies siguen funcionando exactamente igual que hoy — ningún camino de éxito cambia. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | Ningún test `"CONGELA comportamiento actual:"` existente asume acceso sin restricción a `unit-measures`, `/reports/inventory` o `/reports/profitability` (research.md Decisión 5 confirma la búsqueda) — ninguno se modifica. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El comportamiento nuevo (ocultamiento/denegación de las tres superficies) está definido en `spec.md`; el criterio de éxito es conformidad con la spec, no equivalencia con el pasado. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | El cambio de `ReportsService.queries` (de arreglo fijo a arreglo derivado de `inventarioIncluido()`) no es una refactorización ajena: es el mecanismo mínimo necesario para que FR-005/FR-006/FR-007 no rompan el resto de Reportes (research.md Decisión 4) — no se toca ninguna otra query, servicio o componente fuera de las tres superficies de la spec. | PASS |
| **VI. Evolución Incremental** | Las tres superficies son independientemente verificables, igual que sus historias de usuario: Unidades de Medida (US1, backend + ruta) no depende de Reportes (US2/US3, un único servicio compartido) para funcionar ni para probarse. | PASS |
| **VII. Compatibilidad con Datos Históricos** | Ninguna `Sale`/`Payment`/`Invoice` se toca; el cálculo de `profitability_report` no cambia de fórmula, solo gana una comprobación de acceso previa. Los reportes de períodos pasados para un tenant **con** acceso siguen mostrando los mismos números que hoy. | PASS |
| **VIII. Evolución del Modelo de Datos** | No aplica — no hay modelo de datos nuevo ni modificado (ver data-model.md). | N/A |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No aplica — cero dependencias nuevas en `requirements.txt`/`package.json`. | N/A |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a pasos ejecutables. La verificación del wiring backend sigue el mismo criterio ya aceptado en spec 033 (inspección de la dependencia declarada en el router/ruta, sin test de integración HTTP dedicado); la verificación del frontend sí agrega tests nuevos donde spec 033 no dejó precedente reusable (`settings-page.component.spec.ts`, ajustes a `reports.service.spec.ts`) porque ahí sí hay lógica nueva (la señal `inventarioIncluido()` y su efecto en el agregado de estado), no solo wiring. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | La única decisión de negocio (ocultar el Margen por completo en vez de mostrar un estado "no disponible") ya quedó registrada en `spec.md` (Clarifications de `/speckit-specify`, FR-007) antes de este plan. Las decisiones de este plan son puramente técnicas: dónde vive `inventarioIncluido()` (research.md Decisión 3), y el mecanismo de gating de las queries (Decisión 4) — ninguna cambia qué ve el usuario, solo cómo se logra. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec+Clarifications) → este `plan.md`/`research.md` (Decisión técnica) → `tasks.md` (Fase 2, no generada por este comando) → implementación → tests nuevos/ajustados por historia → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

**Re-chequeo post-diseño (Fase 1)**: `research.md` y `data-model.md` no introdujeron ninguna
entidad, dependencia ni decisión que contradiga la tabla anterior. Un punto merece mención
explícita, sin ser una excepción de Complexity Tracking: `inventarioIncluido()` vive en
`ReportsService`, no en `ReportsPageComponent` —a diferencia de `comprasIncluido()` en
`inventory-page.component.ts` (spec 033), que sí vive en el componente— porque aquí la señal
gobierna además el `enabled:` de dos `injectQuery`, y esas queries están definidas en el
servicio, no en el componente (research.md Decisión 3). Es una diferencia de ubicación, no de
criterio: ambas leen `PlanSummaryService.summary()` de la misma forma. Gates siguen en PASS.

## Project Structure

### Documentation (this feature)

```text
specs/062-gating-inventario-ajustes-reportes/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones técnicas y alternativas descartadas
├── data-model.md         # Fase 1 (/speckit-plan) — confirma que no hay entidades nuevas
├── quickstart.md         # Fase 1 (/speckit-plan) — validación ejecutable por historia de usuario
├── contracts/            # Fase 1 (/speckit-plan) — nuevo caso de error sobre endpoints existentes
│   └── inventario-module-gating.md
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
app/api/v1/
├── unit_measures/
│   └── router.py                      # MODIFICADO — el `APIRouter(...)` gana
│                                         dependencies=[Depends(require_module_access("inventario"))]
│                                         a nivel de router (mismo patrón que
│                                         inventory/router.py:28-31, spec 033) — cubre los 5
│                                         endpoints existentes (list/get/create/update/delete) sin
│                                         tocar ninguno individualmente (FR-002/FR-003)
└── reports/
    └── router.py                      # MODIFICADO — las rutas `GET /reports/inventory` (RF-063)
                                          y `GET /reports/profitability` (RF-065) ganan
                                          dependencies=[Depends(require_module_access("inventario"))]
                                          a nivel de ruta individual (mismo patrón que las 4 rutas
                                          `/purchases*` de inventory/router.py con "compras", spec
                                          033) — el resto de rutas del router (sales, products,
                                          top-products, categories, cashiers) no se tocan (FR-006/
                                          FR-007)

# pos-heladeria
src/app/modules/
├── dashboard/
│   └── routes.ts                      # MODIFICADO — el hijo `ajustes/unidades` gana
│                                         canActivate: [planModuleGuard('inventario')] (sin tocar
│                                         el roleGuard ya existente en el padre `ajustes`, ni los
│                                         demás hijos) (FR-002)
├── settings/
│   └── pages/
│       └── settings-page.component.ts # MODIFICADO — `tabs` (arreglo fijo) pasa a `visibleTabs`
│                                         (señal computada), inyecta PlanSummaryService y filtra la
│                                         pestaña "Unidades de medida" con el mismo criterio
│                                         fail-open-mientras-carga que SidebarComponent.visibleItems
│                                         (misma sesión, mismo componente ya modificado hoy) — la
│                                         pestaña es puramente cosmética porque la ruta ya está
│                                         protegida por el guard de arriba (FR-001)
└── reports/
    ├── services/
    │   └── reports.service.ts         # MODIFICADO — nueva señal `inventarioIncluido` (lee
    │                                     PlanSummaryService.summary(), fail-closed mientras el
    │                                     resumen no cargó — a diferencia del fail-open de
    │                                     Sidebar/Ajustes, research.md Decisión 4); `inventoryQuery`
    │                                     y `profitabilityQuery` ganan `enabled: inventarioIncluido()`;
    │                                     el arreglo `queries` usado por `isLoading`/`error` pasa de
    │                                     fijo a `computed()`, incluyendo esas dos queries solo
    │                                     cuando `inventarioIncluido()` es `true` (FR-005/FR-006/
    │                                     FR-007)
    └── pages/
        └── reports-page.component.ts  # MODIFICADO — la sección "Insumos con stock bajo" y la
                                          tarjeta "Margen" (con su `marginHint`) se envuelven en
                                          `@if (svc.inventarioIncluido())`; sin cambios en las
                                          demás tarjetas/secciones (FR-005/FR-007)
```

**Structure Decision**: Web application con dos repositorios (backend FastAPI + frontend
Angular), estructura ya establecida por specs previas (033 y anteriores) — esta spec no agrega
ningún directorio ni módulo nuevo, solo modifica 5 archivos existentes en total (2 backend, 3
frontend).

## Complexity Tracking

*Sin violaciones que justificar — tabla vacía.*
