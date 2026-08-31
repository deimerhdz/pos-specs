# Contrato: catálogo de presentaciones + referencia desde la variante

Endpoints **nuevos** bajo `/presentations`, en `../pos-backend`
(`app/api/v1/presentations/router.py`). Autorización: `require_tenant_admin` (mismo nivel que
`/promotions` y `/products`). Todo el CRUD opera en el schema del tenant.

Este contrato cubre el Incremento A (research.md D17). No hay ningún cambio en endpoints existentes
salvo el campo `presentation_id` que se agrega al payload de variante (§4).

---

## 1. `GET /presentations` — listar

```text
Query: ?active=true|false (opcional), ?search=<texto> (opcional), paginación page/size
→ 200 Page[PresentationResponse]

PresentationResponse {
  id: UUID
  name: string
  active: bool
  applicable_variant_count: int     # variantes ACTIVAS que la referencian (para el panel del
                                    # formulario de promoción y para que el admin vea el alcance)
  created_at: datetime
  updated_at: datetime | null
}
```

## 2. `POST /presentations` — crear

```text
Body: PresentationCreate { name: string (min_length=1, max_length=100, trim) }
→ 201 PresentationResponse
→ 409 "Ya existe una presentación con ese nombre"   (unicidad por tenant, uq__presentations__name)
```

`active` nace en `true` (no es parámetro de creación).

## 3. `PATCH /presentations/{id}` — renombrar / activar / desactivar

```text
Body: PresentationUpdate { name?: string, active?: bool }
→ 200 PresentationResponse
→ 404 "Presentación no encontrada"
→ 409 "Ya existe una presentación con ese nombre"          (si name colisiona)
→ 409 {                                                     (FR-020: si active=false y alguna
        "error": "La presentación está en uso por promociones activas",
        "promotions": [{ "id": UUID, "name": string }, ...]  regla de una promoción `active` la
      }                                                       referencia — CL-2)
```

Renombrar **no** está bloqueado por uso (el alcance se resuelve por `id`, no por nombre, FR-007) —
solo la desactivación y el borrado lo están.

## 4. `DELETE /presentations/{id}` — eliminar

```text
→ 204 No Content
→ 404 "Presentación no encontrada"
→ 409 {                                                     (FR-020: misma condición que PATCH
        "error": "La presentación está en uso por promociones activas",  active=false)
        "promotions": [{ "id": UUID, "name": string }, ...]
      }
```

Al borrar con éxito:
- `product_variants.presentation_id` de las variantes que la apuntaban → `NULL`
  (FK `ondelete="SET NULL"`).
- Filas de `promotion_presentation_rules` de promociones **no activas** (`draft`/`paused`/
  `finished`) que la referencian → se borran en cascada (FK `ondelete="CASCADE"`, research.md D10).
  Una promoción `draft` que pierde una regla se recompleta antes de activar (que revalida todo).

---

## 5. Cambio en el payload de variante (`/catalog/products/{id}/variants` y equivalentes)

`app/api/v1/catalog/schemas.py` — `VariantCreate` y `VariantUpdate` ganan un campo opcional:

```text
VariantCreate  { ..., presentation_id?: UUID | null }     # NUEVO, opcional, default null
VariantUpdate  { ..., presentation_id?: UUID | null }     # NUEVO — enviar null lo desasigna
VariantResponse{ ..., presentation_id: UUID | null }      # NUEVO
```

Reglas:
- `presentation_id` debe referenciar una presentación **existente y activa** del tenant, o ser
  `null` → 422 "Presentación no encontrada o inactiva" si no.
- `null` es el valor por defecto y el de toda variante existente (FR-008): esa variante no entra en
  ninguna regla por presentación. La "Single" de un producto sin tamaños incluida.
- Cambiar la presentación de una variante **no** dispara la verificación de uniformidad de FR-017
  (esa corre solo al guardar una regla de promoción, CL-1b) — la variante nueva/reasignada
  simplemente entra (o sale) del alcance de las reglas que apunten a esa presentación, efectivo en
  el siguiente cobro (FR-019).

---

## 6. Frontend (`../pos-heladeria`)

- **Módulo nuevo `modules/presentations/`**: página de lista + formulario (crear/renombrar/
  activar/desactivar/eliminar), consumiendo los endpoints de §1–§4. Mensaje de 409 de FR-020
  muestra la lista de promociones y enlaza a cada una.
- **`modules/products/pages/product-form.component.ts`**: cada variante/tamaño del formulario gana
  un selector opcional de presentación (lista de presentaciones activas). Texto de ayuda: una
  variante sin presentación no participa de promociones por presentación.
