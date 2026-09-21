# Contrato: Variante sin nombre propio — el nombre lo da la presentación (enmienda 2026-09-20)

Cubre spec.md FR-029 a FR-036 (User Story 7) y anomalía A-79. Reemplaza
[variante-presentacion.md](./variante-presentacion.md) (versión intermedia). Cambio de esquema en
[data-model.md](../data-model.md#enmienda-2026-09-20--productvariant-sin-name-a-79).

**Cambio de contrato no retrocompatible**: ningún schema de variante acepta ni devuelve `name`.
Ver el orden de despliegue en [research.md](../research.md) D10.

## Schemas modificados (`app/api/v1/catalog/schemas.py`, `products/schemas.py`, `menu/schemas.py`)

```
VariantSaveIn:                       # POST/PATCH/PUT /products (guardado consolidado)
  id: UUID | null
  name: —                            # ELIMINADO (antes: obligatorio, ignorado si había presentación)
  presentation_id: UUID | null       # null ⇒ "Presentación única" (get-or-create). Ver reglas.
  price, sku, active, recipe, option_groups   # sin cambio

VariantCreate  (POST /variants):     name → presentation_id: UUID | null   # misma regla
VariantUpdate  (PATCH /variants/{id}): name → presentation_id: UUID | null  # ausente = sin cambio

VariantResponse / VariantSaveOut:
  name: —                            # ELIMINADO
  presentation_id: UUID              # ya no nullable
  presentation_name: str             # NUEVO — Presentation.name

MenuVariantResponse (GET /menu/...): name → presentation_name: str  (+ presentation_id: UUID)
```

`PromotionVariantResponse.description` (texto compuesto `"<producto> - <variante>"`) **no cambia de
forma**: sigue siendo `str`; solo cambia de dónde sale el segundo término (`presentation_name`).
Lo mismo para `variant_display_names()` y el descriptor de regla de `promotions/service.py`
(spec 066): el respaldo "si el nombre de la variante está vacío, usar el del producto" queda sin
efecto (una presentación nunca está vacía) y se retira.

## Reglas del servicio (`_save_variant_entry`, `app/api/v1/catalog/service.py`)

1. **Resolver la presentación**:
   - `presentation_id` no nulo ⇒ debe existir (mismo tenant). Debe estar `active=True` **salvo** si
     es la que esa misma variante ya tiene. Si no: `422` "La presentación seleccionada no existe o
     está inactiva".
   - `presentation_id` nulo ⇒ presentación "Presentación única": `SELECT` por nombre literal, sin
     filtrar `active`; si no existe, `INSERT` (`active=true`); ante `IntegrityError` por carrera,
     re-consultar. (Atajo de entrada para productos sin tamaños, FR-034; lo que se guarda y se
     devuelve nunca es nulo.)
2. **Unicidad por producto**: ninguna otra variante del mismo `product_id`, **activa o
   desactivada**, puede tener ya esa presentación ⇒ `409` "Esta presentación ya está en uso por otra
   variante de este producto" con `variant_index`. Si la que choca está desactivada, el mensaje
   conserva el flujo actual de `variante_duplicada`: "Ya existe la variante «<presentación>»
   desactivada en este producto. Reactívala en vez de crear otra." Dos filas del mismo payload con
   `presentation_id` nulo colisionan por esta misma regla.
3. **SKU autogenerado** (`_unique_sku(... f"{_slug(product.name)}-{_slug(effective_name)}")`): pasa
   a usar `Presentation.name` como sufijo. No cambia para SKUs ya guardados.
4. Se elimina la rama "renombrar la variante" (`effective_name != variant.name` →
   `variante_duplicada` → `variant.name = ...`): cambiar de presentación es solo cambiar
   `presentation_id`, con la misma validación de la regla 2.

## `PATCH /presentations/{id}` (`app/api/v1/presentations/router.py`)

Se **elimina** `_rename_conflict` y el `UPDATE product_variants SET name` (FR-031). Comportamiento
final: valida la unicidad de `Presentation.name` (spec 083 FR-003, `409` existente) y actualiza la
fila. El renombre llega a todas las variantes porque estas leen el nombre por JOIN. El `409`
"No se puede renombrar: el producto '…' ya tiene una variante llamada '…'" **deja de existir**
(y su descripción en `responses=` del decorador).

## Otros puntos de código que dejan de leer `name` de la variante

Inventario por `grep` del 2026-09-20 (la tarea T043 lo repite antes de tocar código; cualquier
lectura nueva que aparezca se agrega aquí):

| Archivo (`pos-backend`) | Uso actual | Pasa a |
|---|---|---|
| `catalog/service.py:74` `variante_duplicada` | busca por nombre normalizado | busca por `presentation_id` |
| `catalog/service.py:96` `ensure_default_variant` | `name="Presentación única"` | `presentation_id` de "Presentación única" |
| `catalog/service.py:321-325` | compara/asigna `variant.name` | se elimina (regla 4) |
| `catalog/router.py:161-165` `PATCH /variants/{id}` | asigna `variant.name` | `presentation_id`, regla 2 |
| `catalog/router.py:235` grupos de opciones en uso | `'Producto · Variante'` | `Presentation.name` vía JOIN |
| `products/service.py:241` `to_save_response` | `name=v.name` | `presentation_id`, `presentation_name` |
| `products/service.py:186` herencia de categoría | `VariantSaveIn(name=p.name)` | `VariantSaveIn(presentation_id=p.id)` |
| `menu/router.py:187` | `MenuVariantResponse(name=v.name)` | `presentation_name` (+ `joinedload`) |
| `catalog_engine/consumption.py:160` | `select(Product.name, ProductVariant.name)` | JOIN a `Presentation.name` |
| `sales/service.py:228`, `orders/checkout.py:294-295,447` | descripción `"<producto> - <variante>"` | igual, con `presentation.name`; el texto ya guardado en `sale_items.description` no cambia |
| `promotions/service.py:464-470, 949-956` | nombre de variante para descriptores | `presentation.name` |
| `presentations/router.py:114` `_rename_conflict` | consulta por nombre | se elimina |

`ProductVariant.presentation` se declara `lazy="joined"` (data-model.md): evita N+1 en el menú
público, que recorre todas las variantes del tenant.

## Casos de error (resumen)

| Caso | Código | Mensaje |
|---|---|---|
| `presentation_id` no existe o está inactiva (y no es la ya asociada) | 422 | "La presentación seleccionada no existe o está inactiva" |
| Presentación ya usada por otra variante del mismo producto | 409 | "Esta presentación ya está en uso por otra variante de este producto" |
| ídem, y la que choca está desactivada | 409 | "Ya existe la variante «<presentación>» desactivada en este producto. Reactívala en vez de crear otra." |
| Payload con `name` de variante | — | se ignora silenciosamente (Pydantic descarta campos extra) durante los pasos A–B del despliegue; a partir del paso C el schema no lo declara |
| ~~Renombrar presentación choca con nombre de otra variante~~ | ~~409~~ | eliminado (FR-031) |

## Frontend

- `VariantDraft`/`VariantSavePayload`/`Variant`/`DeactivatedVariant` (`product.interface.ts`):
  se elimina `name`; se agrega `presentationName: string` en lo que se **lee** (borrador y
  desactivadas) para pintar la fila sin depender de que `PresentationService.allPresentations` ya
  haya cargado; `presentationId: string | null` sigue (null = fila nueva sin elegir).
- `product.service.ts` (mapeos `presentation_id` ↔ `presentationId`, líneas ~299/404/423/540/623):
  agrega `presentation_name` al leer y deja de enviar `name` al guardar.
- `menu.service.ts`: el único punto que lee la respuesta cruda del menú; mapea `presentation_name`
  al `name` del modelo interno del cliente (`MenuVariant`). Por eso `menu-lookup.ts`,
  `dining-cart.service.ts`, `product-select.component.ts` y `promotions-page.component.ts`
  (etiqueta de variante y aplicación masiva de US4) **no requieren cambio**: el modelo interno
  conserva su campo `name`, solo cambia su origen. La aplicación masiva sigue emparejando por la
  etiqueta, que ahora es el nombre de la presentación (único en el catálogo).
- Los `*.spec.ts` que construyen variantes con `name` se actualizan (T051).
