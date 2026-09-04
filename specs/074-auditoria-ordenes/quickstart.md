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
