# Contrato: `GET /cart/orders` y `OrderItemResponse` (US1, CA-6)

Endpoint ya existente (`app/api/v1/cart/router.py:170-180`, `service.list_my_orders`) — mismo
`response_model=list[OrderResponse]`, sin cambio de firma. Cambia únicamente el **shape** de
`OrderItemResponse` (`app/api/v1/orders/schemas.py:139-157`), que también usa `POST /cart/submit`
(ver [cart-submit.md](./cart-submit.md)) y `POST /cart/orders/{id}/cancel`.

## Antes

```python
class OrderItemResponse(BaseModel):
    id: UUID
    product_variant_id: UUID
    participant_id: UUID | None = None
    quantity: int
    unit_price: Decimal          # SIN descuento — inconsistente con CartItemResponse
    estado_cocina: str
    void_de: UUID | None = None
    notes: str | None = None
    combo_id: UUID | None = None
    options: list[OrderItemOptionResponse] = Field(default_factory=list)
    rt_v: int | None = None
```

## Ahora (esta spec)

```python
class OrderItemResponse(BaseModel):
    id: UUID
    product_variant_id: UUID
    participant_id: UUID | None = None
    quantity: int
    unit_price: Decimal                              # SIN CAMBIO — precio unitario sin descuento
    discounted_unit_price: Decimal | None = None      # NUEVO — igual a CartItemResponse
    discounted_line_total: Decimal | None = None      # NUEVO — igual a CartItemResponse
    estado_cocina: str
    void_de: UUID | None = None
    notes: str | None = None
    combo_id: UUID | None = None
    options: list[OrderItemOptionResponse] = Field(default_factory=list)
    rt_v: int | None = None
```

**Semántica de los 2 campos nuevos** (idéntica a `CartItemResponse.discounted_unit_price`/
`discounted_line_total`, `app/api/v1/cart/schemas.py:78-79`):
- `None` cuando ninguna promoción de descuento aplicó a esa línea al momento de confirmar, o la
  línea es de un combo (su ahorro se calcula aparte, nunca se apila con percent/fixed).
- Cuando sí aplicó, `discounted_unit_price`/`discounted_line_total` son el precio/subtotal **ya**
  con el descuento reflejado — no un monto de descuento a restar aparte (research.md Decisión 7).
- Para `OrderItem` creados **antes** de esta spec: ambos campos son `null` siempre (columnas nuevas,
  nullable, sin backfill — FR-015), sin importar si el pedido tenía o no una promoción vigente en su
  momento. El frontend interpreta `null` como "sin descuento" para esas filas, sin recalcular.

**Dónde aparece este cambio de shape**: los tres endpoints que devuelven `OrderResponse` con
`items: list[OrderItemResponse]` lo exponen automáticamente, sin cambios de código propios en cada
uno:
- `POST /cart/submit` (`201`, pedido recién creado) — el caso central de FR-013/CA-6.
- `GET /cart/orders` (`200`, lista de pedidos del comensal — lo que ve al recargar/reabrir).
- `POST /cart/orders/{id}/cancel` (`200`, pedido cancelado por el propio comensal) — sin cambio de
  comportamiento de cancelación, solo hereda el shape nuevo de `items[]`.

## Lo que NO cambia

- `unit_price` sigue siendo el precio sin descuento, snapshot copiado del `CartItem` al momento de
  confirmar — ningún consumidor que ya lo use (KDS, terminal de staff) ve su valor alterado.
- El cálculo de descuento de `orders/checkout.py` (`promotions.evaluate`/`combo_discount_for_lines`,
  usado al cobrar) sigue siendo independiente — no lee ni escribe estos 2 campos nuevos (FR-014).
- `pos-heladeria`: `DiningOrderItem` (`dining.interface.ts:70-99`) gana los mismos 2 campos
  opcionales para tipar correctamente la respuesta — sin ningún cambio de componente/plantilla
  (research.md Decisión 8, ninguna pantalla del comensal renderiza hoy precio de línea de pedido).
