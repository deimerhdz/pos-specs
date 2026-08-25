# Contrato: `POST /orders/{order_id}/checkout-and-send` (comportamiento ampliado)

Endpoint ya existente (spec 028). Esta funcionalidad **no** cambia su autenticación, método
HTTP, ruta, cuerpo de la petición, ni la forma de su respuesta (`Sale`) — solo el valor que
queda en `CustomerOrder.status` tras la operación, visible al consultar el pedido después
(`GET /orders`, `GET /orders/{id}`).

## Antes (comportamiento actual)

```json
// Respuesta de POST /orders/{id}/checkout-and-send: Sale, sin cambios.
// Consulta posterior GET /orders/{id}:
{
  "id": "…",
  "status": "abierta",
  "paid": true,
  "current_payment_attempt": null,
  "items": [ { "estado_cocina": "pendiente", "…": "…" } ]
}
```

El pedido ya tiene una venta (`paid: true`), pero su `status` crudo dice `"abierta"` — el
frontend solo se ve correcto porque la pantalla de Órdenes ya calcula el estado mostrado a
partir de `paid` (spec 029), no de `status`.

## Después (respuesta ampliada por esta funcionalidad)

```json
{
  "id": "…",
  "status": "pagada",
  "paid": true,
  "current_payment_attempt": null,
  "items": [ { "estado_cocina": "pendiente", "…": "…" } ]
}
```

`status` y `paid` ahora coinciden desde el mismo instante del cobro. Nótese que los ítems
pueden seguir `"pendiente"`/`"en_preparacion"` — el pedido ya está pagado, pero cocina puede
seguir trabajando en él (flujo "Cobrar y enviar", spec 028: se cobra antes de preparar).

## Efecto sobre otros endpoints (sin cambio de contrato, cambio de resultado)

- `PATCH /orders/tables/{table_id}/status` (liberar/reservar mesa), `POST /orders/{id}/move`
  (mover orden), `POST /orders/merge` (fusionar mesas): mismos parámetros y mismos códigos de
  respuesta (`200`/`409`) que hoy. Lo que cambia es **cuándo** cada uno responde `409`: una
  orden `'pagada'` por este camino con algún ítem todavía sin `estado_cocina = 'listo'` (o
  `'anulado'`) sigue devolviendo `409` en estas tres operaciones — antes de esta spec, una vez
  `'pagada'` estas operaciones ya no bloqueaban nunca, sin importar el estado de cocina; ahora
  sí lo hacen mientras quede trabajo pendiente. Ver `research.md`, Decisión 2, para el
  predicado exacto.
- `GET /orders/group/{group_id}/bill` (cuenta consolidada de mesas fusionadas): **sin
  cambios** — sigue excluyendo del total cualquier orden `'pagada'`/`'cancelada'`,
  independientemente del estado de sus ítems (decisión ya correcta de specs 017/019, no
  afectada por esta spec).

## Compatibilidad

- Cambio de comportamiento, no de forma — cualquier consumidor que ya lea `status` sigue
  recibiendo un `string` del mismo enum ya documentado; simplemente ahora puede llegar a
  `'pagada'` en un caso donde antes no llegaba.
- No requiere versión de API nueva ni cambio de ruta.
- Pedidos cobrados **antes** de este cambio conservan su `status` histórico (`'abierta'`) —
  esta spec no reescribe datos ya existentes (Edge Case de `spec.md`).
