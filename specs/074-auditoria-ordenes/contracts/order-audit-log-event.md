# Contrato: evento de auditoría de orden (Sentry Logs)

Este feature no expone una API HTTP nueva (spec.md § FR-008: la consulta se hace directamente en el panel de Sentry, no en un endpoint propio). El "contrato" que sí expone — y que cualquier futuro consumidor (otro desarrollador buscando en Sentry, un dashboard futuro, etc.) debe poder asumir estable — es la **forma del payload** que cada tipo de evento envía a Sentry Logs vía `record_order_audit_event(...)`.

## Envoltorio común (todos los eventos)

**Importante — atributos planos, no objetos anidados**: Sentry Logs (`sentry_sdk.logger.info(template, attributes={...})`) solo preserva como campo estructurado/filtrable un valor `bool`/`int`/`float`/`str` (o una lista homogénea de esos) por atributo — verificado contra el código real de `sentry-sdk==2.61.0` (`sentry_sdk/utils.py::format_attribute`). Cualquier objeto o dict anidado como valor de un atributo se degrada a un string de `repr()` de Python, ilegible y no filtrable en el panel. Por eso el payload que efectivamente viaja a Sentry es **plano**: `actor` se descompone en `actor_type`/`actor_id`/`actor_role`, y cada campo de `details` (ver tablas por tipo de evento, todos son escalares) se envía como su propio atributo de nivel superior, sin agrupar bajo una clave `details`. `record_order_audit_event` recibe `actor`/`details` como objeto/dict en su firma interna (API cómoda en Python) pero es responsable de aplanarlos antes de llamar a `sentry_sdk.logger.info` (ver `data-model.md` § Evento de auditoría de orden). El primer argumento posicional de `sentry_sdk.logger.info` (el `template`/mensaje) es el valor de `event_type` (p. ej. `"order.created"`). Un atributo cuyo valor sería `None` (p. ej. `actor_role` para un comensal, `reason` sin registrar) se **omite** del diccionario de atributos en vez de enviarse — de lo contrario `format_attribute(None)` produciría el string literal `"None"`.

```json
{
  "event_type": "order.created",
  "order_id": "b3f1...",
  "tenant_id": 42,
  "actor_type": "comensal",
  "actor_id": "b7a2-...-participant-uuid",
  "occurred_at": "2026-09-03T18:42:10Z"
}
```

(`actor_role` omitido en este ejemplo por ser `None` — comensal no tiene rol. Los campos de `details` de cada tipo de evento, listados en las tablas de abajo, se añaden como atributos adicionales de nivel superior en este mismo diccionario plano — nunca agrupados bajo una clave `details`.)

Reglas generales (ver `data-model.md` § Reglas de validación para el detalle normativo):
- `event_type`, `order_id`, `tenant_id`, `actor_type`, `occurred_at` están presentes en el 100% de los eventos, sin excepción. `actor_id`/`actor_role` están presentes cuando aplican (ver `data-model.md` § Actor).
- Los campos listados como "en `details`" en las tablas de abajo se envían como atributos planos adicionales, con ese mismo nombre, al mismo nivel que `order_id`/`tenant_id`/etc.
- Ningún atributo contiene el nombre del comensal o el comprobante de pago en texto plano; cuando aplica, aparece como `*_hash` (salida de HMAC-SHA256, ver `research.md` § 3).

## `order.created`

Emitido al completarse la creación de la orden (vía QR o manual por staff).

| Campo en `details` | Tipo | Presente cuando | Descripción |
|---|---|---|---|
| `channel` | `"QR_MENU"` \| `"POS"` | siempre | Origen de la orden (`CustomerOrder.channel`). |
| `order_type` | `"DINE_IN"` \| `"TAKEAWAY"` \| `"DELIVERY"` \| `null` | siempre | `CustomerOrder.order_type`. |
| `hold_for_payment` | bool | `channel == "POS"` | Si la orden manual nació en espera de pago (`recibida`) o abierta de inmediato. |
| `diner_name_hash` | string \| `null` | `channel == "QR_MENU"` | HMAC-SHA256 del `display_name` capturado en la sesión de mesa — nunca el nombre en texto plano (FR-005). |

## `order.confirmed`

Emitido al completarse la transición `recibida → abierta`, sin importar si fue manual o automática.

| Campo en `details` | Tipo | Descripción |
|---|---|---|
| `trigger` | `"manual"` \| `"automatic_payment"` | Si se llamó directamente el paso de confirmación (ruta de recuperación) o si ocurrió como efecto colateral de confirmar un pago (ver Edge Case del spec). |

Cuando `trigger == "automatic_payment"`, el `actor` de este evento es `{"type": "sistema", "id": null, "role": null}` — el cajero que confirmó el pago queda registrado en el evento de pago correspondiente (`order.payment.cash_confirmed` o `order.payment.transfer_approved`), correlacionable por el mismo `order_id`.

## `order.payment_attempt.created`

Emitido al registrarse un intento de pago sobre la orden.

| Campo en `details` | Tipo | Presente cuando | Descripción |
|---|---|---|---|
| `payment_method_type` | `"cash"` \| `"transfer"` \| `"other"` | siempre | Tipo del método de pago del catálogo. |
| `payment_method_name` | string | siempre | Nombre del método (p. ej. "Nequi") — no es dato sensible, es un nombre de catálogo del negocio. |
| `receipt_hash` | string \| `null` | `payment_method_type == "transfer"` | HMAC-SHA256 del comprobante adjunto — nunca su contenido/URL original (FR-005). |

## `order.payment.cash_confirmed`

| Campo en `details` | Tipo | Descripción |
|---|---|---|
| `amount_received` | number | Monto recibido, ingresado manualmente por el cajero. |
| `change` | number | Cambio calculado. |

## `order.payment.transfer_approved`

| Campo en `details` | Tipo | Descripción |
|---|---|---|
| `receipt_hash` | string | Mismo HMAC-SHA256 que en `order.payment_attempt.created` para ese comprobante — permite reconocer, sin revelarlo, que es el mismo comprobante (FR-012). |

## `order.payment.transfer_rejected`

| Campo en `details` | Tipo | Descripción |
|---|---|---|
| `receipt_hash` | string | Igual que en `transfer_approved`. |
| `rejection_reason` | string | Motivo del rechazo ingresado por el cajero (texto libre operativo, no dato personal del comensal — se envía tal cual). |

## `order.cancelled`

| Campo en `details` | Tipo | Descripción |
|---|---|---|
| `initiated_by` | `"comensal"` \| `"staff"` | Quién inició la cancelación (coincide con el atributo `actor_type`, se deja explícito igual por legibilidad al filtrar en Sentry). |
| `reason` | string \| `null` | Motivo de cancelación, si se registró. |
| `inventory_loss` | bool | `true` si algún ítem ya estaba `en_preparacion`/`listo` y no se pudo revertir al inventario (FR-009). |

## `order.payment.checkout_and_send`

Adenda post-implementación (FR-014, ver `spec.md` § Mapeo del flujo actual). Emitido cuando el staff cobra y envía a cocina, en un solo paso, una comanda `hold_for_payment` — la orden pasa de `recibida` a `pagada` directamente, sin intento de pago pendiente ni confirmación manual separada. Este `event_type` va acompañado, en el mismo momento, de un `order.confirmed` con `details.trigger = "automatic_payment"` (mismo patrón que el resto de pagos que confirman la orden como efecto colateral — ver § `order.confirmed`).

| Campo en `details` | Tipo | Descripción |
|---|---|---|
| `payment_method_types` | lista de `"cash"` \| `"card"` \| `"transfer"` \| `"other"` | Tipos de método de pago usados, sin duplicados. Puede tener más de un valor (pago dividido entre varios métodos en la misma operación). |
| `total_amount` | number | Suma de los montos de todos los pagos de esta operación. |
| `payment_count` | int | Cuántas líneas de pago distintas componen el cobro (1 si fue un solo método). |

## Compatibilidad hacia adelante

Añadir un campo nuevo dentro de `details` de un `event_type` existente es compatible (los consumidores deben ignorar campos desconocidos). Cambiar el significado de un campo existente, o añadir un `event_type` nuevo, requiere una actualización explícita de este contrato y del spec — no debe hacerse de forma implícita durante la implementación (así se hizo con `order.payment.checkout_and_send`: primero el spec, T040-T043 en `tasks.md`, y este contrato, antes de tocar código).
