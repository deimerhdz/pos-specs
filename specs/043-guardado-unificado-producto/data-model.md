# Data Model: Guardado Unificado de Producto (Crear y Actualizar)

Fase 1 de `/speckit-plan`. **Sin migración**: ninguna tabla, columna, relación ni constraint de
base de datos cambia. Las tres entidades persistidas (`ProductVariant`, `RecipeItem`,
`VariantOptionGroup`) ya tienen hoy todas las columnas que este árbol necesita
(`app/models/product_variant.py`, `recipe_item.py`, `variant_option_group.py`) — esta spec
consolida el **contrato HTTP** de escritura, no el modelo de datos.

## Entidades persistidas (sin cambios — referencia)

| Entidad | Columnas relevantes | Regla de origen |
|---|---|---|
| `Product` | `id`, `category_id`, `name`, `description`, `preparation_type`, `image_url`, `active`, `available`, `tracks_inventory` | spec 002, spec 027 |
| `ProductVariant` | `id`, `product_id`, `name` (único por producto, case/espacio-insensible incl. desactivadas), `sku` (único en el tenant), `price >= 0`, `active`, `display_order` (único por producto) | spec 002 (`RN-CAT-03`, `RN-CAT-06` a `RN-CAT-11`), spec 042 |
| `RecipeItem` | `id`, `product_variant_id`, `inventory_item_id`, `quantity > 0`, único por `(product_variant_id, inventory_item_id)` | spec 003 |
| `VariantOptionGroup` | `id`, `product_variant_id`, `option_group_id`, `min_select >= 0`, `max_select >= min_select`, `quantity_per_option >= 0`, único por `(product_variant_id, option_group_id)` | spec 004 |

## DTOs nuevos (solo forma del request/response — Pydantic, sin tabla propia)

### `RecipeItemIn` / `VariantOptionGroupIn` (reutilizados tal cual)

Ya existen en `app/api/v1/catalog/schemas.py` (líneas 57-62 y 78-96) — el árbol consolidado los usa
sin ningún cambio de forma.

### `VariantSaveIn` (nuevo)

Una presentación dentro del árbol de `POST`/`PATCH /products`. Su posición (índice, 1-based) en la
lista `variants` del body determina `display_order`.

| Campo | Tipo | Notas |
|---|---|---|
| `id` | `UUID \| None` | `None` (u omitido) → crear presentación nueva. Presente → actualizar la presentación existente con ese id (debe pertenecer al producto). Solo tiene sentido en `PATCH`/`PUT /products/{id}`; en `POST /products` siempre es `None` (el producto todavía no tiene presentaciones). |
| `name` | `str` (1-255, recortado) | Igual regla que `VariantCreate.name` hoy — duplicado contra otra presentación del mismo producto (activa o desactivada) rechaza **todo** el guardado (FR-004, `RN-CAT-08`). |
| `price` | `Decimal >= 0` | Igual que `VariantCreate.price` hoy. Default `0`. |
| `sku` | `str \| None` | Igual que hoy: si se omite, se autogenera (`_slug`/`_unique_sku`); si se provee y choca con el de otra variante de cualquier producto del tenant, rechaza todo el guardado (`RN-CAT-11`). |
| `active` | `bool` | Default `True`. `False` explícito desactiva la presentación en el mismo guardado (equivalente a un `DELETE /variants/{id}` de hoy, pero dentro de la transacción consolidada). |
| `recipe` | `list[RecipeItemIn]` | Reemplazo total, igual semántica que `PUT /variants/{id}/recipe` hoy. Lista vacía = sin receta. |
| `option_groups` | `list[VariantOptionGroupIn]` | Reemplazo total, igual semántica que `PUT /variants/{id}/option-groups` hoy. Lista vacía = ningún grupo ofrecido. |

### `ProductCreate` (extendido)

Todos los campos existentes (`category_id`, `name`, `description`, `preparation_type`,
`image_url`, `available`, `tracks_inventory`) sin cambio +:

| Campo nuevo | Tipo | Notas |
|---|---|---|
| `variants` | `list[VariantSaveIn] = []` | Si viene vacío (u omitido), se preserva el comportamiento actual: se crea automáticamente la presentación `"Single"` a precio 0 (`RN-CAT-05`, `ensure_default_variant`). Si trae al menos una entrada, esas son las presentaciones iniciales del producto — `ensure_default_variant` no se invoca (ya hay al menos una). |

### `ProductUpdate` (extendido)

Todos los campos existentes sin cambio +:

| Campo nuevo | Tipo | Notas |
|---|---|---|
| `variants` | `list[VariantSaveIn] \| None = None` | **Ausente del body** (no en `model_fields_set`) → no se toca ninguna presentación, igual que hoy (back-compat total con cualquier llamador que siga mandando solo campos del producto). **Presente** (incluida `[]`) → reemplazo completo del conjunto de presentaciones activas según Decisión 3 de `research.md`: crea las sin `id`, actualiza las que traen `id`, desactiva las activas no listadas. |

### `ProductSaveResponse` (nuevo — respuesta de ambos endpoints)

Extiende `ProductResponse` (todos sus campos, sin cambio) + :

| Campo nuevo | Tipo | Notas |
|---|---|---|
| `variants` | `list[VariantSaveOut]` | Estado final de **todas** las presentaciones activas del producto tras el guardado, en el orden persistido (`display_order` ascendente) — FR-006: el formulario no necesita una lectura adicional. |

### `VariantSaveOut` (nuevo)

Extiende `VariantResponse` (`id`, `product_id`, `name`, `sku`, `price`, `active`) + :

| Campo nuevo | Tipo | Notas |
|---|---|---|
| `display_order` | `int` | Igual que `VariantOrderEntry.display_order` (spec 042). |
| `recipe` | `list[RecipeItemResponse]` | Receta final de la presentación. |
| `option_groups` | `list[VariantOptionGroupResponse]` | Grupos finales de la presentación. |

## Reconciliación de presentaciones (algoritmo, sin cambio de esquema — referencia desde contracts/)

Dado el árbol `variants` recibido y el conjunto de presentaciones **activas** que el producto ya
tiene en base de datos:

| Caso | Acción |
|---|---|
| Entrada sin `id` | Crear presentación nueva (mismas validaciones que `create_variant` hoy: nombre no duplicado, SKU único/autogenerado). |
| Entrada con `id` que existe y pertenece al producto | Actualizar nombre/precio/sku/active de esa fila (mismas validaciones que `update_variant` hoy). |
| Entrada con `id` que no existe o pertenece a otro producto | `404`/`422` — rechaza todo el guardado. |
| Presentación activa existente cuyo `id` **no** aparece en `variants` | Se desactiva (`active=False`, soft-delete, `RN-CAT-10`) — no se toca su `display_order` histórico salvo que se reactive después. |
| Cualquier entrada (nueva o existente) | `display_order` = posición 1-based dentro de la lista `variants` del body, asignado con el patrón de dos pasadas de `reorder_variants` (research.md Decisión 2). |
| Cualquier entrada (nueva o existente) | `recipe`/`option_groups` se reemplazan por completo con lo recibido (igual que `set_recipe`/`set_variant_option_groups` hoy). |

Este algoritmo corre dentro de una única transacción (research.md Decisión 2): cualquier fallo de
validación en cualquier fila aborta el guardado completo sin persistir nada (FR-004).
