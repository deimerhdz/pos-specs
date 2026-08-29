# Quickstart: validación del tipo de orden "Domicilio"

Prerrequisitos: migración aplicada (`alembic upgrade head` en `pos-backend`), `pos-backend`
corriendo local, `pos-heladeria` corriendo local (`ng serve`), sesión de un usuario staff válida,
al menos un producto/variante activa.

## Escenario 1 — Crear un pedido "Domicilio" desde la UI (US1)

1. Iniciar sesión como staff → "Terminal de Mesas" → "Crear Orden Manual" (o la ruta directa
   `/dashboard/mesas-sesiones/manual-order`).
2. En el panel derecho, pestaña "Tipo de Orden": click en "🛵 Domicilio".
   - **Esperado**: la pestaña queda seleccionada; el bloque "Mesas" (buscador de mesa) desaparece;
     no se exige elegir ninguna mesa; aparecen los campos "Cliente" (vacío), "Dirección" (vacío),
     "Teléfono" (vacío) y "Valor del domicilio" (vacío).
3. Agregar 1-2 productos desde el catálogo.
4. Click en "➤ Confirmar y Enviar" sin diligenciar ningún campo nuevo.
   - **Esperado**: el botón está deshabilitado (a diferencia de "Para Llevar", que no exige nada).
5. Escribir un nombre en "Cliente", una dirección en "Dirección", y un valor en "Valor del
   domicilio" (dejar "Teléfono" vacío).
   - **Esperado**: el botón "Confirmar y Enviar" se habilita; el total mostrado en pantalla
     (Subtotal + Domicilio) aumenta exactamente en el valor escrito.
6. Click en "➤ Confirmar y Enviar".
   - **Esperado**: la orden se crea con éxito y vuelve a la Terminal de Mesas.

**Verificación en base de datos** (referencia — no reemplaza la UI):
```sql
SELECT channel, order_type, dining_table_id, customer_name, delivery_address, delivery_phone,
       delivery_fee
FROM tenant.customer_orders
ORDER BY created_at DESC LIMIT 1;
-- Esperado: channel='POS', order_type='DELIVERY', dining_table_id IS NULL,
-- customer_name = <nombre escrito>, delivery_address = <dirección escrita>,
-- delivery_phone IS NULL, delivery_fee = <valor escrito>.
```

## Escenario 2 — Bloqueo por campos obligatorios faltantes (US2)

Repetir el paso 3 del Escenario 1, pero probar cada combinación por separado:

1. Solo "Cliente" y "Dirección" diligenciados, "Valor del domicilio" vacío → botón "Confirmar y
   Enviar" deshabilitado.
2. Solo "Cliente" y "Valor del domicilio" diligenciados, "Dirección" vacía → botón deshabilitado.
3. Solo "Dirección" y "Valor del domicilio" diligenciados, "Cliente" vacío → botón deshabilitado.
4. Los tres diligenciados, "Teléfono" vacío → botón habilitado (teléfono es opcional, FR-008).

Vía API, con un token de staff válido:

```bash
curl -s -X POST http://localhost:8000/orders \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{
    "channel": "POS", "order_type": "DELIVERY",
    "customer_name": "Ana Torres",
    "items": [{"product_variant_id": "<uuid variante activa>", "quantity": 1}]
  }'
```
**Esperado**: `422 Unprocessable Entity` (falta `delivery_address` y `delivery_fee`), ningún
registro creado.

## Escenario 3 — El valor del domicilio queda incluido en la factura final (US3)

1. Completar el Escenario 1 hasta crear una orden "Domicilio" con `delivery_fee = 6000` y productos
   por `subtotal = 25000`.
2. Cobrar y facturar esa orden (checkout normal, cualquier método de pago).
   - **Esperado**: el total cobrado/mostrado en el recibo es `31000` (subtotal + domicilio).

**Verificación en base de datos**:
```sql
SELECT s.subtotal, s.delivery_fee, s.total
FROM tenant.sales s
JOIN tenant.customer_orders o ON o.id = s.customer_order_id
WHERE o.order_type = 'DELIVERY'
ORDER BY s.created_at DESC LIMIT 1;
-- Esperado: total = subtotal + delivery_fee (más impuesto/propina si aplican, hoy siempre 0).
```

Repetir el cobro por el camino de aprobación de transferencia (`POST
/orders/payment-attempts/{attempt_id}/approve`, contracts/orders-checkout-total.md) — es el punto
de mayor riesgo identificado en research.md Decisión 5: verificar que el pago autogenerado cubre
`subtotal + delivery_fee` y no solo `subtotal`, o el endpoint rechazaría el pago con `422`.

## Escenario 4 — No hay efecto sobre pedidos que no son "Domicilio" (US3, no regresión)

1. Crear un pedido "En Mesa" o "Para Llevar" normal, cobrarlo.
   - **Esperado**: el total no cambia respecto al comportamiento anterior a esta mejora — sin fila
     "Domicilio" en el resumen de totales, `Sale.delivery_fee IS NULL` o `0`.
2. Consultar una venta ya emitida **antes** de esta mejora.
   - **Esperado**: su `total` no cambió (no se recalculó ninguna venta histórica, Principio VII).

## Tests automatizados (referencia, no reemplazan lo anterior)

- Backend: `pytest app/characterization_tests/test_orders_service.py
  app/characterization_tests/test_orders_checkout.py` — agregar casos nuevos para la validación de
  campos obligatorios (FR-007) y para `Sale.total`/`Sale.delivery_fee` en los 4 caminos de checkout
  (research.md Decisión 5, Decisión 11), sin dejar en rojo ningún test con prefijo `"CONGELA
  comportamiento actual:"` no autorizado por esta spec.
- Frontend: `ng test` sobre `manual-order-page.component.spec.ts` y `pos-terminal.store.spec.ts` —
  actualizar el caso que asume "Domicilio" deshabilitado (research.md Decisión 11) y agregar casos
  para la pestaña "Domicilio" (campos visibles/obligatorios, sin valor por defecto en "Cliente",
  payload con `order_type: 'DELIVERY'` y los tres campos nuevos, `totals()` con `deliveryFee`
  sumado).
