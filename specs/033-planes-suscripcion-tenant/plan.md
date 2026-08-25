# Implementation Plan: Planes de Suscripción por Tenant (Límites y Accesos a Módulos)

**Branch**: `033-planes-suscripcion-tenant` | **Date**: 2026-08-24 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/033-planes-suscripcion-tenant/spec.md`

## Summary

Hoy `Tenant.plan` (`app/core/models.py`, esquema `shared`) es una columna de texto libre
(`String(100)`, default `"basic"`) que nadie lee: no hay ningún endpoint que la edite, ninguna
validación la consulta, y el único sitio que la toca es un characterization test que la instancia
como ruido del constructor (`test_tenant_timezone.py`). No existe hoy ningún mecanismo real de
límites por plan ni de acceso a módulos — todo tenant puede crear mesas/cajas/usuarios/
productos/métodos de pago sin tope, y acceder a inventario/compras/promociones sin restricción.

Esta spec agrega una entidad `Plan` real (esquema `shared`, ocho características fijas: cinco
límites numéricos con "ilimitado" como `NULL`, tres accesos de módulo booleanos, más dos precios
de referencia — mensual y anual, FR-016) y reemplaza la columna heredada `Tenant.plan` por
`Tenant.plan_id` (FK `NOT NULL` a `shared.plans`) más tres columnas nuevas que representan la
asignación vigente (`ciclo_facturacion`, `plan_iniciado_en`, `plan_vence_en`), que por
construcción garantiza FR-003 (todo tenant tiene siempre exactamente un plan vigente) sin
necesitar una tabla de asignación separada. Un helper centralizado (`app/core/plan_limits.py`)
bloquea, con lock a nivel de fila para evitar condiciones de carrera (FR-015), la creación de
mesas/cajas/usuarios/productos/métodos de pago activos que superarían el límite del plan vigente,
y una dependencia de FastAPI (`require_module_access`) bloquea el acceso a inventario/compras/
promociones cuando el plan no lo incluye. La misma capa agrega una verificación previa,
`ensure_plan_not_expired`, que bloquea automáticamente **todo** lo anterior — límites y módulos
por igual — cuando `Tenant.plan_vence_en` ya pasó y el Super Admin no renovó (FR-019/FR-020/
FR-021); `plan_vence_en` es `NULL` (nunca vence) tanto para el plan transicional sembrado en la
migración como para cualquier asignación futura donde el Super Admin elija explícitamente "sin
vencimiento" en vez de un ciclo mensual/anual. Un endpoint nuevo (`GET /plan`) sirve tanto la
pantalla de consumo del Tenant Admin (US6/FR-013, ahora incluye la fecha de vencimiento) como los
datos que el frontend necesita para ocultar navegación de módulos no incluidos o vencidos antes
de que el usuario choque con un 403.

## Technical Context

**Language/Version**: Backend — Python 3.14.4 (venv `pos-backend/env`). Frontend — TypeScript
5.9.2 (Angular 21.1.x, standalone components + signals, sin NgModules).

**Primary Dependencies**:
- Backend: FastAPI 0.136.3, SQLAlchemy 2.0.50 (sync), Alembic 1.18.4, Pydantic 2.13.4,
  `python-dateutil` 2.9.0.post0 (`relativedelta`, para sumar 1 mes/1 año a `plan_iniciado_en` sin
  reinventar aritmética de calendario — ya está en `requirements.txt`, instalado como
  dependencia transitiva, pero esta spec es la primera en importarlo directamente en código de
  aplicación; no se agrega ninguna línea nueva a `requirements.txt`, así que Principio IX no
  aplica en sentido estricto — ver research.md Decisión 13). Ninguna otra dependencia nueva.
- Frontend: Angular 21 (standalone + signals), `@tanstack/angular-query-experimental` 5.101.4,
  Tailwind 4.1.12. Ninguna dependencia nueva.

**Storage**: PostgreSQL 16, multi-tenant schema-per-tenant.
- Tabla nueva `plans` en el esquema `shared` (junto a `tenants`/`roles`/`users`,
  `app/core/models.py`), vía Alembic plano (sin `@for_each_tenant_schema`), mismo patrón que
  `d6953c4dcf45_payment_method_catalog.py` (spec 032). Columnas `precio_mensual`/`precio_anual`:
  `Numeric(12, 2)` nullable, mismo tipo que `ProductVariant.price`/`Sale.total`
  (`app/models/product_variant.py`, `app/models/sale.py` — research.md Decisión 11).
- Columnas nuevas en `shared.tenants`: `plan_id` (FK `NOT NULL` → `shared.plans.id`),
  `ciclo_facturacion` (`String(10)` nullable, `CHECK IN ('mensual','anual')` o `NULL`),
  `plan_iniciado_en`/`plan_vence_en` (`DateTime` naive UTC, nullable — mismo patrón que
  `SessionParticipant.expires_at`, `app/models/session_participant.py`, cuyo `NULL` ya significa
  "nunca vence" en `app/core/qr_context.py` — research.md Decisión 12). Baja de la columna
  heredada `Tenant.plan` (`String(100)`, sin uso — research.md Decisión 2).
- Ninguna tabla nueva en esquema `tenant`: los cinco recursos gobernados por límite
  (`dining_tables`, `cash_registers`, `users`, `products`, `payment_methods`) ya existen; esta
  spec solo agrega la validación de conteo antes de insertarlos, no cambia su modelo.

**Testing**: `unittest` vía `python -m unittest discover -s app/characterization_tests -p 'test_*.py'`
(sin pytest en el repo). No existe hoy ningún test `"CONGELA comportamiento actual:"` que asuma
creación sin límite de mesas/cajas/usuarios/productos/métodos de pago, ni acceso sin restricción a
inventario/compras/promociones — research.md confirma que ninguno de los characterization tests
existentes necesita modificarse (Principio III no se activa; toda la funcionalidad es nueva,
Principio IV). Se agregan fixtures y tests nuevos por historia de usuario, mismo patrón que spec
032 (`payment_catalog_fixtures.py`). Frontend: Vitest (`ng test`, Angular 21 build system),
specs colocados (`*.spec.ts`). Playwright figura como devDependency pero no está configurado en el
repo (sin `playwright.config.ts` ni carpeta e2e) — no se asume cobertura e2e automatizada.

**Target Platform**: Linux server (API `pos-backend`) + navegador (SPA `pos-heladeria`,
incluyendo el panel de Super Admin ya existente en `src/app/modules/super-admin/`).

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de este repo `pos-specs`).

**Performance Goals**: sin objetivo de throughput nuevo. La validación de límite agrega, por cada
creación de mesa/caja/usuario/producto/método de pago, un `SELECT ... FOR UPDATE` sobre una fila
ya necesaria (`shared.tenants`) más un `COUNT` sobre una tabla ya indexada por PK — sin impacto de
diseño relevante sobre el volumen actual del sistema.

**Constraints**:
- FR-015 exige garantía estricta bajo concurrencia: como máximo una solicitud de creación
  simultánea tiene éxito cuando queda exactamente un cupo. Se resuelve con `with_for_update()`
  sobre la fila de `shared.tenants` del tenant (mismo idiom que `InvoiceCounter._next_number` y
  `table_sessions._load`, research.md Decisión 5) — no con una constraint de base de datos, porque
  el límite es un valor configurable en tiempo de ejecución (`Plan.*_limit`), no un invariante fijo
  del esquema.
- Ningún dato existente se pierde: bajar el límite de un plan o quitarle un módulo a un tenant
  **no** borra ni desactiva recursos ya creados (FR-011/FR-012, edge cases de `spec.md`) — el
  helper de límites solo bloquea creaciones nuevas, nunca actúa sobre filas existentes.
- La columna heredada `Tenant.plan` se elimina en la misma fase de migración de datos que agrega
  `plan_id` (research.md Decisión 2/3) — no se conserva como campo muerto paralelo, porque
  mantenerla sería la clase de comportamiento fantasma que esta spec existe para reemplazar, no una
  refactorización oportunista ajena a la spec (Principio V: reemplazarla es exactamente el alcance
  de esta funcionalidad).
- "Compras" no es un router ni una ruta separada hoy (vive como pestaña dentro de
  `inventory-page.component.ts` / mismo router `app/api/v1/inventory/router.py` que "inventario") —
  su gating de módulo se aplica a nivel de endpoint específico (`/purchases*`), no a nivel de
  router completo (research.md Decisión 6).
- Ningún cobro real ocurre en ningún punto de esta spec (Assumptions de `spec.md`): `precio_mensual`/
  `precio_anual` son datos de referencia que el Super Admin captura manualmente; no hay pasarela de
  pago, ni webhook, ni job programado que cobre o notifique. El vencimiento se evalúa **al vuelo**,
  en cada request (`Tenant.plan_vence_en < utc_now()`), no mediante un proceso en segundo plano que
  cambie el estado de un tenant de forma proactiva — evita introducir un scheduler nuevo para algo
  que ya se resuelve barato en cada validación existente (research.md Decisión 14).
- `Tenant.plan_vence_en IS NULL` MUST significar "nunca vence", tanto para el plan transicional
  sembrado en la migración (research.md Decisión 3, ahora también sin ciclo de facturación) como
  para cualquier tenant al que el Super Admin le asigne explícitamente "sin vencimiento" — la
  migración no puede, bajo ninguna circunstancia, dejar a un tenant existente bloqueado el día del
  despliegue (Principio II).

**Scale/Scope**: 1 tabla nueva (`shared.plans`, con 2 columnas de precio), 4 columnas nuevas
(`plan_id`, `ciclo_facturacion`, `plan_iniciado_en`, `plan_vence_en`) + 1 columna eliminada en
`shared.tenants`, 1 helper de aplicación nuevo (`app/core/plan_limits.py`, con 3 funciones:
`enforce_plan_limit`, `require_module_access`, `ensure_plan_not_expired`) consumido desde 5
endpoints de creación existentes, 1 dependencia de gating de módulo nueva consumida desde 2
routers existentes + 1 conjunto de endpoints dentro de un tercero, ~7 endpoints nuevos/modificados
(CRUD de catálogo de planes + asignación/cambio/renovación de plan de tenant + consumo del tenant)
en `pos-backend`; 1 pantalla nueva en el panel de Super Admin, 1 pantalla nueva de "mi plan" para
el Tenant Admin (ahora con fecha de vencimiento), y guards de navegación nuevos en
`pos-heladeria`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 6 historias priorizadas, 21 FRs y 2 sesiones de clarificación (2026-08-24 original + 2026-08-24 ampliación de precio/duración/renovación) antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El único comportamiento que cambia (creación de mesas/cajas/usuarios/productos/métodos de pago pasa a poder bloquearse; acceso a inventario/compras/promociones pasa a poder denegarse; ahora también: un tenant vencido pasa a poder bloquearse) es exactamente el comportamiento nuevo que `spec.md` define — no es una corrección de una anomalía heredada (Principio IV), así que no requiere entrada en `registro-de-anomalias.md`. La migración de datos (research.md Decisión 3, ampliada) siembra un plan transicional sin límites/restricciones **y sin ciclo de facturación ni vencimiento** para que ningún tenant existente vea su comportamiento actual alterado el día del despliegue — el Super Admin decide después, activamente, bajar a cada tenant a un plan real con su propio ciclo. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | Ningún `"CONGELA comportamiento actual:"` existente asume ausencia de límites, de gating de módulos, ni de vencimiento (research.md lo confirma por búsqueda) — ninguno se modifica. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | Todo el comportamiento nuevo (`Plan` con precios, límites, gating de módulos, vencimiento automático, renovación, pantalla de consumo) está definido en `spec.md`; el criterio de éxito es conformidad con la spec, no equivalencia con el pasado. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Se elimina la columna heredada `Tenant.plan`, pero no es una refactorización ajena: es la misma responsabilidad ("qué plan tiene este tenant") que esta spec reemplaza por una versión real y validada — dejarla intacta sería mantener un campo fantasma que confunde, no una decisión neutral. Ningún otro modelo/router ajeno a las características de plan se toca (ej. `cash/service.py`, `Payment`/`Sale`, no se modifican; ninguna pasarela de pago se integra). | PASS |
| **VI. Evolución Incremental** | El alcance se divide en las mismas unidades que las historias del spec: catálogo de planes con precios (US1) → asignación/renovación a tenants (US2) → bloqueo de límites (US3) → bloqueo de módulos (US4) → bloqueo automático por vencimiento (US5) → pantalla de consumo (US6), cada una verificable por separado (ver Project Structure). La migración de datos (tabla + seed + backfill + `NOT NULL` + drop de columna heredada + columnas de vencimiento) es su propia unidad, separada de la implementación del helper de límites. | PASS |
| **VII. Compatibilidad con Datos Históricos** | Ninguna `Sale`/`Payment`/`Invoice` se toca. El plan transicional sembrado en la migración (research.md Decisión 3) preserva el comportamiento sin límites ni vencimiento que todo tenant existente ya tenía — nada retroactivo se recalcula. | PASS |
| **VIII. Evolución del Modelo de Datos** | Ver data-model.md: tabla nueva (`shared.plans`, con precios), columnas nuevas en `shared.tenants` (`plan_id` `NOT NULL`, `ciclo_facturacion`/`plan_iniciado_en`/`plan_vence_en` nullable-por-diseño) con estrategia de backfill determinística explícita (research.md Decisión 3) y de rollback (`op.drop_table`/`op.drop_column`/restaurar columna `plan`) en research.md. | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se agrega ninguna línea nueva a `requirements.txt`/`package.json`. Se empieza a importar directamente `python-dateutil` (ya instalado como transitiva) por primera vez en código de aplicación — justificado en research.md Decisión 13 como el caso límite de este principio: no es una dependencia nueva en sentido estricto, pero se documenta la razón (aritmética de calendario correcta para sumar 1 mes/1 año) igual que si lo fuera. | PASS |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a comandos `unittest` ejecutables, incluyendo un test de concurrencia dedicado para FR-015 y un test dedicado de vencimiento/renovación para FR-019/FR-020 (US5). | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Decisiones técnicas explícitas en research.md para no confundirse con decisiones de negocio: (a) ocho columnas fijas en `Plan` en vez de un modelo de características extensible (EAV); (b) `Tenant.plan_id`/`ciclo_facturacion`/fechas como columnas directas en vez de una tabla de asignación separada; (c) lock sobre la fila de `Tenant` en vez de una tabla de contadores nueva; (d) vencimiento evaluado al vuelo en cada request en vez de un job en segundo plano; (e) "sin vencimiento" como una elección explícita del Super Admin (`ciclo_facturacion: null`), no un default implícito. El spec no exige ninguna de las cinco — son la forma de implementar sin romper lo existente ni sobre-construir para una extensibilidad que las Assumptions del spec descartan explícitamente. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec, incluyendo la sesión de ampliación) → este `plan.md`/`research.md` (Decisión técnica) → `tasks.md` (Fase 2, no generada por este comando) → implementación → tests nuevos por historia → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

**Re-chequeo post-diseño (Fase 1)**: `research.md` y `data-model.md` no introdujeron ninguna
entidad, dependencia ni decisión que contradiga la tabla anterior. Dos puntos merecen mención
explícita, ninguno una excepción de Complexity Tracking: (a) la Historia de Usuario 6 (pantalla
de consumo) y el gating de navegación del frontend comparten el mismo endpoint `GET /plan` en vez
de crear dos, justificado en research.md Decisión 7; (b) la Historia de Usuario 5 (bloqueo por
vencimiento) no agrega ningún endpoint ni router nuevo — vive enteramente dentro de
`plan_limits.py` como una verificación que `enforce_plan_limit`/`require_module_access` ya
invocan internamente, así que los cinco routers de creación y los dos routers de módulo no
necesitan ningún cambio adicional más allá del que ya requerían las Historias 3 y 4 (research.md
Decisión 14). Gates siguen en PASS.

## Project Structure

### Documentation (this feature)

```text
specs/033-planes-suscripcion-tenant/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones técnicas y alternativas descartadas
├── data-model.md         # Fase 1 (/speckit-plan) — entidades, columnas, validaciones, migraciones
├── quickstart.md         # Fase 1 (/speckit-plan) — validación ejecutable por historia de usuario
├── contracts/            # Fase 1 (/speckit-plan) — contratos HTTP nuevos/modificados
│   ├── super-admin-plans.md
│   └── tenant-plan-enforcement.md
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
app/
├── models/
│   └── plan.py                        # NUEVO — Plan (shared.plans): 8 características +
│                                         precio_mensual/precio_anual (Numeric(12,2) nullable,
│                                         research.md Decisión 11)
├── core/
│   ├── models.py                      # MODIFICADO — Tenant pierde plan (str); gana plan_id (FK
│   │                                     NOT NULL), ciclo_facturacion (str nullable, CHECK
│   │                                     IN ('mensual','anual') o NULL), plan_iniciado_en/
│   │                                     plan_vence_en (DateTime naive UTC nullable — mismo patrón
│   │                                     que SessionParticipant.expires_at, research.md Decisión 12)
│   ├── db.py                          # MODIFICADO — tenant_create() exige plan_id +
│   │                                     ciclo_facturacion, calcula plan_iniciado_en/
│   │                                     plan_vence_en (o los deja NULL si ciclo_facturacion es
│   │                                     null), crea la fila dentro de la misma transacción
│   │                                     (FR-004/FR-017/FR-018)
│   └── plan_limits.py                 # NUEVO — enforce_plan_limit(db, tenant, resource_key),
│                                         require_module_access(module_key) (dependencia FastAPI),
│                                         ensure_plan_not_expired(tenant) (US5, FR-019/FR-021 —
│                                         llamada internamente por las otras dos, ningún caller
│                                         externo la invoca directo, research.md Decisión 14), y
│                                         calculate_plan_vencimiento(inicio, ciclo) (usa
│                                         dateutil.relativedelta, research.md Decisión 13)
│
├── api/v1/
│   ├── admin/
│   │   └── schema.py                  # MODIFICADO — TenantCreateWithUser gana plan_id +
│   │                                     ciclo_facturacion (obligatorio, acepta null — FR-004/
│   │                                     FR-017)
│   │
│   ├── super_admin/
│   │   ├── router.py                  # MODIFICADO — monta plans_router; agrega
│   │   │                                 PATCH /super-admin/tenants/{id} (asignar, cambiar O
│   │   │                                 renovar plan — mismo endpoint cubre FR-010/FR-017/FR-020,
│   │   │                                 research.md Decisión 16)
│   │   ├── plans_router.py            # NUEVO — CRUD de planes, incluye precios (US1)
│   │   └── schemas.py                 # MODIFICADO — PlanCreate/Update/Response (con
│   │                                     precio_mensual/precio_anual), TenantPlanUpdate (con
│   │                                     ciclo_facturacion, valida contra FR-017)
│   │
│   ├── plan/
│   │   ├── router.py                  # NUEVO — GET /plan (US6/FR-013 + datos de gating para el
│   │   │                                 frontend), prefix "/plan", get_current_user
│   │   ├── service.py                 # NUEVO — arma el resumen de consumo (usa las mismas
│   │   │                                 funciones de conteo que plan_limits.py) más
│   │   │                                 vence_en/vencido/ciclo_facturacion
│   │   └── schemas.py                 # NUEVO — PlanSummaryResponse (gana vence_en/vencido)
│   │
│   ├── orders/router.py               # MODIFICADO — create_table() llama a
│   │                                     enforce_plan_limit(db, tenant, "mesas") antes del insert
│   │                                     (ya cubre vencimiento internamente, sin cambio adicional
│   │                                     para US5)
│   ├── cash/router.py                 # MODIFICADO — create_register() llama a
│   │                                     enforce_plan_limit(db, tenant, "cajas")
│   ├── users/router.py                # MODIFICADO — create_user() llama a
│   │                                     enforce_plan_limit(db, tenant, "usuarios")
│   ├── products/router.py             # MODIFICADO — create_product() llama a
│   │                                     enforce_plan_limit(db, tenant, "productos")
│   ├── sales/router.py                # MODIFICADO — creación/activación de payment_methods llama
│   │                                     a enforce_plan_limit(db, tenant, "metodos_pago_activos")
│   │
│   ├── inventory/router.py            # MODIFICADO — router entero gana
│   │                                     Depends(require_module_access("inventario")); los 3
│   │                                     endpoints de /purchases* ganan además
│   │                                     Depends(require_module_access("compras")) (ya cubre
│   │                                     vencimiento internamente, sin cambio adicional para US5)
│   └── promotions/router.py           # MODIFICADO — router entero gana
│                                         Depends(require_module_access("promociones"))
│
├── characterization_tests/
│   ├── plan_fixtures.py               # NUEVO — helpers para crear Plan (con/sin precios) y
│   │                                     Tenant con distintas combinaciones de ciclo/vencimiento
│   ├── test_super_admin_plans.py      # NUEVO — US1 (FR-001/FR-002/FR-007/FR-016)
│   ├── test_tenant_plan_assignment.py # NUEVO — US2 (FR-003/FR-004/FR-010/FR-011/FR-012/FR-017/
│   │                                     FR-018/FR-020)
│   ├── test_plan_resource_limits.py   # NUEVO — US3 (FR-005/FR-006/FR-007), incluye un caso de
│   │                                     concurrencia (dos hilos/conexiones contra el mismo cupo,
│   │                                     FR-015)
│   ├── test_plan_module_access.py     # NUEVO — US4 (FR-008/FR-009)
│   ├── test_plan_expiration.py        # NUEVO — US5 (FR-019/FR-020/FR-021), incluye el caso de
│   │                                     "sin vencimiento nunca bloquea"
│   └── test_plan_summary.py           # NUEVO — US6 (FR-013, incluye vence_en/vencido)
│
└── alembic/versions/
    ├── {rev}_plans_table.py                    # NUEVO — tabla shared.plans (8 características +
    │                                              precio_mensual/precio_anual)
    ├── {rev}_seed_transitional_plan.py         # NUEVO — siembra el plan transicional sin límites
    │                                              (data, no Foundational)
    └── {rev}_tenant_plan_assignment.py         # NUEVO (renombrada de tenant_plan_id) — agrega
                                                    plan_id nullable + ciclo_facturacion/
                                                    plan_iniciado_en/plan_vence_en (nullable para
                                                    siempre); backfill determinístico de plan_id al
                                                    plan transicional (ciclo/fechas quedan NULL);
                                                    ALTER plan_id a NOT NULL; DROP COLUMN plan
                                                    (heredada) — todo en una sola migración por ser
                                                    100% determinístico (research.md Decisión 3,
                                                    contraste explícito con el backfill fuzzy de
                                                    spec 032)

# pos-heladeria
src/app/modules/
├── super-admin/
│   ├── interfaces/
│   │   ├── plan.interface.ts          # NUEVO — Plan gana precio_mensual/precio_anual
│   │   └── tenant.interface.ts        # MODIFICADO — Tenant gana plan_id/plan_name/
│   │                                     ciclo_facturacion/plan_vence_en; TenantCreateWithUser y
│   │                                     TenantPlanUpdate ganan plan_id/ciclo_facturacion
│   ├── services/
│   │   ├── plan.service.ts            # NUEVO — mismo patrón que payment-method-catalog.service.ts
│   │   └── tenant.service.ts          # MODIFICADO — changePlan(tenantId, {planId,
│   │                                     cicloFacturacion}) — mismo método sirve asignar, cambiar
│   │                                     y renovar (research.md Decisión 16)
│   ├── pages/
│   │   └── plans-page.component.ts    # NUEVO — listar/crear/editar planes (con precios)
│   ├── components/
│   │   ├── plan-form.component.ts     # NUEVO — límites numéricos (input + toggle "ilimitado"),
│   │   │                                 accesos de módulo (toggles) y dos campos de precio
│   │   └── tenant-form.component.ts   # MODIFICADO — <select> de plan + <select> de ciclo de
│   │                                     facturación (mensual/anual/sin vencimiento), ambos
│   │                                     obligatorios (FR-004/FR-017)
│   └── routes.ts                      # MODIFICADO — hijo nuevo `plans`, mismo patrón que
│                                         `payment-methods-catalog`
│
├── plan/                              # NUEVO módulo — "mi plan" del Tenant Admin (US6)
│   ├── interfaces/plan-summary.interface.ts   # NUEVO — gana vence_en/vencido/ciclo_facturacion
│   ├── services/plan-summary.service.ts       # NUEVO — GET /plan
│   └── pages/plan-summary-page.component.ts   # NUEVO — muestra fecha de vencimiento o "sin
│                                                 vencimiento"
│
├── core/guards/
│   └── plan-module.guard.ts           # NUEVO — bloquea rutas de módulo no incluido en el plan O
│                                         cuyo plan está vencido (`vencido` de plan-summary), mismo
│                                         patrón que role.guard.ts
│
└── dashboard/
    └── routes.ts                      # MODIFICADO — ruta nueva `mi-plan` (sin roleGuard extra más
                                          allá de estar autenticado, mismo criterio que `mi-cuenta`
                                          — accesible a cualquier usuario del dashboard, no solo
                                          ADMIN, porque la pantalla no expone nada que un CASHIER no
                                          debería ver: solo su propio consumo); `inventario`/
                                          `promotions` ganan `planModuleGuard(...)` encadenado al
                                          `roleGuard` existente
```

**Structure Decision**: cada historia de usuario del spec se mapea a un subconjunto disjunto de
los ficheros de arriba (US1 → `plans_router.py` + módulo `super-admin/` del frontend, incluyendo
precios; US2 → `admin/schema.py` + `core/db.py` + `PATCH /super-admin/tenants/{id}` +
`tenant-form.component.ts`, incluyendo ciclo de facturación y renovación; US3 →
`plan_limits.py::enforce_plan_limit` + los 5 routers de creación; US4 →
`plan_limits.py::require_module_access` + `inventory/router.py`/`promotions/router.py` +
`plan-module.guard.ts`; US5 → enteramente dentro de `plan_limits.py::ensure_plan_not_expired`,
sin tocar ningún router de recurso o módulo (research.md Decisión 14); US6 → `plan/` nuevo en
ambos repos, extendiendo el `GET /plan` que US4 ya construyó), consistente con Principio VI. En
`pos-backend` no se crea ningún paquete nuevo más allá de `models/plan.py` y `api/v1/plan/` — se
extiende `super_admin` (donde ya vive la administración de plataforma) y se agrega un helper
transversal (`core/plan_limits.py`) consumido por los módulos existentes que ya crean los cinco
recursos limitados, evitando duplicar la lógica de conteo+lock+vencimiento cinco (más dos) veces.
En `pos-heladeria` se extiende `super-admin` (mismo patrón que `plans-page.component.ts` de la
analogía con métodos de pago) y se agrega un módulo `plan/` nuevo porque no existe hoy ninguna
pantalla de "mi cuenta de plataforma" del tenant a la que anexar esto sin mezclar
responsabilidades ajenas (`ajustes` es configuración de negocio operable por el tenant; el plan no
lo es).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
