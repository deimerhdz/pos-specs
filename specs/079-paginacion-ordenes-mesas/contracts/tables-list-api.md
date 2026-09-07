# Contrato: `GET /api/v1/orders/tables` — listado de mesas (paginación opt-in)

**Router**: `app/api/v1/orders/router.py::list_tables`
**Cambio**: aditivo y compatible. Sin los parámetros nuevos, la respuesta y el comportamiento son **idénticos a hoy** (FR-023, FR-025).

---

## Parámetros de query

| Parámetro | Tipo | Default | Notas |
|---|---|---|---|
| `page` | int ≥ 1 | *(ausente)* | **NUEVO** (FR-019). Presencia de `page` **o** `size` activa el modo paginado. |
| `size` | int, 1–100 | *(ausente)* | **NUEVO**. UI: 10/20/50/100; default de presentación `20`. |

No se añaden filtros a "Mesas" (Assumptions del spec): solo paginación.

---

## Comportamiento

### Modo compatible — sin `page` ni `size`

Idéntico a hoy:
```
SELECT * FROM dining_tables ORDER BY number
```
`response_model`: `list[TableResponse]`. Sin `ETag` (igual que hoy).

### Modo paginado — con `page` y/o `size`

- `response_model`: `Page[TableResponse]`.
- Consulta: `SELECT * FROM dining_tables ORDER BY number ASC` → `paginate(db, stmt, page, size)`.
  `number` es `UNIQUE NOT NULL` ⇒ orden totalmente determinado, sin desempate (FR-020).
- `total`/`pages` sobre el conjunto completo de mesas del tenant.
- **Clamp** (FR-005, aplicado por consistencia con "Órdenes"): `page` fuera de rango → última página con resultados; `total == 0` → `page = 1`, `items = []`.

Respuesta ejemplo (`page=1&size=20`, 64 mesas):

```json
{ "items": [ { "...": "TableResponse" } ], "total": 64, "page": 1, "size": 20, "pages": 4 }
```

---

## `TableResponse` (sin cambios)

`app/api/v1/orders/schemas.py::TableResponse` — `id, number, name, qr_token, active, status`. Sin cambios.

---

## Consumidores y no-regresión (FR-025)

| Consumidor (`pos-heladeria`) | Llamada | Modo | Efecto |
|---|---|---|---|
| Terminal de Mesas — `pos-terminal.store.ts` (`tableService.loadTables()` ×3) | `GET /orders/tables` | compatible | ninguno |
| Dashboard — `admin-dashboard.component.ts` | `loadTables()` | compatible | ninguno |
| Detalle de orden — `order-detail.component.ts` (resolución de etiqueta de mesa) | `loadTables()` | compatible | ninguno |
| Hoja de impresión de QR — `table-qr-sheet.component.ts` | `loadTables()` | compatible | ninguno |
| Pantalla "Mesas" — `tables-page.component.ts` | `TableService.loadTablesPage()` → `?page=&size=` | paginado | **es el objetivo** |

Las operaciones de la pantalla "Mesas" (`POST /orders/tables`, `PATCH /orders/tables/{id}`, `PATCH /orders/tables/{id}/status`, `GET /orders/tables/{id}/qr-token`) **no cambian** (FR-022); tras completarse, el frontend recarga la página actual del carril paginado (o la última válida).
