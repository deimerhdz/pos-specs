# Research: Notificaciones en Tiempo Real Multi-Tenant

**Spec**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md)

Este documento resuelve las decisiones técnicas necesarias para pasar de la spec al diseño (`data-model.md`, `contracts/`). Cada decisión cita el requisito de spec que resuelve. Ningún punto quedó como `NEEDS CLARIFICATION`: el propio código de `pos-backend`/`pos-heladeria` ya resolvía la mayoría de las incógnitas del contexto técnico sugerido en la spec.

## 1. Backbone de distribución en tiempo real

**Decisión**: Reutilizar tal cual el bus ya existente — `app/core/events.py` (catálogo de eventos + `XADD` a un stream Redis por tenant, `events:tenant:{id}`) y `app/core/event_bus.py` (un `XREAD BLOCK` por proceso+tenant, fan-out en memoria a suscriptores locales vía `asyncio.Queue`, sin sticky sessions). No se introduce Redis Pub/Sub ni un backbone nuevo.

**Rationale**: Ya cumple, en producción, exactamente lo que RNF-001/002/003/004 piden: sin polling (XREAD BLOCK), multi-instancia sin duplicar ni perder (un lector por proceso+tenant, Redis como único punto de serialización), aislamiento derivado del claim firmado del token/ticket (nunca de lo que el cliente pida), y latencia de segundos. Además ya resuelve el replay (`Last-Event-ID` → `XRANGE`) que RF-008 pide para el staff. Reconstruirlo violaría el Principio V (sin refactors no relacionados) y el objetivo explícito de la propia spec de "no rehacer el despachador de eventos".

**Alternativas consideradas**: Redis Streams/Pub/Sub nuevo dedicado a notificaciones — rechazado, sería un segundo bus paralelo al que ya sirve `/realtime/stream`, duplicando infraestructura sin necesidad. WebSockets — rechazado por el mismo motivo que ya documenta `app/api/v1/realtime/router.py`: el flujo es unidireccional servidor→cliente, SSE ya da reconexión/backoff/replay gratis sobre HTTP plano.

## 2. Dónde vive la conexión SSE de staff en el frontend (RF-003)

**Decisión**: Mover las llamadas a `RealtimeService.connectStaff()` / `.disconnect()` desde `pos-terminal.store.ts` (propietario hoy de su ciclo de vida) al shell autenticado — `dashboard-layout.component.ts`, que envuelve todas las secciones del POS (Ventas, Terminal de Mesas, Inventario, Reportes, etc.). `pos-terminal.store.ts` deja de abrir/cerrar la conexión y sigue suscribiéndose con `realtime.on('order.created', …)` exactamente igual que hoy — cero cambio en su lógica de negocio.

**Rationale**: `RealtimeService` ya es `providedIn: 'root'` (singleton) y ya expone un API de suscripción por tipo de evento (`on()`) independiente de quién abrió la conexión — el bug no es de arquitectura de servicio, es de *quién* controla el ciclo de vida. Conectar en el shell (que vive mientras dura la sesión autenticada) en vez de en una página hija resuelve exactamente el escenario que motiva la spec ("cajero en Ventas no se entera").

**Alternativas consideradas**: Un `APP_INITIALIZER` que conecte en el arranque de la app — rechazado porque en el arranque puede no existir sesión válida todavía (login pendiente); el shell ya solo se monta tras autenticar. Mantener la conexión en cada página y multiplicar `connectStaff()` — rechazado: reabrir/cerrar el stream en cada navegación reintroduce el hueco de reconexión que motivó construir el replay por `Last-Event-ID` en primer lugar.

## 3. Persistencia de `NotificationEvent` (RF-006, RNF-005)

**Decisión**: Tabla nueva en el schema `tenant` (mismo `schema_translate_map` que el resto del dominio), siguiendo el patrón ya usado por `AuditLog` (`app/models/audit_log.py`): payload `JSONB` delgado, columnas de primera clase para lo que se filtra/indexa (`related_entity_type`, `related_entity_id`, `created_at` indexado), UUID como PK.

**Rationale**: Es exactamente el mismo problema que ya resolvió `AuditLog` (bitácora tenant-scoped de hechos de negocio con payload variable) — reutilizar el patrón evita inventar una segunda convención de "tabla de eventos" en el mismo repositorio. El stream de Redis (`events:tenant:{id}`) **no sirve** como almacén de los 90 días de retención de RNF-005: su `MAXLEN` (`REALTIME_STREAM_MAXLEN`, hoy 1000) es un tope de tamaño para el replay en caliente, no una política de retención por tenant, y no es consultable vía API REST paginada.

**Alternativas consideradas**: Guardar solo en Redis con TTL largo — rechazado, Redis aquí es infraestructura de tiempo real (`REALTIME_ENABLED` puede desactivarse, y su contenido no es la fuente de verdad en ningún otro punto del sistema: "Postgres sigue siendo la fuente de verdad", cita textual del docstring de `events.py`); además RNF-005 exige que se pueda consultar vía API tras la desconexión, lo que ya es el patrón establecido para todo el dominio (Postgres + endpoints REST).

## 4. Capa de despacho desacoplada de canales (RF-009, RF-010)

**Decisión**: Paquete nuevo `app/core/notifications/` con:
- `dispatch.py`: dos funciones de una línea por punto de llamada, `notify_order_created(...)` y `notify_payment_completed(...)`, análogas a las que ya existen en `events.py`. Cada una (a) inserta una fila en `NotificationEvent` y (b) recorre los canales habilitados para `(tenant_id, event_type)` (tabla de preferencias, ver research §9) llamando a `channel.send(event)`.
- `channels/base.py`: un `Protocol`/ABC `NotificationChannel` con un único método `send(event: NotificationEvent) -> None`.
- `channels/in_app.py`: implementación que llama a `events.publish(tenant_id, type="notification.created", channels=[CH_STAFF], payload={...})` — **reutiliza el bus existente sin tocarlo**, solo agrega un tipo de evento nuevo al catálogo.
- `channels/browser_push.py`: implementación nueva, ver research §5.

Los puntos de llamada de negocio (`cart/router.py` para `order.created`, y los tres sitios que llaman `events.payment_completed` en `orders/router.py`/`table_sessions/service.py`) agregan **una línea** (`notify_order_created(...)` / `notify_payment_completed(...)`) junto a la llamada a `events.*` que ya hacían — no la reemplazan, porque `events.publish()` sigue siendo lo que alimenta las pantallas que ya escuchan tipos de evento específicos (p. ej. la grilla de Terminal de Mesas escuchando `order.created`, `order.confirmed`, etc.), y esa lógica no cambia (Principio II).

**Rationale**: Es el mecanismo mínimo que cumple RF-009 literalmente — "agregar un canal no requiere modificar la lógica de negocio que origina el evento": añadir email/SMS/WhatsApp más adelante es añadir un archivo `channels/email.py` que implemente el mismo `Protocol` y registrarlo en la tabla de preferencias; ningún router ni servicio de negocio se toca. Mantener `notify_*()` separado de `events.*()` (en vez de fusionarlos) evita acoplar "avisar al negocio en tiempo real de un cambio de grilla" (lo que ya hace `events.py`, con eventos muy finos como `order.item_kitchen_changed`) con "generar una notificación persistente y multicanal" (lo que exige esta spec, limitado por clarificación a solo `order.created`/`payment.completed`) — son necesidades distintas con volúmenes de evento muy distintos.

**Alternativas consideradas**: Convertir `events.publish()` mismo en el punto de persistencia+fan-out multicanal — rechazado: dispararía una fila de `NotificationEvent` y un intento de push por cada uno de los ~10 tipos de evento finos del catálogo (`order.item_kitchen_changed`, `bill_changed`, etc.), en contra de la clarificación de spec que acota el disparador a solo 2 tipos, y mezclaría el "bus de sincronización de UI" con el "sistema de notificación al usuario".

## 5. Canal push del navegador (RF-004)

**Decisión**: Backend: librería `pywebpush` (implementa el protocolo Web Push / RFC 8030 y el cifrado de mensaje RFC 8291 con claves VAPID). Frontend: paquete oficial `@angular/service-worker` (registra el Service Worker, expone `SwPush` para pedir permiso y suscribirse con `pushManager.subscribe()`).

**Rationale — nueva dependencia backend (Principio IX)**: el problema que resuelve es implementar correctamente el cifrado de extremo a extremo que exige el estándar Web Push (ECDH + HKDF sobre las claves `p256dh`/`auth` de cada suscripción); reimplementarlo a mano es exactamente el tipo de criptografía que un proyecto no debe reinventar. `cryptography` (dependencia de la que `pywebpush` depende) ya está en `requirements.txt`. Alternativas consideradas: implementar el cifrado ECE a mano sobre `cryptography` — rechazado por riesgo de seguridad y mantenimiento de una implementación de protocolo no trivial; un servicio SaaS de push (Firebase Cloud Messaging, OneSignal) — rechazado porque exige credenciales de terceros y una dependencia operativa externa para un requisito que el estándar Web Push ya resuelve sin intermediario, y porque la spec explícitamente limita el alcance a "push del navegador" sin canales externos.

**Rationale — nueva dependencia frontend (Principio IX)**: `@angular/service-worker` es un paquete de primera parte del mismo framework ya usado (Angular 21), mantenido junto al resto de `@angular/*`; evita escribir a mano el ciclo de vida de registro/actualización del Service Worker. Alternativa considerada: Service Worker manual sin el paquete de Angular — rechazado, sería reimplementar el manejo de versiones/actualización que el paquete oficial ya cubre, sin beneficio para este alcance.

**Deduplicación visual+push con la pestaña enfocada (User Story 3, escenario 2)**: el backend no puede saber si una pestaña concreta tiene foco (múltiples pestañas/dispositivos por cajero, edge case ya documentado en spec), así que el backend **siempre** intenta el push para cada `PushSubscription` del tenant cuando el evento está habilitado para ese canal (fire-and-forget, mismo estilo *fail-open* que `events.publish()`: un push fallido no debe tumbar la operación de negocio). La decisión de **mostrar o no** el `Notification` del sistema operativo se toma del lado del cliente, en el listener `push` del Service Worker, usando `self.clients.matchAll({type: 'window', includeUncontrolled: true})` y comprobando si algún `WindowClient` de la app está `focused`; si lo hay, se omite `showNotification()` (ya se vio el aviso visual+sonoro en la pestaña activa) — esto es el patrón estándar recomendado para evitar doble aviso y es lo único que puede resolver ese escenario, porque solo el navegador conoce el estado de foco de sus propias pestañas.

## 6. Confirmación de pago al comensal en tiempo real (RF-005)

**Decisión**: Un cambio de una línea en `app/core/events.py::payment_completed()`: agregar `session_channel(table_session_id)` a la lista `channels` (hoy solo `[CH_STAFF]`), condicionado a que `table_session_id` no sea `None` — mismo patrón defensivo que ya usan `order_cancelled`/`bill_changed` en el mismo archivo. Y en el frontend, `public-menu.component.ts` (la vista del menú QR del comensal) agrega un handler `this.realtime.on('payment.completed', …)` junto a los que ya tiene para `order.created`/`order.confirmed`/etc.

**Rationale**: Es la brecha exacta que motiva RF-005 — hoy el evento existe y ya viaja por el bus, pero nunca se enruta al canal de sesión del comensal (`session:{table_session_id}`), al que su `EventSource` (`connectDiner(token)`) ya está suscrito desde que abre el menú QR. No hace falta persistencia nueva ni un canal nuevo para este lado: el comensal es anónimo y, según la Asunción de spec, no necesita historial de notificaciones perdidas — solo ver el estado vigente al reabrir, lo que ya resuelve el REST existente de consulta de pedido.

**Alternativas consideradas**: Emitir un evento nuevo dedicado `payment.confirmed_for_diner` en vez de sumar el canal al `payment.completed` existente — rechazado, duplicaría el evento sin necesidad; el mismo evento ya lleva la información que el comensal necesita (`sale_id`, `total`), y `session_channel()` ya filtra correctamente para que solo ese comensal lo reciba (aislamiento de sesión, no solo de tenant).

## 7. Job de purga por retención (RNF-005)

**Decisión**: Un job nuevo en `app/core/scheduler.py`, mismo patrón que `sweep_orphan_sessions`/`expire_promotions`: `AsyncIOScheduler` de APScheduler (ya arrancado en el lifespan de la app), `CronTrigger` diario, lock `SET NX EX` en Redis (mismo mecanismo que `_LOCK_KEY`/`_PROMO_LOCK_KEY`) para que un solo worker lo ejecute por ciclo, e iteración sobre `shared.tenants` abriendo `with_db(schema)` por tenant. La condición de purga por fila es `purge_at < now()` — `purge_at` es una columna calculada al crear cada `NotificationEvent` como `created_at + tenants.notification_retention_days` **vigente en ese momento** (ver data-model.md § Nota de purga); el job no recalcula contra la retención actual del tenant en cada corrida, para que bajar la retención no purgue de golpe notificaciones ya persistidas bajo una ventana mayor. El valor de retención sigue siendo un dato de configuración por tenant, no una constante en código — satisface RNF-005 literalmente; lo único que cambia frente a un recálculo en vivo es *cuándo* se aplica ese valor (al crear la notificación, no en cada barrido).

**Rationale**: Es el mecanismo ya establecido en el propio módulo para "recorrer todos los tenants y aplicar una regla de expiración de forma periódica" (`expire_promotions` es casi el mismo problema: expirar filas vencidas por fecha, por tenant, con lock). No se introduce Celery Beat pese a que Celery ya es una dependencia del proyecto, porque el patrón de "job periódico con lock en Redis sobre APScheduler" ya es el establecido en este mismo archivo para necesidades equivalentes — usar un scheduler distinto para esta tarea rompería la consistencia interna sin ninguna ganancia.

**Alternativas consideradas**: Celery Beat — rechazado, introduciría un segundo mecanismo de programación periódica coexistiendo con el que ya usa `scheduler.py`, sin necesidad (ni Celery Beat está configurado hoy en el proyecto: `celery_task.py` es para tareas asíncronas bajo demanda, no periódicas). Borrado perezoso al leer (filtrar `created_at >= corte` en cada `GET /notifications` sin borrar físicamente) — rechazado, RNF-005 exige explícitamente que se purguen ("dejar de ser accesibles vía API" no basta; deben "purgarse automáticamente").

## 8. Prueba automatizada de aislamiento entre tenants (SC-005)

**Decisión**: Nuevo módulo `app/characterization_tests/test_notifications.py` (a pesar del nombre del paquete, sigue el precedente ya sentado por `test_operational_log.py`/spec 074: comportamiento **nuevo**, verificado contra este spec, no characterization test de comportamiento heredado). Usa `TestClient` con dos tenants reales (fixtures ya existentes de `characterization_tests/fixtures.py`), abre una "conexión" simulada (o llamada directa a `event_bus`/`dispatch`) para cada uno, dispara un `order.created` para el tenant A, y verifica que ningún suscriptor del tenant B lo recibe ni puede abrir un ticket/suscripción sobre el stream del tenant A.

**Rationale**: Sigue la convención de testing ya establecida en el repo (`unittest` + `TestClient`, no pytest) en vez de introducir un framework de test nuevo. El propio aislamiento a nivel de transporte ya está garantizado por diseño (el canal se deriva del claim firmado, nunca de un parámetro del cliente — research §1), así que esta prueba es la evidencia exigida por el checklist de aceptación de la spec, no un parche a una vulnerabilidad conocida.

## 9. Configuración de canal habilitado por tipo de evento (RF-010)

**Decisión**: Tabla `notification_channel_preferences` (schema tenant): `(event_type, channel, enabled)` con `UNIQUE(event_type, channel)`. Regla de lectura: **la ausencia de fila para un `(event_type, channel)` significa habilitado** (default universal para "in_app" y "push" en los 2 tipos de evento de este alcance) — así ningún tenant existente necesita una fila sembrada por migración para seguir recibiendo notificaciones tal como las recibe hoy.

**Rationale**: Es el modelo mínimo que cumple RF-010 ("el modelo debe permitir asociar, por tenant, qué canales están habilitados por tipo de evento") sin construir la interfaz de administración que la propia spec deja fuera de alcance. Vive en el schema tenant (no en `shared`) porque es una preferencia de negocio del tenant, igual que el resto de configuración de dominio (no es metadata de la plataforma como `tenants.schema`/`tenants.host`).

**Alternativas consideradas**: Guardarlo como columna JSON en `tenants` (schema `shared`) — rechazado, mezclaría configuración de dominio (qué notificaciones quiere un tenant) con metadata de plataforma, y RF-010 ya habla de "por tipo de evento", que pide una fila por combinación, no un blob difícil de indexar/validar.

## 10. Estado de lectura/atención compartido por tenant (clarificación de spec, FR-006)

**Decisión**: Dos columnas nullable en `NotificationEvent`: `attended_at: datetime | None` y `attended_by_user_id: UUID | None` (referencia blanda a `shared.users.id`, mismo patrón que `AuditLog.user_id`). `NULL` = pendiente para todo el tenant; una vez cualquier cajero llama `POST /notifications/{id}/attend`, ambos campos se fijan y quedan así para todos — "compartido" significa que solo hay **una** marca de atención por evento, no una marca por usuario.

**Rationale**: Es la traducción directa de la decisión de negocio ya registrada en `spec.md` (bandeja de equipo). Guardar además `attended_by_user_id` no contradice "compartido" — es trazabilidad (quién lo atendió), no una segunda dimensión de lectura por usuario; ningún requisito pide ocultarlo.

## 11. Alcance de eventos disparadores (clarificación de spec, FR-002)

**Decisión**: Solo dos disparadores llaman a `notify_*()`: `order.created` (`cart/router.py`, comensal confirma su carrito) y `payment.completed` (los tres sitios ya identificados en `orders/router.py` y `table_sessions/service.py`). Ningún otro tipo del catálogo de `events.py` (`order.confirmed`, `order.item_kitchen_changed`, `order.item_voided`, `order.cancelled`, `bill_changed`, `session_closed`, `table_status_changed`) genera `NotificationEvent` ni push en el alcance de esta funcionalidad.

**Rationale**: Es la clarificación de negocio ya resuelta en `spec.md` (Sesión 2026-09-07). Acotar el disparador a solo estos dos evita ruido (un `order.item_kitchen_changed` por cada movimiento de un ítem en cocina generaría decenas de notificaciones persistentes y de intentos de push por pedido, muy por encima del volumen que la spec pretende cubrir).
