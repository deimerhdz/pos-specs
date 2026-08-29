# Data Model: Imagen de producto en el catálogo y organización del tipo de orden — creación de orden manual

Sin entidades ni campos nuevos, de backend ni de frontend. Esta spec no modifica ningún modelo de
datos (spec.md, Key Entities; Principio VIII: N/A).

## Dato ya existente que se reutiliza

- **`MenuProduct.image_url: string | null`** (`pos-heladeria/src/app/modules/products/interfaces/
  product.interface.ts:369-384`) — ya lo devuelve `store.catalogProductsFiltered()`
  (`pos-terminal.store.ts:819`), la misma señal derivada que hoy alimenta la grilla del catálogo de
  `manual-order-page.component.ts:125`. Ya es consumido, para el mismo producto, por
  `product-select.component.ts:48-51` (detalle/opciones). Este plan solo agrega un segundo punto de
  lectura del mismo campo, en la tarjeta del catálogo — no cambia su origen, su tipo, ni cómo se
  puebla desde `pos-backend`.

## Sin cambios de estado ni de transición

El listado de mesas (`store.tablesView()`) y las pestañas de tipo de orden no ganan ningún estado
nuevo: la única pestaña funcional sigue siendo "En Mesa" (spec.md, FR-006; research.md D4). El
título nuevo sobre el listado de mesas es texto estático de UI, no un dato derivado del store.
