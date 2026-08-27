# Data Model: Orden de Presentaciones de un Producto

Las decisiones de diseño detrás de cada elección están en [research.md](./research.md); este
documento se limita a la columna, la restricción, la migración y las transiciones.

## ProductVariant (`product_variants`, schema `tenant`) — MODIFICADA

Columna nueva:

| Columna | Tipo | Nulable | Default (ORM) | Default (BD) | Notas |
|---|---|---|---|---|---|
| `display_order` | `Integer` | No | sin default de aplicación — siempre se asigna explícitamente al crear (research.md Decisión 1) | ninguno tras el backfill (research.md Decisión 5) | Posición de la presentación dentro de su producto; determina el orden en el formulario y en el detalle del Menú QR. |

Restricción nueva:

- `UNIQUE (product_id, display_order) DEFERRABLE INITIALLY DEFERRED` —
  `uq__product_variants__product_id__display_order` (research.md Decisión 3). Permite que el
  endpoint de reordenamiento reescriba el `display_order` de todas las presentaciones de un producto
  en una sola transacción sin violar la unicidad en un estado intermedio.

Columnas sin cambio (confirmadas contra `app/models/product_variant.py:14-47`): `id`, `created_at`,
`updated_at`, `product_id`, `name`, `sku`, `price`, `active`; relaciones `recipe_items`/
`option_groups`; constraints `CheckConstraint(price >= 0)`, `UniqueConstraint(product_id, name)`.

## Product (`products`) — SIN CAMBIO DE ESQUEMA

Solo cambia el `order_by` declarado en la relación `Product.variants`
(`app/models/product.py:52`, research.md Decisión 7):

```python
variants: Mapped[list["ProductVariant"]] = relationship(
    ...,
    order_by="ProductVariant.display_order",
)
```

Cualquier código que ya recorra `product.variants` (el detalle del Menú QR, el `GET /products/{id}`
que alimenta el formulario, y cualquier consulta futura) recibe automáticamente la lista ordenada,
sin tocar cada punto de lectura por separado.

## Asignación de `display_order` por operación

| Operación | Efecto sobre `display_order` | Regla |
|---|---|---|
| Crear producto nuevo (`ensure_default_variant`, spec 002) | La variante `"Single"` recibe `display_order = 1` — es la única presentación del producto. | Consistencia con FR-005/FR-009 |
| Crear presentación adicional (`POST /products/{id}/variants`) | Recibe `display_order = MAX(display_order) + 1` entre **todas** las presentaciones del producto (activas e inactivas) — se agrega al final. | FR-005 |
| Reordenar (`PATCH /products/{id}/variants/reorder`) | Reasigna `display_order = 1..N` a las presentaciones **activas** del producto, según la posición de cada ID en la lista recibida. No toca las desactivadas. | FR-002, FR-003, FR-010 |
| Desactivar (`DELETE /variants/{id}`, soft-delete) | Ninguno — `display_order` no se modifica. | FR-007, research.md Decisión 4 |
| Reactivar (`PATCH /variants/{id} {"active": true}`) | Ninguno — conserva el `display_order` que ya tenía. | FR-008, research.md Decisión 4 |
| Editar nombre/precio/SKU (`PATCH /variants/{id}`) | Ninguno. | FR-006 |

## Migración de datos existentes

Ver research.md, Decisión 5, para la migración completa. Resumen de la regla de backfill:

```text
display_order(presentación) = número de orden de esa presentación entre todas las
  presentaciones (activas e inactivas) del mismo producto, ordenadas por id ascendente
  (equivalente al orden de creación, ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY id))
```

Reproduce exactamente el orden implícito que el sistema ya muestra hoy (confirmado: ninguna consulta
actual de `ProductVariant` para listado tiene `.order_by()` propio — el orden visible hoy es el de
inserción/PK), de modo que desplegar esta funcionalidad no reordena nada por sí solo (FR-009, SC-004).

## Contrato del endpoint de reordenamiento (resumen)

Ver [contracts/product-variants-reorder.md](./contracts/product-variants-reorder.md) para el
detalle completo de request/response/validaciones.

## Reglas de validación (resumen por historia de usuario)

| Regla | Dónde se aplica | Historia |
|---|---|---|
| Arrastrar reordena la vista y renumera de inmediato (1..N) | Frontend, `moveItemInArray` sobre `draft().variants` (research.md Decisión 6) | US1 |
| El nuevo orden se persiste al guardar el producto | `product.service.ts`, `reorderVariants()` dentro de `saveExistingProduct` (research.md Decisión 2) | US1 |
| El Menú QR refleja el mismo orden guardado | `order_by` en `Product.variants` (research.md Decisión 7) | US2 |
| Crear/editar/eliminar presentaciones no se ve afectado | `display_order` no interviene en esas rutas salvo para asignarlo al crear (research.md Decisión 1) | US3 |
| Eliminar no deja huecos; reactivar conserva el orden | `display_order` no se recalcula en `DELETE`/reactivación (research.md Decisión 4) | US3, Edge Cases |
| El orden es por producto, no global | `UNIQUE (product_id, display_order)` — el mismo valor puede repetirse entre productos distintos | FR-010 |
