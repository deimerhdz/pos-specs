# Contrato: API REST de notificaciones

**Spec**: [../spec.md](../spec.md) | **Data model**: [../data-model.md](../data-model.md)

Prefijo `app/api/v1/notifications/router.py`, mismo patrón que `app/api/v1/audit/router.py`: `Depends(get_current_user)` (cualquier usuario autenticado del tenant, no solo admin — a diferencia de `/audit-logs`, que exige `require_tenant_admin`), tenant resuelto por `Depends(get_tenant)`/`schema_translate_map` como en todo el dominio. Todas las respuestas están, por construcción, acotadas al schema del tenant del token — no existe ningún parámetro de tenant en la query ni en el body (NFR-003).

## `GET /notifications`

Lista notificaciones del tenant autenticado. Soporta dos modos de uso:

- **Navegación** (abrir el centro de notificaciones): `page`/`size`, igual que `GET /audit-logs`, orden `created_at DESC`.
- **Recuperación tras reconexión** (RF-008): `after_id` — devuelve las posteriores a esa notificación, orden `created_at ASC`, sin paginar por página sino hasta `limit`.

| Parámetro | Tipo | Notas |
|---|---|---|
| `page` | int, default 1 | Ignorado si se pasa `after_id` |
| `size` | int, default 20, max 100 | |
| `after_id` | UUID, opcional | Si se pasa, cambia al modo "recuperación": ignora `page`, usa `limit` |
| `limit` | int, default 100, max 500 | Solo aplica junto con `after_id` |
| `only_pending` | bool, default `false` | Filtra `attended_at IS NULL` |

**Respuesta** `200`:

```json
{
  "items": [
    {
      "id": "uuid",
      "event_type": "order.created",
      "related_entity_type": "customer_order",
      "related_entity_id": "uuid",
      "payload": { "table_number": 4, "customer_name": "...", "items_count": 3, "total": "45000" },
      "created_at": "2026-09-07T18:30:00Z",
      "attended_at": null,
      "attended_by_user_id": null
    }
  ],
  "total": 42,
  "page": 1,
  "size": 20
}
```

En modo `after_id`, `page`/`size`/`total` se omiten (o van `null`) y la respuesta es solo `items` en orden ascendente.

## `POST /notifications/{notification_id}/attend`

Marca una notificación como atendida — compartido por tenant (FR-006 clarificado): si ya estaba atendida por otro usuario, no cambia nada y responde `200` igual (idempotente).

**Respuesta** `200`:

```json
{
  "id": "uuid",
  "attended_at": "2026-09-07T18:31:05Z",
  "attended_by_user_id": "uuid-del-usuario-que-la-marco-primero"
}
```

`404` si la notificación no existe o no pertenece al tenant del token (nunca `403` con detalle — no filtrar por tenant es indistinguible de "no existe", ver `contracts/realtime-events.md` § aislamiento).

## `GET /notifications/push/public-key`

Devuelve la clave pública VAPID que el frontend necesita para `pushManager.subscribe({applicationServerKey})`.

**Respuesta** `200`: `{ "public_key": "BEl62iUY..." }` (misma clave para todos los tenants — es una credencial de la aplicación, no del tenant).

## `POST /notifications/push/subscriptions`

Registra o refresca una suscripción push del usuario autenticado (RF-004, entidad `PushSubscription`).

**Body**:

```json
{
  "endpoint": "https://fcm.googleapis.com/...",
  "keys": { "p256dh": "...", "auth": "..." },
  "user_agent": "Mozilla/5.0 ..."
}
```

**Comportamiento**: `UPSERT` por `endpoint` (único). Si ya existía (mismo endpoint, cualquier usuario/tenant — un endpoint de push es global al dispositivo/navegador), se reasigna al usuario/tenant actual, se marca `active=true` y se actualiza `last_seen_at`. Responde `201` (nueva) o `200` (refrescada).

## `DELETE /notifications/push/subscriptions`

**Body**: `{ "endpoint": "..." }`. Marca `active=false` (no borra físicamente — mismo criterio que una suscripción caducada, research.md §5). Se llama al cerrar sesión o si el usuario revoca el permiso desde el navegador. `204` siempre, sea cual sea el estado previo (idempotente).

## Aislamiento entre tenants (NFR-003, SC-005)

Ningún endpoint de este contrato acepta un identificador de tenant como parámetro. El tenant sale siempre de `Depends(get_tenant)` (resuelto del host/token, igual que el resto de la API), y las consultas ORM corren bajo el `schema_translate_map` de ese tenant — un usuario del tenant B no puede, ni pasando el `id` de una notificación del tenant A, leer o marcar como atendida esa fila: físicamente vive en otro schema de PostgreSQL, invisible a su conexión. Este es el mismo mecanismo de aislamiento que ya protege cualquier otro recurso tenant-scoped del sistema (pedidos, cajas, inventario) — no es un mecanismo nuevo inventado para notificaciones.
