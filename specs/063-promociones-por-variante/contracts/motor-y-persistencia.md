# Contrato: motor de evaluación por conjunto y persistencia del resultado

Reemplaza el contrato del motor de las specs 012 (`evaluate` / `evaluate_detailed`) y 040
(`combined_discount_detailed` / `presentation_package_discount_for_lines`). Ver
[data-model.md](../data-model.md) §"Cálculo del descuento" para los invariantes y
[research.md](../research.md) D5/D6/D7/D14 para el porqué.

---

## 1. Entrada del motor — `promo_lines`

`checkout.promo_lines_for(db, lines)` (`app/api/v1/orders/checkout.py:241`) y sus hermanas
(`cart.service._cart_promo_lines`, `sales.service`) producen **un dict por línea del pedido**:

| Clave | Origen | Uso |
|---|---|---|
| `product_variant_id` | `SaleLine.product_variant_id` | pertenencia al conjunto (`promotion_variants`) |
| `unit_price` | `SaleLine.unit_price` (incluye recargos de opciones) | consumo codicioso (FR-008), reparto (FR-008a) |
| `quantity` | `SaleLine.quantity` | conteo de unidades elegibles |
| `line_id` | `order_items.id` / `cart_items.id` / `sale_items.id` | desempate determinista (FR-008), residuo del redondeo (FR-008a) |
| `_variant_active` | `ProductVariant.active` | FR-011 — una variante desactivada no cuenta |
| `combo_id` | histórico; `None` en toda línea nueva | filtro defensivo (una línea histórica de combo reprocesada no cuenta) |
| `description` | `"{product.name} - {variant.name}"` | texto para `applied_promotions` (no lo usa el cálculo) |

**Ya NO se traen** `product_id`, `category_id` ni `presentation_id` (targets y presentación
eliminados, FR-003 / FR-027). En `checkout.promo_lines_for` eso significa quitar la clave
`presentation_id` (`variant.presentation_id`, hoy ~L264); en `cart.service._cart_promo_lines`,
quitar `ProductVariant.presentation_id` del `select` y del dict.

---

## 2. `evaluate_variant_sets(db, promo_lines, now) -> SetDiscountResult`

```python
@dataclass
class AppliedPromotion:
    promotion_id: UUID
    name: str            # snapshot: sobrevive al borrado de la promoción
    amount: Decimal      # descuento agregado de ESTA promoción en ESTE cobro (>= 0)

@dataclass
class SetDiscountResult:
    total: Decimal                       # Σ by_line, redondeado una sola vez ROUND_HALF_UP
    by_line: dict[int, Decimal]          # line_index -> descuento (para el carrito, spec 038)
    applied: list[AppliedPromotion]      # para applied_promotions (FR-021) y promotion_id único
```

### Algoritmo (normativo)

1. `promos = active_variant_set_promotions(db, now)` — `status == "active"`, `ends_at` no
   vencido (índice `ix_promotions_status_ends_at`), `_valid_now(p, now)` verdadero (hora local
   del tenant; `_valid_now` **sin cambio**, conserva A-57), `selectinload(Promotion.variants)`.
2. Por cada `p` con conjunto `S = {v.product_variant_id for v in p.variants}`:
   1. **Unidades elegibles** `U`: por cada dict de `promo_lines` con `product_variant_id ∈ S`
      **y** `_variant_active` verdadero **y** `combo_id is None`, expandir a `quantity` unidades,
      cada una `(line_index, unit_price, product_variant_id, line_id)`.
   2. `n = len(U)`; `grupos = n // p.min_qty`. Si `grupos == 0`, `p` no aplica.
   3. Ordenar `U` por `(-unit_price, product_variant_id, line_id)` → `U_sorted`.
      `en_grupos = U_sorted[: grupos * p.min_qty]`; el resto es remanente (precio normal).
   4. Trocear `en_grupos` en `grupos` bloques consecutivos de `p.min_qty`. Por bloque `g`:
      - `normal_g = Σ unit_price` de `g`.
      - `descuento_g` =
        - `package_price`: `max(Decimal(0), normal_g - Decimal(p.value))`
        - `percent`: `(normal_g * Decimal(p.value) / 100).quantize(Decimal("1"), ROUND_HALF_UP)`
      - `_distribute_group_discount(g, descuento_g)` → `{line_index: descuento}` (ver §3).
      - acumular en `by_line` y en el `amount` de `p`.
3. `total = Σ by_line.values()`, `.quantize(Decimal("0.01"), ROUND_HALF_UP)` **una sola vez**.
   `applied` = lista de `AppliedPromotion` con `amount > 0`, ordenada por `promotion_id` (orden
   estable).

**No hay reconciliación entre promociones**: FR-014 garantiza que ninguna variante está en dos
promociones vigentes al mismo instante. Si por un bug de datos dos promociones vigentes
compartieran una variante, cada una calcularía su descuento sobre "sus" unidades y ambas sumarían
en `by_line` — se documenta como estado imposible-por-diseño, no como caso soportado; el char
test de FR-014 lo previene en la capa de escritura.

### `promotion_id` único (compatibilidad con `Sale.promotion_id`)

`checkout.auto_discount` deriva `promotion_id`:
- `len(applied) == 1` → `applied[0].promotion_id`.
- `len(applied) != 1` → `None` (como hoy con dos promociones; `applied_promotions` lo cubre —
  **A-29 resuelto**).

---

## 3. `_distribute_group_discount(group_units, discount) -> dict[int, Decimal]`

`group_units`: lista de `(line_index, unit_price, product_variant_id, line_id)` de **un** grupo
completo. `discount`: descuento del grupo, **ya redondeado a peso** (FR-006). Reparte repartiendo
el **importe cobrado** (FR-008a):

1. Agrupar `group_units` por `line_index`. Para cada línea `L`:
   `aporte_L = Σ unit_price` de sus unidades en el grupo.
2. `aporte_total = Σ aporte_L` (= `normal_g`). `objetivo = aporte_total - discount`.
3. Para cada `L`: `cobrado_L = floor(aporte_L - discount * aporte_L / aporte_total)`.
4. `falta = objetivo - Σ cobrado_L` (≥ 0, en pesos enteros). Sumar `falta` a `cobrado_L` de la
   línea cuya **variante tiene el `product_variant_id` más alto**; desempate: `line_id` más alto.
5. Descuento de `L` en este grupo = `aporte_L - cobrado_L`.

**Verificación** (SC-005): `Σ (descuento de L) == discount` exactamente, al peso, para cualquier
orden de `group_units`. El caso que lo ejercita de verdad es la división no exacta (Assumptions:
"3 Pequeños sin licor por $16.000" → residuo $1 a la variante más alta).

---

## 4. `variant_set_condition_text(promo) -> str`

Texto en lenguaje llano, español de Colombia, para el menú QR (FR-022), la terminal (FR-023) y el
resumen del formulario (FR-005). A partir de `(type, value, min_qty, len(variants))`:

| Tipo | `min_qty` | Texto |
|---|---|---|
| `package_price` | `> 1` | `"Llevando {min_qty} de estas {N} variantes pagas {formato_pesos(value)}"` |
| `package_price` | `1` | `"Cada una de estas {N} variantes a {formato_pesos(value)}"` |
| `percent` | `1` | `"{value:g}% en estas {N} variantes"` |
| `percent` | `> 1` | `"{value:g}% llevando {min_qty} de estas {N} variantes"` |

`formato_pesos` = `menu/router._money` (`$12.000`, separador de miles con punto).

---

## 5. Persistencia del resultado (FR-021, A-64)

### `applied_promotions` (JSONB, en `sales`, `invoices`, `customer_orders`)

```json
[
  {"promotion_id": "3f2a…", "name": "2X Pequeños con licor $12.000", "amount": "4000.00"},
  {"promotion_id": "9b1c…", "name": "10% en granizados",             "amount": "2300.00"}
]
```

- Una entrada por promoción con `amount > 0` en ese cobro (de `SetDiscountResult.applied`).
- `name` es **snapshot** — si la promoción se borra o se renombra después, la venta pasada sigue
  mostrando el nombre con que se cobró.
- `amount` es el **agregado por promoción**, no por línea. El desglose por línea de venta
  (variante ↔ promoción ↔ monto) está **fuera de alcance** (FR-021).
- `Σ amount == Sale.discount` menos el descuento **manual** del cajero (mostrador / cierre), que
  no viene de ninguna promoción. `Sale.discount = descuento_manual + Σ amount`.

### Flujo

| Camino | Fija `applied_promotions` en | Vía |
|---|---|---|
| `pay_order` (orden de mesa legada) | `Sale`, `Invoice`, `CustomerOrder` | `auto_discount` → `build_sale(applied_promotions=...)`; `order.discount` / `order.applied_promotions` |
| `sales.service` (mostrador, `checkout_and_send`) | `Sale`, `Invoice` (+ `CustomerOrder` si viene de orden) | ídem |
| `table_sessions._close_unified` | `Sale`, `Invoice`, cada `CustomerOrder` de la sesión | ídem, agregando por sesión |
| `table_sessions._close_split` | una `Sale` + `Invoice` por comensal; `CustomerOrder` del comensal | el motor agrupa por el subconjunto de líneas del comensal (research.md D7; mismo criterio de agrupación que hoy para split) |
| `invoices.issue_for_sale` | `Invoice.applied_promotions = sale.applied_promotions` | copia dentro de la transacción del cobro |

### No retroactivo

`applied_promotions` nace `'[]'`, `customer_orders.discount` nace `0`. Ninguna venta ni factura
emitida antes del despliegue se recalcula ni se re-serializa (Principio VII).

---

## 6. Superficie de preview del carrito del comensal (spec 038)

`cart.service.serialize_cart` llama `evaluate_variant_sets` **una vez** sobre las líneas del
carrito y usa:
- `by_line[i]` → `CartItemResponse.discounted_unit_price` / `discounted_line_total` del ítem `i`.
- `total` → `CartResponse.discounted_total` (`total_bruto - result.total`), `None` si
  `result.total == 0` y ninguna línea trae descuento.

`submit_cart` toma el snapshot de descuento por `OrderItem` del **mismo** `by_line` — así el
snapshot coincide exacto con lo que el carrito mostraba
(`test_submit_cart_snapshot_de_descuento_coincide_con_el_carrito`, spec 038, no CONGELA).

`best_line_discount` y `_line_discount` (por ítem, vía `best_line_discount`) **se eliminan** — el
desglose por línea ahora sale del motor completo, no de una llamada por línea (research.md D7).
