# Implementation Plan: Corrección global de fechas, horas y zonas horarias

**Branch**: `030-correccion-fechas-zona-horaria` | **Date**: 2026-08-24 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/030-correccion-fechas-zona-horaria/spec.md`

## Summary

El defecto es que ninguna capa de la cadena API→Angular convierte los timestamps de instante
absoluto (ya almacenados en UTC, sin marca de zona, vía `TimestampMixin`/`server_default=func.now()`
o `datetime.now(timezone.utc)`) a la hora del negocio antes de mostrarlos, y que dos criterios de
medianoche distintos (UTC en el backend, navegador en el frontend) deciden a qué día pertenece un
registro. La corrección no toca ninguna columna de fecha existente (FR-007): es enteramente de
**serialización, interpretación de filtros y presentación**.

Mecanismo único (FR-002): un módulo nuevo `app/core/timezone.py` en `pos-backend` centraliza (a) la
resolución de la zona horaria del tenant (`resolve_timezone(tenant)`, con `America/Bogota` de
respaldo), (b) un tipo `UtcDatetime` (`Annotated[datetime, PlainSerializer(...)]`) que todo schema de
respuesta con un campo de instante absoluto adopta para que la API emita el offset UTC explícito
(FR-003), (c) `local_day_bounds_utc(day, tz)` para que los filtros "Desde/Hasta/Hoy/Ayer" comparen
contra la medianoche del negocio, no la de UTC (FR-004), y (d) `utc_now()`, un envoltorio de
`datetime.now(timezone.utc)` que reemplaza los ocho sitios citados en spec.md (Edge Cases) que hoy
construyen su propio "ahora" de forma inconsistente (FR-008). En `pos-heladeria`, un pipe Angular
nuevo y único, `TenantDatePipe` (envuelve el `DatePipe` nativo pasándole la zona horaria del tenant
como tercer argumento), reemplaza tanto los `| date` desnudos (5 sitios) como el formateador manual
`toLocaleString` de `cash-session.store.ts` (4 sitios) — mismo mecanismo, uso declarativo en plantilla
e inyección directa en stores.

`Tenant` gana una columna `timezone` (`America/Bogota` por defecto, validada contra la base de datos
IANA en el momento de escribirse — Clarifications) para cerrar A-46; según lo aclarado en
`spec.md` → Clarifications, no se construye ninguna pantalla de autoservicio para editarla, solo un
script interno (`app/scripts/set_tenant_timezone.py`, mismo patrón que `seed_super_admin.py`) y la
validación a nivel de modelo que protege cualquier otro camino de escritura futuro.

No se toca `active_discount_promotions`/`best_line_discount` (A-07, protegida) ni el desempate de
promociones (A-10). El único cambio dentro de `promotions/service.py` es que `_tz()` deja de leer
únicamente `settings.TENANT_TIMEZONE` global y pasa a poder resolver la del tenant — exactamente el
"próximo paso" que el propio comentario de esa función y el "Tratamiento acordado" de A-46 ya
anticipaban — sin alterar ningún criterio de evaluación ni de prioridad.

## Technical Context

**Language/Version**: backend Python 3.14 (venv de `pos-backend`, `env/pyvenv.cfg`; imagen Docker de
despliegue en `python:3.12-slim`, `Dockerfile:1` — misma discrepancia ya documentada por spec 023);
frontend TypeScript 5.9.2 / Angular 21.1 (`pos-heladeria/package.json`)

**Primary Dependencies**: backend — únicamente librería estándar: `zoneinfo.ZoneInfo` (ya usado por
`promotions/service.py:5,53`), `pydantic.PlainSerializer`/`typing.Annotated` (ya disponibles en
Pydantic 2.13.4, sin paquete nuevo), `sqlalchemy.orm.validates` (ya disponible en SQLAlchemy 2.0.50).
Frontend — el `DatePipe` nativo de `@angular/common` (ya usado en 5 plantillas) y `Intl.DateTimeFormat`
del navegador (estándar, sin paquete nuevo). Ninguna dependencia nueva en ningún repo — Principio IX
no exige justificación.

**Storage**: PostgreSQL 16, schema-per-tenant. Una sola columna nueva: `shared.tenants.timezone`
(`String`, `NOT NULL`, `server_default='America/Bogota'`). Ninguna de las columnas `DateTime` de las
once entidades de Key Entities cambia de tipo, de valor ni de default — siguen siendo `TIMESTAMP
WITHOUT TIME ZONE` en UTC (FR-007).

**Testing**: backend — `unittest` vía `python -m unittest` (no hay `pytest` ni `freezegun` instalados,
confirmado por `requirements.txt` y el `env/` actual); las pruebas que necesiten un "ahora" fijo
construyen el `datetime` a mano o hacen `mock.patch`, mismo patrón que
`test_promotions_router.py` de spec 023. Frontend — Vitest `^4.0.8` vía `@angular/build:unit-test`,
`TestBed` + `provideHttpClientTesting()`, mismo harness que `product.service.spec.ts`/
`promotion.service.spec.ts`.

**Target Platform**: backend Linux server (`pos-backend`, FastAPI en producción); frontend navegador
de terminal POS o panel administrativo (`pos-heladeria`, Angular SPA, origen `api.skeilopos.com`
distinto al subdominio de cada tenant — CORS real, igual que spec 023).

**Project Type**: corrección cross-repo (backend + frontend) de alcance amplio pero sin extracción de
módulo ni paquete nuevo: un módulo de utilidades nuevo por repo (`app/core/timezone.py`,
`TenantDatePipe` + `date-format.util.ts`), una columna nueva, y ediciones puntuales — tipo de campo,
filtro de consulta o punto de invocación de formato — en los ficheros existentes de cada una de las
once entidades.

**Performance Goals**: sin objetivo nuevo. La resolución de zona horaria del tenant ya está disponible
sin consulta adicional donde el tenant llega por `Depends(get_tenant)` (la mayoría de endpoints
afectados); donde no (funciones internas de `table_sessions/service.py`/`checkout.py` que hoy no
reciben `tenant`), se añade como parámetro explícito, sin I/O adicional. La serialización con
`PlainSerializer` es una llamada `.isoformat()` extra por campo, despreciable frente al resto de la
respuesta.

**Constraints**: FR-007 — ningún valor histórico almacenado cambia, en ningún caso, bajo ninguna
circunstancia; el motor de promociones (`active_discount_promotions`/`best_line_discount`,
`app/api/v1/promotions/service.py`, A-07 protegida) no cambia su criterio de evaluación ni de
desempate; `promotion-pricing.util.ts` y el desempate del frontend (A-10) no se tocan; `Page[T]`
(`app/core/pagination.py`) no cambia de forma — solo los campos `datetime` de los schemas que ya
envuelve; `Promotion.starts_at`/`ends_at` no se reinterpretan como instante absoluto (FR-009).

**Scale/Scope**: backend — 1 módulo nuevo (`app/core/timezone.py`, ~4 funciones + 1 tipo), 1 columna +
1 validador en `Tenant`, 1 migración Alembic, 1 script nuevo (`set_tenant_timezone.py`), ~13 campos de
schema anotados con `UtcDatetime` en 8 ficheros de `schemas.py`/`router.py` (incluye agregar
`PaymentResponse.paid_at`, hoy ausente — ver research.md Decisión 9), 2 servicios de filtro reescritos
(`sales/service.py`, `reports/service.py`), 8 sitios de "ahora" unificados (los citados en spec.md
Edge Cases), 1 cambio mínimo aditivo en `promotions/service.py:_tz()`, 1 entrada nueva
(`A-50`) + 1 actualización (`A-46`) en `registro-de-anomalias.md`. Frontend — 1 pipe nuevo
(`TenantDatePipe`) + 1 util (`date-format.util.ts`), `TenantInfo`/`TenantInfoResponse` +1 campo,
`reports.service.ts::getDateRange` reescrito para calcular "hoy" en la zona del tenant, 9 sitios de
formato reemplazados (5 `| date` + 4 métodos de `cash-session.store.ts`), tests nuevos/ampliados por
cada entidad tocada (FR-010: mínimo uno por Ventas/Órdenes/Pagos/Caja/Inventario, más el test de
"valor histórico sin cambios").

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` define problema, alcance (once entidades), reglas (`FR-001`–`FR-011`), criterios de aceptación por historia y decisiones de compatibilidad (FR-007, FR-009) antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El cambio de comportamiento (mostrar hora local en vez de UTC cruda; medianoche del negocio en vez de UTC) está autorizado por escrito en `spec.md` → "Autorización de negocio" (propietario, 2026-08-24) y se registra como decisión de negocio nueva (`A-50`) más la actualización de `A-46`, ambas exigidas por `FR-011` y ejecutadas como parte de la implementación de este plan, no como una interpretación técnica no autorizada. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | Ningún test `"CONGELA comportamiento actual:"` existente fija el valor UTC crudo mostrado hoy (no se encontró ninguno en la investigación) — no hay ningún test protegido que este plan deba tocar. `app/scripts/test_promotions_rules.py` (único test de A-07 en CI) no se ve afectado: el cambio en `_tz()` es aditivo (research.md Decisión 10) y no cambia ningún resultado de evaluación para el tenant único que existe hoy. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El nuevo comportamiento (hora de negocio en vez de UTC cruda, filtros por día de negocio, zona horaria configurable por tenant) está definido íntegramente por `spec.md`, no por criterio técnico de esta planeación. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Cada fichero tocado se justifica por un `FR` concreto (ver Project Structure). Explícitamente **no** se tocan: `promotion-pricing.util.ts`, `app/core/pagination.py` (`Page[T]`), la lógica de `active_discount_promotions`/`best_line_discount`, ni ningún fichero fuera de la lista de Key Entities de `spec.md`. | PASS |
| **VI. Evolución Incremental** | El alcance cruza dos repos y once entidades, pero se divide en unidades verificables: primero el mecanismo central de cada repo (Decisión 1–2 de research.md, sin efecto observable hasta que algo lo consuma), luego Ventas (defecto reportado, P1), luego el resto de entidades una por una reutilizando el mismo mecanismo (Historia 2), luego filtros (Historia 3), luego zona horaria por tenant (Historia 4, P2) y garantía de formularios (Historia 5, P2) — cada unidad tiene su propio Independent Test ya definido en `spec.md` y puede desplegarse y verificarse sola. `tasks.md` (Fase 2) hereda esta secuencia. | PASS |
| **VII. Compatibilidad con Datos Históricos** | `FR-007` es una restricción de diseño de primer orden en este plan: ninguna columna de fecha cambia de tipo, valor ni default; el mecanismo entero vive en la capa de serialización/consulta/presentación. `FR-010` exige un test explícito de "valor histórico sin cambios" (quickstart.md, Verificación final), que compara el valor crudo en base de datos antes/después del despliegue. | PASS |
| **VIII. Evolución del Modelo de Datos** | Único cambio de esquema: `Tenant.timezone` (`String`, `NOT NULL`, `server_default='America/Bogota'`) en `shared.tenants` — compatibilidad con filas existentes garantizada por el `server_default` (no requiere backfill de aplicación); migración: una sola revisión Alembic ordinaria (`Tenant` vive en `shared`, no en el esquema `tenant`/`tenant_default` de fan-out por tenant — ver research.md, Migration tooling); rollback: `downgrade()` elimina la columna, seguro porque ningún dato depende todavía de ella al momento de introducirla. | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia nueva en ningún repo (ver Primary Dependencies). No aplica justificación. | PASS (no aplica) |
| **X. Verificación Obligatoria** | `FR-010` exige, como mínimo, un test por Ventas/Órdenes/Pagos/Caja/Inventario que demuestre el comportamiento correcto cerca de medianoche, más un test de valor histórico sin cambios — ambos incorporados a la Fase 1 (Testing, arriba) y detallados en quickstart.md. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | La decisión de negocio (corregir el defecto, reabrir A-46, no requerir migración de datos) está documentada en `spec.md` y en `registro-de-anomalias.md` (`A-50`, actualización de `A-46`); las decisiones técnicas (formato `UtcDatetime` con `PlainSerializer`, `TenantDatePipe`, `local_day_bounds_utc`) son de esta planeación y de `research.md`, sin mezclarse con la autorización de negocio. | PASS |
| **XII. Trazabilidad** | Necesidad (defecto reportado en Ventas, `07:53` vs. `12:49`) → Spec 030 → Decisión (autorización de negocio + `A-50`/`A-46`) → Implementación (este plan) → Tests (`FR-010`) → Verificación (`quickstart.md`, SC-001–SC-006). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y los artefactos que genera (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que las specs 022/023 de las que este plan es directamente comparable en alcance de zona horaria. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/030-correccion-fechas-zona-horaria/
├── plan.md                                  # Este fichero (/speckit-plan)
├── research.md                              # Fase 0 — decisiones del mecanismo central y sus alternativas
├── data-model.md                            # Fase 1 — columna nueva, tipo UtcDatetime, tabla de las 11 entidades
├── quickstart.md                            # Fase 1 — guía de validación end-to-end
├── contracts/
│   ├── instant-datetime-serialization.md    # Fase 1 — contrato transversal: formato UTC explícito en toda la API
│   ├── date-range-filters.md                # Fase 1 — GET /sales, GET /reports/* — semántica de date_from/date_to
│   └── tenant-info-endpoint.md              # Fase 1 — GET/PATCH /tenant — campo timezone nuevo, PATCH lo excluye
└── tasks.md                                 # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (dos repos sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (siblings de este repo, según la Constitución §Alcance). Rutas relativas a la raíz
de cada uno.

```text
# ../pos-backend

app/core/
├── timezone.py                       # NUEVO — mecanismo central (FR-002): resolve_timezone(tenant),
│                                          UtcDatetime (Annotated + PlainSerializer, FR-003),
│                                          local_day_bounds_utc(day, tz) (FR-004), utc_now() (FR-008)
├── config.py                         # SIN CAMBIOS DE VALOR — TENANT_TIMEZONE queda como respaldo
│                                          de último recurso dentro de resolve_timezone() cuando
│                                          tenant.timezone es NULL (no debería serlo tras la migración,
│                                          pero el respaldo evita un None-crash)
└── models.py                         # CAMBIA — clase Tenant gana columna `timezone` (String, NOT
                                           NULL, server_default='America/Bogota') + @validates
                                           ('timezone') con zoneinfo.ZoneInfo(value) (FR-005,
                                           Clarifications: rechazo en escritura)

alembic/versions/
└── <hash>_add_tenant_timezone.py     # NUEVO — ALTER TABLE shared.tenants ADD COLUMN timezone;
                                           downgrade() la elimina (Principio VIII)

app/scripts/
└── set_tenant_timezone.py            # NUEVO — mismo patrón que seed_super_admin.py; único camino de
                                           escritura previsto para Historia 4 (Clarifications: sin
                                           pantalla de autoservicio)

app/api/v1/
├── sales/
│   ├── schemas.py                    # CAMBIA — SaleResponse.sold_at: UtcDatetime (línea 134);
│   │                                      PaymentResponse gana paid_at: UtcDatetime (hoy ausente,
│   │                                      líneas 94-100 — ver research.md Decisión 9)
│   └── service.py                    # CAMBIA — build_sales_query (190-216): date_from/date_to pasan
│                                          por local_day_bounds_utc(day, tz) en vez de comparar contra
│                                          el date crudo (medianoche UTC implícita) (FR-004); recibe
│                                          tenant como parámetro nuevo
├── orders/
│   └── schemas.py                    # CAMBIA — OrderResponse.created_at: UtcDatetime (línea 182);
│                                          PaymentAttemptResponse.created_at/resolved_at: UtcDatetime
│                                          (líneas 213-214)
├── cash/
│   ├── schemas.py                    # CAMBIA — ShiftResponse.opened_at/closed_at (52-53) y
│   │                                      ShiftSummaryResponse (153-154), CashMovementResponse
│   │                                      .occurred_at (77), PartialCountResponse.counted_at (130):
│   │                                      todos a UtcDatetime
│   └── router.py                     # CAMBIA — línea 121 (shift.closed_at = ...) pasa de
│                                          datetime.now(timezone.utc) a utc_now() (FR-008); único
│                                          cambio de este fichero
├── inventory/
│   └── schemas.py                    # CAMBIA — MovementResponse.moved_at (64) y
│                                          PurchaseResponse.purchased_at (125): UtcDatetime
├── table_sessions/
│   ├── schemas.py                    # CAMBIA — TableSessionResponse.opened_at/closed_at (36-37),
│   │                                      ParticipantResponse.joined_at/closed_at (25,27):
│   │                                      UtcDatetime
│   └── service.py                    # CAMBIA — líneas 177/652/739 (now = datetime.now(timezone
│                                          .utc)) a utc_now() (FR-008); sin cambio de lógica, solo de
│                                          dónde sale el valor
├── invoices/
│   └── schemas.py                    # CAMBIA — InvoiceResponse.issued_at (32): UtcDatetime; ningún
│                                          otro campo de Invoice se toca (Principio VII — importe y
│                                          demás datos de facturas ya emitidas quedan intactos)
├── audit/
│   └── router.py                     # CAMBIA — AuditLogResponse.at (19-27, definida inline):
│                                          UtcDatetime; sin filtro de rango que corregir (no existe
│                                          hoy filtro de fecha en este listado)
├── orders/checkout.py                # CAMBIA — líneas 268/460/633/664/811/819/873/925/933
│                                          (datetime.now(timezone.utc)) a utc_now() donde el valor
│                                          alimenta un campo persistido (resolved_at de
│                                          PaymentAttempt); los usos de "now" solo para evaluar
│                                          promociones en memoria (no persistidos) quedan fuera de
│                                          FR-008 (research.md Decisión 8 fija el criterio exacto)
├── core/qr_context.py                # CAMBIA — líneas 85/179 a utc_now() (FR-008)
└── promotions/
    └── service.py                    # CAMBIA MÍNIMO — _tz() (líneas 50-54) resuelve
                                           resolve_timezone(tenant) cuando el caller tiene el tenant
                                           disponible, con TENANT_TIMEZONE como respaldo (Historia 4 /
                                           A-46); active_discount_promotions/best_line_discount/
                                           local_now SIN CAMBIO de criterio (A-07 protegida)

app/models/
└── payment.py                        # SIN CAMBIOS — paid_at ya existe en el modelo (línea 57); el
                                           gap está solo en el schema de respuesta

# ../pos-heladeria

src/app/shared/
├── pipes/
│   └── tenant-date.pipe.ts           # NUEVO — mecanismo central del frontend (FR-002): envuelve
│                                          DatePipe pasándole TenantInfoService.info()?.timezone como
│                                          tercer argumento; usable como `| tenantDate` en plantilla e
│                                          inyectable directamente en stores/servicios
├── pipes/tenant-date.pipe.spec.ts    # NUEVO
└── date-format.util.ts               # NUEVO — businessToday(timezone): string, usado por
                                           reports.service.ts::getDateRange (FR-004, evita depender
                                           del calendario del navegador)

src/app/core/tenant/
└── tenant-info.service.ts            # CAMBIA — interfaz TenantInfo gana `timezone: string` (línea
                                           8-16); update() sigue sin incluir timezone en el tipo de
                                           payload aceptado (Clarifications: sin autoservicio)

src/app/modules/
├── sales/pages/
│   └── sales-page.component.ts       # CAMBIA — líneas 108/155: `| date:'dd/MM/yyyy HH:mm'` →
│                                          `| tenantDate:'dd/MM/yyyy HH:mm'`
├── orders/pages/
│   ├── order-detail.component.ts     # CAMBIA — línea 64: mismo reemplazo de pipe
│   └── orders-page.component.ts      # CAMBIA — línea 92: mismo reemplazo de pipe
├── dashboard/pages/
│   └── admin-dashboard.component.ts  # CAMBIA — línea 123: mismo reemplazo de pipe
├── inventory/pages/
│   └── inventory-page.component.ts   # CAMBIA — líneas 212/297: mismo reemplazo de pipe
├── cash-register/
│   ├── components/
│   │   ├── cash-overview.component.ts  # SIN CAMBIOS DE PLANTILLA — sigue llamando
│   │   │                                    store.formatDate(...), que cambia por dentro
│   │   └── cash-history.component.ts   # SIN CAMBIOS DE PLANTILLA — idem
│   └── services/
│       └── cash-session.store.ts     # CAMBIA — fmtTime/fmtDate/fmtDuration (598-625) dejan de
│                                          llamar new Date(iso).toLocaleString(...) directo y pasan
│                                          por TenantDatePipe (inyectado) con la zona del tenant
├── reports/
│   └── services/
│       └── reports.service.ts        # CAMBIA — getDateRange (250-279): 'today'/'week'/'month'/
│                                          'year' calculan el día de negocio con
│                                          date-format.util.ts::businessToday(tz) en vez de
│                                          new Date()/toLocaleDateString('en-CA') del navegador
│                                          (FR-004); 'specific-date' SIN CAMBIOS — ya usa el string
│                                          elegido por el usuario tal cual (FR-006)
└── promotions/pages/
    └── promotions-page.component.ts  # SIN CAMBIOS — starts_at/ends_at siguen fuera de alcance
                                           (FR-009); el input[type=date] ya round-trippea el string
                                           sin pasar por un Date intermedio

# ../pos-specs (este mismo repo, trazabilidad — FR-011)
specs/000-reconocimiento/
└── registro-de-anomalias.md          # CAMBIA — nueva entrada A-50 (defecto de Ventas y demás
                                           entidades, esta spec) + actualización de A-46
                                           (zona horaria pasa a ser configurable por tenant)
```

**Structure Decision**: dos repos, un módulo central nuevo por repo (`app/core/timezone.py`,
`TenantDatePipe`) del que todo lo demás depende, una columna de esquema nueva, y ediciones puntuales
—tipo de campo, filtro de consulta o punto de invocación de formato— en los ficheros ya existentes de
cada una de las once entidades listadas en Key Entities. No hay extracción de módulo ni paquete nuevo
en ningún repo: el mecanismo central vive en `app/core/` (junto a `config.py`/`models.py`, que ya son
el lugar transversal del backend) y en `shared/` del frontend (junto a otros pipes/utils
transversales). El orden de implementación en `tasks.md` sigue la secuencia de Evolución Incremental
descrita en el Constitution Check: mecanismo central → Ventas (P1, defecto reportado) → resto de
entidades (Historia 2) → filtros (Historia 3) → zona horaria por tenant (Historia 4, P2) → garantía de
formularios (Historia 5, P2) → trazabilidad (`FR-011`).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
