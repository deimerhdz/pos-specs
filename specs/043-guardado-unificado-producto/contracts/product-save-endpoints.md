# Contrato: `POST /products` y `PATCH`/`PUT /products/{id}` (guardado consolidado)

Extiende los endpoints existentes (`app/api/v1/products/router.py`, funciones `create_product` /
`update_product` / `replace_product`) — **misma ruta, mismo verbo, misma autorización**
(`require_tenant_admin`), body y response ampliados de forma **aditiva** (research.md Decisión 1).
Reemplaza, para el flujo del formulario de administración de productos, a los cinco endpoints que
se retiran (FR-007): `POST /products/{id}/variants`, `PATCH /variants/{id}`, `PUT
/variants/{id}/recipe`, `PUT /variants/{id}/option-groups`, `PATCH
/products/{id}/variants/reorder`.

## `POST /products` — Crear

```
POST /products
Authorization: requerido (Depends(require_tenant_admin)) — sin cambios
```

Body (`ProductCreate` extendido, data-model.md):

```json
{
  "category_id": "uuid",
  "name": "Cono Waffle",
  "description": "string | null",
  "preparation_type": "prepared | packaged",
  "image_url": "string | null",
  "available": true,
  "tracks_inventory": false,
  "variants": [
    {
      "name": "Pequeño",
      "price": 8000,
      "sku": null,
      "recipe": [{ "inventory_item_id": "uuid", "quantity": 0.2 }],
      "option_groups": [{ "option_group_id": "uuid", "min_select": 1, "max_select": 1, "quantity_per_option": 0 }]
    },
    {
      "name": "Grande",
      "price": 12000,
      "recipe": [{ "inventory_item_id": "uuid", "quantity": 0.35 }],
      "option_groups": []
    }
  ]
}
```

- `variants` es **opcional**; si se omite o llega `[]`, se preserva el comportamiento actual:
  el producto nace con una presentación `"Single"` a precio 0 autogenerada (`RN-CAT-05`).
- Si trae al menos una entrada, esas son las presentaciones iniciales, en ese orden
  (`display_order = 1, 2, ...`); ninguna trae `id` (el producto aún no existe).

### Response — `201 Created` (`ProductSaveResponse`, data-model.md)

```json
{
  "id": "uuid",
  "category_id": "uuid",
  "name": "Cono Waffle",
  "description": null,
  "preparation_type": "prepared",
  "image_url": null,
  "active": true,
  "available": true,
  "tracks_inventory": false,
  "created_at": "2026-08-27T12:00:00Z",
  "updated_at": null,
  "variants": [
    {
      "id": "uuid", "product_id": "uuid", "name": "Pequeño", "sku": "CONO-PEQU", "price": "8000.00",
      "active": true, "display_order": 1,
      "recipe": [{ "id": "uuid", "inventory_item_id": "uuid", "quantity": "0.200" }],
      "option_groups": [{ "id": "uuid", "product_variant_id": "uuid", "option_group_id": "uuid", "min_select": 1, "max_select": 1, "quantity_per_option": "0.000" }]
    },
    { "id": "uuid", "product_id": "uuid", "name": "Grande", "sku": "CONO-GRAN", "price": "12000.00", "active": true, "display_order": 2, "recipe": [ /* ... */ ], "option_groups": [] }
  ]
}
```

### Errores — todo o nada (FR-004)

Ninguno de los siguientes deja el producto ni ninguna de sus presentaciones creada — el `db.commit()`
único no se ejecuta hasta que todo el árbol pasó validación.

| Código | Causa | Forma del `detail` |
|---|---|---|
| `404` | `category_id` no existe | igual que hoy (string) |
| `404` | `variants[i].recipe[j].inventory_item_id` u `option_groups[j].option_group_id` no existen | igual que hoy (string), identificable vía la posición `i`/`j` que ya trae el 422/404 del insumo u opción referenciado — ver nota abajo |
| `409` | `variants[i].name` duplicado contra otra presentación (nunca aplica en creación, salvo entre sí mismas del propio payload) | `{"error": "...", "variant_index": i, "variant_id": null, "active": null}` |
| `409` | `variants[i].sku` ya existe en el tenant | `{"error": "SKU already exists", "variant_index": i}` |
| `422` | Esquema inválido (`price < 0`, `name` vacío, `max_select < min_select`, etc.) | Formato estándar de Pydantic/FastAPI — `loc` ya incluye `["body", "variants", i, "price"]` |
| `422` | Insumo repetido en `variants[i].recipe`, o grupo repetido/inactivo en `variants[i].option_groups` | `{"error": "...", "variant_index": i}` |

## `PATCH /products/{id}` y `PUT /products/{id}` — Actualizar

```
PATCH /products/{id}
PUT   /products/{id}
Authorization: requerido (Depends(require_tenant_admin)) — sin cambios
```

Body (`ProductUpdate` extendido, data-model.md) — todos los campos opcionales (actualización
parcial), igual que hoy:

```json
{
  "name": "Cono Waffle Grande",
  "tracks_inventory": true,
  "variants": [
    { "id": "uuid-existente", "name": "Pequeño", "price": 8500, "recipe": [ /* ... */ ], "option_groups": [ /* ... */ ] },
    { "name": "Familiar", "price": 20000, "recipe": [], "option_groups": [] }
  ]
}
```

- **`variants` ausente del body** → no se toca ninguna presentación (back-compat, research.md
  Decisión 1) — comportamiento idéntico al `PATCH /products/{id}` de hoy.
- **`variants` presente** (incluido `[]`) → reemplazo completo del conjunto de presentaciones
  activas (data-model.md, tabla de reconciliación): entradas sin `id` se crean, con `id` se
  actualizan, activas no listadas se desactivan; `display_order` = posición en la lista.
- Reactivar una presentación desactivada: incluirla con su `id` real (sin `active: false`) — ver
  research.md Decisión 4.

### Response — `200 OK` (`ProductSaveResponse`, mismo shape que la creación)

### Errores — todo o nada (FR-004)

Mismos casos que en creación, más:

| Código | Causa | Forma del `detail` |
|---|---|---|
| `404` | `variants[i].id` no existe o pertenece a otro producto | `{"error": "Variant not found", "variant_index": i, "variant_id": "<id enviado>"}` |
| `409` | `variants[i].name` duplicado contra una presentación **desactivada** del mismo producto | `{"error": "Ya existe una variante «X» desactivada en este producto. Reactívala en vez de crear otra.", "variant_index": i, "variant_id": "<id de la desactivada>", "active": false}` — mismo mensaje/campos que hoy, con `variant_index` agregado |

En ambos casos, el `404`/`422` de esquema ya no aplica a `category_id` si no se envía (parcial).

## Consumidor (`pos-heladeria`)

`ProductService.saveProduct` (`product.service.ts:439-460`) deja de orquestar `saveNewProduct`/
`saveExistingProduct` como una secuencia de llamadas — construye el body anidado directamente desde
`ProductDraft`/`VariantDraft` (que ya tiene esa forma, ver research.md "Hallazgos de código") y hace
una sola llamada `POST`/`PATCH`. `restoreVariant` deja de llamar red (research.md Decisión 4).

## Fuera de alcance de este contrato

- `GET /products`, `GET /products/{id}` — sin cambio de forma (siguen sin devolver `variants`
  anidado; ese campo solo existe en la respuesta de `POST`/`PATCH`, FR-006).
- `GET /variants/{id}/recipe`, `GET /variants/{id}/option-groups` — sin cambio, fuera de alcance
  (lectura).
- Administración del catálogo de grupos de opciones como entidades independientes (`POST`/`PATCH`/
  `DELETE /option-groups`, `/option-groups/{id}/options`) — sin cambio, fuera de alcance de spec 043.

## Resultado del retiro de endpoints (FR-007, tasks.md T030)

De los cinco endpoints candidatos a retiro, **tres se retiraron** (sin ningún otro consumidor):
`PUT /variants/{id}/recipe`, `PUT /variants/{id}/option-groups`,
`PATCH /products/{id}/variants/reorder`. **Dos quedan excluidos** como excepción documentada:
`POST /products/{id}/variants` y `PATCH /variants/{id}` (`create_variant`/`update_variant`)
siguen existiendo porque `app/scripts/test_variantes_duplicadas.py` (characterization script de la
spec 002, `RN-CAT-08`/`RN-CAT-09`) los llama directamente en proceso, sin pasar por HTTP — ver
[`registro-de-anomalias.md` A-55](../../000-reconocimiento/registro-de-anomalias.md). El formulario
de administración de productos no usa ninguno de los cinco: usa exclusivamente los dos endpoints
consolidados de este contrato.
