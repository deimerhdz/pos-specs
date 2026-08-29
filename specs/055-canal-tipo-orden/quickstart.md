# Quickstart: validación de canal/tipo de orden estandarizados + "Para Llevar"

Prerrequisitos: migración aplicada (`alembic upgrade head` en `pos-backend`, contra un tenant con
al menos una mesa activa), `pos-backend` corriendo local, `pos-heladeria` corriendo local
(`ng serve`), sesión de un usuario staff válida.

## Escenario 1 — Crear un pedido "Para Llevar" desde la UI (US1)

1. Iniciar sesión como staff y entrar a "Terminal de Mesas" → cualquier mesa libre →
   "Crear Orden Manual" (o la ruta directa `/dashboard/mesas-sesiones/manual-order`).
2. En el panel derecho, pestaña "Tipo de Orden": click en "🛍️ Para Llevar".
   - **Esperado**: la pestaña queda seleccionada (mismo estilo visual que "En Mesa" al elegirla);
     el bloque "Mesas" (buscador de mesa) desaparece; no se exige elegir ninguna mesa.
3. Observar el campo "Cliente".
   - **Esperado**: ya muestra "Consumidor final", de solo lectura, con el botón ✏️ visible.
4. Agregar 1-2 productos desde el catálogo.
5. Click en "➤ Confirmar y Enviar" sin editar el campo "Cliente".
   - **Esperado**: el botón no está deshabilitado (a diferencia de "En Mesa" sin mesa elegida); la
     orden se crea con éxito y vuelve a la Terminal de Mesas.
6. Repetir el flujo editando el campo "Cliente" (click en ✏️, escribir un nombre, click fuera) antes
   de confirmar.
   - **Esperado**: la orden se crea con ese nombre como cliente.

**Verificación en base de datos** (referencia — no reemplaza la UI):
```sql
SELECT channel, order_type, dining_table_id, customer_name
FROM tenant.customer_orders
ORDER BY created_at DESC LIMIT 1;
-- Esperado: channel='POS', order_type='TAKEAWAY', dining_table_id IS NULL,
-- customer_name = 'Consumidor final' (o el nombre editado).
```

## Escenario 2 — Rechazo de combinación inválida canal/tipo de orden (US2)

Con un token de staff válido:

```bash
curl -s -X POST http://localhost:8000/orders \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{
    "channel": "WHATSAPP",
    "order_type": "DINE_IN",
    "items": [{"product_variant_id": "<uuid variante activa>", "quantity": 1}]
  }'
```
**Esperado**: `400 Bad Request`, ningún registro creado (`SELECT count(*) FROM
tenant.customer_orders` no cambia).

Repetir con `"channel": "POS", "order_type": "TAKEAWAY"` (mismos items) → **Esperado**: `201
Created`.

## Escenario 3 — Rechazo de mesa en un pedido "Para Llevar" vía API (Decisión 5, research.md)

```bash
curl -s -X POST http://localhost:8000/orders \
  -H "Authorization: Bearer <token>" -H "Content-Type: application/json" \
  -d '{
    "channel": "POS", "order_type": "TAKEAWAY", "dining_table_id": "<uuid mesa>",
    "items": [{"product_variant_id": "<uuid variante activa>", "quantity": 1}]
  }'
```
**Esperado**: `422 Unprocessable Entity`.

## Escenario 4 — Histórico reclasificado (US3)

Contra una base con pedidos anteriores a esta mejora ya migrados:

```sql
-- Ningún valor libre remanente.
SELECT DISTINCT channel FROM tenant.customer_orders;
-- Esperado: solo 'POS', 'QR_MENU' (y eventualmente 'WHATSAPP'/'API' si ya hay pedidos nuevos).

-- Pedidos históricos con mesa quedaron DINE_IN; sin mesa, sin tipo.
SELECT
  count(*) FILTER (WHERE dining_table_id IS NOT NULL AND order_type = 'DINE_IN') AS con_mesa_ok,
  count(*) FILTER (WHERE dining_table_id IS NULL AND order_type IS NULL) AS sin_mesa_sin_tipo,
  count(*) FILTER (WHERE dining_table_id IS NOT NULL AND order_type IS DISTINCT FROM 'DINE_IN') AS deberia_ser_cero
FROM tenant.customer_orders
WHERE created_at < '<fecha de despliegue de esta mejora>';
```

## Escenario 5 — El mesero (consolidación) sigue sin verse afectado (no-regresión, research.md Decisión 2)

1. Abrir una mesa desde el flujo QR (comensal envía un pedido) o desde Terminal de Mesas.
2. Como staff, agregar un ítem directo a esa mesa (botón "+ agregar ítem" de la Terminal de Mesas,
   modo híbrido) dos veces seguidas.
   - **Esperado**: ambos ítems quedan en la **misma** orden (no se crean dos órdenes separadas) —
     mismo comportamiento de siempre.
3. Cobrar y enviar una orden distinta creada vía "Crear Orden Manual" (pestaña "En Mesa") sobre la
   misma mesa, y luego volver a agregar un ítem directo a esa mesa.
   - **Esperado**: el nuevo ítem **no** se agrega a la orden ya cobrada — abre/reusa una orden de
     mesero aparte, exactamente igual que antes de esta mejora (research.md Decisión 2 — el bug de
     colisión que esta spec previene).

## Tests automatizados (referencia, no reemplazan lo anterior)

- Backend: `pytest app/characterization_tests/test_orders_service.py
  app/characterization_tests/test_orders_consolidation.py
  app/characterization_tests/test_cart_single_active_order.py` — deben seguir en verde tras
  actualizar los literales `"waiter"`/`"counter"` (research.md Decisión 2); agregar casos nuevos
  para la validación de combinaciones (FR-006/FR-007) y el rechazo de mesa+TAKEAWAY (Decisión 5).
- Frontend: `ng test` sobre `manual-order-page.component.spec.ts` y `pos-terminal.store.spec.ts` —
  agregar casos para la pestaña "Para Llevar" (sin mesa exigida, payload con `order_type:
  'TAKEAWAY'`, `dining_table_id: null`, `channel: 'POS'`).
