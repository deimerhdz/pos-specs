# Quickstart: validar Pagos de Órdenes en Mesa (Skeilopos)

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite firmas ni
columnas ya detalladas en [data-model.md](./data-model.md) y `contracts/` — solo enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` (Python 3.14), ejecutado desde la raíz de
`../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL ni Cloudflare R2 reales para los tests de backend:
`app/characterization_tests/fixtures.py` crea SQLite en memoria; `generate_presigned_put_url`/
`build_object_key` se mockean con `unittest.mock.patch`, mismo patrón que usa
`test_products_service.py` (spec 021) para mockear `delete_object`.

## Paso 0 — Confirmar la línea base antes de tocar código

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: todo en verde — es la red de no-regresión completa (specs 014-023) contra
la que se compara después de cada historia. Ningún test de este comando se modifica como parte de
esta spec salvo que, durante la implementación, algún `"CONGELA comportamiento actual:"` de
`orders`/`cart` resulte incompatible con FR-005/FR-017 — en ese caso, la actualización cita esta
spec en el mismo commit (Constitución, Principio III).

## US1 — El tenant configura sus métodos de pago

Fichero nuevo: `app/characterization_tests/test_sales_payment_methods.py`.

1. Crear un tenant con solo "Efectivo" activo (fixture existente de `payment.py`/`sales`).
2. `POST /sales/payment-methods` con `type="transfer"`, `payment_info={"cuenta": "..."}` → verificar
   que aparece en `GET /sales/payment-methods` con `payment_info` intacto (Acceptance Scenario 1).
3. `PATCH .../{id}` con `active=false` sobre un método cuando es el único activo restante → `409`
   (Acceptance Scenario 3, contrato en
   [contracts/tenant-payment-methods.md](./contracts/tenant-payment-methods.md)).
4. Con dos métodos activos, desactivar uno → `200`, y verificar que sigue existiendo (no se borra) y
   que un `OrderPaymentAttempt` ya creado con ese método no cambia (Acceptance Scenario 2 y 4).

```bash
python -m unittest app.characterization_tests.test_sales_payment_methods -v
```

**Resultado esperado**: verde solo después de implementar `PATCH /sales/payment-methods/{id}` y la
columna `payment_info` (data-model.md).

## US2 — Transferencia con comprobante revisado por el cajero

Fichero nuevo: `app/characterization_tests/test_cart_payment_attempts.py` (lado comensal) +
extensión de `test_orders_payment_gate.py` (lado cajero).

1. Comensal abre sesión (fixture de `cart_fixtures.py`), agrega ítems, `POST /cart/submit` → orden
   `recibida`.
2. `GET /cart/payment-methods` → solo métodos `active=true` (Acceptance Scenario 1).
3. `POST /cart/orders/{id}/payment-attempts` con el método de transferencia → intento `pendiente`.
4. Presign (mockeado) + `POST .../receipt` con un `file_url` → intento sigue `pendiente`, ahora con
   `receipt_file_url` (Acceptance Scenario 3).
5. Cajero: `GET /orders/{order_id}/payment-attempts` → ve el intento con el archivo.
6. `POST /orders/payment-attempts/{id}/reject` sin `reason` → `422` (Acceptance Scenario 5).
7. `POST .../reject` con `reason="el monto no coincide"` → `200`, intento `rechazado`.
8. Comensal: `GET /cart/orders/{order_id}` → `current_payment_attempt.status == "rechazado"`, **sin**
   campo de motivo en la respuesta (Acceptance Scenario 6, Clarification 3).
9. Repetir 3-4 con un intento nuevo → `POST .../approve` → `200`, `confirmado`.
10. Comensal, antes del paso 9: `GET /cart/orders/{order_id}` sigue mostrando pendiente de pago
    (Acceptance Scenario 7) — verificar en el paso intermedio, no solo al final.

```bash
python -m unittest app.characterization_tests.test_cart_payment_attempts -v
python -m unittest app.characterization_tests.test_orders_payment_gate -v
```

## US3 — Efectivo con cálculo de cambio

Extiende `test_orders_payment_gate.py`.

1. Orden de $18.000, intento de pago con método `is_cash=true` → `pendiente`.
2. `POST .../confirm-cash` con `amount_received=15000` → `422` (Acceptance Scenario 4, FR-010a).
3. `POST .../confirm-cash` con `amount_received=20000` → `200`, `change_amount=2000` (Acceptance
   Scenario 1).
4. Mismo flujo con `amount_received=18000` exacto → `change_amount=0` (Acceptance Scenario 2).
5. Antes de confirmar: `GET /cart/orders/{order_id}` desde el comensal sigue "pendiente de pago"
   (Acceptance Scenario 3).

```bash
python -m unittest app.characterization_tests.test_orders_payment_gate -v
```

## US4 — Gate de comanda

Fichero nuevo: `app/characterization_tests/test_orders_payment_gate.py` (compartido con US2/US3
arriba — es el mismo fichero, agrupa todos los casos de `confirm_order`).

1. Orden con único intento `pendiente` → `POST /orders/{id}/confirm` → `409` (paso 3a del contrato,
   [contracts/order-confirm-gate.md](./contracts/order-confirm-gate.md)).
2. Orden con único intento `rechazado` → `409`.
3. Orden con intento `confirmado` → `200`, `status` pasa a `abierta`, inventario descontado (mismo
   comportamiento que hoy, solo que ahora exige el paso 3a primero).
4. Llamar `confirm_order` dos veces seguidas sobre la misma orden → la segunda `409` (paso 3, sin
   cambio de comportamiento respecto a hoy, solo confirmando que sigue funcionando con el gate
   nuevo).
5. Sobre el mismo `attempt_id` ya `confirmado`, llamar `approve`/`confirm-cash` de nuevo → `409` (no
   `pendiente`) — verifica FR-018/SC-007 a nivel de intento, no solo de orden.

```bash
python -m unittest app.characterization_tests.test_orders_payment_gate -v
```

## US5 — Reintento tras rechazo

Fichero nuevo: `app/characterization_tests/test_cart_payment_attempts.py` (mismo fichero de US2,
casos adicionales).

1. Repetir pasos 1-7 de US2 (llega a un intento `rechazado`).
2. `POST /cart/orders/{id}/payment-attempts` de nuevo (mismo o distinto método) → `201`, un intento
   `pendiente` **nuevo**, el `rechazado` sigue existiendo (Acceptance Scenario 2).
3. Verificar que `order.items` no cambió entre el paso 1 y el 2 (Acceptance Scenario 1 — el
   reintento no toca `OrderItem`, ver data-model.md tabla de reglas).
4. `GET /orders/{order_id}/payment-attempts` (cajero) → lista con 2 filas, la vieja `rechazado` con
   su motivo, la nueva `pendiente` (Acceptance Scenario 3).
5. Antes de que el rechazo se resuelva, intentar `POST .../payment-attempts` una tercera vez sobre la
   orden mientras la segunda sigue `pendiente` → `409` (FR-015a, ya cubierto en US2 pero repetido
   aquí en contexto de reintento).

```bash
python -m unittest app.characterization_tests.test_cart_payment_attempts -v
```

## US6 — Una orden activa por comensal

Fichero nuevo: `app/characterization_tests/test_cart_single_active_order.py`.

1. Comensal con una orden `recibida` (o `abierta`/`bloqueada`) → `POST /cart/submit` de un carrito
   nuevo → `409` (Acceptance Scenario 1, FR-005).
2. Marcar esa orden `pagada` (o `cancelada`) directamente en la fixture → `POST /cart/submit` de un
   carrito nuevo → `201` (Acceptance Scenario 2, FR-006).

```bash
python -m unittest app.characterization_tests.test_cart_single_active_order -v
```

## Verificación de no regresión — specs 007/008/015/016/017

```bash
python -m unittest app.characterization_tests.test_cart_service -v
python -m unittest app.characterization_tests.test_orders_confirm -v   # nombre exacto según lo ya existente
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: los characterization tests preexistentes de `cart`/`orders` (specs
015/016/017) siguen en verde sin modificación — `submit_cart` y `confirm_order` ganan una
precondición nueva, pero ningún caso ya cubierto por esos tests deja de comportarse como antes
(ninguno de ellos ejercita "dos órdenes activas" ni "confirmar sin intento de pago", porque esos
conceptos no existían antes de esta spec).

## Verificación final — SC-001 a SC-007

| Criterio | Cómo se verifica |
|---|---|
| SC-001 (100% de órdenes en comanda con pago confirmado) | US4, pasos 1-3 — ninguna orden `pendiente`/`rechazada` pasa el gate |
| SC-002 (alta de método en <2 min) | US1, paso 2 — sin despliegue, un `POST`/`PATCH` |
| SC-003 (comensal nunca ve método desactivado) | US1 paso 3 + US2 paso 2 juntos: desactivar y re-consultar `GET /cart/payment-methods` |
| SC-004 (100% de rechazos con motivo) | `CHECK` de BD (data-model.md) + US2 paso 6 (`422` sin motivo) |
| SC-005 (cambio exacto) | US3 pasos 3-4 |
| SC-006 (reintento conserva ítems) | US5 paso 3 |
| SC-007 (una sola decisión válida) | US4 pasos 4-5 |

## Frontend — validación manual (no automatizada en esta guía)

`pos-heladeria` usa Vitest para specs unitarios de componentes/servicios nuevos
(`payment-attempt-review-panel.component.spec.ts`, extensión de
`dining-cart.service.spec.ts`/`payment-method.service.spec.ts`), pero la validación end-to-end del
flujo visual (comensal escaneando QR → pantalla de pago → cajero aprobando) requiere `ng serve` +
navegación manual, siguiendo la guía general del proyecto — no se detalla aquí un script E2E nuevo
(Playwright está en `devDependencies` pero no hay suite E2E activa hoy, confirmado en
research/exploración de esta spec).
