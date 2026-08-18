# Quickstart: validar Revisión y Pago Antes de Enviar el Pedido (Skeilopos)

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite firmas ni
columnas ya detalladas en [data-model.md](./data-model.md) y `contracts/` — solo enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` (Python 3.14), ejecutado desde la raíz de
`../pos-backend` (sibling de este repo `pos-specs`), sobre el código de spec 024 ya implementado.

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL ni Cloudflare R2 reales: `app/characterization_tests/cart_fixtures.py`
crea SQLite en memoria; el presign se firma localmente sin I/O de red (mismo mecanismo que ya
usan los tests de spec 024).

## Paso 0 — Confirmar la línea base antes de tocar código

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: todo en verde (231 tests de spec 024 y anteriores) — es la red de
no-regresión contra la que se compara después de cada historia.

## US1 — El comensal revisa su pedido y elige cómo pagar antes de enviarlo

1. Sembrar un comensal con carrito no vacío (`cart_fixtures.make_cart`/`make_cart_item`).
2. Verificar que **no existe ninguna `CustomerOrder`** para ese comensal todavía (nada que
   consultar — es la ausencia de fila, no un estado nuevo, research.md Decisión 3).
3. Confirmar que `service.list_payment_methods(db)` (spec 024, sin cambios) devuelve solo los
   métodos `active=true` — es lo que alimentaría la pantalla de revisión.

```bash
python -m unittest app.characterization_tests.test_cart_payment_attempts -v
```

## US2 — El comensal paga en efectivo y el pedido queda para el cajero

1. Con el carrito sembrado, llamar `service.submit_cart(db, participant, efectivo.id)` (sin
   `receipt_file_url`).
2. Verificar: se creó una `CustomerOrder` (`status='recibida'`) y su `current_payment_attempt`
   tiene `status='pendiente'`, `is_cash=True`, `receipt_file_url=None` (contracts/
   submit-cart-with-payment.md, camino feliz).
3. Verificar que `submit_cart(db, participant, efectivo.id, receipt_file_url="algo")` falla con
   `422` (efectivo no admite comprobante).
4. Sobre la orden creada en el paso 1, confirmar que `checkout.confirm_cash_payment_attempt`
   (spec 024, sin cambios) sigue funcionando exactamente igual.

```bash
python -m unittest app.characterization_tests.test_cart_payment_attempts -v
```

## US3 — El comensal paga por transferencia y el pedido solo se registra tras cargar el comprobante

1. Con el carrito sembrado, llamar `service.presign_payment_receipt(db, "tenant_test",
   participant.id, "image/jpeg")` — verificar que no exige ningún `attempt_id` y devuelve
   `upload_url`/`public_url` (contracts/payment-receipt-presign.md).
2. Verificar que, en este punto, **sigue sin existir ninguna `CustomerOrder`**.
3. Llamar `service.submit_cart(db, participant, nequi.id, receipt_file_url=public_url)` —
   verificar que ahora sí se crea la orden, con `current_payment_attempt.receipt_file_url ==
   public_url`.
4. Verificar que `submit_cart(db, participant, nequi.id)` (sin `receipt_file_url`) falla con
   `422` (transferencia exige comprobante, FR-006).
5. Sobre la orden creada en el paso 3, confirmar que `checkout.approve_payment_attempt`/
   `reject_payment_attempt` (spec 024, sin cambios) siguen funcionando igual, incluyendo el
   reintento tras rechazo (`POST /cart/orders/{id}/payment-attempts`, sin cambios en esta spec).

```bash
python -m unittest app.characterization_tests.test_cart_payment_attempts -v
python -m unittest app.characterization_tests.test_orders_payment_gate -v
```

## US4 — El comensal cambia de método de transferencia antes de subir el comprobante

1. Llamar `presign_payment_receipt` dos veces con contenidos distintos (simulando que el
   comensal probó un método, se arrepintió, y probó otro) — verificar que ambas llamadas
   funcionan sin ningún error de "ya existe algo" (no hay ningún recurso previo que gestionar,
   research.md Decisión 2).
2. Llamar `submit_cart` una sola vez, con el segundo `public_url` — verificar que solo existe una
   orden, con el método finalmente elegido, y que el primer `public_url` (nunca usado) no aparece
   en ningún lado de la base de datos.

## Edge case — Confirmación duplicada (FR-013, SC-006)

1. Sembrar un comensal con carrito no vacío.
2. Llamar `submit_cart(db, participant, metodo.id)` dos veces seguidas con la misma sesión de
   base de datos, simulando un doble toque — verificar que la segunda llamada falla con `409` y
   que solo existe **una** `CustomerOrder` para ese comensal al final (data-model.md,
   `idx_active_order_per_participant`).

```bash
python -m unittest app.characterization_tests.test_cart_single_active_order -v
```

## Edge case — Reintento tras fallo de creación sin volver a subir el archivo (FR-012)

1. Llamar `presign_payment_receipt` + simular la subida (sin R2 real, basta con el `public_url`
   devuelto).
2. Forzar un fallo en `submit_cart` (p. ej. mockeando `db.commit` para que lance una excepción la
   primera vez) — verificar que no queda ninguna `CustomerOrder` ni `OrderPaymentAttempt` creados.
3. Llamar `submit_cart` de nuevo con el **mismo** `public_url` (sin volver a llamar al presign) —
   verificar que esta vez sí se crea la orden, con ese mismo `receipt_file_url`.

## Verificación de no regresión — spec 024 completa

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: toda la suite en verde. Los tests de spec 024 que llamaban
`submit_cart(db, participant)` (sin argumentos de pago) y luego `create_payment_attempt` como
llamada separada se actualizan para usar la nueva firma en una sola llamada — citando esta spec,
con evidencia de que ningún otro comportamiento protegido se ve afectado (Constitución, Principio
III). Los endpoints de aprobar/rechazar/confirmar-efectivo/`confirm_order` (spec 024) no deberían
requerir ningún cambio: reciben una orden con su intento ya creado, exactamente como antes.

## Verificación final — SC-001 a SC-006

| Criterio | Cómo se verifica |
|---|---|
| SC-001 (100% de pedidos llegan con método+intento) | Todo `submit_cart` exitoso crea ambos juntos — no hay camino que cree uno sin el otro |
| SC-002 (0% de pedidos fantasma al abandonar) | US1 paso 2, US3 paso 2 — ausencia de fila mientras no se completa el pago |
| SC-003 (≤2 toques para pagar en efectivo) | Validación de UI (pos-heladeria), no automatizable desde `unittest` — verificar manualmente en `ng serve` |
| SC-004 (ningún producto se pierde en transferencia) | US3 — los `OrderItem` creados en `submit_cart` son un snapshot exacto del carrito, igual que en spec 024 |
| SC-005 (cambiar de método no deja rastro) | US4 |
| SC-006 (0% de confirmaciones duplicadas crean 2 pedidos) | Edge case "Confirmación duplicada" |

## Frontend — validación manual (no automatizada en esta guía)

Igual que spec 024: requiere `ng serve` + backend real con Postgres/Redis/R2, no disponible en un
entorno de test automatizado. La pantalla de revisión+pago reutiliza casi todo el modal de
selección de método y carga de comprobante que spec 024 ya construyó en
`public-menu.component.ts` — la validación manual debe confirmar, sobre todo, que ese modal ahora
se abre **antes** de que el pedido aparezca en el panel del staff, no después.
