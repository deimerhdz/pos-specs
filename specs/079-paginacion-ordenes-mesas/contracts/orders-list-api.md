# Contrato: `GET /api/v1/orders` — listado de comandas (paginación + filtros opt-in)

**Router**: `app/api/v1/orders/router.py::list_orders`
**Cambio**: aditivo y compatible. Sin los parámetros nuevos, la respuesta y el comportamiento son **idénticos a hoy** (FR-023, "Decisiones de compatibilidad").

---

## Parámetros de query

| Parámetro | Tipo | Default | Notas |
|---|---|---|---|
| `status` | string enum | *(ausente)* | **YA EXISTÍA** como alias `status`. Valores permitidos ahora: `recibida`, `abierta`, `bloqueada`, `pagada`, `cancelada`. Ausente = "Todos". Otro valor → `422`. Semántica = "estado que ve la persona" (ver más abajo). |
| `order_type` | string enum | *(ausente)* | **NUEVO** (FR-009). `DINE_IN` \| `TAKEAWAY` \| `DELIVERY`. Ausente = "Todos". Otro valor → `422`. |
| `page` | int ≥ 1 | *(ausente)* | **NUEVO** (FR-001). Presencia de `page` **o** `size` activa el modo paginado. |
| `size` | int, 1–100 | *(ausente)* | **NUEVO**. Valores esperados de UI: 10, 20, 50, 100 (`DEFAULT_PAGE_SIZES`); el backend acepta cualquier 1–100. Default de presentación cuando solo llega `page`: `20`. |
| `active_sessions_only` | bool | `false` | **YA EXISTÍA** (spec 029). Solo lo usa la Terminal de Mesas. Se **ignora** en el modo paginado (§Comportamiento). |

---

## Comportamiento

### Modo compatible — sin `page` ni `size`

Idéntico a hoy:
- `service.list_orders(db, status_filter, active_sessions_only)` con `ORDER BY created_at DESC`.
- El router asigna `paid` y `staff_user_name` en bloque.
- Respuesta vía `json_or_304(...)`: `200` con cuerpo `list[OrderResponse]` + `ETag` + `Cache-Control: no-cache`, o `304` si `If-None-Match` coincide.
- `status` mantiene su comportamiento previo para cualquier consumidor que ya lo pasara (filtra por `customer_orders.status` crudo). *(La ampliación de semántica de `status` solo aplica en modo paginado; ver nota de compatibilidad al final.)*

### Modo paginado — con `page` y/o `size`

- `response_model`: `Page[OrderResponse]`.
- Consulta: `service.list_orders_query(status_mostrado=status, order_type=order_type)` →
  `SELECT ... FROM customer_orders WHERE <predicado estado> AND <predicado tipo> ORDER BY created_at DESC, id DESC`.
- `total` y `pages` = del conjunto **ya filtrado** (FR-007).
- **Clamp** (FR-005): si `page` > `pages`, se devuelve la última página con resultados; si `total == 0`, `page = 1` con `items = []`. Nunca `404`/error por página fuera de rango.
- `paid` y `staff_user_name` se asignan en bloque **solo sobre los `items` de la página**.
- **Sin `ETag`** (la pantalla no se sondea).
- `active_sessions_only` se ignora (ningún consumidor legítimo lo combina con paginación).

Respuesta ejemplo (`page=2&size=20`, 130 órdenes):

```json
{
  "items": [ { "...": "OrderResponse" } ],
  "total": 130,
  "page": 2,
  "size": 20,
  "pages": 7
}
```

---

## Semántica del filtro `status` (modo paginado) — FR-016

`TIENE_VENTA` ≔ `EXISTS (SELECT 1 FROM sales s WHERE s.customer_order_id = customer_orders.id)`

| `status` | Predicado |
|---|---|
| *(ausente)* | — (incluye `bloqueada`, FR-015) |
| `recibida` | `status='recibida' AND NOT TIENE_VENTA` |
| `abierta` | `status='abierta' AND NOT TIENE_VENTA` |
| `bloqueada` | `status='bloqueada' AND NOT TIENE_VENTA` |
| `pagada` | `status<>'cancelada' AND (TIENE_VENTA OR status='pagada')` |
| `cancelada` | `status='cancelada'` |

## Semántica del filtro `order_type` — FR-018

`order_type IS NULL` (órdenes históricas) → excluidas por cualquier valor concreto, incluidas cuando el parámetro está ausente.

Combinación `status` + `order_type` = `AND` (FR-011).

---

## `OrderResponse` (sin cambios)

`app/api/v1/orders/schemas.py::OrderResponse` ya incluye `order_type: str | None` y `paid: bool`. No se añade ni se quita ningún campo.

---

## Consumidores y no-regresión

| Consumidor (`pos-heladeria`) | Llamada | Modo | Efecto de este cambio |
|---|---|---|---|
| Terminal de Mesas — `pos-terminal.store.ts` | `listOrders(undefined, true)` | compatible | ninguno |
| Dashboard — `admin-dashboard.component.ts` | `listOrders()` | compatible | ninguno |
| Pantalla "Órdenes" — `orders-page.component.ts` | `orders-list.service.ts` → `?page=&size=&status=&order_type=` | paginado | **es el objetivo** |
| "Pagos por confirmar" y demás vistas de cobro | *no consumen `GET /orders`* | — | ninguno (FR-024) |

---

## Nota de compatibilidad sobre `status`

Hoy ningún consumidor externo pasa `status` a `GET /orders` (el frontend filtra en cliente; la Terminal solo pasa `active_sessions_only`). La ampliación de semántica (de `status` crudo a "estado que ve la persona": `pagada` = tiene `Sale` **o** `status='pagada'` y no cancelada; los no terminales excluyen los que ya tienen `Sale`) se activa **solo en modo paginado**; el modo compatible conserva el filtro por `status` crudo por si algún cliente futuro lo usa sin paginar. Los characterization tests cubren ambos caminos.
