# Contrato: motor de evaluación por regla y persistencia del resultado

> **Reemplaza** la versión de este contrato escrita para el modelo plano (una promoción = una
> combinación). Esa versión describe con exactitud lo que **ya está implementado** en la rama de
> feature `refactor/063-promociones-por-variante` de `pos-backend`
> (`app/api/v1/promotions/service.py`, verificado 2026-09-01: `evaluate_variant_sets` en la línea
> 211, `_greedy_units` en 164, `_distribute_group_discount` en 175). Este documento describe **solo
> el delta**: el motor pasa de agrupar por **promoción vigente** a agrupar por **regla cuya
> promoción está vigente**. Ver [data-model.md](../data-model.md) para `PromotionRule` y
> [research.md](../research.md) D-R1/D-R6 para el porqué.

---

## 1. Entrada del motor — `promo_lines`

**Sin cambio.** `checkout.promo_lines_for` (`checkout.py:241`) y sus hermanas
(`cart.service._cart_promo_lines`, `sales.service`) siguen produciendo un dict por línea del
pedido con `product_variant_id`, `unit_price`, `quantity`, `line_id`, `_variant_active`, `combo_id`
(histórico), `description`. Nada de esto depende de si el conjunto vive en una promoción o en una
regla — el motor sigue cruzando por `product_variant_id`.

---

## 2. `active_variant_set_promotions` → `active_variant_set_rules(db, now) -> list[PromotionRule]`

**Cambia de nombre y de qué carga.** Antes: `select(Promotion).where(status=='active', ends_at
válido, _valid_now).options(selectinload(Promotion.variants))`. Ahora:

```python
select(PromotionRule).join(PromotionRule.promotion).where(
    Promotion.status == "active",
    # ends_at / _valid_now sin cambio de criterio — se evalúan sobre Promotion, no sobre la regla
).options(
    selectinload(PromotionRule.variants),
    contains_eager(PromotionRule.promotion),   # ya viene del join, evita un segundo round-trip
)
```

`_valid_now(promotion, now)` **no cambia de cuerpo** — sigue evaluando `days_of_week`/
`start_time`/`end_time`/`ends_at` de la `Promotion` (A-57 intacto); la única diferencia es que
ahora se llama pasando `rule.promotion` en vez de la promoción directamente.

---

## 3. `evaluate_variant_sets(db, promo_lines, now) -> SetDiscountResult`

```python
@dataclass
class AppliedPromotion:
    promotion_id: UUID
    rule_id: UUID         # NUEVO — qué regla (qué tramo de precio/tipo) generó este monto
    name: str             # snapshot del nombre de la PROMOCIÓN (sin cambio)
    amount: Decimal        # descuento agregado de ESTA regla en ESTE cobro (>= 0)

@dataclass
class SetDiscountResult:
    total: Decimal                       # Σ by_line, redondeado una sola vez ROUND_HALF_UP
    by_line: dict[int, Decimal]          # line_index -> descuento (para el carrito, spec 038)
    applied: list[AppliedPromotion]      # para applied_promotions (FR-021) y promotion_id único
```

### Algoritmo (normativo) — mismo esqueleto, unidad de iteración cambia de promoción a regla

1. `rules = active_variant_set_rules(db, now)`.
2. Por cada `rule` con conjunto `S = {v.product_variant_id for v in rule.variants}`:
   1. **Unidades elegibles `U`**: sin cambio de criterio (mismo filtro `product_variant_id ∈ S`,
      `_variant_active`, `combo_id is None`), ahora sobre el conjunto de **esta regla**.
   2. `n = len(U)`; `grupos = n // rule.min_qty`. Si `grupos == 0`, la regla no aplica.
   3. Ordenar y trocear en grupos — **sin cambio de fórmula** (FR-008): `(-unit_price,
      product_variant_id, line_id)`.
   4. Por bloque `g`: `descuento_g` según `rule.type`/`rule.value` — **misma fórmula que antes con
      `promo.type`/`promo.value`** (FR-006), solo que ahora lee de la regla.
      `_distribute_group_discount(g, descuento_g)` — **sin cambio de cuerpo** (§4).
      Acumular en `by_line` y en el `amount` de un `AppliedPromotion(promotion_id=rule.promotion_id,
      rule_id=rule.id, name=rule.promotion.name, amount=...)`.
3. `total`, redondeo — sin cambio. `applied` ordenada por `(promotion_id, rule_id)` (orden estable).

**Invariante que sigue garantizando "sin reconciliación entre reglas"**: FR-014 (entre reglas de
promociones distintas) + FR-001a (entre reglas de la misma promoción) garantizan juntos que
**ninguna variante está en dos reglas vigentes al mismo instante**, sin importar si esas reglas
pertenecen a la misma promoción o a promociones distintas. El motor sigue sin necesitar ninguna
reconciliación por prioridad — exactamente la misma garantía que ya sostenía el modelo plano,
ahora extendida por FR-001a a un caso que antes no podía ocurrir (una promoción no podía competir
consigo misma porque solo tenía una combinación).

### `promotion_id` único (compatibilidad con `Sale.promotion_id`)

**Sin cambio de criterio, redefinido sobre `applied`**: `len({a.promotion_id for a in applied}) ==
1` → esa promoción; si no, `None`. Nota: dos entradas de `applied` con el **mismo** `promotion_id`
pero distinto `rule_id` (dos reglas de la misma promoción, ambas con descuento en este cobro)
**siguen contando como una sola promoción** para este criterio de compatibilidad — es exactamente
el caso que motivó la partición (Springfield: varias reglas, una promoción), y `Sale.promotion_id`
ya era un campo de compatibilidad de mejor esfuerzo antes de esto (A-29, resuelto de fondo por
`applied_promotions`, no por este campo).

---

## 4. `_distribute_group_discount(group_units, discount) -> dict[int, Decimal]`

**Sin cambio de cuerpo.** Reparte por importe cobrado (FR-008a) sobre un grupo que ya pertenece a
una sola regla — la función nunca necesitó saber a qué promoción o regla pertenece el grupo que
recibe, solo las unidades y el descuento ya calculado.

---

## 5. `variant_set_condition_text(rule) -> str`

**Cambia el parámetro** de `promo` (una `Promotion`) a `rule` (una `PromotionRule`). La tabla de
formato (`package_price`/`percent` × `min_qty` = 1 / > 1) **no cambia** — sigue leyendo
`(type, value, min_qty, len(variants))`, ahora de la regla en vez de la promoción.

`menu_unit_discount(rules, variant_id, price)`: mismo cambio de parámetro — itera `rules` (lista de
`PromotionRule`) en vez de `promos`.

---

## 6. Persistencia del resultado (FR-021)

### `applied_promotions` (JSONB, en `sales`, `invoices`, `customer_orders`) — gana `rule_id`

```json
[
  {"promotion_id": "3f2a…", "rule_id": "a1b2…", "name": "2X entre semana", "amount": "4000.00"},
  {"promotion_id": "3f2a…", "rule_id": "c3d4…", "name": "2X entre semana", "amount": "5000.00"},
  {"promotion_id": "9b1c…", "rule_id": "e5f6…", "name": "10% en granizados", "amount": "2300.00"}
]
```

- Una entrada por **regla** con `amount > 0` en ese cobro (antes: una entrada por promoción). Dos
  entradas pueden compartir `promotion_id` (dos reglas de la misma promoción activas en el mismo
  cobro — el caso Springfield: "2X entre semana" con la regla de Pequeños y la de Medianos ambas
  aplicando en el mismo pedido).
- `name` sigue siendo el nombre de la **promoción** (snapshot) — a nivel de regla no hay nombre
  propio en el modelo (`spec.md` Key Entities: la regla no tiene `nombre`), así que dos entradas de
  la misma promoción comparten `name`; se distinguen por `rule_id`.
- `amount` sigue siendo el agregado por (ahora) regla, no por línea — el desglose por línea de
  venta sigue fuera de alcance (FR-021, sin cambio).
- `Σ amount == Sale.discount − descuento_manual`, sin cambio de fórmula.

### Compatibilidad con entradas ya escritas (Incremento G/H en adelante)

Las entradas de `applied_promotions` escritas por el modelo plano (antes de este refactor) no
tienen `rule_id`. **No se les hace backfill** (Principio VII, "el registro NO es retroactivo",
FR-021). Cualquier lector nuevo (p. ej. una futura pantalla de arqueo que agrupe por regla) DEBE
tolerar `rule_id` ausente en entradas históricas y no asumir que siempre está presente.

### Flujo

**Sin cambio de tabla de caminos** (`pay_order`, `sales.service`, `table_sessions._close_unified`,
`table_sessions._close_split`, `invoices.issue_for_sale`) — todos siguen recibiendo
`SetDiscountResult.applied` desde `evaluate_variant_sets` a través de `checkout.auto_discount`, cuya
firma externa `(total, promotion_id, applied) -> ...` **no cambia**; solo el contenido de cada
`AppliedPromotion` gana `rule_id`.

---

## 7. Superficie de preview del carrito del comensal (spec 038)

**Sin cambio de contrato.** `cart.service.serialize_cart` sigue llamando `evaluate_variant_sets`
una vez sobre las líneas del carrito; `by_line`/`total` tienen la misma forma. El snapshot que
`submit_cart` guarda en `OrderItem` sigue viniendo del mismo `by_line`
(`test_submit_cart_snapshot_de_descuento_coincide_con_el_carrito`, spec 038, no CONGELA, sin
cambio).
