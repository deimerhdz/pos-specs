# Data Model: Notificaciones en Tiempo Real Multi-Tenant

**Spec**: [spec.md](spec.md) | **Research**: [research.md](research.md)

Convenciones seguidas (ya vigentes en `pos-backend`, ver `app/core/models.py` y `app/models/audit_log.py`): entidades tenant-scoped viven en el schema `tenant` (`Base.metadata = MetaData(schema="tenant")`, resuelto por `schema_translate_map`); UUID como PK vía `UUIDPrimaryKeyMixin`; nomenclatura de constraints por la `naming_convention` ya declarada; referencias a `shared.users`/`shared.tenants` son **blandas** (sin `ForeignKey` cruzando de `tenant` a `shared`), igual que `AuditLog.user_id`.

## Entidades nuevas (schema `tenant`)

### NotificationEvent

Evento de notificación persistido y dirigido al staff de un tenant (RF-006). Es la copia durable de un hecho de negocio ya publicado en el bus de tiempo real (`app/core/events.py`), no lo reemplaza.

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID, PK | `UUIDPrimaryKeyMixin` |
| `event_type` | String(50) | `"order.created"` \| `"payment.completed"` — mismos literales que el catálogo de `events.py` (research §11) |
| `related_entity_type` | String(50) | `"customer_order"` \| `"sale"` |
| `related_entity_id` | UUID | id del pedido o de la venta según `event_type` |
| `payload` | JSONB | Delgado, mismo estilo que `events.py` ("eventos delgados"): ids + campos mínimos para render (mesa, cliente, total) — nunca el objeto completo |
| `created_at` | DateTime, index | `server_default=func.now()` |
| `purge_at` | DateTime, index | Calculado al insertar = `created_at + tenants.notification_retention_days` (vigente en ese momento). Ver Nota de purga abajo |
| `attended_at` | DateTime, nullable | `NULL` = pendiente. Compartido por tenant (research §10) |
| `attended_by_user_id` | UUID, nullable | Referencia blanda a `shared.users.id`. Trazabilidad, no una segunda dimensión de lectura |
| `delivery` | JSONB, nullable | Resumen de intentos por canal: `{"in_app": {"delivered_at": "..."}, "push": {"attempted": 3, "delivered": 2, "failed": 1}}`. Informativo — el estado operativo de cada `PushSubscription` (activa/caducada) vive en esa entidad, no aquí |

**Índices**: `(created_at)` para el listado/purga; `(purge_at)` para la condición `WHERE` del job de purga; `(attended_at)` opcional si el listado filtra "solo pendientes" con frecuencia.

**Reglas de negocio**:
- Se inserta exactamente una fila por invocación de `notify_order_created()` / `notify_payment_completed()` (research §4, §11) — nunca por los demás tipos del catálogo de `events.py`.
- `attended_at`/`attended_by_user_id` solo se fijan una vez (idempotente: marcar como atendida una notificación ya atendida no cambia quién ni cuándo, responde 200 igual — FR-006 clarificado).
- Nunca se actualiza `payload`/`event_type`/`related_entity_*` tras el insert (inmutable, como `AuditLog`).
- **Nota de purga**: `purge_at` se calcula con la retención vigente **al momento de crear la notificación**, no se recalcula si el tenant cambia su retención después — evita que bajar la retención purgue de golpe notificaciones que un cajero aún no vio; el job de purga (research §7) filtra por `purge_at < now()`, no recalculando por tenant en cada corrida.

### PushSubscription

Suscripción push de un dispositivo/navegador de un usuario del staff (RF-004, entidad `PushSubscription` de la spec).

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID, PK | |
| `user_id` | UUID | Referencia blanda a `shared.users.id` (mismo patrón que `AuditLog.user_id`) |
| `endpoint` | String(500), unique | URL del push service (FCM/Mozilla/etc.), tal como la entrega `PushSubscription.endpoint` del navegador |
| `p256dh_key` | String(255) | Clave pública de cifrado (`keys.p256dh` del `PushSubscriptionJSON`) |
| `auth_key` | String(255) | Secreto de autenticación (`keys.auth`) |
| `user_agent` | String(255), nullable | Diagnóstico (qué navegador/dispositivo es) |
| `created_at` | DateTime | `server_default=func.now()` |
| `last_seen_at` | DateTime, nullable | Se actualiza cada vez que el cliente vuelve a registrar la misma suscripción (heartbeat de vigencia) |
| `active` | Boolean, default `true` | Se pone en `false` cuando el push service responde 404/410 (suscripción caducada/revocada) — el envío al canal push nunca borra filas en caliente dentro de una request de negocio, solo las marca inactivas; una limpieza física es housekeeping fuera de esta spec |

**Índices/constraints**: `UNIQUE(endpoint)` — un endpoint de push corresponde a una única suscripción viva; si el navegador rota el endpoint, el cliente registra uno nuevo y el viejo queda `active=false` por el flujo normal de envío fallido.

**Reglas de negocio**:
- El canal push (research §5) intenta entregar a **todas** las filas `active=true` de todos los usuarios del tenant (no filtra por el usuario que disparó el evento) — coherente con FR-003 ("llega a todos los usuarios conectados del tenant").
- Un intento fallido con 404/410 marca `active=false`; otros errores (5xx, timeout) no desactivan la suscripción (podría ser un fallo transitorio del push service, no de la suscripción).

### NotificationChannelPreference

Asociación por tenant de qué canales están habilitados para cada tipo de evento (RF-010).

| Campo | Tipo | Notas |
|---|---|---|
| `id` | UUID, PK | |
| `event_type` | String(50) | Mismo dominio que `NotificationEvent.event_type` |
| `channel` | String(20) | `"in_app"` \| `"push"` (hoy); `"email"`/`"sms"`/`"whatsapp"` en el futuro sin cambiar la forma de la tabla |
| `enabled` | Boolean | |

**Constraint**: `UNIQUE(event_type, channel)`.

**Regla de lectura** (research §9): la **ausencia** de fila para un `(event_type, channel)` se interpreta como `enabled=true`. No se siembra ninguna fila por defecto en la migración — todo tenant existente sigue recibiendo ambos canales para los dos tipos de evento sin intervención.

## Cambios sobre entidades existentes

### `Tenant` (schema `shared`, `app/core/models.py`)

Se agrega una columna, mismo patrón que la ya existente `timezone`:

| Campo | Tipo | Notas |
|---|---|---|
| `notification_retention_days` | Integer, `nullable=False`, `server_default="90"` | Política de retención de RNF-005. Dato de configuración, no constante en código; hoy sin pantalla de edición (fuera de alcance), editable a futuro por el mismo mecanismo manual que hoy fija `timezone` (`app/scripts/set_tenant_timezone.py` tiene un equivalente implícito: un script o `UPDATE` directo) |

### `app/core/events.py` (no es un modelo, pero cambia comportamiento)

`payment_completed()` agrega `session_channel(table_session_id)` a `channels` cuando `table_session_id is not None` (research §6). Es el único cambio a una función ya existente en esta spec; el resto de funciones del catálogo no se tocan.

## Migración (Alembic)

Una sola revisión nueva bajo `alembic/versions/`, con:
1. `ALTER TABLE shared.tenants ADD COLUMN notification_retention_days INTEGER NOT NULL DEFAULT 90` (server-side default: no requiere backfill, ningún tenant existente queda con `NULL`).
2. `CREATE TABLE tenant.notification_events (...)` con sus índices.
3. `CREATE TABLE tenant.push_subscriptions (...)` con su `UNIQUE(endpoint)`.
4. `CREATE TABLE tenant.notification_channel_preferences (...)` con su `UNIQUE(event_type, channel)`.

Las tablas 2–4 se crean **por cada schema de tenant** — mismo mecanismo ya usado por toda migración de tenant en este proyecto (Alembic recorre `shared.tenants` para aplicar DDL en cada schema; no es un mecanismo nuevo de esta spec).

**Rollback**: `DROP TABLE` en orden inverso (3, 2, no rompen FKs porque son blandas hacia `shared`) + `ALTER TABLE shared.tenants DROP COLUMN notification_retention_days`. Sin pérdida de datos fuera de las propias notificaciones/suscripciones (no hay ninguna otra tabla que dependa de estas tres), consistente con el Principio VIII (estrategia de rollback declarada de antemano).

**Compatibilidad con datos existentes**: no hay backfill de filas en las tablas nuevas (arrancan vacías); el único dato retro-poblado es `notification_retention_days=90` vía `server_default`, sin tocar ninguna fila de negocio existente.
