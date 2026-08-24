# Contratos de API: Correcciones de Cobro, Anulación y Descuento en la Terminal de Mesas

**Spec**: [spec.md](./../spec.md) | **Datos**: [data-model.md](./../data-model.md)

Todos los endpoints están bajo el prefijo existente `pos-backend` `app/api/v1/`. Autenticación:
staff autenticado (`User`), sin cambios respecto del esquema de auth ya existente. Esta spec no
agrega ningún endpoint nuevo — modifica el contrato de dos endpoints existentes y extiende un
esquema de respuesta también existente.

## Endpoints reutilizados sin cambios de contrato (Out of Scope de esta spec)

| Endpoint | Uso en esta spec |
|---|---|
| `GET /invoices?order_id=` / `GET /sales/{sale_id}` | Sigue siendo la base de la única acción de impresión que sobrevive, "Imprimir Factura" (FR-001/FR-002) — ya usada por `resolveSaleForOrder`/`printOrderInvoice`; sin cambios de contrato. |
| `POST /table-sessions/{id}/release` (`release_paid_session`, spec 028) | Fuera de alcance — esta spec no cambia cuándo se acepta o rechaza el cierre de mesa, solo corrige la insignia mostrada mientras la mesa sigue activa. |
| `PATCH /orders/items/{item_id}/kitchen` (`transition_kitchen`) | Sin cambios — sigue sin validar el estado de pago de la orden (ver research.md D3, decisión explícita de **no** tocar esta función). |

## Endpoints con contrato modificado

### `POST /orders/items/{item_id}/void` — nuevo guard de rechazo (sustenta FR-005/FR-006/FR-007)

**Comportamiento nuevo**: si la orden padre del ítem ya tiene una `Sale` asociada
(`Sale.customer_order_id == item.order_id`) o su `status == "pagada"`, responde:

```jsonc
// 409 Conflict
{
  "detail": "El pedido ya fue pagado y no puede anularse"
}
```

sin mutar el ítem, sin registrar `OrderItemVoidLog`, sin reversa de inventario ni ejecución del
reemplazo si `data.replacement` venía incluido en el request.

**Comportamiento sin cambios**: sobre una orden sin pago registrado, `void_item` se comporta
exactamente igual que hoy (mismas validaciones de `estado_cocina != "anulado"`, misma reversa de
inventario condicional a `pendiente`, mismo mecanismo de reemplazo).

**Request/Response**: sin cambios de forma — mismo `VoidItemIn` de entrada, mismo `CustomerOrder`
serializado de salida en el caso de éxito. El único cambio de contrato es el nuevo caso `409`.

### `POST /orders/{order_id}/checkout-and-send` — `discount` deja de aceptar valores distintos de cero (sustenta FR-009/FR-010/FR-011)

**Antes**:
```jsonc
{ "discount": 0 }   // cualquier valor >= 0 era válido
```

**Después**:
```jsonc
{ "discount": 0 }   // único valor válido; cualquier otro → 422
```

```jsonc
// 422 Unprocessable Entity (formato estándar de validación de Pydantic/FastAPI)
{
  "detail": [
    {
      "loc": ["body", "discount"],
      "msg": "Input should be less than or equal to 0",
      "type": "less_than_equal"
    }
  ]
}
```

**Comportamiento sin cambios**: si el cliente omite `discount` o envía `0` explícitamente, el
comportamiento es idéntico al actual — el total sigue reflejando únicamente el descuento
automático de promociones/combos (`promo_discount`/`combo_discount`, ya sumados en el handler,
sin relación con este campo).

**No afecta**: el campo `discount` compartido de `sales/schemas.py` (usado por
`POST /sales` y `POST /orders/{id}/pay`, mostrador y cierre de mesa unificado/dividido) — ese
campo no se modifica en esta spec (ver research.md D4, alcance de spec 011).

## Campo de respuesta nuevo

### `OrderResponse.paid: bool` (sustenta FR-012/FR-013/FR-014, y la ocultación de "Anular" de FR-007)

Aparece en cualquier respuesta que ya serializa `OrderResponse` — el listado de pedidos de la
Terminal de Mesas y el detalle de un pedido — sin nuevo endpoint ni parámetro de request.

```jsonc
{
  "id": "…",
  "channel": "waiter",
  "status": "abierta",
  "version": 3,
  "items": [ /* … */ ],
  "current_payment_attempt": null,
  "paid": true   // NUEVO — true si existe una Sale con customer_order_id == este id
}
```

**Compatibilidad**: campo aditivo; ningún consumidor existente deja de recibir los campos que ya
tenía.

## Contrato de frontend (sin backend nuevo más allá de lo anterior)

| Elemento | Contrato |
|---|---|
| Botón "Imprimir Factura" (unifica D1) | Reemplaza tanto "🧾 Reimprimir Factura POS" (barra lateral) como el botón de impresión del diálogo de éxito para el caso de un solo comprobante — ambos pasan a invocar `store.printOrderInvoice(orderId)`. El caso de cuenta dividida (`lastReceipts().length > 1`) conserva sus botones "Imprimir todos"/por comensal, sin cambios. |
| Insignia de mesa "Listo" (D2) | `deriveTableStatus` recibe las mismas `orders` de siempre (ahora con el campo `paid` incluido) — sin cambio de firma de la función, solo de su cuerpo. |
| Texto de cabecera del pedido (D2) | Nueva rama de tres estados en `pos-order-panel.component.ts`, derivada de `store.kitchenReady()` (sin cambios) + el nuevo `store.selectedOrder()?.paid`. |
| Botón "Anular" (D3) | Oculto (`@if`) cuando `store.selectedOrder()?.paid === true`; sin cambio en la llamada a `voidPersistedItem`/`voidPersistedCombo` para el caso no pagado. |
| Atajo `F4` y popover de descuento (D4) | Eliminados por completo — `table-sessions.component.ts:206-211`, `pos-order-panel.component.ts:122,151-167`, y las señales/métodos correspondientes de `pos-terminal.store.ts`. |
