# Contrato: impacto del valor del domicilio en el total facturado (checkout)

Cubre FR-011, FR-012, FR-013. No agrega ni cambia la forma (request/response) de ningún endpoint
de checkout — cambia únicamente el **valor** de `total` que cada uno produce cuando la orden
facturada es de tipo `DELIVERY`. Ver research.md Decisión 5 para el porqué de tocar los tres
puntos de cálculo listados abajo, no solo `build_sale`.

## Endpoints afectados (sin cambio de forma, solo de valor cuando `order_type == DELIVERY`)

| Endpoint | Función (`checkout.py`) |
|---|---|
| `POST /orders/{order_id}/pay` | `pay_order` |
| `POST /orders/{order_id}/checkout-and-send` | `checkout_and_send` |
| `POST /orders/payment-attempts/{attempt_id}/approve` | `approve_payment_attempt` |
| `POST /orders/payment-attempts/{attempt_id}/confirm-cash` | `confirm_cash_payment_attempt` |

## Regla

Para toda orden con `order_type == "DELIVERY"`, el `total` de la `Sale` generada por cualquiera de
los cuatro endpoints anteriores es:

```text
total = subtotal - discount + tax + tip + delivery_fee
```

donde `delivery_fee` es el valor guardado en `CustomerOrder.delivery_fee` al crear la orden
(contracts/orders-create.md). Para cualquier orden que no sea `DELIVERY`, `delivery_fee` es `0` y
el `total` no cambia respecto al comportamiento actual (FR-012).

- **Ejemplo — `POST /orders/{order_id}/checkout-and-send` sobre una orden `DELIVERY`**: orden con
  `subtotal = 25000`, `delivery_fee = 6000`, sin descuento/impuesto/propina → `Sale.total =
  31000.00`. La `Sale` resultante también persiste `delivery_fee = 6000.00` (data-model.md).
- **Ejemplo — `POST /orders/payment-attempts/{attempt_id}/approve` sobre una orden `DELIVERY`**:
  el pago autogenerado que este endpoint construye internamente para pasarlo a `build_sale` debe
  cubrir `subtotal + delivery_fee` (menos descuentos), no solo `subtotal` — de lo contrario
  `build_sale` rechaza el pago con `422` por insuficiente (research.md Decisión 5, punto de mayor
  riesgo encontrado).
- **Ejemplo — `POST /orders/payment-attempts/{attempt_id}/confirm-cash` sobre una orden
  `DELIVERY`**: el chequeo previo "`amount_received >= total`" debe considerar `delivery_fee`
  dentro de ese `total` — un cajero que reciba efectivo solo por el valor de los productos, sin el
  domicilio, debe ver el mismo rechazo que vería si el efectivo no cubriera el subtotal de
  productos.

## Garantía de no regresión (FR-013)

Ninguna `Sale` ya emitida antes de esta mejora se recalcula. La fórmula extendida solo se ejecuta
para ventas nuevas generadas a partir de esta mejora, y únicamente agrega un término
(`delivery_fee`) que vale `0`/`NULL` para toda orden que no sea `DELIVERY` — el valor de `total`
para pedidos "En Mesa"/"Para Llevar" (nuevos o históricos) queda idéntico al que produciría el
sistema sin esta mejora.
