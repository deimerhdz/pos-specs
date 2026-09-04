# Quickstart: validar la auditoría del ciclo de vida de una orden

Dos formas de validar este feature: automatizada (sin depender de Sentry real) y manual end-to-end (con un proyecto Sentry de prueba). Usa la primera para verificación rutinaria; la segunda solo para confirmar una vez que el envío real funciona.

## Prerrequisitos

- Repo `pos-backend` con este feature implementado (`app/core/order_audit.py` y los puntos de integración de `research.md` § 4).
- Variable de entorno `AUDIT_HASH_SECRET` definida (ver `research.md` § 3) — sin ella, el hash de datos sensibles no debe poder calcularse; los tests unitarios deben fallar de forma explícita si falta, no usar un valor por defecto silencioso.
- Para la validación manual end-to-end: un DSN de un proyecto Sentry de prueba (no el de producción) y `ENVIRONMENT=prod` local (ver `research.md` § 2 — fuera de `prod` el cliente Sentry nunca se inicializa).

## 1. Validación automatizada (recomendada, sin red)

```bash
cd pos-backend
python -m unittest app.characterization_tests.test_order_audit_log -v
```

Qué debe confirmar esta corrida (ver `research.md` § 6 y `data-model.md` § Reglas de validación):
- El helper `record_order_audit_event` construye el payload correcto para cada uno de los 8 `event_type` (contracts/order-audit-log-event.md).
- `order_id`, `tenant_id` y `actor` (con `type` ∈ {comensal, cajero, sistema}) están presentes en el 100% de los payloads generados en la prueba.
- El nombre del comensal y el comprobante de pago nunca aparecen en el payload en texto plano — solo su `*_hash`.
- El mismo valor de entrada produce siempre el mismo hash (determinismo de FR-012).
- Si la transición de negocio simulada falla (p. ej. `confirm_cash_payment_attempt` con monto insuficiente), el helper **no** se invoca.
- Si se fuerza una excepción dentro del envío a Sentry (mockeado), la función de servicio que la rodea completa su transacción igual y no la propaga (FR-011).

También corre la suite completa de characterization tests para confirmar que no se rompió ningún comportamiento existente:

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py'
```

## 2. Validación manual end-to-end (con Sentry real de prueba)

1. Configura localmente `ENVIRONMENT=prod` y `SENTRY_DSN=<dsn del proyecto de prueba>` (nunca el DSN de producción del negocio).
2. Levanta `pos-backend` localmente y ejecuta, contra un tenant de prueba, la secuencia:
   a. Escanea/abre una sesión de mesa y envía un pedido (`POST /cart/submit`) con un comensal de nombre real, p. ej. "María López".
   b. Confirma el pago en efectivo desde la Terminal (`POST /orders/payment-attempts/{id}/confirm-cash`).
   c. Cancela una segunda orden de prueba, una vez creada y confirmada, desde la Terminal (`POST /orders/{id}/cancel`).
3. En el panel de Sentry (Logs), filtra por el `order_id` de cada orden usada en el paso 2.
4. Verifica:
   - Aparecen los eventos esperados por orden (`order.created`, `order.confirmed`, `order.payment.cash_confirmed` para la primera; `order.created`, `order.confirmed`, `order.cancelled` para la segunda), en orden cronológico.
   - El nombre "María López" no aparece en ningún atributo del log — solo un valor tipo hash en el atributo plano `diner_name_hash`.
   - El actor de cada evento corresponde a lo esperado (comensal para `order.created`, cajero con su `user_id`/rol para la confirmación de pago y la cancelación, o `sistema` si la confirmación fue automática).
   - Los eventos de auditoría son reconocibles como categoría propia dentro de Sentry (no se confunden con los logs de error operativos existentes).

## Notas

- No hay endpoint propio de este sistema para "ver el historial de una orden" (FR-008) — la consulta siempre es en el panel externo de Sentry, filtrando por `order_id`.
- Los eventos dejan de ser recuperables una vez pasada la ventana de retención del plan de Sentry contratado (7 a 30 días, SC-004) — no hay respaldo ni exportación adicional en este feature.
- Referencias completas de payload: `contracts/order-audit-log-event.md`. Reglas de validación completas: `data-model.md`.

## 3. Validación automatizada de la extensión (logging operativo, FR-015–FR-021)

```bash
cd pos-backend
python -m unittest app.characterization_tests.test_operational_log -v
```

Qué debe confirmar esta corrida (ver `research.md` § 7-11 y `data-model.md` § Reglas de validación — extensión):

- Una petición `POST`/`PUT`/`PATCH`/`DELETE` a cualquier ruta fuera de `/api/v1/super-admin` genera una entrada con `method`/`route`/`status`/`duration_ms`/`request_id` (`contracts/operational-log-entry.md`); `route` es el patrón registrado (p. ej. `/orders/{order_id}/cancel`), no la URL con el UUID real.
- Una petición `GET`/`HEAD`/`OPTIONS` no genera ninguna entrada.
- Una petición a `/api/v1/super-admin` no genera ninguna entrada de esta extensión (sigue con su mecanismo propio, sin cambios).
- El nivel de severidad usado al llamar a `sentry_sdk.logger.*` corresponde al `status`: `info` (`<400`), `warning` (`400-499`), `error` (`>=500`).
- Ningún atributo de la entrada contiene el cuerpo de la petición ni de la respuesta.
- Las 3 dependencias modificadas (`get_tenant`, `get_current_user`, `get_session_context`) siguen devolviendo exactamente lo mismo que antes a quien las invoca — el side-effect en `request.state` no cambia su valor de retorno.
- Una de las 8 rutas ya auditadas (p. ej. `POST /orders/{order_id}/cancel`) genera **ambas** entidades: el evento de auditoría de orden (con su `request_id` incluido, FR-021) y la entrada de log operativo — ninguna reemplaza a la otra.
- Si se fuerza una excepción dentro del middleware nuevo (mockeado), la petición real completa su respuesta igual, sin propagar el error.

También corre la suite completa (incluye lo anterior, más `test_order_audit_log.py` y todos los characterization tests preexistentes — este cambio se monta sobre casi todas las rutas del backend, así que una regresión aquí tendría el radio de impacto más amplio de todo el spec):

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py'
```

## 4. Validación manual end-to-end de la extensión

1. Con `ENVIRONMENT=prod` y `SENTRY_DSN` de prueba (igual que en el paso 2), ejecuta cualquier acción mutativa fuera de las 8 ya auditadas — p. ej. `PATCH` sobre un método de pago desde la Terminal, o crear una categoría.
2. En el panel de Sentry (Logs), filtra por el `request_id` de esa petición (visible en la cabecera de respuesta o en los logs locales de la petición).
3. Verifica que aparece una entrada con método/ruta/status/duración/`request_id`, sin ningún campo con el cuerpo enviado.
4. Repite el flujo del paso 2 de la sección anterior (crear + confirmar + pagar una orden) y confirma que, para cada una de esas peticiones, aparecen **ambas** entidades — el evento `order.*` y la entrada operativa genérica — con el mismo `request_id`.
5. Ejecuta una petición `GET` cualquiera (p. ej. listar categorías) y confirma que no aparece ninguna entrada operativa para ella.
