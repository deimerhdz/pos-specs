# Contrato: orden de categorías en el filtro del Menú QR

Cubre FR-001 a FR-009. No se agrega ningún endpoint nuevo — se modifican los `schemas` y el
efecto de tres endpoints ya existentes de `POST/GET/PATCH /categories`, y el `order_by` de
`GET /menu` / `GET /menu/qr-token/{token}`.

## `POST /categories` — cambio de forma (request) y de efecto

- **Request** (`CategoryCreate`), campo nuevo:
  ```json
  {
    "name": "Bebidas",
    "description": "Gaseosas, jugos y aguas",
    "display_order": 10
  }
  ```
  `display_order` es **opcional** (`int | None`, `ge=0`). Si se omite o se envía `null`, el
  backend asigna automáticamente `MAX(display_order) + 1` entre todas las categorías existentes
  (data-model.md, tabla de asignación; FR-004).

- **Response 201** (`CategoryResponse`), campo nuevo:
  ```json
  {
    "id": "...",
    "name": "Bebidas",
    "description": "Gaseosas, jugos y aguas",
    "active": true,
    "display_order": 10,
    "created_at": "...",
    "updated_at": null
  }
  ```

- **Validaciones y errores**:
  - `422` si `display_order` es negativo o no es un entero (Pydantic `ge=0`) — mensaje de error
    estándar de validación de FastAPI (FR-003).
  - Sin cambio en las validaciones existentes de `name` (único, `409` si duplicado).

## `PATCH /categories/{id}` — cambio de forma (request) y de efecto

- **Request** (`CategoryUpdate`), campo nuevo:
  ```json
  { "display_order": 25 }
  ```
  `display_order` es **opcional**. Si se omite, el valor existente de la categoría no cambia
  (igual que `name`/`description`/`active` cuando no vienen en el body) — FR-002.

- **Response 200** (`CategoryResponse`): mismo *shape* que en `POST`, con `display_order`
  actualizado si vino en el request.

- **Validaciones y errores**: igual que en `POST` (`422` si negativo/no numérico).

## `GET /categories` — cambio de forma (response), sin cambio de orden

- **Response 200** (`Page<CategoryResponse>`): cada categoría incluye `display_order` (para que
  la tabla de administración pueda mostrarlo, User Story 3).
- El orden de la página **no cambia** — sigue `order_by(Category.name)` (data-model.md,
  "Listado de administración — SIN CAMBIO DE ORDEN"). Fuera de alcance de esta spec.

## `GET /menu` y `GET /menu/qr-token/{token}` — sin cambio de forma, cambio de orden

- **Response**: sin cambio de *shape* — `MenuCategoryResponse` sigue siendo `{id, name, products}`,
  sin exponer `display_order` (igual que `MenuVariantResponse` no expone el suyo, spec 042).
- **Efecto**: las categorías activas dentro de `menu`/`categories` de la respuesta vienen ahora
  ordenadas por `display_order` descendente y, en caso de empate, por `name` ascendente
  (data-model.md, "Menú QR — consulta modificada"; FR-005, FR-006). Un cliente que solo lea el
  contenido de cada categoría sin fijarse en el orden del arreglo no nota ningún cambio de forma.
- Categorías inactivas siguen ausentes de la respuesta, sin importar su `display_order` (FR-007,
  comportamiento ya existente, sin cambio).

## Sin cambios de forma en otros endpoints

- `DELETE /categories/{id}` (soft-delete): sin cambio de *shape* ni de `display_order` — la
  categoría desactivada conserva el valor que ya tenía (data-model.md, tabla de asignación).
