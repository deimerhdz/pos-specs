---

description: "Task list for Planes de Suscripción por Tenant (Límites, Accesos, Precios y Renovación)"
---

# Tasks: Planes de Suscripción por Tenant (Límites y Accesos a Módulos, Precio/Duración/Renovación)

**Input**: Design documents from `/specs/033-planes-suscripcion-tenant/` (plan.md, spec.md,
research.md, data-model.md, contracts/, quickstart.md)

**Tests**: incluidos — `plan.md` (Project Structure) y `quickstart.md` fijan de antemano qué
characterization test crea cada historia (Constitución, Principio X: Verificación Obligatoria),
así que no son opcionales para esta spec.

**Organization**: tareas agrupadas por historia de usuario (US1-US6, prioridades de `spec.md`)
para que cada una sea implementable y verificable de forma independiente, per `quickstart.md`. La
migración de datos existentes es 100% determinística (research.md Decisión 3) y por eso vive
completa en Foundational, no como fase aparte al final. Esta versión de `tasks.md` reemplaza a la
anterior tras la sesión de ampliación de clarificaciones (2026-08-24 — precio, duración y
renovación): agrega la Historia de Usuario 5 (bloqueo automático por vencimiento) y renumera la
antigua Historia 5 (pantalla de consumo) a Historia 6, además de extender US1/US2/US4 con precio,
ciclo de facturación y renovación.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (ficheros distintos, sin dependencia de una tarea sin
  terminar)
- **[Story]**: historia de usuario a la que pertenece (US1-US6)
- Cada tarea incluye la ruta de fichero exacta, relativa a la raíz del repo sibling que corresponda
  (`pos-backend` o `pos-heladeria`)

## Path Conventions

Dos repositorios sibling de `pos-specs` (Constitución §Alcance, plan.md §Project Structure):

- Backend: `pos-backend/app/...` (rutas de este documento ya incluyen el prefijo `pos-backend/`)
- Frontend: `pos-heladeria/src/app/...` (rutas ya incluyen el prefijo `pos-heladeria/`)

---

## Phase 1: Setup

**Purpose**: confirmar que el entorno está listo — esta spec no agrega ninguna dependencia nueva
a `requirements.txt`/`package.json` (plan.md Technical Context; `python-dateutil` ya está
instalado, research.md Decisión 13).

- [X] T001 Confirmar entorno: `pos-backend` con el venv activado (`source env/bin/activate`,
  Python 3.14.4) y `pos-heladeria` con `npm install` ya corrido; confirmar
  `python -c "import dateutil; print(dateutil.__version__)"` funciona sin instalar nada nuevo.
  Agregar `python-dateutil==2.9.0.post0` como línea explícita en
  `pos-backend/requirements.txt` (hoy solo está presente como dependencia transitiva) — esta spec
  es la primera en importarlo directamente en código de aplicación (T007), así que debe quedar
  pinneado por su propio nombre, no depender de que otra dependencia lo siga arrastrando.
  **Confirmado**: ya estaba pinneado explícitamente en `requirements.txt:59` (no era solo
  transitiva); `dateutil.__version__` == `2.9.0.post0`; `node_modules/` presente en
  `pos-heladeria`. Se crearon además las ramas `033-planes-suscripcion-tenant` en `pos-backend` y
  `pos-heladeria` (ambas partían de `develop`, limpias)

**Checkpoint**: entornos listos, sin instalar nada nuevo.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: la entidad `Plan` (con precios), el reemplazo de `Tenant.plan` por
`Tenant.plan_id`/`ciclo_facturacion`/`plan_iniciado_en`/`plan_vence_en`, y las funciones base de
validación de límites/módulos — nada de `Phase 3+` puede empezar sin esto (data-model.md,
research.md Decisiones 1-13). El bloqueo por vencimiento en sí (`ensure_plan_not_expired`) se
construye en la Historia de Usuario 5, no aquí — ver esa fase.

**⚠️ CRITICAL**: ninguna historia de usuario arranca hasta que esta fase esté completa.

- [X] T002 Crear migración Alembic `pos-backend/alembic/versions/{rev}_plans_table.py`: tabla
  `shared.plans` (`id`, `name` único, `description`, cinco columnas `*_limit` `Integer` nullable
  default `0`, tres columnas `*_access` `Boolean` default `false`, `precio_mensual`/
  `precio_anual` `Numeric(12,2)` nullable, `created_at`/`updated_at`), migración plana **sin**
  `@for_each_tenant_schema` (mismo patrón que `d6953c4dcf45_payment_method_catalog.py`);
  `downgrade()` con `op.drop_table("plans", schema="shared")` (data-model.md, research.md
  Decisiones 1 y 11)
- [X] T003 [P] Crear modelo `Plan` en `pos-backend/app/models/plan.py` (columnas de
  data-model.md incluyendo `precio_mensual`/`precio_anual: Mapped[Optional[Decimal]]`,
  `__table_args__ = {"schema": "shared"}`)
- [X] T004 Crear migración Alembic
  `pos-backend/alembic/versions/{rev}_seed_transitional_plan.py`: siembra una fila
  `name="Ilimitado (transición)"` con los cinco límites en `NULL`, los tres accesos en `true`, y
  ambos precios en `NULL` (`op.execute`, sin ORM); `downgrade()` borra esa fila por `name` —
  depende de T002 (research.md Decisión 3)
- [X] T005 Crear migración Alembic
  `pos-backend/alembic/versions/{rev}_tenant_plan_assignment.py`: agrega `plan_id` (UUID, FK →
  `shared.plans.id`) nullable, `ciclo_facturacion` (`String(10)`, `CHECK IN ('mensual','anual')`
  o `NULL`), `plan_iniciado_en`/`plan_vence_en` (`DateTime` naive UTC, nullable para siempre) en
  `shared.tenants`; `UPDATE shared.tenants SET plan_id = (SELECT id FROM shared.plans WHERE name
  = 'Ilimitado (transición)') WHERE plan_id IS NULL` (dejando `ciclo_facturacion`/
  `plan_iniciado_en`/`plan_vence_en` en `NULL`); `ALTER COLUMN plan_id SET NOT NULL`; `DROP
  COLUMN plan` (la columna heredada `String(100)`); `downgrade()` re-agrega `plan` (`String(100)`,
  default `"basic"`) y elimina las cuatro columnas nuevas — depende de T004 (research.md
  Decisiones 2, 3, 10, 12, 17)
- [X] T006 [P] Modificar `Tenant` en `pos-backend/app/core/models.py`: reemplazar
  `plan: Mapped[str]` por `plan_id: Mapped[uuid.UUID]` (FK `NOT NULL` → `shared.plans.id`),
  `ciclo_facturacion: Mapped[Optional[str]]`, `plan_iniciado_en`/`plan_vence_en:
  Mapped[Optional[datetime]]`, y agregar `relationship("Plan")` — depende de T003
- [X] T007 [P] Crear helper `pos-backend/app/core/plan_limits.py`: diccionario de configuración
  por recurso (modelo/tabla, columna de límite en `Plan`, si filtra por `active`, per
  data-model.md tabla de reglas de conteo), `enforce_plan_limit(db: Session, tenant: Tenant,
  resource_key: str)` (lock `with_for_update()` sobre la fila de `Tenant`, contar, comparar,
  `HTTPException(403, ...)` si corresponde — research.md Decisiones 4, 5, 8),
  `require_module_access(module_key: str)` (dependencia FastAPI, `HTTPException(403, ...)` si
  `Plan.<módulo>_access` es `false` — research.md Decisión 8),
  `calculate_plan_vencimiento(inicio: datetime, ciclo: str | None) -> datetime | None` (usa
  `dateutil.relativedelta`, `None` si `ciclo` es `None` — research.md Decisión 13), y
  `validate_billing_cycle_price(plan: Plan, ciclo: str | None)` (lanza `HTTPException(409, ...)`
  si el plan no tiene precio para el ciclo elegido — research.md Decisión 15, reutilizada por
  T008 y T024). **`ensure_plan_not_expired` NO se agrega aquí** — se construye en la Historia de
  Usuario 5 (T048) para que US3/US4 sean entregables sin ella — depende de T006
- [X] T008 [P] Modificar `tenant_create()` en `pos-backend/app/core/db.py`: agregar parámetros
  `plan_id`/`ciclo_facturacion` obligatorios, llamar `validate_billing_cycle_price` y calcular
  `plan_iniciado_en = utc_now()`/`plan_vence_en` con `calculate_plan_vencimiento` antes de crear
  la fila `Tenant`, todo dentro de la misma transacción que ya crea el esquema y el primer
  usuario ADMIN (FR-004/FR-017/FR-018) — depende de T006, T007
- [X] T009 [P] Crear fixtures compartidos `pos-backend/app/characterization_tests/plan_fixtures.py`
  (`make_plan` con distintas combinaciones de límites/accesos/precios, `make_tenant_with_plan`
  con distintas combinaciones de ciclo/fechas, incluyendo un caso con `plan_vence_en` en el
  pasado para la Historia de Usuario 5) — depende de T003, T006
- [X] T010 Aplicar T002/T004/T005 contra una base de datos de prueba y verificar el `downgrade()`
  de cada una (rollback limpio, sin dejar filas/columnas/tabla huérfanas) — depende de T002, T004,
  T005. **VERIFICACIÓN PARCIAL en este entorno**: sin acceso a Docker/Postgres en este sandbox
  (`docker ps` → "permission denied"; sin `pg_isready` — mismo límite que spec 032). Se verificó:
  (a) `python -m py_compile` sobre las tres migraciones, sin errores; (b) `alembic heads` reconoce
  una sola cabeza (`5a77a91b482d`), encadenada limpio sobre el head previo (`04b3d1d3e15f`) sin
  bifurcaciones (`alembic history` confirma la cadena completa); (c) el modelo `Plan`/`Tenant` con
  las columnas nuevas se ejercitó extensamente contra SQLite vía `plan_fixtures.new_session()`
  (T009), incluyendo el caso crítico de "NULL explícito ≠ omitido" (ver nota en
  `app/models/plan.py` — un bug real de SQLAlchemy encontrado y corregido durante esta
  verificación: un `server_default`/`default` en una columna nullable hace que el ORM omita del
  INSERT cualquier asignación explícita de `None`, dejando que el default la reemplace en
  silencio; se resolvió quitando esos defaults de las cinco columnas de límite y moviendo el
  "0 si no se configura" a la capa Pydantic, T012). Pendiente: aplicar `alembic upgrade head`
  contra Postgres real antes de desplegar (mismo pendiente que spec 032 dejó documentado)

**Checkpoint**: `Plan` (con precios), `Tenant.plan_id`/`ciclo_facturacion`/fechas, y las
funciones base de límite/módulo listas — las historias de usuario pueden empezar (en paralelo si
hay más de una persona).

---

## Phase 3: User Story 1 - El Super Admin define planes con límites, accesos y precios (Priority: P1) 🎯 MVP

**Goal**: el Super Admin crea y edita planes indicando límites numéricos (con "ilimitado"),
accesos on/off a módulos, y un precio mensual y/o anual.

**Independent Test**: crear/editar un plan vía `contracts/super-admin-plans.md`, sin que ningún
tenant participe — verificar que queda listado, disponible para asignarse, y con sus precios
guardados (quickstart.md §US1).

### Tests for User Story 1

- [X] T011 [P] [US1] Characterization test
  `pos-backend/app/characterization_tests/test_super_admin_plans.py` — FR-001/FR-002/FR-007/
  FR-014/FR-016, Acceptance Scenarios 1-4 (quickstart.md §US1)

### Implementation for User Story 1

- [X] T012 [US1] Crear `PlanCreate`/`PlanUpdate`/`PlanResponse` en
  `pos-backend/app/api/v1/super_admin/schemas.py` (todos los campos de característica y de
  precio opcionales en `PlanCreate`, `precio_mensual`/`precio_anual: Decimal | None`,
  `contracts/super-admin-plans.md`)
- [X] T013 [US1] Crear router `pos-backend/app/api/v1/super_admin/plans_router.py`: `GET
  /super-admin/plans`, `POST /super-admin/plans` (`409` si `name` duplicado), `PATCH
  /super-admin/plans/{id}` (`404`/`409`) — depende de T012
- [X] T014 [US1] Montar `plans_router` en `pos-backend/app/api/v1/super_admin/router.py`
  (`router.include_router(...)`, mismo patrón que `payment_methods_catalog_router`) — depende de
  T013
- [X] T015 [P] [US1] Crear `pos-heladeria/src/app/modules/super-admin/interfaces/plan.interface.ts`
  (`Plan`, `PlanCreate`, `PlanUpdate` con `precioMensual`/`precioAnual`, mismo shape que
  `contracts/super-admin-plans.md`)
- [X] T016 [US1] Crear `pos-heladeria/src/app/modules/super-admin/services/plan.service.ts`
  (`list`/`create`/`update`, mismo patrón que `payment-method-catalog.service.ts`) — depende de
  T015
- [X] T017 [US1] Crear
  `pos-heladeria/src/app/modules/super-admin/components/plan-form.component.ts` (inputs numéricos
  con toggle "ilimitado" por cada límite, toggles de acceso por módulo, dos campos de precio) —
  depende de T016
- [X] T018 [US1] Crear
  `pos-heladeria/src/app/modules/super-admin/pages/plans-page.component.ts` (listar/crear/editar)
  — depende de T016, T017
- [X] T019 [US1] Agregar ruta hija `plans` en
  `pos-heladeria/src/app/modules/super-admin/routes.ts` — depende de T018

**Checkpoint**: en este punto, la Historia de Usuario 1 debe ser completamente funcional y
verificable de forma independiente.

---

## Phase 4: User Story 2 - El Super Admin asigna, cambia y renueva el plan de un tenant (Priority: P2)

**Goal**: la creación de un tenant exige elegir un plan y un ciclo de facturación; el Super Admin
puede cambiar el plan vigente de un tenant o renovarlo (mismo o distinto plan) en cualquier
momento.

**Independent Test**: crear un tenant de prueba sin poder completar el alta sin elegir plan y
ciclo; luego cambiarlo a otro plan y confirmar que rige de inmediato con una nueva fecha de
vencimiento; renovar el mismo plan y confirmar que la fecha se recalcula (quickstart.md §US2).

### Tests for User Story 2

- [X] T020 [P] [US2] Characterization test
  `pos-backend/app/characterization_tests/test_tenant_plan_assignment.py` —
  FR-003/FR-004/FR-010/FR-011/FR-012/FR-017/FR-018/FR-020, Acceptance Scenarios 1-7
  (quickstart.md §US2)

### Implementation for User Story 2

- [X] T021 [P] [US2] Agregar `plan_id: uuid.UUID` y `ciclo_facturacion: Literal["mensual",
  "anual"] | None` (sin default — research.md Decisión 15) obligatorios a `TenantCreateWithUser`
  en `pos-backend/app/api/v1/admin/schema.py`
- [X] T022 [US2] Modificar `create_tenant()` en `pos-backend/app/api/v1/admin/router.py`: pasar
  `body.plan_id`/`body.ciclo_facturacion` a `tenant_create()` — depende de T021, T008
- [X] T023 [P] [US2] Crear `TenantResponse` (gana `plan_id`/`plan_name`/`ciclo_facturacion`/
  `plan_vence_en`, pierde `plan`) y `TenantPlanUpdate` (`plan_id`, `ciclo_facturacion` sin
  default) en `pos-backend/app/api/v1/super_admin/schemas.py`
- [X] T024 [US2] Agregar `PATCH /super-admin/tenants/{id}` en
  `pos-backend/app/api/v1/super_admin/router.py` (body `TenantPlanUpdate`, `404` si el tenant o
  el `plan_id` no existen, llama `validate_billing_cycle_price` — `409` si el ciclo no tiene
  precio — y recalcula `plan_iniciado_en`/`plan_vence_en` con `calculate_plan_vencimiento`
  siempre, sea cambio de plan o renovación del mismo, `contracts/super-admin-plans.md`) — depende
  de T023, T007
- [X] T025 [P] [US2] Modificar
  `pos-heladeria/src/app/modules/super-admin/interfaces/tenant.interface.ts`: `Tenant` gana
  `plan_id`/`plan_name`/`ciclo_facturacion`/`plan_vence_en`, pierde `plan`;
  `TenantCreateWithUser`/`TenantPlanUpdate` ganan `plan_id`/`ciclo_facturacion`
- [X] T026 [US2] Modificar
  `pos-heladeria/src/app/modules/super-admin/components/tenant-form.component.ts`: `<select>` de
  plan y `<select>` de ciclo de facturación (mensual/anual/sin vencimiento), ambos obligatorios,
  poblados desde `PlanService` — depende de T025, T016
- [X] T027 [US2] Modificar
  `pos-heladeria/src/app/modules/super-admin/services/tenant.service.ts`: método
  `changePlan(tenantId, { planId, cicloFacturacion })` (sirve cambiar y renovar, research.md
  Decisión 16) — depende de T025
- [X] T028 [US2] Modificar
  `pos-heladeria/src/app/modules/super-admin/pages/tenants-page.component.ts`: acción "cambiar/
  renovar plan" por fila (selector de plan + ciclo + confirmación) — depende de T026, T027

**Checkpoint**: en este punto, las Historias de Usuario 1 y 2 deben funcionar de forma
independiente.

---

## Phase 5: User Story 3 - El sistema bloquea la creación de recursos que exceden el límite del plan (Priority: P3)

**Goal**: crear una mesa/caja/usuario/producto/método de pago activo que superaría el límite del
plan vigente se bloquea con un mensaje explicativo; nunca se supera el límite ni bajo concurrencia.

**Independent Test**: tenant con límite de 5 mesas y 5 ya creadas → la sexta se rechaza con mensaje
que menciona el límite; con 4 creadas, la quinta se permite sin aviso (quickstart.md §US3).

### Tests for User Story 3

- [X] T029 [P] [US3] Characterization test
  `pos-backend/app/characterization_tests/test_plan_resource_limits.py` — FR-005/FR-006/FR-007,
  Acceptance Scenarios 1-4, más un caso de concurrencia dedicado para FR-015 (quickstart.md §US3)

### Implementation for User Story 3

- [X] T030 [P] [US3] Modificar `create_table()` en `pos-backend/app/api/v1/orders/router.py`:
  llamar `enforce_plan_limit(db, tenant, "mesas")` antes de `db.add(table)` — depende de T007
- [X] T031 [P] [US3] Modificar `create_register()` en `pos-backend/app/api/v1/cash/router.py`:
  llamar `enforce_plan_limit(db, tenant, "cajas")` — depende de T007
- [X] T032 [P] [US3] Modificar `create_user()` en `pos-backend/app/api/v1/users/router.py`: llamar
  `enforce_plan_limit(db, tenant, "usuarios")` (conteo por `tenant_id`, no por esquema) — depende
  de T007
- [X] T033 [P] [US3] Modificar `create_product()` en `pos-backend/app/api/v1/products/router.py`:
  llamar `enforce_plan_limit(db, tenant, "productos")` — depende de T007
- [X] T034 [P] [US3] Modificar el endpoint de creación/activación de método de pago en
  `pos-backend/app/api/v1/sales/router.py`: llamar `enforce_plan_limit(db, tenant,
  "metodos_pago_activos")` (conteo solo sobre `active=true`, data-model.md) — depende de T007

**Checkpoint**: en este punto, las Historias de Usuario 1, 2 y 3 deben funcionar de forma
independiente.

---

## Phase 6: User Story 4 - El sistema deniega el acceso a módulos que el plan no incluye (Priority: P3)

**Goal**: acceder a inventario/compras/promociones cuando el plan no lo incluye se deniega con un
mensaje claro, no un error técnico.

**Independent Test**: tenant sin acceso a compras → cualquier endpoint de `/inventory/purchases*`
responde `403` con mensaje claro; `GET /inventory/items` sigue funcionando si `inventario_access`
es `true` (quickstart.md §US4).

**Nota de alcance compartido**: esta historia crea el endpoint `GET /plan` porque el guard de
navegación del frontend lo necesita para decidir qué ocultar (research.md Decisión 7) — la
Historia 6 solo agrega la pantalla que lo consume, reutilizando este mismo endpoint sin
duplicarlo. El campo `vencido` de `PlanSummaryResponse` se agrega aquí en el schema, pero solo se
calcula de forma significativa una vez que la Historia 5 construya la lógica de vencimiento — ver
nota en T037.

### Tests for User Story 4

- [X] T035 [P] [US4] Characterization test
  `pos-backend/app/characterization_tests/test_plan_module_access.py` — FR-008/FR-009, Acceptance
  Scenarios 1-2 (quickstart.md §US4)

### Implementation for User Story 4

- [X] T036 [US4] Crear `PlanSummaryResponse` en `pos-backend/app/api/v1/plan/schemas.py` (nuevo
  paquete, shape de `contracts/tenant-plan-enforcement.md` §`GET /plan`, incluye
  `ciclo_facturacion`/`plan_vence_en`/`vencido`)
- [X] T037 [US4] Crear `pos-backend/app/api/v1/plan/service.py`: `build_plan_summary(db, tenant)`
  reutilizando la configuración de conteo de `plan_limits.py` (sin lock — lectura de consulta,
  research.md Decisión 7); `vencido = tenant.plan_vence_en is not None and tenant.plan_vence_en <
  utc_now()` (la misma comparación que la Historia 5 usará en `ensure_plan_not_expired` — no hay
  dependencia de código entre ambas, solo la misma fórmula) — depende de T036, T007
- [X] T038 [US4] Crear `pos-backend/app/api/v1/plan/router.py`: `GET /plan`, prefix `/plan`,
  `Depends(get_current_user)` (no `require_tenant_admin` — research.md Decisión 7) — depende de
  T037
- [X] T039 [US4] Montar el router de `plan` en el router principal de la API (donde se registran el
  resto de routers `v1`) — depende de T038
- [X] T040 [US4] Agregar `dependencies=[Depends(require_module_access("inventario"))]` al
  `APIRouter` de `pos-backend/app/api/v1/inventory/router.py` — depende de T007
- [X] T041 [US4] Agregar `Depends(require_module_access("compras"))` a los cuatro endpoints de
  `/purchases*` dentro de `pos-backend/app/api/v1/inventory/router.py` (research.md Decisión 6) —
  depende de T040
- [X] T042 [P] [US4] Agregar `dependencies=[Depends(require_module_access("promociones"))]` al
  `APIRouter` de `pos-backend/app/api/v1/promotions/router.py` — depende de T007
- [X] T043 [P] [US4] Crear
  `pos-heladeria/src/app/modules/plan/interfaces/plan-summary.interface.ts` (gana
  `cicloFacturacion`/`planVenceEn`/`vencido`) y
  `pos-heladeria/src/app/modules/plan/services/plan-summary.service.ts` (`GET /plan`) — depende de
  T039
- [X] T044 [US4] Crear `pos-heladeria/src/app/core/guards/plan-module.guard.ts` (mismo patrón que
  `role.guard.ts`, consume `PlanSummaryService`; bloquea si el módulo no está incluido **o** si
  `vencido` es `true`, sin importar lo que digan los flags de módulo individuales) — depende de
  T043
- [X] T045 [US4] Aplicar `planModuleGuard('inventario')` / `planModuleGuard('promociones')`
  encadenado al `roleGuard` existente en las rutas `inventario`/`promotions` de
  `pos-heladeria/src/app/modules/dashboard/routes.ts` — depende de T044
- [X] T046 [US4] Ocultar la pestaña "compras" en
  `pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts` cuando el plan no
  incluye `compras_access` o el tenant está `vencido` — depende de T043

**Checkpoint**: en este punto, las Historias de Usuario 1 a 4 deben funcionar de forma
independiente. `vencido` ya se calcula correctamente en `GET /plan` (T037), pero **todavía no
bloquea nada** — ningún endpoint de creación o de módulo lo consulta hasta la Historia 5.

---

## Phase 7: User Story 5 - El sistema bloquea automáticamente a un tenant cuyo plan vence sin renovarse (Priority: P3)

**Goal**: cuando `Tenant.plan_vence_en` ya pasó sin que el Super Admin haya renovado, todo intento
de crear un recurso limitado o acceder a un módulo se bloquea con el mismo tipo de mensaje que un
límite alcanzado o un módulo no incluido — sin borrar ningún dato.

**Independent Test**: con un tenant cuya `plan_vence_en` está en el pasado (fixture de T009), un
intento de crear cualquier recurso limitado o de acceder a cualquier módulo se bloquea con `403` y
un mensaje de vencimiento; con `plan_vence_en IS NULL`, nunca se bloquea por este motivo
(quickstart.md §US5).

**Nota de alcance mínimo**: esta historia no crea ningún endpoint ni router nuevo — extiende las
dos funciones que US3 y US4 ya construyeron y ya están conectadas a los 5+2 puntos de entrada
(research.md Decisión 14). Por eso su fase de implementación es deliberadamente pequeña: una
función nueva más dos líneas de wiring, en un único fichero ya existente.

### Tests for User Story 5

- [X] T047 [P] [US5] Characterization test
  `pos-backend/app/characterization_tests/test_plan_expiration.py` — FR-019/FR-020/FR-021,
  Acceptance Scenarios 1-5 (quickstart.md §US5) — este test debe **fallar** contra el código de
  T007-T046 tal cual (el tenant vencido de la fixture T009 todavía no se bloquea, porque
  `ensure_plan_not_expired` no existe hasta T048)

### Implementation for User Story 5

- [X] T048 [US5] Agregar `ensure_plan_not_expired(tenant: Tenant)` a
  `pos-backend/app/core/plan_limits.py` (`HTTPException(403, "Tu plan venció. Debe renovarse...")`
  si `tenant.plan_vence_en is not None and tenant.plan_vence_en < utc_now()`, sin efecto si es
  `NULL` — FR-019/FR-021) y llamarla como primera línea dentro de `enforce_plan_limit` y
  `require_module_access` (mismas dos funciones de T007) — depende de T007, T029 (US3) y T035
  (US4) ya en verde para no introducir una regresión al modificar funciones compartidas

**Checkpoint**: en este punto, las Historias de Usuario 1 a 5 deben funcionar de forma
independiente — el bloqueo por vencimiento ya protege los cinco recursos limitados y los tres
módulos sin haber tocado ninguno de sus routers.

---

## Phase 8: User Story 6 - El Tenant Admin consulta su plan, su consumo y su vencimiento (Priority: P4)

**Goal**: una sola pantalla muestra el nombre del plan, el consumo de cada límite ("4 de 5" o
"ilimitado"), el estado de cada acceso de módulo, y la fecha de vencimiento (o "sin
vencimiento").

**Independent Test**: tenant con límite de 5 mesas y 4 creadas, y `plan_vence_en` en el futuro →
la pantalla de plan muestra "4 de 5" para mesas, el estado de cada módulo, y la fecha de
vencimiento (quickstart.md §US6).

**Nota**: el endpoint `GET /plan` ya existe desde la Historia 4 (T038), y ya devuelve
`plan_vence_en`/`vencido` correctamente calculados desde esa misma historia (T037) — esta
historia solo agrega la pantalla que lo consume, reutilizando `plan-summary.service.ts` (T043)
sin duplicarlo.

### Tests for User Story 6

- [X] T049 [P] [US6] Characterization test
  `pos-backend/app/characterization_tests/test_plan_summary.py` — FR-013, Acceptance Scenarios
  1-5, incluye el caso de acceso por un usuario `CASHIER` y los casos con/sin vencimiento
  (quickstart.md §US6, research.md Decisión 7)

### Implementation for User Story 6

- [X] T050 [US6] Crear `pos-heladeria/src/app/modules/plan/pages/plan-summary-page.component.ts`
  (nombre del plan, "usado de límite" o "ilimitado" por recurso, estado de cada módulo, fecha de
  vencimiento o "sin vencimiento") — depende de T043
- [X] T051 [US6] Agregar ruta `mi-plan` en
  `pos-heladeria/src/app/modules/dashboard/routes.ts` (sin `roleGuard` adicional — accesible a
  cualquier usuario autenticado del dashboard, mismo criterio que `mi-cuenta`) — depende de T050

**Checkpoint**: todas las historias de usuario deben ser ahora funcionales de forma independiente.

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: verificación final que atraviesa todas las historias.

- [X] T052 [P] Ejecutar
  `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` en `pos-backend` —
  confirmar que ningún characterization test preexistente se rompe (Principio III) y que los seis
  ficheros nuevos (T011, T020, T029, T035, T047, T049) pasan en verde. **383/383 en verde**
  (ejecutado repetidamente tras cada historia, no solo al final) — 0 characterization tests
  preexistentes rotos, 28 tests nuevos de esta spec, todos en verde
- [X] T053 [P] Ejecutar `ng build --configuration development` en `pos-heladeria` — confirmar que
  todo el código nuevo/modificado compila sin errores de tipos. **Build limpio**, sin errores de
  tipos, ejecutado tras cada historia
- [X] T054 Ejecutar `quickstart.md` paso a paso completo, incluyendo la verificación de migración
  (`alembic upgrade head` contra una base con tenants preexistentes — cero filas con `plan_id IS
  NULL`, comportamiento sin restricciones ni vencimiento preservado) y la prueba de concurrencia
  manual contra Postgres real (FR-015) si SQLite no la ejercita de forma representativa.
  **VERIFICACIÓN PARCIAL en este entorno** (mismo límite que T010/T052 de spec 032 — sin
  Docker/Postgres disponible aquí: `docker ps` → "permission denied"). Completado: los 6 casos
  `python -m unittest` de quickstart.md por historia (§US1-US6), todos en verde; `alembic heads`
  confirma cadena única sin bifurcaciones; `ng build` limpio. Pendiente antes de desplegar:
  `alembic upgrade head` contra Postgres real con tenants preexistentes, y la prueba de
  concurrencia manual de dos terminales contra Postgres real (documentada explícitamente en
  quickstart.md §Verificación de concurrencia contra Postgres, con el caveat correspondiente ya
  registrado en T029/`test_plan_resource_limits.py` sobre por qué SQLite no puede probarla)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — arranca de inmediato.
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA todas las historias de usuario.
- **User Stories (Phase 3-8)**: todas dependen de Foundational. Hay además dependencias
  funcionales de orden entre ellas (no solo de prioridad):
  - **US1 antes que US2**: el Super Admin necesita planes creados (con precios) para poder
    asignarlos a un tenant (T012-T013 antes de que T021-T024 sean observables de punta a punta),
    aunque Foundational (`plan_fixtures.py`, T009) ya alcanza para escribir los tests de US2 en
    paralelo sin pasar por el router de US1.
  - **US4 crea el endpoint que US6 reutiliza**: `GET /plan` (T036-T039) nace en US4 porque el
    guard de navegación lo necesita antes que exista ninguna pantalla de consumo; US6 no repite
    ese trabajo, solo agrega la pantalla (research.md Decisión 7).
  - **US5 depende de que US3 y US4 ya estén en verde**: T048 modifica las mismas dos funciones
    (`enforce_plan_limit`, `require_module_access`) que T030-T034 y T040-T042 ya conectaron a sus
    routers — hacerlo después de que esos tests pasen evita introducir una regresión al tocar
    código compartido ya probado (research.md Decisión 14).
  - **US3 y US4 son independientes entre sí**: ambas dependen solo de Foundational
    (`plan_limits.py`, T007) y tocan ficheros disjuntos — pueden desarrollarse en paralelo, y
    ambas deben completarse antes de US5.
- **Polish (Phase 9)**: depende de que todas las historias que se vayan a entregar estén
  completas.

### Dentro de cada historia

- Tests antes que implementación (T011 antes de T012-T019; T020 antes de T021-T028; T029 antes de
  T030-T034; T035 antes de T036-T046; T047 antes de T048; T049 antes de T050-T051) — escritos
  para fallar primero, per Constitución Principio X. T047 en particular está escrito para fallar
  incluso después de que US3/US4 estén completas, precisamente porque prueba el comportamiento
  que solo T048 agrega.
- Schemas antes que router del mismo endpoint (mismo fichero de router depende del de schemas
  correspondiente).
- Backend antes que frontend dentro de la misma historia (el frontend consume el contrato ya
  implementado).

### Parallel Opportunities

- Foundational: T003 en paralelo con T002 (ficheros distintos); T006/T007/T009 en paralelo entre
  sí una vez completo T003 (T008 depende de T006 y T007, así que no es paralelo con ellos).
- US1: T011 (test backend) y T015 (interfaz frontend) en paralelo — ficheros distintos, sin
  dependencia entre sí.
- US2: T021 y T023 en paralelo (ficheros distintos); T025 en paralelo con el bloque backend
  (T021-T024).
- US3: las cinco tareas de implementación (T030-T034) son completamente paralelas entre sí —
  cinco ficheros distintos, cada uno depende solo de Foundational (T007).
- US4: T042 (promociones) en paralelo con T040-T041 (inventario/compras, mismo fichero entre sí,
  secuenciales); T043 depende de que el backend (T039) esté montado.
- US5: fase deliberadamente secuencial y pequeña (T047 → T048) — no hay paralelismo real posible
  ni necesario en una fase de dos tareas sobre el mismo fichero.
- US6: T049 (test) puede escribirse en paralelo con T050 (frontend) una vez que T043 (Historia 4)
  está listo.
- Distintas historias de usuario **no** se recomiendan en paralelo más allá de arrancar los tests
  de la siguiente historia mientras se termina la actual — US2/US4 comparten ficheros con US1/US3
  respectivamente en algunos puntos de montaje de router, y US5 requiere que US3/US4 ya estén en
  verde antes de tocar `plan_limits.py` de nuevo.

---

## Parallel Example: User Story 3

```bash
# Las cinco tareas de implementación de US3 son independientes entre sí:
Task: "Modificar create_table() en app/api/v1/orders/router.py"
Task: "Modificar create_register() en app/api/v1/cash/router.py"
Task: "Modificar create_user() en app/api/v1/users/router.py"
Task: "Modificar create_product() en app/api/v1/products/router.py"
Task: "Modificar el endpoint de payment methods en app/api/v1/sales/router.py"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (CRÍTICO — bloquea todas las historias)
3. Completar Phase 3: User Story 1
4. **DETENER y VALIDAR**: el Super Admin ya puede administrar el catálogo de planes con precios,
   aunque ningún tenant lo use todavía
5. Desplegar/demostrar si está listo

### Incremental Delivery

1. Completar Setup + Foundational → `Plan`/`Tenant.plan_id` (y sus columnas de ciclo/fechas)
   listos, sin comportamiento nuevo observable todavía (todo tenant existente queda en el plan
   transicional sin restricciones ni vencimiento)
2. Agregar US1 → probar independientemente → demo (catálogo de planes con precios)
3. Agregar US2 → probar independientemente → demo (los tenants ya se crean/cambian/renuevan con
   un plan real y un ciclo de facturación)
4. Agregar US3 → probar independientemente → demo (límites numéricos ya bloquean)
5. Agregar US4 → probar independientemente → demo (módulos ya se deniegan; `GET /plan` nace aquí)
6. Agregar US5 → probar independientemente → demo (el vencimiento ya bloquea límites y módulos
   por igual, sin haber tocado ningún router de nuevo)
7. Agregar US6 → probar independientemente → demo (pantalla de consumo del Tenant Admin, con
   fecha de vencimiento)
8. Cada historia agrega valor sin romper las anteriores

### Parallel Team Strategy

Con más de una persona:

1. El equipo completa Setup + Foundational junto.
2. Una vez lista Foundational:
   - Persona A: User Story 1 (catálogo de planes con precios)
   - Persona B: escribe los tests de User Story 2 contra fixtures (T020) mientras A termina US1,
     luego continúa con la implementación de US2
3. Una vez US1+US2 completas, User Story 3 y User Story 4 pueden repartirse entre dos personas en
   paralelo (ficheros disjuntos, ambas dependen solo de Foundational).
4. User Story 5 empieza solo cuando User Story 3 y User Story 4 están ambas en verde (dependencia
   real sobre código compartido, no solo de prioridad).
5. User Story 6 empieza solo cuando User Story 4 completó `GET /plan` (T038).
