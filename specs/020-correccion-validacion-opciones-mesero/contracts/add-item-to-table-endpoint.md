# Contrato: `POST /orders/tables/{table_id}/items`

Endpoint existente (`app/api/v1/orders/router.py:202-216`, función `add_table_item` → llama
`consolidation.add_item_to_table`), **sin cambios de forma** en esta spec — mismo esquema de
request y de response en ambos códigos de estado. Lo único que cambia es **cuándo** el endpoint
devuelve `422` en vez de `201` para una variante con opciones obligatorias (Out of Scope de
`spec.md`: "esta delta es de comportamiento en backend... la terminal ya consume el mismo
endpoint").

## Request

```
POST /orders/tables/{table_id}/items
Authorization: requerido (Depends(get_current_user)) — sin cambios
```

| Parámetro | Tipo | Origen |
|---|---|---|
| `table_id` | UUID | Path |

Body (`OrderItemIn`, `orders/schemas.py:100-113`) — **sin cambios de esquema**:

```json
{
  "product_variant_id": "uuid | null",
  "combo_id": "uuid | null",
  "quantity": 1,
  "option_ids": ["uuid", "..."],
  "notes": "string | null"
}
```

Restricciones ya existentes, sin cambio: exactamente uno de `product_variant_id`/`combo_id`; si
`combo_id` está presente, `option_ids` debe estar vacío.

## Response — `201 Created` (`OrderResponse`)

Esquema **sin cambios**. Se devuelve cuando la selección de opciones cumple `min_select`/
`max_select`/pertenencia de grupo de la variante (si el grupo existe) — antes de esta corrección,
se devolvía también cuando la selección **no** cumplía esas reglas (A-04); después, solo cuando sí
las cumple, igual que ya hace `POST /orders` (`create_order`) para el mismo caso.

## Response — `422 Unprocessable Entity`

**Comportamiento nuevo para este endpoint** ante una selección de opciones incompleta o fuera de
grupo (FR-001/FR-002) — mismo código y mismo mecanismo (`HTTPException` desde
`validate_option_selection`, `app/catalog_engine/pricing.py`) que ya usa `POST /orders` hoy para el
caso equivalente. Ya existía `422` en este endpoint para variante inactiva
(`consolidation.py:197-198`, sin cambio); esta corrección agrega el caso de selección de opciones
inválida a la misma familia de respuestas.

Ningún `OrderItem` se crea ni se descuenta inventario cuando la respuesta es `422` — verificado en
`add_item_to_table`: la excepción se lanza antes de `get_or_create_open_order`, dentro de la misma
rama que carga `options`.

## Response — `404 Not Found`

Sin cambios: mesa inexistente (`get_or_404(db, DiningTable, table_id, ...)`) o variante/combo
inexistentes.

## Consumidor (`pos-heladeria`)

La terminal del mesero ya consume este endpoint para el botón de alta directa. El único cambio de
comportamiento visible es que un ítem con sabores/opciones obligatorios incompletos, que hoy se
agrega sin aviso, empieza a devolver `422` — el frontend ya maneja `422` como error de validación
para el mismo endpoint en el flujo de variante inactiva, así que no requiere manejo nuevo de
código de estado, solo mostrará el mensaje de error que ya sabe renderizar. Sin cambios de
contrato de tipos; fuera de alcance de esta spec cualquier ajuste de UI/UX del mensaje mostrado
(Out of Scope de `spec.md`).
