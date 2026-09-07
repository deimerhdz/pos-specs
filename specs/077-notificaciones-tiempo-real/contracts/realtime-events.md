# Contrato: extensión del stream SSE existente (`/realtime/stream`)

**Spec**: [../spec.md](../spec.md) | **Research**: [../research.md](../research.md) §§ 1, 4, 6

Este contrato **extiende** `app/api/v1/realtime/router.py` (ya existente, sin cambios de endpoint ni de protocolo SSE); no crea un stream nuevo. Documenta solo lo que esta funcionalidad agrega al catálogo de eventos.

## Evento nuevo: `notification.created`

Emitido por el canal `in_app` (`app/core/notifications/channels/in_app.py`) inmediatamente después de persistir una `NotificationEvent`, vía `events.publish(tenant_id, type="notification.created", channels=[CH_STAFF], payload={...})` — reutiliza el mismo `events.publish()` de siempre, no una ruta nueva.

**Canal**: `staff` únicamente (nunca `session:{table_session_id}` — este evento es para el personal, no para el comensal).

**Frame SSE** (mismo formato que cualquier otro evento del stream, ver `_event_frame()`):

```
id: 1725730000123-0
event: notification.created
data: {"type":"notification.created","v":1725730000123000,"at":"2026-09-07T18:30:00Z","notification_id":"uuid","event_type":"order.created","related_entity_type":"customer_order","related_entity_id":"uuid","summary":"Mesa 4 · 3 ítems · $45.000"}
```

`summary` es texto ya formateado para mostrar en el toast/campanita sin que el frontend tenga que conocer la forma del payload de cada `event_type` — mantiene al centro de notificaciones desacoplado de los tipos de evento de negocio (research §4: escucha **un** tipo genérico, no cada evento fino del catálogo).

**Consumo esperado (frontend)**: el nuevo `NotificationCenterService` (en el shell, `core/notifications/`) se suscribe **únicamente** a `notification.created` — no a `order.created`/`payment.completed` directamente, que siguen siendo consumidos como hoy por los componentes que ya los usan (p. ej. `pos-terminal.store.ts`). Al recibirlo: incrementa el contador, dispara el toast + sonido (si `document.hasFocus()`), y opcionalmente hace `GET /notifications/{id}` bajo demanda si necesita el detalle completo (el `summary` ya alcanza para el aviso).

## Evento existente que cambia de enrutamiento: `payment.completed`

**Antes**: `channels=[CH_STAFF]` únicamente (`app/core/events.py::payment_completed`).

**Después**: `channels=[CH_STAFF, session_channel(table_session_id)]` cuando `table_session_id is not None` (ventas de mostrador sin mesa asociada, `table_session_id=None`, siguen sin llegar a ningún comensal — no aplica RF-005 porque no hay comensal QR involucrado).

**Payload**: sin cambios (`sale_id`, `table_session_id`, `total`, `customer_name`, `billing_mode`, `invoice`) — el comensal recibe exactamente el mismo evento que ya recibe el staff, no una versión reducida; `public-menu.component.ts` decide qué mostrar de ese payload (ya filtra así para el resto de eventos que ya escucha).

**Consumo esperado (frontend, comensal)**: `public-menu.component.ts` agrega `this.realtime.on('payment.completed', …)` junto a los handlers de `order.created`/`order.confirmed`/etc. que ya tiene (línea ~1218-1226 hoy) — mismo patrón, sin infraestructura nueva.

## Sin cambios

- Formato de frame SSE, heartbeat, `retry:`, replay por `Last-Event-ID`/`resync`: todo tal como está.
- Canal `session:{table_session_id}` para el resto de eventos de pedido (`order.created`, `order.confirmed`, `order.item_kitchen_changed`, `order.item_voided`, `order.cancelled`): sin cambios, siguen llegando al comensal exactamente igual que hoy.
- Autenticación/aislamiento (`/realtime/ticket` para staff, `token` firmado para comensal): sin cambios — el canal de un cliente se deriva del servidor, nunca de un parámetro que el cliente controle (NFR-003, ya vigente).
