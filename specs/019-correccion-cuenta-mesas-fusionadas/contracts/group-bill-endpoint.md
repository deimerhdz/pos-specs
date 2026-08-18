# Contrato: `GET /orders/group/{group_id}/bill`

Endpoint existente (`app/api/v1/orders/router.py:124-131`), **sin cambios de forma** en esta spec —
solo cambian los valores que devuelve, no su esquema. Documentado aquí como referencia del contrato
que la corrección debe seguir respetando (Out of Scope de `spec.md`: "esta delta es de cálculo en
backend... sin cambios de contrato de respuesta").

## Request

```
GET /orders/group/{group_id}/bill
Authorization: requerido (Depends(get_current_user)) — sin cambios
```

| Parámetro | Tipo | Origen |
|---|---|---|
| `group_id` | UUID | Path |

## Response — `200 OK` (`GroupBillResponse`, `orders/schemas.py:77-88`)

Esquema **sin cambios** — mismos campos, mismos tipos:

```json
{
  "merged_group_id": "uuid",
  "total": "decimal",
  "orders": [
    {
      "order_id": "uuid",
      "dining_table_id": "uuid | null",
      "status": "string",
      "subtotal": "decimal"
    }
  ]
}
```

**Cambia el significado de los valores, no la forma**:

| Campo | Antes | Después |
|---|---|---|
| `total` | Suma bruta de todas las órdenes del grupo (terminales incluidas), sin descuentos | Suma de `orders[].subtotal` solo de las órdenes billables (`status not in ("pagada", "cancelada")`) — FR-001/FR-002 |
| `orders[]` | Todas las órdenes del grupo, con `subtotal` bruto (sin descuento) | Todas las órdenes del grupo siguen listadas (research.md Decisión 3); las billables llevan `subtotal` neto de promociones/combos (FR-002); las terminales llevan `subtotal=0` (no contribuyen al `total`, pero quedan visibles para trazabilidad — la Historia 1 exige justamente que el cajero pueda ver que esa orden ya se pagó aparte) |

## Response — `404 Not Found`

Sin cambios: el grupo no existe (ninguna `CustomerOrder` con ese `merged_group_id`, de ningún
status) — research.md Decisión 3. Un grupo que existe pero cuyas órdenes son todas terminales
**no** es 404 (Edge Case de `spec.md`): responde `200` con `total=0`.

## Consumidor (`pos-heladeria`)

`table.service.ts:132-134` (`groupBill()`) y `table.interface.ts:16-20` (`GroupBill`) consumen este
contrato sin cambios de tipo — verificado que `groupBill()` no tiene ningún llamador actual en el
código de `pos-heladeria` (no hay pantalla que hoy dependa del desglose `orders[].subtotal`), así
que el cambio de valores no requiere ningún ajuste en el frontend para no romper una pantalla
existente. Fuera de alcance de esta spec (Out of Scope de `spec.md`).
