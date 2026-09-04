# Contrato: entrada de log operativo (Sentry Logs)

Extensión de logging operativo (FR-015–FR-021, Clarifications tercera ronda). Al igual que `order-audit-log-event.md`, este feature no expone una API HTTP nueva para consultar este log — se consulta directamente en el panel de la herramienta externa de monitoreo. Este documento describe la **forma del payload** que el middleware nuevo (`app/core/error_middleware.py`, ver `research.md` § 7-10) envía a Sentry Logs por cada petición mutativa en alcance.

## Envoltorio (todas las entradas)

Igual que `order-audit-log-event.md`, atributos **planos** — nunca objetos anidados (mismo motivo técnico: `sentry_sdk` solo preserva como atributo filtrable un valor escalar, ver `research.md` § 1 del feature original). El primer argumento posicional (`template`) de `sentry_sdk.logger.*` es la etiqueta derivada de método + ruta (research.md § 7, regla 11 de `data-model.md`).

```json
{
  "method": "POST",
  "route": "/orders/{order_id}/cancel",
  "status": 200,
  "duration_ms": 42.7,
  "actor_id": "b7a2-...-user-uuid",
  "actor_type": "staff",
  "tenant_id": 7,
  "request_id": "e4d1...-request-uuid"
}
```

(`actor_id`/`actor_type`/`tenant_id` se omiten del diccionario, no se envían como `null`, cuando no se pudieron resolver — p. ej. una petición a `/api/v1/auth/login` fallida antes de identificar al usuario.)

## Reglas del payload

- `method`, `route`, `status`, `duration_ms`, `request_id` están presentes en el 100% de las entradas.
- `actor_id`/`actor_type`/`tenant_id` están presentes cuando la petición pasó por una de las 3 dependencias que los resuelven (`get_tenant`, `get_current_user`, `get_session_context` — research.md § 8); ausentes en caso contrario.
- **Nunca** hay una clave `body`, `payload`, `request_body`, `response_body` ni ninguna variante — el cuerpo de la petición/respuesta no viaja en esta entidad bajo ninguna forma (FR-018).
- El nivel de severidad del log (`info`/`warning`/`error`) se deriva de `status` (research.md § 9) — no es un campo del payload, es el nivel con el que se llama a `sentry_sdk.logger.*`.

## Nivel de severidad según `status`

| Rango de `status` | Nivel |
|---|---|
| `< 400` | `info` |
| `400`–`499` | `warning` |
| `>= 500` | `error` |

## Relación con el evento de auditoría de orden

Cuando la misma petición dispara uno de los 8 eventos de `OrderAuditEventType` (ver `order-audit-log-event.md`), ambas entradas existen para esa petición, correlacionadas por el mismo `request_id` (FR-021, FR-020) — pero son entidades independientes, con su propio `template`/mensaje en Sentry, y ninguna reemplaza a la otra. Un administrador investigando una orden filtra por `order_id` (como ya hacía); un desarrollador depurando un incidente técnico filtra por `request_id` y encuentra ambas si la petición correspondía a una de esas 8 transiciones, o solo la entrada operativa si no.

## Compatibilidad hacia adelante

Esta entidad no tiene un catálogo de valores fijos que versionar (a diferencia de `OrderAuditEventType`): `route` crece automáticamente con cada endpoint nuevo del backend, sin tocar este contrato. Cambiar la lista de campos del envoltorio (por ejemplo, agregar un campo nuevo) sí requiere actualizar este documento y el spec correspondiente.
