# Contrato: `PATCH /products/{id}` y `PUT /products/{id}`

Endpoints existentes (`app/api/v1/products/router.py:79-115`, funciones `update_product` /
`replace_product`, ambas llaman `service.update_product`), **sin cambios de forma** en esta spec —
mismo esquema de request y de response, mismos códigos de estado, misma autorización
(`require_tenant_admin`). Lo único que cambia es **cuándo**, internamente, se borra el objeto de
imagen anterior en Cloudflare R2 respecto al commit en base de datos — un detalle no observable
por el cliente HTTP en el camino feliz (ver Response 200 más abajo).

## Request

```
PATCH /products/{id}
PUT   /products/{id}
Authorization: requerido (Depends(require_tenant_admin)) — sin cambios
```

| Parámetro | Tipo | Origen |
|---|---|---|
| `id` | UUID | Path |

Body (`ProductUpdate`, `products/schemas.py:26-33`) — **sin cambios de esquema**:

```json
{
  "category_id": "uuid | null",
  "name": "string | null",
  "description": "string | null",
  "preparation_type": "string | null",
  "image_url": "string | null",
  "active": "boolean | null",
  "available": "boolean | null"
}
```

Todos los campos son opcionales (actualización parcial); esta delta solo afecta el procesamiento
interno cuando `image_url` viene presente y distinto al valor actual del producto.

## Response — `200 OK` (`ProductResponse`)

Esquema **sin cambios**. Comportamiento observable **sin cambios** en el camino feliz: cuando el
guardado tiene éxito, la respuesta refleja el `image_url` nuevo y el objeto viejo ya fue borrado de
R2 — exactamente igual que antes de esta corrección (FR-003, spec.md). El cliente HTTP no puede
distinguir, por la respuesta, si el borrado ocurrió antes o después del commit.

## Response — `404 Not Found`

Sin cambios: producto, categoría o unidad de medida inexistentes (`get_or_404`).

## Response — `422 Unprocessable Entity`

Sin cambios: datos de entrada inválidos (validación de `ProductUpdate`). Esta delta no agrega
ningún caso nuevo de `422` — a diferencia del caso de fallo de guardado por otras causas (ver
abajo), que ya devolvía un error de servidor antes de esta corrección y lo sigue haciendo después;
lo único que cambia ante ese fallo es el estado de R2 tras la respuesta de error, no el código de
estado HTTP en sí.

## Comportamiento interno ante un fallo de guardado (no es un código de estado nuevo)

Antes de esta corrección, un fallo de `db.commit()` posterior al borrado en R2 (causa ajena a
`image_url`, p. ej. un error de otro campo) dejaba el objeto de imagen anterior borrado en R2 pese
a que la respuesta de error revierte `product.image_url` a la URL vieja en base de datos — la URL
devuelta en una consulta posterior del producto quedaba apuntando a un objeto inexistente (A-44).
Después de esta corrección, ante el mismo fallo, el objeto de imagen anterior **permanece** en R2 —
la respuesta de error es la misma (el código de estado que ya producía ese fallo no cambia), pero
el estado de R2 tras el error es ahora consistente con `product.image_url`.

## Consumidor (`pos-heladeria`)

El panel de administración ya consume ambos endpoints para el flujo de reemplazo de imagen de
producto. Ningún cambio de contrato de tipos ni de código de estado — fuera de alcance de esta spec
cualquier ajuste de UI/UX (Out of Scope de `spec.md`).
