# Contrato: Catálogo de Presentaciones (`/presentations`)

Cubre FR-001, FR-002, FR-003, FR-019, US1. Nuevo router `app/api/v1/presentations/` (directorio
ya existe vacío, solo `__pycache__` huérfano de spec 040/063 — D1). Sigue el molde exacto de
`app/api/v1/categories/router.py` + `schemas.py`.

Autenticación/autorización: `AccessTokenBearer` + `get_current_user`, igual que Categorías. Sin
`require_module_access` (D3) — cualquier admin del tenant con acceso al panel puede administrar
el catálogo, mismo criterio que Categorías/Productos.

## Endpoints

### `GET /presentations`

Lista paginada (`Page[PresentationResponse]`, mismo wrapper `app/core/pagination.py` que
`GET /categories`). Query params:

| Param | Tipo | Default | Efecto |
|---|---|---|---|
| `page` | int, ≥1 | 1 | Página. |
| `size` | int, 1–100 | 20 | Tamaño de página. |
| `active` | bool \| null | null (sin filtro) | Filtra por estado. |
| `search` | str \| null | null | Búsqueda `ilike` por nombre. |

Orden: por `name` ascendente (igual que `GET /categories`).

### `GET /presentations/{id}`

`PresentationResponse` o 404.

### `POST /presentations`

Body `PresentationCreate`:

```
name: str (1..255, requerido)
```

- Nace `active=true` (FR-001, "queda disponible... con estado activo").
- 409 si `name` ya existe en el tenant (`ensure_unique`, FR-003).
- Response 201: `PresentationResponse`.

### `PATCH /presentations/{id}`

Body `PresentationUpdate` (todos opcionales, solo se aplican los enviados):

```
name: str | None (1..255)
active: bool | None
```

- Si `name` cambia y ya lo usa otra presentación del tenant → 409 (`ensure_unique(...,
  exclude_id=id)`, FR-003 "también aplica al editar").
- `active` se puede cambiar en cualquier sentido (`false→true` reactiva, US1 Escenario 5) — sin
  ninguna validación de "¿está referenciada?": una presentación se puede desactivar aunque tenga
  asociaciones o variantes que coincidan en nombre, porque desactivar no las borra (FR-002).
- Response 200: `PresentationResponse`.

**Sin `DELETE /presentations/{id}`** (D4): el borrado físico no es una operación expuesta. FR-002
("impedir la eliminación física... permitir desactivarla") se satisface porque la única forma de
retirar una presentación de circulación es `PATCH {active: false}`.

### `PresentationResponse`

```
id: UUID
name: str
active: bool
created_at: datetime
updated_at: datetime | None
```

## Errores

| Código | Caso |
|---|---|
| 401 | No autenticado. |
| 404 | `id` no existe (`GET {id}`, `PATCH {id}`). |
| 409 | `name` duplicado en `POST` o `PATCH` (FR-003). |
| 422 | `name` vacío o > 255 caracteres. |

## Siembra inicial (FR-019)

No es un endpoint — ver [data-model.md](../data-model.md) §Migración Alembic. Al desplegar esta
funcionalidad, `GET /presentations` ya devuelve, para cada tenant, una fila activa por cada
nombre de variante distinto que existiera antes del despliegue (incluyendo, por ejemplo,
`"Single"` si algún producto ya tenía una variante con ese nombre literal — la siembra no filtra
ni excluye ningún nombre existente).
