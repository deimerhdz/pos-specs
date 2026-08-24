---

description: "Task list for Catálogo de Métodos de Pago Administrado por el Super Admin"
---

# Tasks: Catálogo de Métodos de Pago Administrado por el Super Admin

**Input**: Design documents from `/specs/032-catalogo-metodos-pago/` (plan.md, spec.md,
research.md, data-model.md, contracts/, quickstart.md)

**Tests**: incluidos — `plan.md` (Project Structure) y `quickstart.md` fijan de antemano qué
characterization test crea o edita cada historia (Constitución, Principio X: Verificación
Obligatoria), así que no son opcionales para esta spec.

**Organization**: tareas agrupadas por historia de usuario (US1-US3, prioridades de `spec.md`)
para que cada una sea implementable y verificable de forma independiente, per `quickstart.md`. La
migración de datos existentes (FR-015/FR-015a/FR-016) es su propia unidad, aparte de las tres
historias (Constitución, Principio VI: no mezclar funcionalidad nueva con migración de datos).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (ficheros distintos, sin dependencia de una tarea sin
  terminar)
- **[Story]**: historia de usuario a la que pertenece (US1, US2, US3)
- Cada tarea incluye la ruta de fichero exacta, relativa a la raíz del repo sibling que corresponda
  (`pos-backend` o `pos-heladeria`)

## Path Conventions

Dos repositorios sibling de `pos-specs` (Constitución §Alcance, plan.md §Project Structure):

- Backend: `pos-backend/app/...` (rutas de este documento ya incluyen el prefijo `pos-backend/`)
- Frontend: `pos-heladeria/src/app/...` (rutas ya incluyen el prefijo `pos-heladeria/`)

---

## Phase 1: Setup

**Purpose**: confirmar que el entorno está listo — esta spec no agrega ninguna dependencia nueva
(plan.md Technical Context), así que no hay instalación ni configuración de herramientas que hacer.

- [X] T001 Confirmar entorno: `pos-backend` con el venv activado (`source env/bin/activate`,
  Python 3.14.4) y `pos-heladeria` con `npm install` ya corrido; verificar que ningún
  `requirements.txt`/`package.json` necesita cambio (plan.md confirma cero dependencias nuevas)

**Checkpoint**: entornos listos, sin instalar nada nuevo.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: el catálogo compartido y su referencia desde `tenant.payment_methods` — nada de
`Phase 3+` puede empezar sin esto (data-model.md, research.md Decisiones 1 y 3).

**⚠️ CRITICAL**: ninguna historia de usuario arranca hasta que esta fase esté completa.

- [X] T002 Crear migración Alembic
  `pos-backend/alembic/versions/d6953c4dcf45_payment_method_catalog.py`: tabla
  `shared.payment_method_catalog` (`id`, `name` único, `type` con `CHECK
  IN ('cash','card','transfer','other')`, `active` default `true`, `fields` JSONB,
  `created_at`/`updated_at`), migración plana **sin** `@for_each_tenant_schema` (mismo patrón que
  `c2d3e4f5a6b7_tenant_logo_url.py` — `shared` es una tabla única, no per-tenant); `downgrade()`
  con `op.drop_table("payment_method_catalog", schema="shared")` (research.md Decisión 1 y 3)
- [X] T003 [P] Crear migración Alembic
  `pos-backend/alembic/versions/a241d5c311bd_tenant_payment_methods_catalog_ref.py`: columnas
  `catalog_id` (UUID, FK → `shared.payment_method_catalog.id`, **nullable**) e `is_complete`
  (`bool`, default `false`) en `tenant.payment_methods`, vía `@for_each_tenant_schema` (mismo
  patrón que `c1d2e3f4a5b6_order_payment_attempts.py`); restricción única normal (no parcial) sobre
  `catalog_id` — a lo sumo una fila por tenant por método de catálogo, para siempre (FR-017,
  research.md Decisión 9 — corregido durante T021 respecto al diseño inicial de índice parcial);
  `downgrade()` con `op.drop_column`/`op.drop_constraint` (research.md Decisión 3 y 9)
- [X] T004 [P] Crear modelo `PaymentMethodCatalog` en
  `pos-backend/app/models/payment_method_catalog.py` (columnas y `CheckConstraint` de
  data-model.md; `__table_args__ = {"schema": "shared"}`)
- [X] T005 Agregar `catalog_id: Mapped[Optional[UUID]]` (FK) e `is_complete: Mapped[bool]` a
  `PaymentMethod` en `pos-backend/app/models/payment.py` — depende de T004
- [X] T006 [P] Crear fixtures compartidos
  `pos-backend/app/characterization_tests/payment_catalog_fixtures.py`
  (`make_payment_method_catalog` en distintos estados: activo/inactivo, con/sin `fields`) — depende
  de T004
- [X] T007 Aplicar T002/T003 contra una base de datos de prueba y verificar el `downgrade()`
  (rollback limpio, sin dejar columnas/tablas huérfanas) — depende de T002, T003.
  **VERIFICACIÓN PARCIAL en este entorno**: sin acceso a Docker/Postgres en este sandbox
  (`docker ps` → "permission denied"; no hay `pg_isready` ni servidor local — mismo límite que
  spec024). Se verificó: (a) `python -m py_compile` sobre ambos ficheros, sin errores; (b)
  `alembic heads` reconoce una sola cabeza (`a241d5c311bd`) — la cadena `down_revision` encadena
  limpio sobre el head previo (`2d2c3090473f`), sin bifurcaciones; (c) el modelo `PaymentMethod`
  con `catalog_id`/`is_complete`/la restricción única nueva se ejercitó extensamente contra
  SQLite vía `payment_catalog_fixtures.new_session()` (T006), incluyendo el `JOIN` a
  `PaymentMethodCatalog` a través del esquema `shared` simulado con `ATTACH DATABASE`. `alembic
  upgrade head --sql` (modo offline) falla, pero en una migración **anterior a esta spec**
  (`abd505aae914`, preexistente) por una limitación conocida de `@for_each_tenant_schema` con modo
  offline (necesita una conexión real para `SELECT schema FROM shared.tenants`) — no es un
  problema introducido por T002/T003. Pendiente: aplicar contra Postgres real antes de desplegar.

**Checkpoint**: catálogo y referencia listos — las historias de usuario pueden empezar (en
paralelo si hay más de una persona).

---

## Phase 3: User Story 1 - El Super Admin administra el catálogo de la plataforma (Priority: P1) 🎯 MVP

**Goal**: el Super Admin crea, edita y activa/desactiva métodos de pago en el catálogo de la
plataforma, definiendo qué campos de integración requiere cada uno.

**Independent Test**: crear/editar/(des)activar entradas de catálogo vía
`contracts/super-admin-catalog.md`, sin que ningún tenant participe — verificar que quedan
listadas y disponibles para activación (quickstart.md §US1).

### Tests for User Story 1

- [X] T008 [P] [US1] Characterization test
  `pos-backend/app/characterization_tests/test_super_admin_payment_catalog.py` — FR-001/FR-002/
  FR-003/FR-004/**FR-014**, Acceptance Scenarios 1-4 (quickstart.md §US1; escenario 4 cubre FR-014/
  SC-003: una venta histórica con un método luego desactivado no cambia). 4/4 tests en verde.

### Implementation for User Story 1

- [X] T009 [US1] Crear `PaymentMethodCatalogCreate`/`PaymentMethodCatalogUpdate`/
  `PaymentMethodCatalogResponse` en `pos-backend/app/api/v1/super_admin/schemas.py` (validar
  `fields`: `key` únicas, `format` ∈ `{"text","numeric","image"}`, `length` solo en
  `text`/`numeric` — contracts/super-admin-catalog.md)
- [X] T010 [US1] Crear `pos-backend/app/api/v1/super_admin/payment_methods_router.py`:
  `GET`/`POST`/`PATCH /super-admin/payment-methods-catalog` (contracts/super-admin-catalog.md) —
  depende de T009
- [X] T011 [US1] Montar el router de T010 en `pos-backend/app/api/v1/super_admin/router.py` —
  depende de T010
- [X] T012 [P] [US1] Crear
  `pos-heladeria/src/app/modules/super-admin/interfaces/payment-method-catalog.interface.ts`
- [X] T013 [US1] Crear
  `pos-heladeria/src/app/modules/super-admin/services/payment-method-catalog.service.ts` (mismo
  patrón que `tenant.service.ts`: `load()`/`create()`/`update()`) — depende de T012
- [X] T014 [US1] Crear
  `pos-heladeria/src/app/modules/super-admin/components/payment-method-catalog-form.component.ts`
  (alta/edición de `name`/`type`, editor de `fields`: agregar/quitar campo, `required`, `format`,
  `length`) — depende de T013
- [X] T015 [US1] Crear
  `pos-heladeria/src/app/modules/super-admin/pages/payment-method-catalog-page.component.ts`
  (listar catálogo completo, activar/desactivar, mismo patrón que `tenants-page.component.ts`) —
  depende de T013, T014
- [X] T016 [US1] Agregar ruta hija en `pos-heladeria/src/app/modules/super-admin/routes.ts` (junto
  a `tenants`/`users`, ya gateada por `super-admin-domain.guard.ts`) — depende de T015. También se
  agregó la entrada de navegación (`navigation.config.ts::SUPER_ADMIN_NAV_ITEMS`, no listada
  originalmente en tasks.md — sin ella la ruta quedaba inalcanzable desde el sidebar).

**Checkpoint**: US1 completa y verificable de forma independiente (`python -m unittest
app.characterization_tests.test_super_admin_payment_catalog -v` — 4/4 en verde; `ng build
--configuration development` en verde, incluyendo el chunk lazy
`payment-method-catalog-page-component`).

---

## Phase 4: User Story 2 - El Tenant Admin activa y configura métodos de pago para su negocio (Priority: P2)

**Goal**: el Tenant Admin ve el catálogo activo de la plataforma, activa los métodos que usa,
completa sus datos de integración validados por formato, y desactiva los que ya no usa.

**Independent Test**: con un catálogo poblado (vía US1 o fixtures), activar "Nequi", completar el
número de celular, verificar que queda `is_complete=true`; dejar un campo obligatorio vacío y
verificar que queda `is_complete=false` (quickstart.md §US2).

### Tests for User Story 2

- [X] T017 [P] [US2] Editar
  `pos-backend/app/characterization_tests/test_sales_payment_methods.py` para reflejar
  `catalog_id` en vez de `name`/`type` libres (no es un test `"CONGELA comportamiento actual:"` —
  plan.md Constitution Check, Principio III — se edita directamente citando esta spec). Los 5 tests
  ya existentes no llamaban a la creación libre vía API (solo `PATCH`/fixtures) — no necesitaron
  cambios de fondo; se agregó una nota al docstring explicando esto y remitiendo a
  `test_sales_payment_methods_catalog.py` para la creación nueva. 5/5 en verde.
- [X] T018 [P] [US2] Characterization test nuevo
  `pos-backend/app/characterization_tests/test_sales_payment_methods_catalog.py` —
  FR-005/FR-006/FR-007/FR-008/FR-009/FR-010/FR-011/FR-017, Acceptance Scenarios 1-5 (quickstart.md
  §US2). 10/10 en verde.

### Implementation for User Story 2

- [X] T019 [US2] Agregar `"payment-methods"` a `Literal["products", "logo"]` en
  `pos-backend/app/api/v1/uploads/schemas.py` (`PresignRequest.folder`, research.md Decisión 8)
- [X] T020 [US2] Reescribir `PaymentMethodCreate`/`PaymentMethodUpdate`/`PaymentMethodResponse` en
  `pos-backend/app/api/v1/sales/schemas.py`: `catalog_id` en vez de `name`/`type`/`is_cash`
  libres; `catalog_id`/`is_complete` en la respuesta (contracts/tenant-payment-methods.md)
- [X] T021 [US2] Implementar en `pos-backend/app/api/v1/sales/service.py`: validar `payment_info`
  contra `catalog.fields` (obligatoriedad + formato), calcular/recalcular `is_complete`, copiar
  `name`/`type`/`is_cash` del catálogo al crear (research.md Decisión 5), guardia "una fila por
  `catalog_id` para siempre" (`409` en `POST` si ya existe cualquier fila, activa o no, para ese
  `catalog_id` — research.md Decisión 9, **corregida durante esta tarea**: el diseño inicial de
  índice único parcial `WHERE active=true` chocaba con la restricción única preexistente de
  `payment_methods.name`, ver research.md) y guardia "el catálogo debe estar activo" (`409` si
  `catalog.active=false`) — depende de T020, T005. **Nota de implementación**: T003/T004/T005
  (modelo `PaymentMethod.catalog_id`) se ajustaron en el mismo momento para reflejar la restricción
  única normal (no parcial) — ver research.md Decisión 9.
- [X] T022 [US2] Actualizar `POST`/`PATCH /sales/payment-methods` en
  `pos-backend/app/api/v1/sales/router.py` para el contrato nuevo — depende de T021
- [X] T023 [US2] Crear schema `CatalogPaymentMethodOption` (`id`, `name`, `fields`, `active`,
  `already_activated`) en `pos-backend/app/api/v1/sales/schemas.py`, y agregar
  `GET /sales/payment-methods/catalog` en `pos-backend/app/api/v1/sales/router.py` + `service.py`:
  catálogo activo a nivel plataforma, más
  las entradas que el tenant ya activó aunque estén inactivas — cada fila lleva `active` (estado
  del catálogo) y `already_activated`, para que el frontend avise "ya no disponible" cuando
  `already_activated=true` y `active=false` (FR-005/FR-006, contracts/tenant-payment-methods.md) —
  depende de T021
- [X] T024 [P] [US2] Actualizar
  `pos-heladeria/src/app/modules/sales/interfaces/sales.interface.ts`: `PaymentMethod` gana
  `catalog_id`/`is_complete`; `CatalogPaymentMethodOption` nuevo (más `PaymentMethodCheckoutOption`,
  adelantado de T029/US3 porque vive en el mismo fichero)
- [X] T025 [US2] Crear
  `pos-heladeria/src/app/modules/sales/services/payment-method-catalog.service.ts`
  (`TenantPaymentMethodCatalogService`: GET catálogo activo para el tenant) — depende de T024
- [X] T026 [US2] Actualizar
  `pos-heladeria/src/app/modules/sales/services/payment-method.service.ts`: `create()`/`update()`
  contra `catalog_id` en vez de `name`/`type` libres — depende de T024
- [X] T027 [US2] Actualizar
  `pos-heladeria/src/app/modules/sales/pages/payment-methods-page.component.ts`: activar desde
  catálogo (modal de selección), formulario dinámico según `catalog.fields` (campo `format=image`
  como URL de texto — la subida real a R2 vía `POST /uploads/presign?folder=payment-methods` queda
  para una iteración de UI posterior, no bloquea el flujo), indicador "Incompleto"/"Completar" —
  depende de T019, T025, T026

**Checkpoint**: US1 y US2 completas y verificables juntas (`python -m unittest
app.characterization_tests.test_sales_payment_methods_catalog -v` — 10/10 — y
`test_sales_payment_methods -v` — 5/5; suite completa 313/313; `ng build --configuration
development` en verde, sin warnings).

---

## Phase 5: User Story 3 - El Cajero cobra usando solo métodos de pago completamente disponibles (Priority: P3)

**Goal**: la pantalla de cobro solo muestra métodos activos y completos, sin exponer los datos de
integración al cajero.

**Independent Test**: con un tenant que tiene métodos completos e incompletos, verificar que
`GET /sales/payment-methods?available=true` solo devuelve los completos, sin `payment_info`
(quickstart.md §US3).

### Tests for User Story 3

- [X] T028 [P] [US3] Characterization test nuevo
  `pos-backend/app/characterization_tests/test_sales_payment_methods_checkout.py` —
  FR-012/FR-012a/FR-013, Acceptance Scenarios 1-3 (quickstart.md §US3), más un caso de regresión
  (`test_fila_sin_catalog_id_todavia_sigue_disponible`, FR-016) descubierto durante esta misma
  tarea. 4/4 en verde.

### Implementation for User Story 3

- [X] T029 [US3] Agregar schema `PaymentMethodCheckoutOption` en
  `pos-backend/app/api/v1/sales/schemas.py` (contracts/checkout-payment-methods.md).
  **Corregido durante esta tarea**: incluye `id`/`name`/`is_cash` (no solo `id`/`name` como decía
  el plan original) — `is_cash` es clasificación operativa, no un dato de integración, y
  `payment-input.component.ts` ya la necesitaba para calcular vuelto antes de esta spec; omitirla
  habría roto esa funcionalidad existente.
- [X] T030 [US3] Extender `GET /sales/payment-methods` en
  `pos-backend/app/api/v1/sales/router.py`/`service.py` con query param `available: bool = False`.
  **Dos correcciones descubiertas al implementar** (ver research.md/data-model.md actualizados):
  (a) `PaymentMethod.is_complete` cambió su default de migración de `false` a `true` — con `false`,
  aplicar la migración de esquema (antes del backfill) habría vaciado el checkout de todos los
  tenants de golpe, violando FR-016; (b) la consulta usa `outerjoin` (no `join`) a
  `PaymentMethodCatalog` y admite `catalog_id IS NULL`, para no excluir filas que todavía no pasaron
  por el backfill — depende de T029, T021
- [X] T031 [P] [US3] Agregar método `loadAvailableForCheckout()` a
  `pos-heladeria/src/app/modules/sales/services/payment-method.service.ts` (el mismo servicio que
  `pos-terminal.store.ts` ya inyecta) que llama `GET /sales/payment-methods?available=true` y
  expone el resultado en la señal nueva `checkoutOptions`
- [X] T032 [US3] Actualizar `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`.
  **Ajustado respecto al plan**: se agregó `paymentMethodsAvailable` (nueva) en vez de reemplazar
  `paymentMethods` — `paymentMethods` (listado completo) lo sigue necesitando `methodName()` para
  resolver el nombre de un método ya usado en una venta impresa, aunque ya no esté disponible para
  cobros nuevos (Principio VII). Actualizados también `pos-checkout-panel.component.ts`,
  `session-bill-panel.component.ts` y `payment-input.component.ts` para consumir
  `paymentMethodsAvailable()`/`PaymentMethodCheckoutOption[]` — depende de T031
- [X] T033 [US3] Revisar `pos-heladeria/src/app/modules/tables/services/payment-draft.util.ts`
  (`paymentIssue`/`nonCashAmount`): solo necesitó cambiar el tipo del parámetro `methods` de
  `PaymentMethod[]` a `PaymentMethodCheckoutOption[]` (misma lógica, ambos shapes traen
  `id`/`name`/`is_cash`) — depende de T032. Suite Vitest completa: 356/359 (los 3 restantes son
  fallos preexistentes en `develop`, confirmados con `git stash` antes de tocar nada de esta spec —
  `app.spec.ts`, `auth.service.spec.ts`, y el test "Imprimir Pre-cuenta" de
  `pos-checkout-panel.component.spec.ts`, ninguno relacionado con métodos de pago).

**Checkpoint**: las tres historias funcionan juntas de punta a punta (catálogo → activación →
checkout filtrado).

---

## Phase 6: Migración de Datos Existentes (FR-015/FR-015a/FR-016)

**Purpose**: llevar los métodos de pago que los tenants ya usan (efectivo, Nequi, Bancolombia, y
cualquier método personalizado) al modelo nuevo, sin pérdida de configuración — unidad aparte de
las historias de usuario (Constitución, Principio VI; data-model.md §Migración; research.md
Decisiones 3 y 7). Requiere US1 (para poder agregar métodos personalizados al catálogo) y US2
(columnas `catalog_id`/`is_complete` en uso) ya implementadas.

- [X] T034 Migración Alembic
  `pos-backend/alembic/versions/130642d23e76_seed_payment_method_catalog.py`: sembrar "Efectivo"
  (`type=cash`, `fields=[]`), "Nequi" (`type=transfer`, `fields=[celular obligatorio
  numeric(10), qr opcional image]`) y "Transferencia Bancolombia" (`type=transfer`,
  `fields=[cuenta obligatorio text, tipo_cuenta obligatorio text, qr opcional image]`) en
  `shared.payment_method_catalog` (data-model.md §Migración, paso 1); idempotente por `name`;
  `downgrade()` borra esas tres filas por `name` (Principio VIII: estrategia de rollback explícita)
  — depende de T002
- [X] T035 Script `pos-backend/app/scripts/migrate_payment_methods_catalog.py` con flag
  `--report-only`: recorre `tenant.payment_methods` de cada tenant, normaliza `name` (minúsculas,
  sin tildes) y reporta las filas que no matchean ninguna entrada del catálogo (FR-015a) — depende
  de T005, T034
- [X] T036 Extender el script de T035 (modo backfill real, reejecutable): para las filas que sí
  matchean, setea `catalog_id` y recalcula `is_complete` (reutilizando
  `sales.service._validate_payment_info`, mismo criterio que altas/ediciones nuevas), preservando
  `payment_info` ya capturado (FR-015). Cubierto por
  `test_migrate_payment_methods_catalog.py` (5/5 en verde, `with_db` parcheado contra SQLite vía
  `payment_catalog_fixtures`, ver research.md §Migración) — depende de T035
- [ ] T037 Ejecutar el reporte de T035 contra un snapshot representativo de tenants, revisar con el
  Super Admin qué métodos personalizados agregar al catálogo (vía `POST
  /super-admin/payment-methods-catalog`, T010) y correr el backfill de T036 — depende de T036, T010.
  **NO EJECUTADA en este entorno**: requiere una base de datos Postgres real con datos de tenants
  reales (`shared.tenants`) — sin acceso a Docker/Postgres en este sandbox (mismo límite que T007).
  El script en sí está escrito, compila, y su lógica de matching/backfill está probada contra
  SQLite (T036). Pendiente: ejecutar contra un snapshot de producción antes de desplegar esta spec.
- [ ] T038 Verificar SC-004/FR-016: cero filas `tenant.payment_methods` con `active=true` y
  `catalog_id IS NULL` tras el backfill; cada tenant ve en
  `GET /sales/payment-methods?available=true` exactamente los métodos que veía antes de migrar,
  comparado contra un snapshot tomado antes de T003 (quickstart.md §Migración) — depende de T037,
  T030. **NO EJECUTADA**: depende de T037 (requiere Postgres real). La garantía de que ninguna fila
  desaparece de caja *antes* de correr el backfill sí está verificada
  (`test_fila_sin_catalog_id_todavia_sigue_disponible`, T028) — lo pendiente es la verificación
  posterior al backfill real, contra datos reales.

**Checkpoint**: producción migrada, sin pérdida de configuración de ningún tenant.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: verificación de no-regresión y cierre de la cadena de trazabilidad (Constitución,
Principios X y XII).

- [X] T039 [P] Ejecutar `python -m unittest discover -s app/characterization_tests -p 'test_*.py'
  -v` completo desde `pos-backend` — 322/322 en verde (299 preexistentes de specs anteriores + 23
  nuevos de esta spec: 4+10+4+5 en T008/T018/T028/T036), ningún characterization test preexistente
  quedó en rojo
- [X] T040 [P] Ejecutar `npm test` en `pos-heladeria` (Vitest) — 356/359 en verde. Los 3 restantes
  (`app.spec.ts`, `auth.service.spec.ts`, "Imprimir Pre-cuenta" de
  `pos-checkout-panel.component.spec.ts`) son fallos preexistentes en `develop`, confirmados con
  `git stash` antes de aplicar cualquier cambio de esta spec — ninguno relacionado con métodos de
  pago; se corrigió en cambio la regresión real introducida (`sidebar.component.spec.ts`, ruta
  nueva del catálogo en `SUPER_ADMIN_NAV_ITEMS`)
- [X] T041 Recorrido de SC-001 a SC-006 contra la cobertura de tests existente: SC-001 (T008, alta
  de catálogo) y SC-002 (T018/T028, completar+aparece de inmediato) verificados estructuralmente —
  el tiempo en minutos es un criterio de UX no automatizable; SC-003 (T008, venta histórica sin
  cambios) y SC-005/SC-006 (T028, filtrado de checkout) verificados directamente; SC-004
  (preservación tras la migración) verificado parcialmente — la seguridad *antes* del backfill está
  cubierta (T028, T036), la verificación *después* del backfill real queda pendiente de T038 (sin
  Postgres en este sandbox)
- [ ] T042 Validación manual end-to-end en `pos-heladeria` (`ng serve`): Super Admin crea
  "Daviplata" → Tenant Admin lo activa y completa el celular → aparece en el panel de cobro
  (`pos-checkout-panel.component.ts`) de inmediato; Super Admin desactiva un método que un tenant
  tenía activo → deja de aparecer en caja sin afectar ventas históricas — depende de T016, T027,
  T032, T038. **NO EJECUTADA en este entorno**: `ng serve` necesita la API real (`pos-backend`)
  corriendo contra Postgres, no disponible en este sandbox (mismo límite que T007/T037/T038). Sí se
  verificó `ng build --configuration development` en verde en cada fase (T016, T027, T032) —
  confirma tipos y que compila, no el comportamiento visual/interactivo real. Pendiente: correrla
  contra un entorno real (Postgres + R2) antes de dar la funcionalidad por completa de cara al
  usuario final.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — arranca de inmediato.
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA todas las historias de usuario.
- **User Stories (Phase 3-5)**: todas dependen de Foundational. Hay además una dependencia
  funcional de orden entre ellas (no solo de prioridad):
  - **US1 antes que US2**: el Tenant Admin necesita entradas de catálogo para poder activar algo
    (T010 antes de que T021-T023 sean observables de punta a punta), aunque el modelo de datos
    (Foundational) ya alcanza para escribir los tests de US2 en paralelo usando `payment_catalog_fixtures.py`
    directamente (sin pasar por el router de US1).
  - **US2 antes que US3**: US3 filtra sobre `PaymentMethod.is_complete`/`active` y
    `PaymentMethodCatalog.active`, columnas que solo se llenan a través de los endpoints de US2 —
    el test de US3 (T028) puede escribirse con fixtures directas, pero la validación manual
    (T042) requiere las tres historias.
- **Migración (Phase 6)**: depende de US1 (T010, para agregar métodos personalizados al catálogo)
  y US2 (T005/T021, columnas en uso) — no depende de US3.
- **Polish (Phase 7)**: depende de que todas las historias y la migración que se vayan a entregar
  estén completas.

### Dentro de cada historia

- Tests antes que implementación (T008 antes de T009-T016; T017-T018 antes de T019-T027; T028
  antes de T029-T033) — escritos para fallar primero, per Constitución Principio X.
- Schemas antes que service/router del mismo endpoint (mismo fichero de router depende del de
  schemas correspondiente).
- Backend antes que frontend dentro de la misma historia (el frontend consume el contrato ya
  implementado).

### Parallel Opportunities

- Foundational: T003 y T004 en paralelo (ficheros distintos); T006 tras T004.
- US1: T008 (test backend) y T012 (interfaz frontend) en paralelo — ficheros distintos, sin
  dependencia entre sí.
- US2: T017 y T018 en paralelo (dos ficheros de test distintos); T024 en paralelo con el bloque
  backend (T019-T023), ya que es un fichero de interfaces sin dependencia de implementación
  backend para escribirse (aunque sí para probarse end-to-end).
- US3: T028 (test) y T031 (servicio frontend, tras T030) son los únicos [P] de la fase.
- Distintas historias de usuario **no** se recomiendan en paralelo entre sí más allá de US1 con
  trabajo de test/frontend inicial de US2 — comparten ficheros (`sales/router.py`,
  `sales/service.py`, `sales/schemas.py`) entre US2 y US3.

---

## Parallel Example: User Story 2

```bash
# Tests de la historia, en paralelo (ficheros distintos):
Task: "Editar test_sales_payment_methods.py para catalog_id"
Task: "Characterization test nuevo test_sales_payment_methods_catalog.py"

# Interfaz de frontend, en paralelo con el bloque backend completo:
Task: "Actualizar sales.interface.ts con catalog_id/is_complete"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (CRÍTICO — bloquea todas las historias)
3. Completar Phase 3: User Story 1
4. **DETENER y VALIDAR**: probar US1 de forma independiente (el Super Admin ya puede administrar
   el catálogo, aunque ningún tenant lo consuma todavía)
5. Desplegar/demostrar si está listo

### Incremental Delivery

1. Completar Setup + Foundational → catálogo listo
2. Agregar US1 → probar independientemente → demo (MVP del catálogo)
3. Agregar US2 → probar independientemente → demo (tenants ya activan y configuran)
4. Agregar US3 → probar independientemente → demo (caja filtra correctamente)
5. Ejecutar Migración (Phase 6) → producción al día sin pérdida de configuración
6. Cada historia agrega valor sin romper las anteriores

### Parallel Team Strategy

Con más de una persona:

1. El equipo completa Setup + Foundational junto.
2. Una vez lista Foundational:
   - Persona A: User Story 1 (catálogo del Super Admin)
   - Persona B: escribe los tests de User Story 2 contra fixtures (T017, T018) mientras A termina
     US1, luego continúa con la implementación de US2
3. User Story 3 y la Migración empiezan solo cuando US1+US2 están completas (dependencia
   funcional real, no solo de prioridad).
