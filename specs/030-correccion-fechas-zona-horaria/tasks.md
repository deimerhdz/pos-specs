---

description: "Task list template for feature implementation"
---

# Tasks: Corrección global de fechas, horas y zonas horarias

**Input**: Design documents from `/specs/030-correccion-fechas-zona-horaria/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: `FR-010` exige explícitamente un test mínimo por Ventas/Órdenes/Pagos/Caja/Inventario más
un test de valor histórico sin cambios — los tests de este documento no son opcionales, son parte del
contrato de la spec.

**Repos involucrados**: esta spec vive en `pos-specs`, pero el código está en `../pos-backend` y
`../pos-heladeria` (siblings de este repo). Todas las rutas de archivo son relativas a la raíz de cada
uno de esos dos repos, tal como en `plan.md` → Project Structure.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1..US6)
- Include exact file paths in descriptions

---

## Phase 1: Setup

**Purpose**: Verificación de entorno — no hay dependencias nuevas que instalar (Principio IX no
aplica, ver plan.md → Primary Dependencies) ni estructura de proyecto que crear.

- [X] T001 Verificar `SHOW timezone;` contra la instancia real de PostgreSQL y confirmar ausencia de
      `TZ=` distinto de UTC en `../pos-backend/docker-compose.yml` y
      `../pos-backend/docker-compose.prod.yml` (research.md Assumptions — último paso de verificación
      recomendado antes de implementar)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: mecanismo central por repo del que dependen las seis historias de usuario (FR-002)

**⚠️ CRITICAL**: ninguna historia de usuario puede implementarse hasta que esta fase esté completa

- [X] T002 Crear módulo `../pos-backend/app/core/timezone.py` con `resolve_timezone(tenant) ->
      ZoneInfo`, el tipo `UtcDatetime` (`Annotated[datetime, PlainSerializer(...)]`),
      `local_day_bounds_utc(day, tz) -> tuple[datetime, datetime]` y `utc_now() -> datetime`
      (research.md Decisiones 1-2, data-model.md)
- [X] T003 [P] Agregar columna `timezone` (`String`, `NOT NULL`, `server_default='America/Bogota'`) a
      la clase `Tenant` en `../pos-backend/app/core/models.py`, más validador `@validates('timezone')`
      que ejecuta `zoneinfo.ZoneInfo(value)` y rechaza valores no reconocidos (research.md Decisión 3,
      FR-005, Clarifications)
- [X] T004 Generar y aplicar la migración Alembic en
      `../pos-backend/alembic/versions/<hash>_add_tenant_timezone.py` (`alembic revision
      --autogenerate -m "add_tenant_timezone"`, revisar `server_default='America/Bogota'` y
      `nullable=False`, `alembic upgrade head`; `downgrade()` elimina la columna — Principio VIII)
- [X] T005 [P] Agregar campo `timezone: str` (solo lectura) a `TenantInfoResponse` en
      `../pos-backend/app/api/v1/tenant/schemas.py`; confirmar que `TenantUpdateRequest` NO lo incluye
      (contracts/tenant-info-endpoint.md)
- [X] T006 [P] Crear `TenantDatePipe` (standalone) en
      `../pos-heladeria/src/app/shared/pipes/tenant-date.pipe.ts`, envolviendo el `DatePipe` nativo de
      `@angular/common` con `TenantInfoService.info()?.timezone ?? 'America/Bogota'` como tercer
      argumento (research.md Decisión 6)
- [X] T007 [P] Crear `../pos-heladeria/src/app/shared/date-format.util.ts` con
      `businessToday(timezone: string): string` usando `Intl.DateTimeFormat('en-CA', { timeZone })`
      (research.md Decisión 7)
- [X] T008 [P] Agregar `timezone: string` a la interfaz `TenantInfo` en
      `../pos-heladeria/src/app/core/tenant/tenant-info.service.ts` (contracts/tenant-info-endpoint.md)

**Checkpoint**: mecanismo central listo en ambos repos — las historias de usuario pueden implementarse

---

## Phase 3: User Story 1 - Una venta muestra la hora real en que ocurrió (Priority: P1) 🎯 MVP

**Goal**: corregir el defecto reportado — `SaleResponse.sold_at` viaja con offset UTC explícito y se
muestra convertido a la hora de Bogotá en el listado/detalle de Ventas.

**Independent Test**: crear una venta con el reloj fijado a un instante conocido, consultar el
listado/detalle de Ventas inmediatamente después, verificar que la hora mostrada coincide con la hora
de Bogotá de ese instante, no con la UTC cruda.

### Tests for User Story 1 ⚠️

- [X] T009 [P] [US1] Test de serialización `sold_at` con offset UTC explícito en
      `../pos-backend/app/characterization_tests/test_sales_timezone.py::TestSaleSoldAtUtcSerialization`
      (quickstart.md Paso 3)
- [X] T010 [P] [US1] Test de filtro "venta a las 23:59 Bogotá incluida en el día correcto" en el mismo
      fichero `::TestSalesFiltroMedianocheBogota` (quickstart.md Paso 3; el filtro real se implementa
      en Fase 6/US3, este test documenta el contrato desde ya)
- [X] T011 [P] [US1] Test `TenantDatePipe` formatea en la zona del tenant, no la del navegador, en
      `../pos-heladeria/src/app/shared/pipes/tenant-date.pipe.spec.ts` (quickstart.md Paso 12)

### Implementation for User Story 1

- [X] T012 [US1] Cambiar `SaleResponse.sold_at: datetime` → `UtcDatetime` en
      `../pos-backend/app/api/v1/sales/schemas.py:134` (import desde `app.core.timezone`)
- [X] T013 [US1] Agregar `PaymentResponse.paid_at: UtcDatetime` (campo nuevo) en
      `../pos-backend/app/api/v1/sales/schemas.py:94-100` (research.md Decisión 9)
- [X] T014 [US1] Reemplazar `| date: 'dd/MM/yyyy HH:mm'` por `| tenantDate: 'dd/MM/yyyy HH:mm'` en
      `../pos-heladeria/src/app/modules/sales/pages/sales-page.component.ts:108,155`
- [X] T015 [US1] Confirmar en verde `python3 -m unittest
      app.characterization_tests.test_sales_timezone -v` (desde `../pos-backend`) y `npm test --
      tenant-date.pipe.spec.ts sales-page.component.spec.ts` (desde `../pos-heladeria`)

**Checkpoint**: Ventas muestra la hora real de Bogotá — defecto reportado corregido y verificable de
forma independiente

---

## Phase 4: User Story 6 - Ningún dato histórico ya registrado cambia de valor (Priority: P1)

**Goal**: garantizar que la corrección no recalcula ni altera ningún valor almacenado (Principio VII).

**Independent Test**: comparar, para un registro existente, su valor almacenado en base de datos antes
y después de desplegar el cambio — debe ser idéntico.

- [X] T016 [US6] Escribir `test_valor_almacenado_no_cambia` en
      `../pos-backend/app/characterization_tests/test_sales_timezone.py`: crear una venta, ejecutar la
      ruta de lectura/serialización introducida por US1, hacer `db.refresh(sale)` y comparar
      `sale.sold_at` antes/después — debe ser idéntico (quickstart.md Paso 6, FR-007/FR-010)
- [X] T017 [US6] Confirmar en verde `python3 -m unittest
      app.characterization_tests.test_sales_timezone -v`

**Checkpoint**: valor histórico de Ventas verificado como intacto — mismo patrón se reutiliza al cerrar
cada entidad en la Fase 5

---

## Phase 5: User Story 2 - Todas las entidades usan el mismo mecanismo (Priority: P1)

**Goal**: aplicar `UtcDatetime` a las diez entidades restantes (Órdenes, Pagos/Intentos de pago, Caja,
Inventario, Mesas, Facturas, Compras, Auditoría) y unificar los ocho sitios de "ahora" (FR-008).

**Independent Test**: para cada entidad listada, verificar que su fecha se muestra convertida a la
hora del negocio usando el mismo mecanismo central (no una implementación por módulo).

### Schemas de respuesta → `UtcDatetime`

- [X] T018 [P] [US2] `OrderResponse.created_at: UtcDatetime` en
      `../pos-backend/app/api/v1/orders/schemas.py:182`
- [X] T019 [P] [US2] `PaymentAttemptResponse.created_at/resolved_at: UtcDatetime` en
      `../pos-backend/app/api/v1/orders/schemas.py:213-214`
- [X] T020 [P] [US2] `ShiftResponse.opened_at/closed_at` y `ShiftSummaryResponse` (mismos campos) a
      `UtcDatetime` en `../pos-backend/app/api/v1/cash/schemas.py:52-53,153-154`
- [X] T021 [P] [US2] `CashMovementResponse.occurred_at: UtcDatetime` en
      `../pos-backend/app/api/v1/cash/schemas.py:77`
- [X] T022 [P] [US2] `PartialCountResponse.counted_at: UtcDatetime` en
      `../pos-backend/app/api/v1/cash/schemas.py:130`
- [X] T023 [P] [US2] `MovementResponse.moved_at: UtcDatetime` en
      `../pos-backend/app/api/v1/inventory/schemas.py:64`
- [X] T024 [P] [US2] `PurchaseResponse.purchased_at: UtcDatetime` en
      `../pos-backend/app/api/v1/inventory/schemas.py:125`
- [X] T025 [P] [US2] `TableSessionResponse.opened_at/closed_at: UtcDatetime` en
      `../pos-backend/app/api/v1/table_sessions/schemas.py:36-37`
- [X] T026 [P] [US2] `ParticipantResponse.joined_at/closed_at: UtcDatetime` en
      `../pos-backend/app/api/v1/table_sessions/schemas.py:25,27`
- [X] T027 [P] [US2] `InvoiceResponse.issued_at: UtcDatetime` en
      `../pos-backend/app/api/v1/invoices/schemas.py:32`
- [X] T028 [P] [US2] `AuditLogResponse.at: UtcDatetime` en
      `../pos-backend/app/api/v1/audit/router.py:19-27`

### Unificación de "ahora" (FR-008, research.md Decisión 8 — solo los 8 sitios citados en Edge Cases)

- [X] T029 [US2] Reemplazar `datetime.now(timezone.utc)` por `utc_now()` en
      `../pos-backend/app/api/v1/cash/router.py:121`
- [X] T030 [US2] Reemplazar `datetime.now(timezone.utc)` por `utc_now()` en
      `../pos-backend/app/api/v1/orders/checkout.py:811,819,873,925,933`
- [X] T031 [US2] Reemplazar `datetime.now(timezone.utc)` por `utc_now()` en
      `../pos-backend/app/core/qr_context.py:85,179`
- [X] T032 [US2] Reemplazar `datetime.now(timezone.utc)` por `utc_now()` en
      `../pos-backend/app/api/v1/table_sessions/service.py:177,652,739`

### Tests por entidad (FR-010: mínimo uno por Órdenes/Pagos/Caja/Inventario)

- [X] T033 [P] [US2] Test de serialización con offset UTC para Órdenes en
      `../pos-backend/app/characterization_tests/test_orders_timezone.py`
- [X] T034 [P] [US2] Test de serialización con offset UTC para Pagos en
      `../pos-backend/app/characterization_tests/test_payments_timezone.py`
- [X] T035 [P] [US2] Test de serialización con offset UTC para Caja (turno, movimiento, arqueo) en
      `../pos-backend/app/characterization_tests/test_cash_timezone.py`
- [X] T036 [P] [US2] Test de serialización con offset UTC para Inventario (movimiento y compra) en
      `../pos-backend/app/characterization_tests/test_inventory_timezone.py`

### Frontend — resto de sitios `| date` y formateador manual

- [X] T037 [P] [US2] `| date` → `| tenantDate` en
      `../pos-heladeria/src/app/modules/dashboard/pages/admin-dashboard.component.ts:123`
- [X] T038 [P] [US2] `| date` → `| tenantDate` en
      `../pos-heladeria/src/app/modules/inventory/pages/inventory-page.component.ts:212,297`
- [X] T039 [P] [US2] `| date` → `| tenantDate` en
      `../pos-heladeria/src/app/modules/orders/pages/order-detail.component.ts:64`
- [X] T040 [P] [US2] `| date` → `| tenantDate` en
      `../pos-heladeria/src/app/modules/orders/pages/orders-page.component.ts:92`
- [X] T041 [US2] Reemplazar `new Date(iso).toLocaleString(...)` por `TenantDatePipe` inyectado en
      `fmtTime`/`fmtDate`/`fmtDuration` de
      `../pos-heladeria/src/app/modules/cash-register/services/cash-session.store.ts:598-625`
      (research.md Decisión 6; `cash-overview.component.ts`/`cash-history.component.ts` no cambian de
      plantilla)
- [X] T042 [US2] Confirmar en verde `npm test -- cash-session.store.spec.ts`

**Checkpoint**: las once entidades de Key Entities usan el mismo mecanismo central (SC-002)

---

## Phase 6: User Story 3 - Los filtros de fecha respetan el día del negocio (Priority: P1)

**Goal**: "Desde/Hasta/Hoy/Ayer" de Ventas y Reportes calculan los límites del día en la zona horaria
del tenant, no medianoche UTC ni la del navegador.

**Independent Test**: crear registros a las 23:59, 00:00 y 00:01 hora de Bogotá y verificar que los
filtros los asignan al día de Bogotá correcto.

- [X] T043 [US3] Agregar parámetro `tenant` y reemplazar la comparación cruda por
      `local_day_bounds_utc(date_from, tz)`/`local_day_bounds_utc(date_to, tz)` en `build_sales_query`
      de `../pos-backend/app/api/v1/sales/service.py:190-216` (contracts/date-range-filters.md,
      research.md Decisión 5)
- [X] T044 [US3] Pasar `tenant` (ya resuelto por `Depends(get_tenant)`) a `build_sales_query` en
      `../pos-backend/app/api/v1/sales/router.py`
- [X] T045 [US3] Reescribir `_paid_sales_filter` y el bucketing por día/mes (`func.date_trunc`,
      `func.date`) en `../pos-backend/app/api/v1/reports/service.py:1-27` usando
      `local_day_bounds_utc` para el filtro y `func.timezone(tz_name, func.timezone('UTC',
      Sale.sold_at))` para el bucketing (research.md Decisión 5)
- [X] T046 [P] [US3] Test de filtros de medianoche (23:59 dentro, 00:01 fuera, rango exacto de un día)
      en `../pos-backend/app/characterization_tests/test_reports_timezone.py`
      (contracts/date-range-filters.md, tabla de casos obligatorios)
- [X] T047 [US3] Reescribir `getDateRange` para `'today'`/`'week'`/`'month'`/`'year'` usando
      `businessToday(tz)` en vez de `new Date()`/calendario del navegador, en
      `../pos-heladeria/src/app/modules/reports/services/reports.service.ts:250-279` (research.md
      Decisión 7; `'specific-date'` no cambia — FR-006)
- [X] T048 [P] [US3] Actualizar `../pos-heladeria/src/app/modules/reports/services/reports.service.spec.ts`
      inyectando una zona horaria de prueba fija y comparando contra `businessToday(tz)`, no contra el
      reloj del entorno de ejecución (quickstart.md Paso 16)

**Checkpoint**: filtros de Ventas y Reportes respetan la medianoche de Bogotá (SC-003)

---

## Phase 7: User Story 4 - El negocio puede configurar la zona horaria por tenant (Priority: P2)

**Goal**: reabrir A-46 — cada tenant puede tener su propia zona horaria vía script interno, con
`America/Bogota` como valor por defecto.

**Independent Test**: configurar dos tenants con zonas horarias distintas (vía script/BD, sin UI) y
verificar que cada uno muestra sus fechas en su propia zona sin afectar al otro.

- [X] T049 [US4] Crear `../pos-backend/app/scripts/set_tenant_timezone.py` (mismo patrón que
      `seed_super_admin.py`: `argparse` + función pura reusable + `with_db(None)`; valida contra
      `zoneinfo` antes de persistir — research.md Decisión 4)
- [X] T050 [US4] Cambio aditivo en `_tz(tenant: Tenant | None = None)` en
      `../pos-backend/app/api/v1/promotions/service.py:50-54`: si se pasa `tenant`, devuelve
      `resolve_timezone(tenant)`; si no, conserva `ZoneInfo(settings.TENANT_TIMEZONE)` (research.md
      Decisión 10; `active_discount_promotions`/`best_line_discount` sin cambio — A-07 protegida)
- [X] T051 [P] [US4] Test de dos tenants con zonas horarias distintas mostrando sus propias fechas sin
      afectarse entre sí, en `../pos-backend/app/characterization_tests/test_tenant_timezone.py`
      (Historia 4, Escenarios 1-2)
- [X] T052 [P] [US4] Test de rechazo de zona horaria inválida (`ZoneInfoNotFoundError`, nunca se
      persiste) en `../pos-backend/app/characterization_tests/test_tenant_timezone.py`
      (Clarifications, FR-005)

**Checkpoint**: zona horaria configurable por tenant, `America/Bogota` como respaldo (SC-005)

---

## Phase 8: User Story 5 - El valor de fecha/hora de un formulario no cambia (Priority: P2)

**Goal**: garantía verificable de que ningún selector de fecha/hora corre el día por offset de zona
horaria (research.md Decisión 11 — sin defecto activo encontrado, solo test de regresión).

**Independent Test**: seleccionar una fecha en un formulario, enviarla, recuperarla y mostrarla de
nuevo — el valor debe ser idéntico al seleccionado.

- [X] T053 [P] [US5] Test de regresión del filtro Desde/Hasta de Ventas (string-a-string vía `ngModel`,
      sin `new Date()` intermedio) en
      `../pos-heladeria/src/app/modules/sales/pages/sales-page.component.spec.ts`
- [X] T054 [P] [US5] Test de regresión del filtro Desde/Hasta de Reportes en
      `../pos-heladeria/src/app/modules/reports/pages/reports-page.component.spec.ts`
- [X] T055 [P] [US5] Test de regresión de la ventana de vigencia (`starts_at`/`ends_at`) del formulario
      de Promoción en `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.spec.ts`

**Checkpoint**: los tres formularios de fecha/hora quedan protegidos por test (FR-006)

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: trazabilidad (FR-011), no regresión del motor de promociones (A-07), y verificación final
de los seis Success Criteria.

- [X] T056 Agregar entrada `A-50` (siguiente disponible tras `A-49`) y actualizar la entrada `A-46` en
      `specs/000-reconocimiento/registro-de-anomalias.md`, siguiendo el formato existente (`###
      A-NN — <título>`, `**Descripción**`, `**CÓDIGO**`, `**Clasificación**`, `**Depende de esto**`,
      `**Tratamiento acordado**`) — FR-011
- [X] T057 Ejecutar `python3 -m unittest app.scripts.test_promotions_rules -v` desde `../pos-backend` y
      confirmar sin cambios de resultado (A-07 protegida, quickstart.md Paso 9)
- [X] T058 [P] Verificar SC-006: `grep -rn "timedelta(hours" ../pos-backend/app ../pos-heladeria/src`
      no debe devolver resultados nuevos en los ficheros tocados por esta spec (comparar contra el
      estado previo al fix)
- [X] T059 Ejecutar la suite completa — backend `python3 -m unittest discover -s
      app/characterization_tests -p "test_*.py"` desde `../pos-backend`, frontend `npm test` desde
      `../pos-heladeria` — y confirmar ambas en verde (quickstart.md, Verificación final SC-001–SC-006)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciar de inmediato
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA todas las historias de usuario
- **US1 (Phase 3)**: depende de Foundational — es el MVP, defecto reportado
- **US6 (Phase 4)**: depende de US1 (reutiliza el fixture de Ventas ya corregido en Fase 3)
- **US2 (Phase 5)**: depende de Foundational; independiente de US1/US6 salvo que reutiliza el mismo
  patrón ya validado en Ventas
- **US3 (Phase 6)**: depende de Foundational; usa `sales/service.py` (ya tocado por US1 solo en
  `schemas.py`, sin conflicto de archivo) y `reports/service.py`
- **US4 (Phase 7)**: depende de Foundational (la columna `Tenant.timezone` de T003/T004 ya existe)
- **US5 (Phase 8)**: depende de Foundational; sin dependencia de código de otras historias (son tests
  de regresión)
- **Polish (Phase 9)**: depende de que todas las historias deseadas estén completas

### User Story Dependencies

- **US1 (P1)**: solo depende de Foundational — es el punto de entrada natural (MVP)
- **US6 (P1)**: depende de US1 para tener un caso concreto que verificar; el mismo patrón se reutiliza
  para el resto de entidades al cerrarse en US2
- **US2 (P1)**: depende de Foundational; puede ejecutarse en paralelo con US1/US6 si hay más de un
  desarrollador, ya que toca ficheros de entidades distintas a Ventas
- **US3 (P1)**: depende de Foundational; toca `sales/service.py`/`sales/router.py` (no
  `sales/schemas.py`, que es de US1) y `reports/service.py` — sin conflicto de archivo con US1/US2
- **US4 (P2)**: depende de Foundational únicamente
- **US5 (P2)**: depende de Foundational únicamente; sin dependencia de ninguna otra historia

### Within Each User Story

- Tests (si se incluyen) antes de o junto con la implementación, no después
- Schemas (`UtcDatetime`) antes de que cualquier test de serialización pueda pasar en verde
- Backend antes que el frontend que lo consume, dentro de la misma historia

### Parallel Opportunities

- Todas las tareas [P] de la Fase 2 (Foundational) pueden ejecutarse en paralelo — son ficheros
  distintos sin dependencias entre sí
- Dentro de la Fase 5 (US2), las diez tareas de schemas (T018-T028) son totalmente paralelas entre sí
  (ficheros distintos); igual los cuatro tests por entidad (T033-T036) y los cuatro reemplazos de pipe
  de plantilla (T037-T040)
- US1, US2, US4 y US5 pueden trabajarse en paralelo por desarrolladores distintos una vez completada
  la Fase 2 — US3 y US6 tienen una dependencia de secuencia más fuerte (ver arriba)

---

## Parallel Example: User Story 2 (Fase 5)

```bash
# Lanzar juntos los 10 cambios de schema (ficheros de schemas.py distintos):
Task: "OrderResponse.created_at: UtcDatetime en ../pos-backend/app/api/v1/orders/schemas.py:182"
Task: "PaymentAttemptResponse.created_at/resolved_at: UtcDatetime en .../orders/schemas.py:213-214"
Task: "ShiftResponse/ShiftSummaryResponse: UtcDatetime en .../cash/schemas.py:52-53,153-154"
Task: "CashMovementResponse.occurred_at: UtcDatetime en .../cash/schemas.py:77"
Task: "PartialCountResponse.counted_at: UtcDatetime en .../cash/schemas.py:130"
Task: "MovementResponse.moved_at: UtcDatetime en .../inventory/schemas.py:64"
Task: "PurchaseResponse.purchased_at: UtcDatetime en .../inventory/schemas.py:125"
Task: "TableSessionResponse.opened_at/closed_at: UtcDatetime en .../table_sessions/schemas.py:36-37"
Task: "ParticipantResponse.joined_at/closed_at: UtcDatetime en .../table_sessions/schemas.py:25,27"
Task: "InvoiceResponse.issued_at: UtcDatetime en .../invoices/schemas.py:32"

# Lanzar juntos los 4 tests por entidad:
Task: "Test serialización Órdenes en app/characterization_tests/test_orders_timezone.py"
Task: "Test serialización Pagos en app/characterization_tests/test_payments_timezone.py"
Task: "Test serialización Caja en app/characterization_tests/test_cash_timezone.py"
Task: "Test serialización Inventario en app/characterization_tests/test_inventory_timezone.py"
```

---

## Implementation Strategy

### MVP First (User Story 1 únicamente)

1. Completar Fase 1: Setup
2. Completar Fase 2: Foundational (CRÍTICO — bloquea todas las historias)
3. Completar Fase 3: User Story 1 (Ventas)
4. **PARAR y VALIDAR**: verificar que Ventas muestra la hora real de Bogotá, de forma independiente
5. Desplegar/demostrar si está listo — es el defecto reportado, corregido

### Incremental Delivery

1. Setup + Foundational → mecanismo central listo en ambos repos
2. US1 (Ventas) → probar de forma independiente → desplegar (MVP, defecto reportado resuelto)
3. US6 (valor histórico intacto) → confirma que US1 no violó Principio VII
4. US2 (resto de entidades) → probar cada entidad de forma independiente → desplegar
5. US3 (filtros de medianoche) → probar con los tres casos de la tabla de contrato → desplegar
6. US4 (zona horaria por tenant) → probar con dos tenants → desplegar
7. US5 (formularios) → tests de regresión, sin cambio de producción → desplegar
8. Polish → trazabilidad (A-50/A-46), no regresión de promociones, verificación final SC-001–SC-006

### Parallel Team Strategy

Con más de un desarrollador, tras completar Foundational:

- Desarrollador A: US1 → US6 (secuencia natural, mismo fichero de test)
- Desarrollador B: US2 (entidades restantes, ficheros disjuntos de US1)
- Desarrollador C: US3 (filtros) — coordinar con Desarrollador A solo si ambos tocan
  `sales/router.py` el mismo día
- US4 y US5 pueden asignarse a quien quede libre primero — no dependen de ninguna otra historia

---

## Notes

- [P] = ficheros distintos, sin dependencias entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- `FR-007`/Principio VII: ninguna tarea de este documento modifica el tipo, valor o default de una
  columna `DateTime` existente — verificado explícitamente por US6 y por T058 (SC-006)
- `Promotion.starts_at`/`ends_at` y el motor de evaluación de promociones (A-07) quedan fuera de
  alcance de toda tarea de este documento salvo T050 (cambio aditivo, no de criterio) — ver FR-009
- Commitear después de cada tarea o grupo lógico
- Parar en cada checkpoint para validar la historia de forma independiente
