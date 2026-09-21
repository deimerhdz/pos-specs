# Contrato: Asociación de variante con presentación del catálogo (bug 2)

> **Enmienda 2026-09-20 (A-79)** — este contrato describe la versión intermedia (presentación
> opcional, `name` conservado, cascada de renombre). Quedan **vigentes** solo el schema
> `presentation_id` en `VariantSaveIn`/`VariantResponse` y las validaciones 422/409 de asociación.
> Quedan **sustituidos** el uso de `name`, la opción "Sin presentación", el `PATCH /presentations/{id}`
> con cascada y su guarda de colisión, y el frontend de nombre de solo lectura. Fuente vigente:
> [variante-sin-nombre.md](./variante-sin-nombre.md).

Cubre spec.md FR-001 a FR-007. Ver [data-model.md](../data-model.md) para el cambio de esquema.

## `POST /products` y `PUT /products/{id}` (o el endpoint equivalente de guardado de variantes)

**Schema modificado**: `VariantSaveIn` (`app/api/v1/catalog/schemas.py`)

```
VariantSaveIn:
  id: UUID | null            # sin cambio
  name: str (min_length=1)   # sin cambio de tipo; su valor se IGNORA si presentation_id no es null
  price: Decimal              # sin cambio
  sku: str | null             # sin cambio
  active: bool                 # sin cambio
  recipe: ... | null           # sin cambio
  option_groups: [...] | null  # sin cambio
  presentation_id: UUID | null # NUEVO — presentación del catálogo asociada a esta variante
```

**Comportamiento del servicio** (`_save_variant_entry`, `app/api/v1/catalog/service.py`):

1. Si `presentation_id` es `null` → comportamiento idéntico a hoy: `name` del payload se usa tal
   cual, validación de unicidad `(product_id, name)` sin cambio.
2. Si `presentation_id` no es `null`:
   - Debe existir una `Presentation` con ese id, del mismo tenant, con `active=True` — si no,
     `422` con mensaje "La presentación seleccionada no existe o está inactiva".
   - Ninguna otra variante del mismo `product_id` puede tener ya ese `presentation_id` — si la
     hay, `409` con mensaje "Esta presentación ya está en uso por otra variante de este producto"
     (mismo estilo que el error `variante_duplicada` ya existente).
   - El `name` efectivo que se guarda es `Presentation.name` (el `name` recibido en el payload se
     descarta silenciosamente; el frontend nunca debería enviar uno distinto porque el campo es
     de solo lectura en el formulario mientras haya una presentación asociada, FR-002).

**Schema modificado**: `VariantResponse` gana `presentation_id: UUID | null` (para que el
frontend sepa qué opción del `<select>` marcar al editar).

## `PATCH /presentations/{id}`

**Sin cambio de schema** (`PresentationUpdate`/`PresentationResponse` idénticos). **Cambio de
comportamiento del servicio** (`app/api/v1/presentations/router.py`): cuando el `PATCH` incluye un
`name` distinto al actual, en la misma transacción se ejecuta:

```sql
UPDATE product_variants
SET name = :nuevo_nombre
WHERE presentation_id = :presentation_id;
```

Antes del `COMMIT` de la validación de unicidad de nombre de `Presentation` ya existente (spec 083
FR-003) — si el nombre nuevo ya lo usa otra presentación, la operación completa (incluida la
cascada) se rechaza como hoy, sin ejecutar ningún `UPDATE`.

**Guarda adicional (spec.md FR-004, edge case)**: antes de ejecutar la cascada, el servicio valida
que ningún producto con una variante asociada a esta presentación tenga ya otra variante (sin esta
presentación) con `name = :nuevo_nombre` — ver la consulta exacta en
[data-model.md](../data-model.md#efecto-en-cascada). Si la hay, el `PATCH` se rechaza por completo
(ni `Presentation.name` ni ninguna variante cambian).

## Casos de error (resumen)

| Caso | Código | Mensaje |
|---|---|---|
| `presentation_id` no existe o está inactiva | 422 | "La presentación seleccionada no existe o está inactiva" |
| `presentation_id` ya usado por otra variante del mismo producto | 409 | "Esta presentación ya está en uso por otra variante de este producto" |
| Renombrar una presentación colisionaría con el nombre de otra variante del mismo producto | 409 | "No se puede renombrar: el producto '<producto>' ya tiene una variante llamada '<nuevo_nombre>'" |

## Frontend (`product-form.component.ts`)

- Cada fila de variante gana un `<select>` de presentación, poblado desde
  `PresentationService.allPresentations` (ya existe, `loadAllPresentations()`) filtrado a
  `active=true` en el cliente, con la opción "Sin presentación" (`presentationId: null`).
- Al elegir una presentación: el `<input>` de `name` de esa fila pasa a `readonly`/`disabled` y
  muestra el nombre de la presentación elegida (autocompletado desde el mismo `allPresentations`,
  sin esperar la respuesta del servidor).
- Al volver a "Sin presentación": el `<input>` de `name` vuelve a ser editable, conservando el
  texto que tenía en ese momento (no lo limpia).
- `VariantDraft`/`VariantSavePayload` (`product.interface.ts`) ganan `presentationId: string |
  null`, enviado como `presentation_id` en el payload.
