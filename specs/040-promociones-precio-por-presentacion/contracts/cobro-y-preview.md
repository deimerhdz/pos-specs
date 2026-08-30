# Contrato: efecto sobre los caminos de cobro y de previsualización

Ninguno de los endpoints de cobro o de cuenta cambia de **forma** — mismo método, ruta, body y
`response_model`, mismos códigos de estado. Lo que cambia es el **cálculo interno del descuento**:
donde hoy se suman "promociones automáticas (percent/fixed/qty_price línea a línea)" +
"ahorro de combos", ahora se suma además el "descuento de paquetes por presentación", reconciliado
por línea con el primero (FR-013). Cubre el Incremento C (research.md D17).

`../pos-heladeria` **no requiere** cambio en estos endpoints (los totales ya vienen calculados del
backend); sí gana el render del carrito del comensal (§4) y del menú QR (contrato aparte).

---

## 1. Función nueva y orquestador

En `../pos-backend/app/api/v1/promotions/service.py`:

```text
presentation_package_discount_for_lines(db, lines, now) -> PresentationDiscountResult
    # total: Decimal   +   por_linea: dict[line_index -> Decimal]
    # Hermana de combo_discount_for_lines (service.py:399). Algoritmo: research.md D5.

combined_discount_detailed(db, promo_lines, now) -> CombinedDiscountResult
    # Orquesta los 3 mecanismos y reconcilia con recálculo del pool hasta punto fijo (research.md D6):
    #   1. evaluate_detailed        -> LineDiscount por línea (percent/fixed/qty_price producto);
    #                                  una sola vez sobre todas las líneas.
    #   2. combo_discount_for_lines -> ahorro de combos (líneas con combo_id; no compiten)
    #   3. presentation_package_discount_for_lines -> descuento por presentación, sobre `pool`
    #   Bucle: una línea sale de `pool` si (1) la deja con total ESTRICTAMENTE menor (empate → se
    #   queda) o si el descuento por presentación la deja peor que sin promoción (FR-023); si alguna
    #   salió, se recalcula (3) sobre el `pool` más chico y se repite. Ninguna línea acumula dos.
    # Devuelve: total_descuento, promotion_id_unico_si_aplica (regla de single_promotion_id).
```

`checkout.promo_lines_for` (`checkout.py:240-256`) agrega a cada dict de línea: `presentation_id`,
`product_variant_id`, `unit_price` y el `id` de la fila de línea — además de los
`product_id`/`category_id`/`quantity`/`line_total` de hoy. `_presentation_reference_unit_price` usa
`unit_price`; el desempate de FR-011 usa `product_variant_id` y el `id` de línea (ver data-model.md
§"Definición operativa de identificador más alto"). `evaluate` y `combo_discount_for_lines` ignoran
los campos nuevos. Los demás call-sites que construyen sus propias listas de línea
(`table_sessions/service.py`, `sales/service.py`, `cart/service.py`) agregan los mismos campos.

## 2. Caminos que pasan a usar `combined_discount_detailed`

Sustituyen el par `promotions.evaluate(...)` + `promotions.combo_discount_for_lines(...)` (y el
cálculo de `final_promotion_id`) por una sola llamada. Verificado con
`grep -rn "promotions.evaluate\|combo_discount_for_lines"`:

| Fichero:línea | Función | Tipo | Nota |
|---|---|---|---|
| `orders/checkout.py:275` | `pay_order` | cobro real | orden de mesa legada |
| `orders/checkout.py:466` | `checkout_and_send` | cobro real | mostrador/mesero cobra antes de cocina |
| `orders/checkout.py:849` | `approve_payment_attempt` | cobro real | transferencia aprobada |
| `orders/checkout.py:969` | `confirm_cash` | cobro real | efectivo confirmado |
| `orders/checkout.py:136` | `compute_bill` (por mesa) | preview | cuenta de mesa |
| `table_sessions/service.py:186` | `compute_bill` (por comensal) | preview | agrupa por comensal (research.md D16) |
| `table_sessions/service.py:656` | `release_paid_session` | cobro real | |
| `table_sessions/service.py:753` | `_close_split` | cobro real | cierre dividido — agrupa por comensal |
| `sales/service.py:279` | venta de mostrador | cobro real | |
| `orders/tables_advanced.py:155` | `group_bill` | preview | mesas fusionadas — agrupa por orden |

**Comportamiento observable**:

```text
Antes: total_descuento = descuento_percent_fixed_qtyprice_por_linea + ahorro_combos
Ahora: total_descuento = combined_discount_detailed(...).total
       (= lo anterior, PERO cada línea que también sería elegible para un paquete por presentación
        recibe el MENOR de los dos descuentos, nunca la suma; y una presentación con >= min_qty
        unidades combinadas de varias líneas genera su descuento de paquete aunque ninguna línea
        sola alcanzara un qty_price de producto)

Sin ninguna promoción qty_price_presentation activa y vigente:
       combined_discount_detailed(...).total == (evaluate + combo_discount_for_lines) de hoy,
       exacto — la modalidad nueva es aditiva-segura (research.md D6, verificado en tests).
```

`Sale.promotion_id` (spec 012 FR-026): se rellena si una única promoción explica todas las líneas
descontadas — la promoción por presentación cuenta como esa "única promoción" cuando corresponde.

## 3. FR-023 — nunca empeora

`combined_discount_detailed` nunca aplica el descuento por presentación a una línea si el total de
esa línea con el descuento sería mayor que sin ninguna promoción — mismo criterio "mejor promoción
por línea" del motor (spec 012), que ya contempla "ninguna promoción" como alternativa. En un
paquete completo esto no debería ocurrir salvo configuración marcada con `confirm_sin_descuento`
(contrato de promociones §4); el motor lo protege igual.

## 4. Carrito del comensal por QR (`cart/service.py`)

- `serialize_cart` (`cart/service.py:269`) y el snapshot de `submit_cart` (`~635`): el
  `discounted_total` del carrito refleja el descuento por presentación cuando las cantidades del
  carrito alcanzan el `min_qty` de alguna regla vigente — llamando `combined_discount_detailed`
  sobre las líneas del carrito. **No se persiste** (FR-014); es preview, se recalcula en cada
  `GET /cart` y al cobrar.
- El precio unitario por variante en el menú (`menu/router.py:157`, `best_line_discount`) **no**
  cambia: el descuento por presentación depende de la combinación de unidades del pedido, que en el
  menú no existe (mismo motivo por el que `qty_price` no baja precio en el menú,
  `menu/router.py:152-153`). El menú lo comunica por el **anuncio** (contrato `menu-qr-anuncio.md`),
  no por `discounted_price`.

## 5. Determinismo (SC-005, CA-5)

El reparto por línea de `presentation_package_discount_for_lines` no depende del orden de las líneas
del pedido: las unidades sobrantes y el residuo del redondeo se asignan por identificador de
variante más alto (desempate: identificador de línea más alto). Dos cobros del mismo pedido con las
líneas en distinto orden devuelven el mismo total y el mismo desglose por línea.
