# Data Model: Auditoría del ciclo de vida de una orden

Este feature no persiste datos en base de datos (spec.md § Clarifications, decisión de alcance: sin tabla ni almacenamiento interno). Lo que sigue no es un esquema de tabla — es la forma del **payload estructurado** que cada evento envía a Sentry Logs, y las reglas de validación que ese payload debe cumplir para satisfacer los requisitos funcionales del spec.

## Evento de auditoría de orden

Entrada de log estructurada (no persistida por este sistema), emitida por `record_order_audit_event(...)` (`app/core/order_audit.py`, ver `research.md` § 1 y § 4).

**Dos representaciones distintas — no confundir**: la firma de `record_order_audit_event` (API interna en Python, tabla de abajo) recibe `actor`/`details` como objeto/dict anidado, por comodidad de quien la llama. Pero el payload que efectivamente sale hacia Sentry Logs es **plano** (sin objetos anidados) — `record_order_audit_event` es responsable de aplanar `actor`/`details` en atributos individuales antes de llamar a `sentry_sdk.logger.info`. La razón es técnica, no de diseño: Sentry Logs solo preserva como atributo filtrable un valor escalar (`bool`/`int`/`float`/`str`); un objeto anidado se degrada a un string de `repr()` ilegible (verificado contra `sentry-sdk==2.61.0`). Ver `contracts/order-audit-log-event.md` para la forma exacta del payload plano.

| Parámetro (API interna) | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `event_type` | `OrderAuditEventType` (enum de string) | Sí | Uno de los 7 valores fijos listados abajo. Se envía a Sentry como el `template`/mensaje del log y como atributo `event_type`. |
| `order_id` | string/UUID (id de `CustomerOrder`) | Sí | Identificador de la orden a la que pertenece el evento (FR-002). Atributo plano `order_id`. |
| `tenant_id` | int (`Tenant.id`) | Sí | Negocio activo en el momento del evento, capturado explícitamente por quien llama al helper (FR-004) — nunca inferido dentro del helper. Atributo plano `tenant_id`. |
| `actor` | `OrderAuditActor` (objeto) | Sí | Quién originó el evento (FR-003). Ver entidad abajo. Se aplana en los atributos `actor_type`/`actor_id`/`actor_role` (este último omitido, no `None`, cuando no aplica). |
| `occurred_at` | datetime (UTC, ISO 8601) | Sí | Momento en que la transición de negocio se completó (no el momento del intento, si hubo reintento de envío). Atributo plano `occurred_at`. |
| `details` | dict (forma específica por `event_type`) | Depende del tipo | Datos propios del evento — ver `contracts/order-audit-log-event.md` para la forma exacta por tipo. Cada clave de `details` se envía como su propio atributo plano de nivel superior (nunca agrupado bajo una clave `details`). Cualquier campo sensible dentro de `details` (nombre del comensal, comprobante de pago) va transformado con HMAC-SHA256 (FR-005, FR-012), nunca en texto plano. |
| `request_id` | string \| `None` | No (FR-021) | **Añadido en la extensión de logging operativo** — el `request_id` de la petición HTTP que originó este evento, cuando esa petición pasa por una ruta cubierta por el middleware de FR-015 (ver § Entrada de log operativo, más abajo). `None` cuando el evento no tiene una petición HTTP directa asociada con `request_id` resuelto. Atributo plano `request_id`, omitido (no enviado como `None`) cuando no aplica. |

### `OrderAuditEventType` (valores fijos)

```
order.created
order.confirmed
order.payment_attempt.created
order.payment.cash_confirmed
order.payment.transfer_approved
order.payment.transfer_rejected
order.cancelled
order.payment.checkout_and_send
```

No hay un noveno valor genérico ni una forma de extender este conjunto sin modificar el código — coincide 1:1 con los 8 puntos de integración de `research.md` § 4 (el octavo, `order.payment.checkout_and_send`, se añadió como adenda post-implementación — ver `spec.md` § Mapeo del flujo actual y FR-014).

## Actor (`OrderAuditActor`)

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `type` | enum: `comensal` \| `cajero` \| `sistema` | Sí | Exactamente uno de los tres (FR-003). No existe un cuarto valor ni un actor nulo — ver Regla de validación 1. |
| `id` | string \| `None` | Condicional | `participant_id` (UUID) cuando `type == comensal`; `user_id` cuando `type == cajero`; `None` cuando `type == sistema`. Nunca el nombre del comensal. |
| `role` | string \| `None` | Solo cuando `type == cajero` | El rol del usuario (`User.role_name`, p. ej. `ADMIN`, `CASHIER`). `None` para los otros dos tipos. |

Esta entidad nunca es una referencia directa (FK) a la tabla de usuarios — es un valor plano incluido en el payload del log, igual que el patrón de soft-reference (`user_id` sin FK) ya usado en el resto del módulo de órdenes (`OrderCancelLog.user_id`, `OrderPaymentAttempt.resolved_by_user_id`, etc.).

## Relación con la orden existente

La "orden" (`CustomerOrder`, entidad ya existente en `pos-backend`, no introducida por este feature) atraviesa el estado `recibida → abierta → (bloqueada) → pagada`, con `cancelada` como terminal alternativo desde cualquier estado no terminal (documentado en `customer_order.py`). Este feature no modifica esa máquina de estados — solo observa cuándo cada transición se completa y emite el evento correspondiente. La correspondencia entre transición de negocio y `event_type` es la de la tabla de `research.md` § 4.

## Reglas de validación

1. **Actor exhaustivo y excluyente** (FR-003): `actor.type` DEBE ser exactamente uno de `comensal`/`cajero`/`sistema`; nunca vacío, nulo ni un cuarto valor. Verificado por SC-002.
2. **Tenant siempre presente** (FR-004): `tenant_id` DEBE estar poblado en el 100% de los eventos; el helper rechaza (falla rápido, en tiempo de desarrollo/test) una llamada sin `tenant_id`.
3. **Orden siempre presente** (FR-002): `order_id` DEBE estar poblado en el 100% de los eventos.
4. **Sin texto plano sensible** (FR-005, FR-012, SC-003): cuando `details` referencia el nombre del comensal o el comprobante de pago, el valor DEBE ser la salida de HMAC-SHA256 (ver `research.md` § 3) — nunca el valor original. La misma entrada produce siempre el mismo hash (mientras la clave `AUDIT_HASH_SECRET` no cambie), permitiendo reconocer repeticiones sin revelar el valor.
5. **Emisión solo tras éxito** (FR-010): el helper se invoca únicamente después de que el `commit` de la transición correspondiente fue exitoso; si la transición falla o se revierte, no se emite ningún evento para ese intento.
6. **No bloqueante** (FR-011): una excepción o fallo dentro de `record_order_audit_event` (p. ej. al enviar a Sentry) nunca se propaga hacia la función de servicio que la invoca — se captura y se reintenta/descarta internamente, sin afectar la respuesta HTTP de la transición de negocio ya completada.
7. **Categoría distinguible** (FR-007): todo evento de auditoría se emite con un `event_type` reconocible (prefijo `order.`), distinto de los mensajes de log operativo/error existentes.

---

# Extensión: Entrada de log operativo (FR-015–FR-021)

Entrada de log estructurada, sin curar, emitida por el middleware nuevo (`app/core/error_middleware.py`, ver `research.md` § 7-10) — entidad distinta del "Evento de auditoría de orden" de arriba: más genérica, cubre cualquier ruta mutativa del backend (no solo las 8 de orden), y su etiqueta se deriva automáticamente en vez de curarse a mano.

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `method` | string (`POST`\|`PUT`\|`PATCH`\|`DELETE`) | Sí | Método HTTP de la petición. Las peticiones `GET`/`HEAD`/`OPTIONS` nunca llegan a generar esta entidad (FR-016). |
| `route` | string | Sí | Patrón de ruta registrado en FastAPI (p. ej. `/orders/{order_id}/cancel`), no la URL con los valores reales de los parámetros (research.md § 10). |
| `status` | int | Sí | Código de estado HTTP de la respuesta. Determina la severidad del log (research.md § 9). |
| `duration_ms` | number | Sí | Duración de la petición, en milisegundos, medida desde que el middleware la recibe hasta que la respuesta está lista. |
| `actor_id` | string \| ausente | No | Identificador del actor que ejecutó la petición (`user_id` o `participant_id`), si se pudo resolver durante el procesamiento de la petición (research.md § 8). Ausente, no `null`, cuando no se resolvió ninguno. |
| `actor_type` | string (`staff`\|`comensal`) \| ausente | No | Tipo de actor, cuando `actor_id` está presente. |
| `tenant_id` | int \| ausente | No | Negocio/tenant activo, si se pudo resolver (research.md § 8). |
| `request_id` | string | Sí | Identificador único de la petición — el mismo que, cuando aplica, viaja también en el evento de auditoría de orden correspondiente (FR-021). |

**Nunca incluye** (FR-018): el cuerpo de la petición ni de la respuesta, en ninguna forma ni parcialmente.

## Reglas de validación (extensión)

8. **Solo mutativas** (FR-016): esta entidad solo se genera para peticiones `POST`/`PUT`/`PATCH`/`DELETE`. Una petición `GET`/`HEAD`/`OPTIONS` NUNCA produce una.
9. **Nunca super-admin** (FR-015): una petición a `/api/v1/super-admin` NUNCA produce esta entidad — ese prefijo sigue teniendo su propio mecanismo, sin cambios.
10. **Sin cuerpo, sin excepciones** (FR-018): ningún campo de esta entidad contiene, ni siquiera parcialmente, el cuerpo de la petición o de la respuesta.
11. **Etiqueta automática** (FR-019): el `template`/mensaje enviado a `sentry_sdk.logger.*` se deriva de `method` + `route` (p. ej. `"POST /orders/{order_id}/cancel"`), sin una tabla de nombres curada a mano por endpoint.
12. **Convive con la auditoría de orden, no la reemplaza** (FR-020): una petición que dispara uno de los 8 eventos de `OrderAuditEventType` DEBE generar también su "Entrada de log operativo" correspondiente — ambas entidades coexisten para la misma petición.
13. **No bloqueante** (mismo principio que la regla 6, aplicado al middleware): una falla al construir o enviar esta entrada NUNCA se propaga hacia la petición real ni altera su respuesta.
