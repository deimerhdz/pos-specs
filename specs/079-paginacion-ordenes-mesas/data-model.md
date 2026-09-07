# Data Model: Paginación y filtros en Órdenes y Mesas

**Cambios de esquema: ninguno.** No hay entidades, campos, relaciones, valores por defecto ni migraciones nuevas (Principio VIII, FR-026). Este documento describe (a) las columnas ya existentes que la funcionalidad lee, (b) la forma de la respuesta paginada reutilizada, y (c) el mapeo de los filtros de la pantalla "Órdenes" a predicados de consulta.

---

## 1. Entidades existentes leídas (sin cambios)

### `customer_orders` (esquema por tenant) — `app/models/customer_order.py`

| Columna | Tipo | Uso en esta funcionalidad |
|---|---|---|
| `id` | `UUID` (`uuid4` aleatorio) | Desempate del orden (`ORDER BY ... , id DESC`) para paginación determinista (FR-003, SC-008). **No** es una secuencia incremental; sirve solo como criterio estable. |
| `status` | `String(12)` — `recibida` \| `abierta` \| `bloqueada` \| `pagada` \| `cancelada` | Filtro de estado (FR-008). Ver §3. |
| `order_type` | `String(10)` *nullable* — `DINE_IN` \| `TAKEAWAY` \| `DELIVERY` \| `NULL` | Filtro de tipo (FR-009) y columna visible por fila (FR-017). `NULL` = orden histórica sin clasificar → "Sin especificar" (FR-018). |
| `created_at` | `DateTime` (naive) `server_default=func.now()` | Orden principal, descendente (FR-003). Conjunto y orden idénticos a hoy. |

Índices ya presentes que la consulta aprovecha: `idx_customer_orders_order_type`, `idx_customer_orders_channel`. No se crean índices nuevos.

### Relación `customer_orders` → `sales` (existente) — `app/models/sale.py`

`sales.customer_order_id` (*nullable*, referencia blanda). La existencia de **al menos una** `Sale` para un pedido es la señal real de "Pagada" que ve la persona (spec 029/035). Se consulta con `EXISTS (SELECT 1 FROM sales WHERE customer_order_id = customer_orders.id)` — mismo patrón que `orders/service.py::order_has_sale` / `paid_order_ids`, sin tocar esas funciones.

### `dining_tables` (esquema por tenant) — `app/models/dining_table.py`

| Columna | Tipo | Uso |
|---|---|---|
| `number` | `Integer` `UNIQUE NOT NULL` | Único criterio de orden de "Mesas", ascendente (FR-020). `UNIQUE` ⇒ sin empates ⇒ no necesita desempate. |
| `id`, `name`, `qr_token`, `active`, `status` | — | Se devuelven en `TableResponse` tal cual hoy. |

---

## 2. Envoltura de paginación (reutilizada, sin cambios)

`app/core/pagination.py` — `Page[T]` (Pydantic genérico) y `paginate(db, stmt, page, size) -> dict`:

```
Page[T] = {
  items: list[T],   # la página (≤ size elementos)
  total: int,        # nº de resultados del conjunto YA filtrado (FR-007)
  page:  int,        # página efectivamente devuelta
  size:  int,        # tamaño de página aplicado
  pages: int,        # ceil(total / size), o 0 si total == 0
}
```

Frontend: `Page<T>` en `src/app/core/interfaces/page.interface.ts` (idéntico). Barra: `src/app/shared/pagination/pagination-bar.component.ts` con `DEFAULT_PAGE_SIZES = [10, 20, 50, 100]` (FR-001).

### Regla de página fuera de rango (FR-005) — se resuelve en la capa de servicio antes de `paginate()`

1. Construir `stmt` filtrado y contar `total`.
2. `pages = ceil(total / size)` (0 si `total == 0`).
3. `page_efectiva = 1` si `total == 0`; si no, `min(max(page_solicitada, 1), pages)` (clamp al máximo válido).
4. Llamar a `paginate()` con `page_efectiva`. La respuesta lleva `page = page_efectiva`, de modo que el frontend se recoloca sin error.

> El clamp lo aplica la funcionalidad nueva; `paginate()` no cambia. Cambiar de filtro o de tamaño de página **siempre** lleva a `page = 1` (FR-004), lo decide el frontend antes de llamar.

---

## 3. Mapeo del filtro de estado de "Órdenes" → predicado (FR-008, FR-016)

Parámetro `status` (query) — `Enum` cerrado de exactamente 6 valores. Semántica = la de `displayOrderStatus()` (`order-status.util.ts`), resuelta en servidor.

Sea `TIENE_VENTA` ≔ `EXISTS (SELECT 1 FROM sales s WHERE s.customer_order_id = customer_orders.id)`.

| `status` (valor de query) | Etiqueta | Predicado añadido al `WHERE` |
|---|---|---|
| *(ausente)* / `todos` | Todos | *(ninguno)* |
| `recibida` | Por confirmar | `status = 'recibida' AND NOT TIENE_VENTA` |
| `abierta` | Abierta | `status = 'abierta' AND NOT TIENE_VENTA` |
| `bloqueada` | Bloqueada | `status = 'bloqueada' AND NOT TIENE_VENTA` |
| `pagada` | Pagada | `status <> 'cancelada' AND (TIENE_VENTA OR status = 'pagada')` |
| `cancelada` | Cancelada | `status = 'cancelada'` |

Notas:
- Sin filtro de estado, las órdenes `bloqueada` **siguen apareciendo** (FR-015): "Todos" no aplica ningún predicado de estado.
- El caso límite del spec (pedido `status='abierta'` con venta emitida) cae bajo `pagada` y **no** bajo `abierta`.
- Simétricamente, una orden legada con `status='pagada'` crudo pero **sin** `Sale` también cae bajo `pagada`: la persona la ve como "Pagada" vía `displayOrderStatus()`, que devuelve `'pagada'` para ese `status` sin mirar `paid`. Por eso el predicado es `OR status = 'pagada'`, no solo `TIENE_VENTA` (FR-016). Si el negocio confirma que ese caso no existe en los datos del tenant —todos los caminos de `checkout.py` emiten la `Sale` en la misma transacción—, el `OR` es inocuo y solo blinda datos históricos.

## 4. Mapeo del filtro de tipo de orden → predicado (FR-009, FR-018)

Parámetro `order_type` (query) — `Enum` de 4 valores (`DINE_IN`, `TAKEAWAY`, `DELIVERY`) + ausente.

| `order_type` | Etiqueta | Predicado |
|---|---|---|
| *(ausente)* / `todos` | Todos | *(ninguno)* — incluye las de `order_type IS NULL` |
| `DINE_IN` | En mesa | `order_type = 'DINE_IN'` |
| `TAKEAWAY` | Para llevar | `order_type = 'TAKEAWAY'` |
| `DELIVERY` | Domicilio | `order_type = 'DELIVERY'` |

`order_type IS NULL` nunca satisface un `=` concreto (lógica ternaria de SQL) ⇒ las órdenes históricas sin tipo quedan fuera al filtrar por un tipo y dentro en "Todos" (FR-018), sin `COALESCE` ni centinela.

**Combinación** (FR-011): los predicados de estado y de tipo se unen con `AND`. El `total`/`pages` de la respuesta corresponden a la intersección.

---

## 5. Presentación del tipo de orden (frontend, FR-017/FR-018)

`order-status.util.ts` — mapa nuevo, análogo a `ORDER_STATUS`:

| `order.order_type` | Etiqueta visible |
|---|---|
| `'DINE_IN'` | En mesa |
| `'TAKEAWAY'` | Para llevar |
| `'DELIVERY'` | Domicilio |
| `null` / `undefined` / `''` | Sin especificar |

Se pinta en cada fila de la lista de "Órdenes"; no cambia `OrderResponse` ni `DiningOrder` (el campo `order_type?: string | null` ya existe en ambos).

---

## 6. Estado del frontend

### "Órdenes" — `orders-list.service.ts` (nuevo, patrón `SalesService`)

| Signal de entrada | Tipo | Default | Efecto al cambiar |
|---|---|---|---|
| `page` | `number` | `1` | recarga la query |
| `size` | `number` | `20` | `page → 1`, recarga |
| `status` | `'' \| 'recibida' \| 'abierta' \| 'bloqueada' \| 'pagada' \| 'cancelada'` | `''` | `page → 1`, recarga |
| `orderType` | `'' \| 'DINE_IN' \| 'TAKEAWAY' \| 'DELIVERY'` | `''` | `page → 1`, recarga |

Derivados (`computed`): `orders`, `total`, `totalPages`, `loading` (`isFetching`), `error`. `queryKey` = `['orders','page',{page,size,status,orderType}]`. `enabled` tras el primer `list()`.

### "Mesas" — carril añadido a `TableService`

| Signal | Tipo | Default |
|---|---|---|
| `tablesPage` | `number` | `1` |
| `tablesSize` | `number` | `20` |

Derivados: `pagedTables`, `tablesTotal`, `tablesTotalPages`, `tablesLoading`. Método `loadTablesPage(page, size)`. `queryKey` = `['tables','page',{page,size}]`. **Coexiste** con `loadTables()` / `tables()` / mutaciones, que no cambian. Tras una mutación disparada desde `tables-page.component.ts`, se refresca la página actual (o la última válida si la mutación la dejó fuera de rango).
