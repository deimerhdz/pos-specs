---

description: "Task list for Notificaciones en Tiempo Real Multi-Tenant"
---

# Tasks: Notificaciones en Tiempo Real Multi-Tenant

**Input**: Design documents from `/specs/077-notificaciones-tiempo-real/`
**Prerequisites**: [plan.md](plan.md), [spec.md](spec.md), [research.md](research.md), [data-model.md](data-model.md), [contracts/](contracts/), [quickstart.md](quickstart.md)

**Tests**: Solo se incluye una tarea de prueba automatizada explícita — la de aislamiento entre tenants (US2), porque es la única que la spec exige literalmente en su checklist de aceptación ("Existe una prueba automatizada que demuestra que un evento del tenant A nunca llega a un suscriptor del tenant B", SC-005). El resto de historias se valida con los pasos manuales de `quickstart.md`.

**Organization**: Tareas agrupadas por historia de usuario de `spec.md`, en su orden de prioridad (P1, P1, P2, P2, P3). Repos: `../pos-backend` (API) y `../pos-heladeria` (Angular), hermanos de `pos-specs` — todas las rutas de este documento son relativas a la raíz del repo correspondiente, indicada en cada tarea.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: Historia de usuario a la que pertenece (US1..US5, según `spec.md`)

---

## Phase 1: Setup

**Purpose**: Preparar las dos dependencias nuevas que el resto de fases necesitan (research.md §5).

- [X] T001 Agregar `pywebpush` a `../pos-backend/requirements.txt` e instalarlo en el entorno del proyecto
- [X] T002 [P] Agregar `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY` y `VAPID_SUBJECT` a `../pos-backend/app/core/config.py`, siguiendo el patrón `Field(default=..., env="...")` ya usado por los ajustes `REALTIME_*`
- [X] T003 [P] Ejecutar `ng add @angular/service-worker` en `../pos-heladeria` (agrega la dependencia, genera `ngsw-config.json` y registra `provideServiceWorker` en `src/app/app.config.ts`)

**Checkpoint**: dependencias listas para Foundational.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Modelo de datos, migración y la capa de despacho/canal mínima de la que dependen todas las historias.

**⚠️ CRITICAL**: ninguna historia de usuario puede empezar hasta que esta fase esté completa.

- [X] T004 [P] Agregar la columna `notification_retention_days` (Integer, `nullable=False`, `server_default="90"`) a la clase `Tenant` en `../pos-backend/app/core/models.py`, junto a la columna `timezone` ya existente (data-model.md § Cambios sobre entidades existentes)
- [X] T005 [P] Crear el modelo `NotificationEvent` en `../pos-backend/app/models/notification_event.py`, siguiendo el patrón de `app/models/audit_log.py` (schema `tenant`, `payload` JSONB, `created_at`/`purge_at` indexados, `attended_at`/`attended_by_user_id` nullable) — ver data-model.md § NotificationEvent
- [X] T006 [P] Crear el modelo `PushSubscription` en `../pos-backend/app/models/push_subscription.py` (schema `tenant`, `endpoint` único, `p256dh_key`/`auth_key`, `active` boolean) — ver data-model.md § PushSubscription
- [X] T007 [P] Crear el modelo `NotificationChannelPreference` en `../pos-backend/app/models/notification_channel_pref.py` (schema `tenant`, `UNIQUE(event_type, channel)`) — ver data-model.md § NotificationChannelPreference
- [X] T008 Generar la migración Alembic en `../pos-backend/alembic/versions/` que agrega la columna de T004 y crea las tres tablas de T005-T007 en cada schema de tenant, con su estrategia de rollback documentada (data-model.md § Migración) — depende de T004, T005, T006, T007
- [X] T009 [P] Crear el protocolo `NotificationChannel` (método `send(event) -> None`) en `../pos-backend/app/core/notifications/channels/base.py` (research.md §4)
- [X] T010 [P] Crear `InAppChannel` en `../pos-backend/app/core/notifications/channels/in_app.py`, que reutiliza `events.publish()` para emitir el tipo `notification.created` en el canal `staff` (contracts/realtime-events.md § Evento nuevo) — depende de T009
- [X] T011 Crear `../pos-backend/app/core/notifications/dispatch.py` con `notify_order_created(...)` y `notify_payment_completed(...)`: cada una inserta una fila `NotificationEvent` y reparte a los canales habilitados en `NotificationChannelPreference` (ausencia de fila = habilitado) — depende de T005, T007, T009, T010
- [X] T012 [P] Crear `../pos-backend/app/api/v1/notifications/router.py` y `schemas.py` con `GET /notifications` (modos `page`/`size` y `after_id`/`limit`, `only_pending`) tal como especifica contracts/notifications-api.md — depende de T005
- [X] T013 Registrar `notifications_router` en `../pos-backend/app/main.py` (import + `app.include_router(notifications_router, prefix="/api/v1")`, mismo patrón que `audit_router` en la línea 157) — depende de T012
- [X] T014 [P] Crear el esqueleto de `NotificationCenterService` y `notification.model.ts` en `../pos-heladeria/src/app/core/notifications/` (estado con signals, cliente HTTP hacia `GET /notifications`)

**Checkpoint**: modelo, migración, despacho y endpoint base listos — las historias de usuario pueden empezar.

---

## Phase 3: User Story 1 - El cajero ve el pedido nuevo sin importar en qué página del POS esté (Priority: P1) 🎯 MVP

**Goal**: el cajero recibe, en cualquier sección del POS, un aviso visual y sonoro inmediato de un pedido nuevo o un pago confirmado, y puede marcarlo como atendido para todo el equipo.

**Independent Test**: con el POS abierto en "Ventas", confirmar un pedido desde el menú QR de una mesa del mismo tenant y verificar que el aviso (visual + sonido) aparece sin navegar a Terminal de Mesas (quickstart.md §1).

### Implementation for User Story 1

- [X] T015 [US1] Agregar la llamada a `notify_order_created(...)` en `../pos-backend/app/api/v1/cart/router.py`, junto a la llamada existente a `events.order_created(...)` (~línea 159)
- [X] T016 [US1] Agregar la llamada a `notify_payment_completed(...)` en los dos sitios de `../pos-backend/app/api/v1/orders/router.py` que ya llaman a `events.payment_completed(...)` (~líneas 453 y 483)
- [X] T017 [US1] Agregar la llamada a `notify_payment_completed(...)` en `../pos-backend/app/api/v1/table_sessions/service.py`, junto a la llamada existente a `events.payment_completed(...)` (~línea 311)
- [X] T018 [US1] Mover el ciclo de vida de `RealtimeService.connectStaff()`/`.disconnect()` desde `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` hacia `../pos-heladeria/src/app/modules/dashboard/layout/dashboard-layout.component.ts`; `pos-terminal.store.ts` conserva únicamente sus `realtime.on(...)` (research.md §2)
- [X] T019 [US1] En `../pos-heladeria/src/app/core/notifications/notification-center.service.ts`, suscribirse al tipo SSE `notification.created` y exponer un contador/lista de pendientes como signal — depende de T014, T018
- [X] T020 [US1] Agregar el icono/campanita con contador en `../pos-heladeria/src/app/modules/dashboard/layout/header.component.ts`, conectado a `NotificationCenterService` — depende de T019
- [X] T021 [US1] Disparar el aviso visual (reutilizando `../pos-heladeria/src/app/shared/feedback/toast.service.ts`) y un sonido audible cuando llega `notification.created` con la pestaña enfocada, desde `NotificationCenterService` (FR-003) — depende de T019
- [X] T022 [US1] Implementar `POST /notifications/{notification_id}/attend` en `../pos-backend/app/api/v1/notifications/router.py` (marca compartida por tenant, idempotente) — depende de T012
- [X] T023 [US1] Conectar la acción "marcar atendida" desde la campanita de `header.component.ts` a ese endpoint — depende de T020, T022

**Checkpoint**: US1 funcional y verificable de forma independiente (quickstart.md §1).

---

## Phase 4: User Story 2 - Aislamiento estricto de notificaciones entre tenants (Priority: P1)

**Goal**: demostrar, con una prueba automatizada, que ningún evento ni endpoint de notificaciones de un tenant es alcanzable desde otro tenant.

**Independent Test**: `python -m unittest app.characterization_tests.test_notifications -v` pasa en verde, cubriendo tanto el stream SSE como los endpoints REST nuevos (quickstart.md §4).

### Tests for User Story 2 ⚠️

> Requisito explícito del checklist de aceptación de la spec (SC-005) — no es TDD genérico, es la prueba que la propia spec exige como entregable.

- [X] T024 [P] [US2] Escribir la prueba de aislamiento entre tenants en `../pos-backend/app/characterization_tests/test_notifications.py`: dos tenants, un `notify_order_created(...)` para el tenant A, y aserciones de que ningún suscriptor/ticket ni `GET /notifications` del tenant B puede recibir o leer ese evento (research.md §8, contracts/notifications-api.md § Aislamiento) — depende de T011, T012
- [X] T025 [US2] Ejecutar `python -m unittest app.characterization_tests.test_notifications -v` y corregir hasta que pase en verde — depende de T024, T041

**Checkpoint**: SC-005 satisfecho; ítem del checklist de aceptación marcado.

---

## Phase 5: User Story 3 - El cajero recibe notificación push cuando la pestaña no tiene foco (Priority: P2)

**Goal**: cuando la pestaña del POS no está enfocada (o el navegador minimizado), el cajero recibe una notificación push equivalente, sin duplicarse cuando sí está enfocada.

**Independent Test**: con permiso push concedido y la pestaña sin foco, generar un pedido nuevo y verificar que llega la notificación del sistema operativo/navegador (quickstart.md §2).

### Implementation for User Story 3

- [X] T026 [P] [US3] Implementar `BrowserPushChannel` en `../pos-backend/app/core/notifications/channels/browser_push.py` usando `pywebpush` + las claves VAPID de T002: envía a todas las `PushSubscription` `active=true` del tenant, marca `active=false` ante 404/410 (research.md §5, data-model.md § PushSubscription) — depende de T002, T006, T009
- [X] T027 [US3] Registrar `BrowserPushChannel` en la lista de canales de `../pos-backend/app/core/notifications/dispatch.py` — depende de T011, T026
- [X] T028 [P] [US3] Implementar `GET /notifications/push/public-key`, `POST /notifications/push/subscriptions` y `DELETE /notifications/push/subscriptions` en `../pos-backend/app/api/v1/notifications/router.py` (contracts/notifications-api.md) — depende de T006, T013
- [X] T029 [P] [US3] Crear `push-registration.service.ts` en `../pos-heladeria/src/app/core/notifications/push-registration.service.ts`: solicita permiso, llama `SwPush.requestSubscription()` y registra el resultado contra `POST /notifications/push/subscriptions` — depende de T003
- [X] T030 [US3] Conectar el aviso/botón de "activar notificaciones push" en `../pos-heladeria/src/app/modules/dashboard/layout/header.component.ts` a `push-registration.service.ts` — depende de T020, T029
- [X] T031 [P] [US3] Implementar el listener `push` de un Service Worker propio (p. ej. `../pos-heladeria/src/push-sw.ts`, registrado junto a `ngsw-worker.js`) que decide mostrar la notificación del sistema solo si ningún `WindowClient` de la app está enfocado (`clients.matchAll()`, research.md §5) — depende de T003
- [X] T032 [US3] Dar de baja la suscripción push (`DELETE /notifications/push/subscriptions`) en el flujo de cierre de sesión de `../pos-heladeria/src/app/core/services/auth.service.ts` — depende de T028, T029

**Checkpoint**: US3 funcional y verificable de forma independiente (quickstart.md §2).

---

## Phase 6: User Story 4 - El comensal anónimo ve el estado de su pedido y la confirmación de pago en tiempo real (Priority: P2)

**Goal**: el comensal con el menú QR abierto ve la confirmación de pago en tiempo real, sin recargar.

**Independent Test**: cobrar un pedido desde el POS y verificar que la pestaña del menú QR del comensal muestra la confirmación sin recargar (quickstart.md §3).

**Nota**: esta historia no depende de Foundational (no usa `NotificationEvent` ni los canales) — es un ajuste puntual sobre el bus SSE ya existente (research.md §6). Puede implementarse en paralelo con cualquier otra fase.

### Implementation for User Story 4

- [X] T033 [US4] En `../pos-backend/app/core/events.py::payment_completed()`, agregar `session_channel(table_session_id)` a `channels` cuando `table_session_id is not None` (contracts/realtime-events.md § Evento existente que cambia de enrutamiento)
- [X] T034 [US4] Agregar `this.realtime.on('payment.completed', ...)` en `../pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts`, junto a los handlers ya existentes (~líneas 1218-1226), para mostrar la confirmación de pago — depende de T033

**Checkpoint**: US4 funcional y verificable de forma independiente (quickstart.md §3).

---

## Phase 7: User Story 5 - El cajero recupera las notificaciones perdidas tras una desconexión (Priority: P3)

**Goal**: al reconectarse, el cajero ve las notificaciones de su tenant generadas mientras estuvo desconectado, sin duplicados.

**Independent Test**: simular pérdida de conexión, generar pedidos durante la desventana, reconectar y verificar que aparecen en el centro de notificaciones (quickstart.md §5).

### Implementation for User Story 5

- [X] T035 [US5] En `../pos-heladeria/src/app/core/notifications/notification-center.service.ts`, al detectar la reconexión de `RealtimeService.status` (o al iniciar el shell), llamar `GET /notifications?after_id=<último-id-visto>` y fusionar el resultado sin duplicados — depende de T012, T019
- [X] T036 [US5] Persistir localmente el id de la última notificación vista (p. ej. signal respaldado en `localStorage`) en `notification-center.service.ts`, para que la recuperación funcione también tras recargar la pestaña completa, no solo en una reconexión en caliente — depende de T035

**Checkpoint**: US5 funcional y verificable de forma independiente (quickstart.md §5).

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: retención/purga (RNF-005) y validación final — no ligadas a una sola historia de usuario.

- [X] T037 [P] Implementar `purge_expired_notifications()` en `../pos-backend/app/core/notifications/purge.py`: por cada schema de tenant, borra `NotificationEvent` con `purge_at < now()` (research.md §7, data-model.md § Nota de purga) — depende de T005
- [X] T038 Registrar el job de purga en `../pos-backend/app/core/scheduler.py::start_scheduler()` (`CronTrigger` diario + lock en Redis, mismo patrón que `sweep_orphan_sessions`/`expire_promotions`) — depende de T037
- [X] T039 [P] Ejecutar la validación completa de `quickstart.md` (los 8 escenarios) y registrar el resultado — depende de todas las fases anteriores. **Resultado**: §4 (SC-005) y §7 (SC-008) verificados con la suite automatizada (`test_notifications.py`, en verde) y con llamadas reales contra Postgres/Redis del entorno de desarrollo. §1/§6 (SC-001/002/007) verificados por HTTP real contra el backend en marcha (`notify_order_created` → `GET /notifications` → `POST .../attend`, ciclo completo con JWT real) y por inspección de código (`dispatch.py` es el único punto que conoce los canales). §8 (SC-009) no requiere prueba nueva (research.md, diseño ya existente). §2/§3/§5 (US3 push, US4 confirmación al comensal, US5 recuperación) requieren un navegador real con permiso de push y dos pestañas simultáneas (cajero + menú QR) — **no ejecutables en este entorno sin interfaz gráfica**; quedan pendientes de una validación manual en navegador antes de considerar la spec completamente cerrada
- [X] T040 [P] Actualizar `../pos-backend/.env.example` con las variables `VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` agregadas en T002
- [X] T041 [US2] Ampliar `../pos-backend/app/characterization_tests/test_notifications.py` (creado en T024) con la aserción de aislamiento de `POST /notifications/{notification_id}/attend`: el tenant B no puede marcar como atendida una notificación del tenant A — depende de T022 (Historia 1) y T024. Numerada fuera de secuencia con la Fase 4 para no forzar un renumerado del resto del documento; es, junto con T024/T025, parte de la prueba de aislamiento de US2 y debe ejecutarse antes de dar por cerrada esa historia (nota: es la única pieza de US2 que depende de una tarea de US1, ver "User Story Dependencies" arriba)

---

## Fase 9: Corrección post-implementación (bugs reportados en pruebas manuales)

**Propósito**: T039 dejó pendiente la validación manual en navegador de US1 (vivo, sin recargar), US3 (push) y US4 (comensal). Al hacerla, el usuario reportó 4 síntomas; se diagnosticaron con agentes de exploración de solo lectura contra el código real, confirmando 2 bugs de código concretos y 1 limitación esperada de Angular (con un bug latente de producción real dentro de ella). No se crea una spec nueva — esta fase corrige la 077 ya implementada.

- [X] T042 [P] Agregar `'notification.created'` a `KNOWN_EVENT_TYPES` en `../pos-heladeria/src/app/core/realtime/sse-client.ts` — sin esto, `EventSource.addEventListener` nunca se registraba para ese nombre de evento (lista hardcodeada) y el frame se descartaba en silencio antes de llegar a `RealtimeService.emit()`. Causa raíz de "la campanita no se actualiza si no recargo" y "no se muestra el aviso fuera de Terminal de Mesas" (T020/T021)
- [X] T043 [US4] Mover la conexión SSE del comensal de `PublicMenuComponent` a un nuevo `DinerShellComponent` (padre sin `path` que envuelve `menu/t/:token` y `menu/t/:token/checkout/**` en `../pos-heladeria/src/app/app.routes.ts`) — las dos rutas eran hermanas, así que navegar al checkout destruía `PublicMenuComponent` y con él la única conexión SSE del comensal durante todo el asistente de pago, justo cuando envía su pedido o el cajero puede cobrar. Mismo patrón que T018 ya aplicó del lado del staff (`DashboardLayoutComponent`). Incluye: `diner-shell.component.ts` nuevo (reacciona a `DinerTokenStore.token()` con un `effect()`), `public-menu.component.ts::connectRealtime()`/`disconnectRealtime()` dejan de abrir/cerrar la conexión directamente (solo gestionan sus propios `rtOff`), y `exit()` ahora llama `tokenStore.clear()` para seguir cerrando el stream al salir — depende de T018, T034
- [X] T044 Robustez del canal push en producción: `push-registration.service.ts::register()` ahora detecta `navigator.serviceWorker.controller === null` (primera visita, Service Worker instalado pero todavía sin controlar la página — `ngsw-worker.js` no llama `clients.claim()`) y devuelve `{ok: false, reason: 'needs-reload'}` en vez de dejar `requestSubscription()` colgado indefinidamente sin ningún error visible; `header.component.ts::enablePush()` interpreta el resultado y muestra un mensaje claro (`ToastService`) pidiendo recargar o reportando el error, en vez de ocultarlo en silencio — depende de T029, T030
- [X] T045 [P] Actualizar `quickstart.md` §2 (push) con la aclaración de que probar US3 exige un build de producción (`ng build --configuration production` o `ng serve --configuration production`) — `ng serve` normal nunca activa el Service Worker (`app.config.ts`: `enabled: !isDevMode()`), y §3 (comensal) con el escenario de checkout de T043 — depende de T042, T043, T044

**Checkpoint**: build limpio (`tsc --noEmit`, `ng build` dev+producción, `push-sw.js`/`ngsw-worker.js` presentes en el build de producción) y los 727 tests de `pos-backend` sin regresión (backend no se tocó en esta fase). T042 verificado end-to-end con una conexión SSE real: se abrió `GET /realtime/stream` con un ticket real y, al disparar `notify_order_created(...)` en otro proceso, el frame `event: notification.created` llegó completo por esa conexión — confirma que el backend siempre emitió correctamente el evento que el frontend descartaba. T043/T044 requieren clic-a-clic en un navegador real (checkout del comensal con el Network tab abierto; primera visita con push en un build de producción) — pendiente de que el usuario los confirme manualmente.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias, arranca de inmediato.
- **Foundational (Phase 2)**: depende de Setup — bloquea todas las historias **excepto** US4, que no toca el modelo de datos nuevo.
- **User Stories (Phase 3-7)**: US1, US2, US3, US5 dependen de Foundational; US4 es independiente y puede correr en paralelo con Foundational.
- **Polish (Phase 8)**: depende de que todas las historias que se vayan a entregar estén completas (T039 en particular corre al final).

### User Story Dependencies

- **US1 (P1)**: depende de Foundational. Es la base de la UI de notificaciones que US3 y US5 reutilizan (campanita, `NotificationCenterService`), pero es funcional y verificable por sí sola sin ellas.
- **US2 (P1)**: la cobertura de SSE/`GET /notifications` (T024) depende solo de Foundational (T011, T012), no de US1. La única excepción es T041 (aislamiento del endpoint `attend`), que depende de que la Historia 1 haya implementado T022 — es la única dependencia real de US2 sobre US1, documentada aquí en vez de ocultarse.
- **US3 (P2)**: depende de Foundational; reutiliza los puntos de disparo de US1 (T015-T017) para tener eventos que empujar por push, pero el canal push en sí (T026-T032) es un archivo/endpoint separado.
- **US4 (P2)**: sin dependencias de Foundational ni de otra historia — puede implementarse primero, en paralelo, o al final.
- **US5 (P3)**: depende de Foundational (`GET /notifications?after_id=`, T012) y de US1 (`NotificationCenterService`/campanita ya existente, T019) para tener dónde mostrar lo recuperado.

### Within Each User Story

- Modelos/migración antes que servicios; servicios antes que endpoints; backend antes que el frontend que lo consume, salvo cuando el frontend es un esqueleto de estado (T014) que no depende de ningún endpoint todavía.

### Parallel Opportunities

- T002 y T003 (Setup) en paralelo.
- T004-T007, T009-T010, T012, T014 (Foundational) en paralelo entre sí donde no comparten archivo (ver marcas `[P]`); T008 (migración) espera a T004-T007; T011 espera a T005/T007/T009/T010; T013 espera a T012.
- US4 completa (T033-T034) puede ejecutarse en paralelo con cualquier otra fase — no comparte archivos con ninguna.
- Dentro de US3: T026, T028, T029, T031 en paralelo (archivos distintos); T027, T030, T032 esperan a sus prerrequisitos respectivos.

---

## Parallel Example: Foundational

```bash
# Modelos nuevos, en paralelo (archivos distintos):
Task: "Crear el modelo NotificationEvent en app/models/notification_event.py"
Task: "Crear el modelo PushSubscription en app/models/push_subscription.py"
Task: "Crear el modelo NotificationChannelPreference en app/models/notification_channel_pref.py"
Task: "Agregar notification_retention_days a Tenant en app/core/models.py"

# Tras los modelos, la migración:
Task: "Generar la migración Alembic para las 3 tablas + la columna"
```

## Parallel Example: User Story 3

```bash
Task: "Implementar BrowserPushChannel en app/core/notifications/channels/browser_push.py"
Task: "Implementar los 3 endpoints de push subscriptions en app/api/v1/notifications/router.py"
Task: "Crear push-registration.service.ts en pos-heladeria"
Task: "Implementar el listener push del Service Worker con dedupe por foco"
```

---

## Implementation Strategy

### MVP First (US1 + US2 — las dos historias P1)

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (bloquea US1/US2/US3/US5; US4 puede adelantarse en paralelo).
3. Completar Fase 3: US1 — el cajero ya ve el aviso en cualquier sección del POS.
4. Completar Fase 4: US2 — se demuestra el aislamiento entre tenants con la prueba automatizada.
5. **Detener y validar**: correr quickstart.md §§1 y 4. Esto ya resuelve el problema central que motiva la spec y satisface las dos condiciones P1 (funcionalidad + seguridad no negociable).

### Incremental Delivery

1. Setup + Foundational → base lista (US4 puede ir en paralelo desde ya, es independiente).
2. US1 → validar independientemente → esto ya es un incremento demostrable (el cajero deja de perderse pedidos).
3. US2 → validar con la prueba automatizada → cierra el requisito de seguridad no negociable.
4. US3 → validar independientemente → cubre el caso de pestaña sin foco/navegador cerrado.
5. US4 → validar independientemente (si no se hizo antes) → cierra el lado del comensal.
6. US5 → validar independientemente → red de seguridad ante desconexiones.
7. Polish (purga por retención + validación completa de quickstart.md).

### Parallel Team Strategy

Con más de una persona disponible: Setup + Foundational en equipo; luego una persona en US1 (y detrás, US2 sobre lo mismo), otra en US4 desde el principio (sin esperar a Foundational), y quien termine primero se suma a US3 o US5.

---

## Notes

- `[P]` = archivos distintos, sin dependencias pendientes entre sí.
- `[Story]` mapea cada tarea a su historia de `spec.md` para trazabilidad (Principio XII de la constitución).
- US4 es la única historia sin dependencia de Foundational — candidata natural a adelantarse si se busca la entrega más rápida de valor visible al comensal.
- La prueba automatizada de US2 (T024, T025, T041) es la única exigida explícitamente por la spec; el resto de historias se validan con los pasos manuales de `quickstart.md`.
- Commitear tras cada tarea o grupo lógico; detenerse en cada checkpoint para validar la historia de forma independiente antes de continuar.
