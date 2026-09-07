# Implementation Plan: Notificaciones en Tiempo Real Multi-Tenant

**Branch**: `077-notificaciones-tiempo-real` | **Date**: 2026-09-07 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/077-notificaciones-tiempo-real/spec.md`

## Summary

Hoy la única notificación de "pedido nuevo" o "pago confirmado" es el flujo SSE que `Terminal de Mesas` abre para sí misma (`pos-terminal.store.ts` llama `RealtimeService.connectStaff()`); si el cajero navega a otra sección o pierde el foco de la pestaña, no se entera. El backend ya tiene un bus de eventos en tiempo real robusto (Redis Streams por tenant, `app/core/events.py` + `app/core/event_bus.py` + `/realtime/stream` SSE con replay por `Last-Event-ID`) que resuelve la mayor parte de los requisitos no funcionales (sin polling, multi-instancia, aislamiento por tenant derivado del token firmado, latencia de segundos). El trabajo de esta funcionalidad **no es reconstruir ese bus**, sino:

1. Mover la conexión SSE de staff del store de una sola página al *shell* autenticado, para que sobreviva a la navegación (RF-003).
2. Añadir una capa de notificación persistente y desacoplada por canal (`NotificationEvent` + `NotificationChannel`) por encima del bus existente, de forma que agregar canales futuros (email/SMS/WhatsApp) sea añadir una clase, no tocar los puntos de negocio que ya llaman a `events.order_created` / `events.payment_completed` (RF-006, RF-009, RF-010).
3. Sumar el canal push del navegador (Web Push + VAPID) para cuando la pestaña no tiene foco o está cerrada (RF-004).
4. Cerrar una brecha puntual ya identificada: `events.payment_completed` hoy solo publica en el canal `staff`, nunca en el canal de sesión del comensal (`session:{table_session_id}`), así que el comensal anónimo nunca se entera del pago por tiempo real (RF-005) — se corrige agregando ese canal a la función existente.
5. Job de purga por retención (90 días por defecto, configurable por tenant) siguiendo el mismo patrón que `sweep_orphan_sessions`/`expire_promotions` en `app/core/scheduler.py` (APScheduler + lock en Redis, ya son dependencias del proyecto).

## Technical Context

**Language/Version**: Backend: Python 3.11+ (FastAPI 0.136, SQLAlchemy 2.0, Alembic). Frontend: TypeScript 5.9 / Angular 21 (standalone components, signals).

**Primary Dependencies**:
- Ya presentes y reutilizadas tal cual: `redis` 8.0 (Streams + Pub/Sub, ya es el backbone de `/realtime`), `APScheduler` 3.11 (ya programa `sweep_orphan_sessions` y `expire_promotions`), `celery` 5.6 (broker ya configurado, no se usa aquí porque APScheduler ya resuelve el patrón "job periódico con lock" en este mismo módulo), `SQLAlchemy`/`Alembic` (persistencia tenant-scoped vía `schema_translate_map`).
- Nuevas, justificadas en `research.md`: `pywebpush` (backend, protocolo Web Push + cifrado VAPID/RFC 8291) y `@angular/service-worker` (frontend, Service Worker + `SwPush` de primera parte de Angular).

**Storage**: PostgreSQL 16, schema-per-tenant (`schema_translate_map`, patrón ya usado por `AuditLog`). Redis 8 para el bus de tiempo real (ya en producción) y para los locks de los jobs periódicos (ya en producción).

**Testing**: Backend: `unittest` + `fastapi.testclient.TestClient`, siguiendo el precedente de `app/characterization_tests/test_operational_log.py` (spec 074) para comportamiento **nuevo** verificado contra este spec, no characterization tests de comportamiento heredado. Frontend: Karma/Jasmine (`ng test`), patrón ya usado por `sse-client.spec.ts`.

**Target Platform**: Servidor Linux (backend, ya en producción) + navegador (frontend Angular, ya en producción). El canal push depende de navegadores con soporte de Push API/Service Workers (Chrome, Edge, Firefox; Safari con matices — ver `research.md`).

**Project Type**: Web application (backend `pos-backend` + frontend `pos-heladeria`, dos repositorios ya existentes).

**Performance Goals**: Latencia de entrega casi inmediata (NFR-004, SC-002/SC-003/SC-004: <5 s en condiciones normales) — ya alcanzable con el bus Redis Streams existente (`REALTIME_HEARTBEAT_SECONDS`, `REALTIME_READER_BLOCK_MS`); no se requiere nueva infraestructura de bajo nivel.

**Constraints**: Sin polling contra PostgreSQL (NFR-001, ya cumplido por el bus existente). Multi-instancia sin pérdida ni duplicado (NFR-002, ya cumplido: un lector por proceso+tenant sobre Redis Streams, sin sticky sessions). Aislamiento por tenant validado en servidor (NFR-003, ya cumplido: el canal se deriva del claim firmado del JWT/ticket, nunca de lo que pida el cliente) — se añade una prueba automatizada explícita (SC-005) porque el spec la exige como entregable, no porque el mecanismo esté roto.

**Scale/Scope**: Mismo orden de magnitud que el bus ya en producción (~50 eventos/s en el peor caso por tenant, según el propio comentario de `app/core/events.py`). Alcance de esta funcionalidad: 2 tipos de evento disparadores (`order.created`, `payment.completed`, según la clarificación de spec), 2 canales de entrega (en-aplicación, push), 1 job de purga.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Evaluado contra `.specify/memory/constitution.md` v3.0.0 (fase de Evolución Funcional):

- **I. Nace de un spec** ✅ — `specs/077-notificaciones-tiempo-real/spec.md`, con `## Clarifications` resolviendo las 3 decisiones de negocio sin default obvio.
- **II. Comportamiento existente protegido** ✅ con una precisión: `events.payment_completed()` pasa a publicar también en `session_channel(table_session_id)`, algo que hoy **no ocurre**. No es un comportamiento protegido que se modifique (nada depende hoy de que el comensal *no* reciba ese evento; su ausencia es la brecha que este spec autoriza a cerrar vía RF-005) — es comportamiento nuevo aditivo, cubierto por el Principio IV, no una modificación de uno existente. No se requiere entrada en `registro-de-anomalias.md` porque no hay comportamiento previo que se esté reemplazando, solo una entrega que antes no llegaba a nadie.
- **III. Characterization tests protegidos** ✅ — no hay tests con prefijo `"CONGELA comportamiento actual:"` sobre `/realtime`, `events.py` o `event_bus.py` (son scripts de verificación manual en `app/scripts/test_realtime_stream.py`, no characterization tests); ninguno se toca.
- **IV. Nuevo comportamiento vía spec** ✅ — toda la funcionalidad (persistencia de notificaciones, canal push, purga, entrega al comensal) es comportamiento nuevo definido por este spec.
- **V. Sin refactors oportunistas** ✅ — no se reescribe el bus Redis Streams ni el endpoint `/realtime/stream` existentes; se extienden con wrappers nuevos (`app/core/notifications/`), y el único cambio a código existente es la línea de canales en `payment_completed()`.
- **VI. Evolución incremental** ✅ — el plan separa persistencia+dispatcher (backend), canal push (backend+frontend+SW), reubicación de la conexión SSE (frontend), y job de purga (backend) como unidades verificables independientes; `/speckit-tasks` las secuenciará.
- **VII. Datos históricos** N/A — no toca facturas ni importes.
- **VIII. Evolución del modelo de datos** ✅ — ver `data-model.md`: entidades nuevas, columna nueva en `shared.tenants`, estrategia de migración (Alembic, con default de servidor) y de rollback documentadas.
- **IX. Dependencias nuevas justificadas** ✅ — `pywebpush` y `@angular/service-worker`, justificadas en `research.md` con alternativas consideradas.
- **X. Verificación obligatoria** ✅ — ver `research.md` § pruebas: TestClient con dos tenants para SC-005, más pruebas de los nuevos endpoints y del job de purga.
- **XI. Negocio vs. técnico** ✅ — las 3 decisiones de negocio están en `## Clarifications` de spec.md; este plan solo decide *cómo* implementarlas.
- **XII. Trazabilidad** ✅ — cada decisión de `research.md` cita el FR/NFR/SC que resuelve.
- **XIII. Español de Colombia** ✅ — todos los artefactos de este plan están en español.

**Resultado**: sin violaciones. No se requiere `Complexity Tracking`.

### Re-chequeo posterior al diseño (Fase 1)

Con `research.md` y `data-model.md` ya escritos, se revisan de nuevo los puntos que dependían de decisiones de diseño concretas:

- **VIII. Evolución del modelo de datos** ✅ confirmado — `data-model.md` documenta las 3 tablas nuevas, la columna nueva en `shared.tenants`, la migración (una revisión Alembic, aplicada por schema de tenant como ya es el mecanismo estándar del proyecto) y el rollback explícito (`DROP TABLE`/`DROP COLUMN`, sin FKs duras que romper).
- **IX. Dependencias nuevas** ✅ confirmado — exactamente 2 dependencias nuevas (`pywebpush` backend, `@angular/service-worker` frontend), ambas con problema/alternativas/impacto documentados en `research.md` §5.
- **II. Comportamiento existente protegido** ✅ confirmado tras el diseño — el único cambio a código ya existente en producción es una línea en `events.py::payment_completed()` (agregar un canal a una llamada a `publish()` que ya existe) y el traslado de `connectStaff()`/`disconnect()` de un store a un componente de shell (mismo `RealtimeService`, mismo API público, sin tocar su implementación). Ningún endpoint, modelo o comportamiento ya en producción cambia su contrato.

Sin violaciones nuevas. La spec está lista para `/speckit-tasks`.

## Project Structure

### Documentation (this feature)

```text
specs/077-notificaciones-tiempo-real/
├── plan.md              # Este archivo (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
├── data-model.md         # Fase 1 (/speckit-plan)
├── quickstart.md         # Fase 1 (/speckit-plan)
├── contracts/             # Fase 1 (/speckit-plan)
│   ├── notifications-api.md
│   └── realtime-events.md
└── tasks.md              # Fase 2 (/speckit-tasks — no la crea /speckit-plan)
```

### Source Code (dos repositorios existentes, hermanos de `pos-specs`)

Proyecto tipo **web application** ya establecido: `../pos-backend` (API FastAPI, schema-per-tenant) + `../pos-heladeria` (Angular). Esta funcionalidad no crea repos ni módulos de alto nivel nuevos; extiende paquetes ya existentes y añade paquetes hermanos a los que ya tienen el mismo rol.

```text
../pos-backend/
├── app/
│   ├── core/
│   │   ├── events.py                    # YA EXISTE — catálogo de eventos; se edita
│   │   │                                 #   payment_completed() para publicar
│   │   │                                 #   también en session_channel() (RF-005)
│   │   ├── event_bus.py                 # YA EXISTE — sin cambios (bus Redis Streams)
│   │   ├── scheduler.py                 # YA EXISTE — se añade el job de purga junto
│   │   │                                 #   a sweep_orphan_sessions/expire_promotions
│   │   ├── models.py                    # YA EXISTE — Tenant gana la columna
│   │   │                                 #   notification_retention_days (shared)
│   │   └── notifications/               # NUEVO — capa de persistencia + fan-out
│   │       ├── dispatch.py              #   notify_order_created(), notify_payment_completed()
│   │       ├── channels/
│   │       │   ├── base.py              #   Protocol NotificationChannel
│   │       │   ├── in_app.py            #   reutiliza events.publish() (bus existente)
│   │       │   └── browser_push.py      #   pywebpush + VAPID
│   │       └── purge.py                 #   purga por retención (llamado desde scheduler.py)
│   ├── models/
│   │   ├── notification_event.py        # NUEVO (schema tenant, patrón audit_log.py)
│   │   ├── push_subscription.py         # NUEVO (schema tenant)
│   │   └── notification_channel_pref.py # NUEVO (schema tenant) — RF-010
│   └── api/v1/
│       └── notifications/               # NUEVO — mismo patrón que app/api/v1/audit/
│           ├── router.py                # GET /notifications, POST .../attend,
│           │                            #   POST/DELETE /notifications/push/subscriptions
│           └── schemas.py
├── app/characterization_tests/
│   └── test_notifications.py            # NUEVO — comportamiento nuevo (precedente
│                                         #   test_operational_log.py, spec 074),
│                                         #   incluye la prueba de aislamiento SC-005
└── alembic/versions/
    └── <hash>_notification_event_push_subscription_channel_pref.py   # NUEVO

../pos-heladeria/
├── src/app/
│   ├── core/
│   │   ├── realtime/                    # YA EXISTE — sin cambios de API pública
│   │   └── notifications/               # NUEVO — homólogo de core/realtime
│   │       ├── notification-center.service.ts   # estado (signals), API REST, sonido
│   │       ├── push-registration.service.ts     # SwPush, alta/baja de PushSubscription
│   │       └── notification.model.ts
│   ├── modules/dashboard/layout/
│   │   ├── dashboard-layout.component.ts # SE EDITA — arranca aquí RealtimeService
│   │   │                                 #   .connectStaff() (antes solo lo hacía
│   │   │                                 #   pos-terminal.store.ts) y el
│   │   │                                 #   NotificationCenterService
│   │   └── header.component.ts           # SE EDITA — icono/campanita de notificaciones
│   └── modules/tables/services/
│       └── pos-terminal.store.ts         # SE EDITA — deja de llamar
│                                          #   connectStaff()/disconnect(); sigue
│                                          #   usando realtime.on(...) igual que hoy
├── ngsw-config.json                      # NUEVO (config de @angular/service-worker)
└── src/
    └── sw-push.ts o public/custom-sw.js  # NUEVO — listener 'push' con dedupe por
                                           #   foco (clients.matchAll)
```

**Structure Decision**: se mantiene la separación de dos repositorios ya vigente (`pos-backend` API, `pos-heladeria` frontend Angular). En backend, la nueva capa vive en `app/core/notifications/` (paralela a `app/core/events.py`/`event_bus.py`, mismo nivel de abstracción) más un router `app/api/v1/notifications/` (mismo patrón que `app/api/v1/audit/`) y tres modelos nuevos en `app/models/` (mismo patrón tenant-scoped que `audit_log.py`). En frontend, la conexión de tiempo real se reubica del store de una página (`pos-terminal.store.ts`) al shell autenticado (`dashboard-layout.component.ts`), y la UI de notificaciones vive en un paquete nuevo `core/notifications/` paralelo a `core/realtime/`, consumido desde el `header.component.ts` del shell para estar disponible en cualquier sección del POS (RF-003).

## Complexity Tracking

*Sin violaciones de la Constitution Check — sección no aplica.*
