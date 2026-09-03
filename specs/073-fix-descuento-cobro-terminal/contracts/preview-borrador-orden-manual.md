# Contrato: Preview del borrador de una orden manual (sin guardar)

**Cubre**: FR-013 a FR-015a — User Story 5 (spec.md).
**Research**: [research.md](../research.md) D5, D6.

## `POST /orders/draft-preview`

Desglose de un pedido **hipotético**, que todavía no existe como `CustomerOrder`, mientras el
cajero arma una orden manual (para llevar o domicilio, o mesa). No persiste nada, no exige
`participant_id`, no exige mesa.

### Request — `DraftPreviewIn`

Mismo shape de líneas que `OrderCreate.items` (`orders/schemas.py:124-144`), para que el frontend
arme este cuerpo con el mismo borrador que luego usará para el `POST /orders` real:

```json
{
  "items": [
    { "product_variant_id": "…", "quantity": 2, "options": [] }
  ],
  "delivery_fee": null
}
```

- `items`: `list[OrderItemIn]`, mínimo 1 — igual validación que `OrderCreate`.
- `delivery_fee`: `Decimal | None`, igual que `OrderCreate.delivery_fee` — el frontend lo manda
  solo si el borrador es de tipo domicilio (FR-003, mismo criterio que hoy usa
  `orderTypeTab() === 'domicilios'`).

### Response `200 OK` — `CheckoutPreviewResponse`

Misma forma exacta que [preview-cobro-pedido.md](./preview-cobro-pedido.md) — `subtotal`,
`discount`, `delivery_fee`, `total`, `promotion_evaluated_at`. En este caso
`promotion_evaluated_at` es siempre el instante de la llamada (`datetime.now(timezone.utc)`, aware
UTC, sin congelar — research.md D5): el borrador no tiene todavía un pedido cuyo instante congelar.

### Errores

| Código | Cuándo |
|---|---|
| `422` | `items` vacío o con un `product_variant_id` inexistente — misma validación que ya aplica `OrderCreate`/`create_order` a las líneas. |

No hay `404`/`409` de estado: no hay ningún recurso persistido que pueda estar en el estado
equivocado.

### Implementación (referencia)

Reusa `promo_lines_for`/`auto_discount` construyendo `SaleLine` directamente desde
`DraftPreviewIn.items` (sin pasar por `order_sale_lines`, que exige un `order_id` real) — mismo
cálculo de precio unitario que ya usa `create_order` para poblar `OrderItem.unit_price`
(`compute_line_price(variant, options)`), para que el subtotal del preview coincida centavo a
centavo con el que tendrá el pedido real una vez creado.

## Consumo desde `pos-heladeria` (referencia, ver research.md D10)

- `manual-order-page.component.ts` (su propia instancia de `PosTerminalStore`, providers línea
  27-34) dispara `loadDraftPreview()` en cada cambio del borrador: agregar línea
  (`addDraftFromSelection`), quitar, cambiar cantidad — FR-013.
- El resumen (líneas 272-286, hoy `store.totals()`) pasa a leer `store.draftPreview()`, con una
  fila `Descuento` nueva (hoy ausente pese a que `tot.discount` existe en el objeto local) cuando
  `discount > 0` (FR-004/FR-014).
- FR-015 (sin conexión / error): si `loadDraftPreview()` falla, el resumen muestra el subtotal
  local (sin descuento) con un aviso de que el descuento se confirma al cobrar, y **no** deshabilita
  "Confirmar pedido" — a diferencia del panel de cobro (FR-007a), aquí no hay dinero
  comprometiéndose todavía.
- FR-015a: justo antes de `confirm()` (línea 403-409 → `store.createManualOrderFromDraft()`), se
  vuelve a pedir el preview una vez más; si el `total` cambió respecto al último mostrado, se
  detiene, se presenta el total nuevo y se exige una segunda confirmación explícita antes de
  llamar a `POST /orders` — mismo patrón de doble chequeo que D11 aplica al cobro.
