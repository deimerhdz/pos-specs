# Contrato: Asociación Categoría↔Presentaciones y herencia al crear producto

Cubre FR-004 a FR-008, US2. Extiende `app/api/v1/categories/` (schemas + router) y
`ProductService.create_product` (`app/api/v1/products/service.py`) — ningún endpoint nuevo, solo
campos nuevos sobre los ya existentes.

## `POST /categories` y `PATCH /categories/{id}`

`CategoryCreate`/`CategoryUpdate` ganan:

```
presentation_ids: list[UUID] | None = None
```

- **`POST /categories`**: `presentation_ids` ausente o `[]` → la categoría nace sin ninguna
  presentación asociada (comportamiento hoy vigente, sin cambio). Cada id debe existir en
  `presentations` (404 si no) — no se exige `active=true` en el momento de guardar (una categoría
  puede conservar una asociación a una presentación que se desactivó después, FR-002).
- **`PATCH /categories/{id}`**: `presentation_ids` ausente (`None`, campo no enviado) → no toca la
  asociación existente (mismo criterio "solo se modifican los campos enviados" que ya tienen
  `name`/`description`/`active`/`display_order`). `presentation_ids: []` enviado explícitamente →
  desasocia todas. Cualquier otra lista → **reemplazo total** (research.md D5): se borran las
  asociaciones que ya no estén en la lista y se crean las nuevas: 400/404 si algún id no existe.
- Cambiar `presentation_ids` de una categoría **no** modifica ninguna `ProductVariant` de
  productos ya creados en ella (FR-008) — el cambio solo se lee la próxima vez que se cree un
  producto en esa categoría.

## `CategoryResponse`

Gana:

```
presentations: list[PresentationSummary]   # solo lectura
```

```
PresentationSummary:
  id: UUID
  name: str
```

Incluye las presentaciones asociadas **actualmente**, sin filtrar por `active` de la
`Presentation` referenciada (una asociación a una presentación luego desactivada sigue apareciendo
aquí — solo deja de ofrecerse como opción nueva al editar, FR-002). El picker del frontend al
**editar** la asociación sí filtra por `active=true` sobre `GET /presentations` para construir las
opciones seleccionables, pero pre-marca cualquier id ya asociado aunque ya no esté en esa lista
filtrada, para no perderlo al guardar sin querer.

## `POST /products` — herencia automática (sin cambio de contrato de entrada)

`ProductCreate` **no cambia**: sigue aceptando `variants: list[VariantSaveIn] | None`. El cambio
es interno a `ProductService.create_product` (`app/api/v1/products/service.py:76-82`):

```
si data.variants no es vacío:
    _save_variant_tree(...)          # sin cambio — el cliente ya mandó variantes explícitas
si no:
    presentaciones = presentaciones activas asociadas a product.category_id, ordenadas por nombre
    si presentaciones no está vacío:
        crear una VariantSaveIn sintética por presentación (name=presentación.name, price=0)
        _save_variant_tree(db, product, [esas entradas sintéticas])   # FR-005
    si no:
        ensure_default_variant(db, product)   # FR-006, ahora crea "Presentación única" (D7)
```

- **FR-005**: un producto creado en una categoría con presentaciones asociadas nace con una
  `ProductVariant` por cada una, `price=0`, en el mismo `display_order` con el que fueron
  asociadas o alfabético (decisión de detalle, no observable por ningún Acceptance Scenario).
- **FR-006**: un producto creado en una categoría sin presentaciones asociadas (o sin categoría)
  nace con una única variante `"Presentación única"` — mismo mecanismo, nuevo literal (D7, A-74).
- **FR-007**: sin cambio — el administrador sigue pudiendo agregar/editar/eliminar variantes
  manualmente después vía `PATCH /products/{id}` (`_reconcile_variants`), heredadas o no.
- **FR-008**: sin cambio de contrato — es una propiedad del punto anterior (la resolución de
  presentaciones ocurre una sola vez, en `create_product`; `update_product` nunca vuelve a
  consultar `category_presentations`).

## Errores nuevos

| Código | Caso |
|---|---|
| 404 | Un id de `presentation_ids` no existe, en `POST`/`PATCH /categories`. |
| 409 | Nombre de categoría duplicado (sin cambio, ya existente). |

Ningún error nuevo en `POST /products`: si la categoría tiene 0 presentaciones activas asociadas,
el camino es exactamente el mismo que hoy (`ensure_default_variant`), solo con otro nombre.
