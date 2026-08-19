# Data Model: Control de Inventario por Producto (Switch de Insumos)

Las decisiones de diseño detrás de cada elección están en [research.md](./research.md); este
documento se limita a las columnas, restricciones, transiciones y a la consulta de migración.

## Product (`products`, schema `tenant`) — MODIFICADA

Columna nueva:

| Columna | Tipo | Nulable | Default (ORM) | Default (BD) | Notas |
|---|---|---|---|---|---|
| `tracks_inventory` | `Boolean` | No | `True` (research.md Decisión 3) | `server_default="true"` | Determina si las presentaciones del producto exigen y aplican la validación/descuento de spec 003. |

Columnas sin cambio: `id`, `category_id`, `name`, `description`, `preparation_type`, `image_url`,
`active`, `available`.

**Asimetría intencional de defaults** (research.md Decisión 3): el default `True` a nivel de modelo
protege cualquier construcción de `Product(...)` que no mencione este campo (tests existentes,
scripts, código futuro) — preserva el comportamiento actual (Principio II). El default `False` que
pide FR-001 ("apagado por defecto en todo producto nuevo") vive **solo** en el schema Pydantic
`ProductCreate`, el único punto por el que pasa la creación desde el formulario nuevo.

## ProductVariant (`product_variants`) — SIN CAMBIO DE ESQUEMA

Sigue siendo la unidad real de venta (spec 002). Cambia únicamente su **comportamiento derivado**:
su receta fija (`recipe_items`) y sus grupos de opciones que descuentan (`variant_option_groups`)
solo se exigen (guard) y se aplican (descuento real) cuando `Product.tracks_inventory` del producto
al que pertenece es `True` — evaluado de forma independiente por presentación (FR-006), nunca
agregado a nivel de producto.

## RecipeItem / VariantOptionGroup / Option — SIN CAMBIO DE ESQUEMA

Sin cambios de estructura ni de validación (spec 003). Lo único nuevo es que su exigencia y su
aplicación en el momento de la venta ahora pasan primero por el chequeo de `tracks_inventory` del
producto (research.md Decisión 1/2), antes de llegar a la lógica ya existente.

## Migración de datos existentes

Ver research.md, Decisión 4, para la migración completa (`op.add_column` +
`op.execute(UPDATE ... EXISTS ...)`). Resumen de la regla de backfill:

```text
tracks_inventory(producto) = EXISTS(
    alguna presentación ACTIVA del producto que tenga:
      receta fija (RecipeItem) para esa presentación
      O un grupo de opciones vinculado con quantity_per_option > 0
      O un grupo de opciones vinculado con alguna opción activa,
        con inventory_item_id y item_quantity > 0
)
```

Replica exactamente la lógica de `load_recipe`/`group_discounts`
(`app/catalog_engine/consumption.py:50-88`), restringida a presentaciones activas (una presentación
desactivada con receta vieja no cuenta para decidir si el producto migra encendido).

## Transición del switch (`tracks_inventory`) en el formulario

```text
                    ┌───────────┐
   (crear producto) │  false    │  ← default de ProductCreate (FR-001)
                    └─────┬─────┘
                          │
                 activar (sin confirmación)
                          │
                          ▼
                    ┌───────────┐
                    │   true    │──── guardar sin insumos ──► advertencia visible (FR-013)
                    └─────┬─────┘                              el producto sigue sin venderse
                          │                                     hasta configurar algo (FR-006)
             desactivar, PRODUCTO SIN insumos
             configurados (guarda directo)
                          │
                          ▼
                    ┌───────────┐
                    │  false    │
                    └───────────┘

                    ┌───────────┐
                    │   true    │  (con receta o grupo configurado)
                    └─────┬─────┘
                          │
             desactivar, PRODUCTO CON insumos
             configurados → confirmación explícita (FR-014)
                          │
              ┌───────────┴───────────┐
         cancelar                  aceptar
              │                       │
              ▼                       ▼
        vuelve a true           false (insumos
        (sin guardar)           persisten, FR-008)
```

- Los insumos (`RecipeItem`, `VariantOptionGroup`) nunca se borran por un cambio del switch — solo
  se exigen/aplican o no, según el valor vigente al momento de la venta (FR-008/FR-009).
- No existe ningún estado nuevo que persistir aparte de la propia columna booleana — no hay un
  tercer valor ni un estado intermedio "confirmando".

## Reglas de validación (resumen por historia de usuario)

| Regla | Dónde se aplica | Historia |
|---|---|---|
| Switch apagado por defecto en producto nuevo | `ProductCreate.tracks_inventory: bool = False` | US1 |
| Guardar sin insumos con switch apagado no bloquea | `ProductService.create_product`/`update_product` — sin validación nueva que lo impida | US1 |
| Venta de producto con switch apagado no se rechaza ni descuenta | `plan_line_consumption`, `ensure_lines_consume_inventory` (`app/catalog_engine/consumption.py`) | US1, US3 |
| Venta de producto con switch encendido sigue las reglas de spec 003 sin cambios | Mismas dos funciones, camino sin modificar cuando `tracks_inventory=True` | US2 |
| Insumos persisten al apagar el switch | Sin `DELETE` en `RecipeItem`/`VariantOptionGroup` al hacer `PATCH` del switch — solo se actualiza la columna del producto | US3 |
| Confirmación al apagar con insumos configurados | Frontend (`ConfirmService.ask`), antes de enviar el `PATCH` | US3, FR-014 |
| Advertencia al guardar con switch encendido y sin insumos | Frontend, derivada de `draft()` en memoria, sin llamada nueva al backend | US2, FR-013 |
| Migración: switch según si ya superaba `RN-CAT-34` | `op.execute(UPDATE ... EXISTS ...)` de la migración nueva | US4 |
| Switch uniforme por producto, no por presentación | No existe columna equivalente en `product_variants` — el atributo vive solo en `products` | Edge case, FR-007 |
