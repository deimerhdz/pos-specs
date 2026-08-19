# Contrato: campo `tracks_inventory` en los endpoints de producto

Cubre FR-001, FR-004, FR-008/FR-009 sobre los endpoints que ya existen de creación y edición de
producto. **El shape cambia** (campo nuevo) — a diferencia de los contratos de spec 026, aquí sí hay
un cambio de forma, no solo de efecto.

## `POST /products`

- **Request** — `ProductCreate` (`app/api/v1/products/schemas.py:14-23`), campo nuevo:
  ```json
  {
    "category_id": "…",
    "name": "Domicilio",
    "description": null,
    "preparation_type": "packaged",
    "image_url": null,
    "available": true,
    "tracks_inventory": false
  }
  ```
  `tracks_inventory` es **opcional en el request** (tiene default): si se omite, el sistema asume
  `false` (FR-001) — un cliente HTTP que no conozca este campo todavía sigue creando productos sin
  inventario por defecto, no productos que exigen receta.
- **Response 201** — `ProductResponse`/`ProductDetailResponse` (líneas 36-56), incluye
  `"tracks_inventory": false` en el body devuelto.
- **Efecto**: `ProductService.create_product` (`service.py:41-65`) pasa `tracks_inventory` de forma
  explícita al constructor de `Product`, igual que ya hace con `available` — el `ensure_default_variant`
  posterior (spec 002) no cambia.

## `PATCH /products/{id}` (y su alias `PUT /products/{id}`)

- **Request** — `ProductUpdate` (líneas 26-33), campo nuevo opcional:
  ```json
  { "tracks_inventory": true }
  ```
  Mismo patrón que `active`/`available`: `null`/ausente no toca el valor actual; solo se actualiza
  cuando el campo viene explícito en el body (`if data.tracks_inventory is not None: product.tracks_inventory = data.tracks_inventory`,
  mismo estilo que `service.py:80-84`).
- **Response 200** — `ProductResponse`, con el valor ya actualizado.
- **Efecto**: cambiar este campo **nunca** toca `RecipeItem` ni `VariantOptionGroup` de ninguna
  presentación del producto — no hay ningún `DELETE` asociado a este endpoint por este campo
  (FR-008). El frontend es responsable de pedir confirmación (FR-014) **antes** de enviar este
  `PATCH` cuando corresponda — el backend no la exige ni la valida, es una decisión de UX, no de
  contrato de datos.
- **Sin validación cruzada nueva**: el backend NO rechaza apagar el switch de un producto con
  insumos configurados, ni exige tenerlos para encenderlo — ambas reglas (FR-013 advertencia,
  FR-014 confirmación) son enteramente responsabilidad del frontend sobre datos que ya puede leer
  del `draft()` cargado (research.md Decisión 8/9). El único lugar donde `tracks_inventory=true` sin
  insumos tiene una consecuencia real es en el momento de vender (ver
  `sale-consumption-guard.md`), no al guardar el producto.

## Sin cambios

- `GET /products`, `GET /products/{id}`: agregan `tracks_inventory` al body de respuesta
  (`ProductListResponse`/`ProductDetailResponse` heredan de `ProductResponse`), sin cambios de
  comportamiento adicionales.
- `POST /products/{id}/variants`, `PATCH /variants/{id}`, `PUT /variants/{id}/recipe`
  (spec 002/003): **sin ningún cambio de forma ni de validación** — siguen aceptando y guardando
  recetas/grupos exactamente igual, sin importar el valor de `tracks_inventory` de su producto. La
  diferencia solo aparece más adelante, al vender (ver el otro contrato de esta carpeta).
