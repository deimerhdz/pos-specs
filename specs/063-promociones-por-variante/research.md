# Research: Refactorización del módulo de promociones — modelo por conjunto explícito de variantes

Fase 0 de `/speckit-plan`. `spec.md` no dejó ningún `[NEEDS CLARIFICATION]` (checklist
`requirements.md` al 100%, sesión de clarificación del 2026-08-31 con 24 preguntas resueltas y
8 cambios de comportamiento ya listados). Este documento no resuelve ambigüedades del contrato
funcional: investiga el código real de `../pos-backend` y `../pos-heladeria` para decidir **cómo**
modelar el conjunto de variantes, **cómo** rehacer el motor, **cómo** borrar `Presentation` /
`combo` / `priority` sin romper histórico, y **cómo** persistir el resultado, dejando registro de
qué alternativas se descartaron y por qué.

Numeración de línea: estado del código al 2026-08-31 (rama `develop` de `pos-backend`, tras el
merge de la spec 040 — PR #47).

---

## Decisión 1 — El conjunto de variantes es una tabla puente sin atributos de precio (`promotion_variants`)

**Decisión**: tabla nueva `promotion_variants` (schema `tenant`): `id`, `promotion_id`
(FK → `promotions.id` `ondelete="CASCADE"`, indexado), `product_variant_id`
(FK → `product_variants.id` `ondelete="CASCADE"`), `UniqueConstraint("promotion_id",
"product_variant_id")`. `Promotion` gana `variants: list[PromotionVariant]` con
`cascade="all, delete-orphan"` (igual que hoy `targets` / `combo_items`). **La fila no lleva
`type`, `value` ni `min_qty`** — solo expresa pertenencia al conjunto elegible.

**Por qué**:
- La clarification del 2026-08-31 es explícita: el diagrama original proponía una regla por
  variante con `type` / `value` / `minQuantity` propios; se rechazó. "La regla solo expresa
  pertenencia al conjunto elegible (`promotionId`, `productVariantId`). `type`, `value` y
  `minQuantity` viven en la promoción, una sola combinación por promoción. Es lo único que
  permite que un paquete combine variantes distintas del conjunto" (FR-001, FR-002).
- `PromotionComboItem` (`app/models/promotion.py:192-217`) ya es el precedente estructural exacto
  de una tabla hija de `promotions` con `UniqueConstraint(promotion_id, product_variant_id)` y
  `ondelete="CASCADE"`. `promotion_variants` es `PromotionComboItem` **sin** la columna
  `quantity` — más simple, no más compleja.
- `ondelete="CASCADE"` en `product_variant_id`: FR-011 exige que "una variante eliminada sale del
  conjunto de toda promoción que la incluyera". `CASCADE` lo hace en la base sin código. (Una
  variante **desactivada** —no borrada— sí se filtra en el motor, D5, sin tocar la fila.)

**Alternativas consideradas**:
- *Ampliar `PromotionTarget` con `product_variant_id`*: rechazada — `PromotionTarget` tiene
  `product_id` XOR `category_id`, dos índices únicos parciales, `TargetIn._one_scope`
  (`schemas.py:79-85`) y `value`/`min_qty` de precio por target. El modelo nuevo **borra**
  `PromotionTarget` entero (FR-003, no hay alcance por producto ni categoría); reusarlo sería
  arrastrar el XOR y el precio-por-target que la spec elimina.
- *Regla por variante con `type`/`value`/`min_qty` (el diagrama original)*: rechazada por la
  clarification citada — rompe el caso central (un paquete que combina variantes distintas del
  conjunto necesita **una sola** combinación (tipo, valor, cantidad) para el grupo entero).
- *Un array `product_variant_id[]` en `promotions`*: rechazada — sin FK no hay `ON DELETE
  CASCADE` (FR-011), y el `SELECT` del motor y la validación de solape (FR-014) quieren un `JOIN`
  indexado, no un `= ANY(array)`.

---

## Decisión 2 — Dos tipos: `percent` (se conserva) y `package_price` (valor nuevo), no reusar `qty_price`

**Decisión**: el tipo de precio de paquete usa el string **`package_price`** (13 caracteres, cabe
en el `varchar(50)` actual sin migrar el ancho); `percent` **conserva su semántica y su string**.
El `PromotionType` de **entrada** (`schemas.py`, `promotion.interface.ts`) queda en
`{PERCENT, PACKAGE_PRICE}` — ninguna promoción nueva nace fuera de eso. Pero `PROMOTION_TYPES` del
modelo y `ck_promotion_type` de la BD **se quedan con los valores viejos + `package_price`** (revisión
`063a` amplía; `063b` estrecha `ck_promotion_type` solo **con escape** `OR status='finished'`),
porque las promociones que la migración pasó a `finished` conservan su `type` original como registro
histórico (aviso de FR-025) y `PromotionResponse.type` debe poder serializarlas — es `str` libre en
la respuesta. Ver [data-model.md](./data-model.md) §"Migración y rollback".

**Por qué**:
- FR-002 retira explícitamente los identificadores `buy_x_get_y`, `qty_price`,
  `qty_price_presentation`, `combo` y `fixed`. Reusar el string `qty_price` para "precio de
  paquete" contradice el FR y, sobre todo, haría **ambigua la migración**: hay que distinguir las
  `qty_price` viejas (que van a `Finalizada`, FR-025) de las nuevas — mismo problema que la
  spec 040 D4 evitó creando un `type` explícito en vez de un sub-modo.
- `percent` **no cambia de forma** salvo que su alcance pasa de targets a conjunto de variantes;
  conservar el string minimiza el diff de la migración de datos (FR-026: `percent` se migra
  automáticamente) y de los tests.
- `package_price` describe lo que es (precio total de `min_qty` unidades, FR-006) sin colisionar
  con la palabra "presentación" (que se elimina) ni con `qty_price` (retirado).

**Alternativa considerada**: `type = "qty_price"` reusado — rechazada (ambigüedad de migración,
FR-002).
**Alternativa considerada**: `bundle_price` — equivalente; se elige `package_price` porque
`spec.md` habla de "precio de paquete" y "paquete" en todo el texto.

---

## Decisión 3 — Se elimina `priority` por completo; no se reemplaza por nada

**Decisión**: se borra `promotions.priority` (columna), su `server_default`, el criterio de
desempate en el motor, el `order_by(Promotion.priority.desc())` de `list_query`
(`service.py:790`), el campo en todos los schemas y el input del formulario. `list_query` ordena
por `Promotion.name`.

**Por qué**:
- Hoy `priority` solo existe para resolver el conflicto cuando dos promociones tocan la misma
  línea (`_best_line_match`, `service.py:291`; `best_line_discount` docstring). Con el **bloqueo
  de solape real** de FR-014 (variante compartida + ventanas que se intersectan), dos promociones
  **nunca** pueden aplicar a la misma línea en el mismo instante — la clarification del 2026-08-31
  lo dice literalmente: "Con el bloqueo de solapamiento real de FR-014, dos promociones nunca
  pueden aplicar a la misma línea en el mismo instante, así que no queda ningún conflicto que
  desempatar."
- Reglas de negocio afectadas: RN-PROMO-13, RN-PROMO-14 (desempate por prioridad), RN-PROMO-43
  (listado ordenado por prioridad) — todas INTENCIONAL en `reglas-de-negocio.md`; su derogación
  es el cambio de comportamiento **A-58** (D18).

**Alternativa considerada**: conservar `priority` "por si acaso" — rechazada: un campo que no
gobierna nada es deuda y ruido en el formulario; si el negocio quisiera acumulación o desempate
en el futuro sería otra spec (Principio V).

---

## Decisión 4 — El solape pasa de advertencia a bloqueo, con el criterio acotado de FR-014/FR-014a

**Decisión**: función nueva `_guard_variant_overlap(db, promo, variant_ids)` en
`promotions/service.py`, invocada desde `create`, `update_shape` **y** `change_status` cuando el
destino es `active`. Rechaza con **409** si existe otra promoción en estado
`draft`/`active`/`paused` cuyo conjunto comparta ≥1 `product_variant_id` **y** cuyas tres
dimensiones de vigencia se intersecten con las de `promo`:

1. **Fecha**: `[starts_at, ends_at]` se intersecta con `[c.starts_at, c.ends_at]`. `ends_at`
   ausente = indefinido (cubre hasta el infinito). `starts_at` siempre está (obligatoria,
   FR-012).
2. **Días**: `days_of_week` (conjunto) se intersecta con el de `c`. Vacío/`NULL` en cualquiera =
   los siete días (se intersecta con todo).
3. **Horas**: `[start_time, end_time]` (con cruce de medianoche) se intersecta con la de `c`.
   `NULL` en cualquiera = 00:00–24:00 (se intersecta con todo).

El bloqueo solo se produce cuando **las tres** se intersectan a la vez y hay ≥1 variante
compartida (FR-014a). El detalle del 409 nombra la(s) promoción(es) en conflicto y la(s)
variante(s) compartida(s).

**Por qué**:
- `find_overlaps` (`service.py:750-773`) hoy compara ventanas y devuelve **advertencia**
  (`PromotionWithOverlaps.overlaps`), con la nota "un bloqueo duro haría imposibles los propios
  casos de uso del RF". Ese razonamiento dependía de `priority`; sin `priority` (D3) el
  solapamiento real es exactamente lo que hay que impedir. RN-PROMO-30 (solape = advertencia)
  queda derogada — cambio de comportamiento **A-59** (D18).
- Se reutilizan las primitivas de intersección que `find_overlaps` ya tiene
  (`_ranges_overlap`, `_csv_overlap`, `_times_overlap`, `service.py:708-727`) — su lógica de
  "dimensión nula = se solapa con todo" ya coincide con FR-014a. Lo que cambia es (a) el alcance
  se compara por **variante compartida** (`promotion_variants` ∩), no por target/categoría; (b)
  el resultado **bloquea** en vez de advertir; (c) exige intersección **simultánea** en las tres
  dimensiones, no en cualquiera.
- `change_status → active` también: mismo patrón que hoy revalida `>= 2` componentes de combo al
  activar (`service.py:1108-1112`) y que la spec 040 usó para el solape de presentación
  (`service.py:1113-1123`). Una promo creada en `draft` sin conflicto puede chocar cuando otra
  ocupa la variante antes de que se active.

**Alternativa considerada**: bloquear ante cualquier variante compartida sin mirar ventanas —
rechazada: FR-014 es explícito ("**si además** sus ventanas de fecha y día y hora se
intersectan"); "10% en granizados de 08:00 a 15:00" y "15% en granizados de 15:00 a 22:00" deben
poder coexistir (US3-CA3).

---

## Decisión 5 — Motor único `evaluate_variant_sets`: consumo codicioso descendente, grupos completos, reparto por importe cobrado

**Decisión**: función nueva en `promotions/service.py` que **reemplaza** `evaluate`,
`evaluate_detailed`, `combined_discount_detailed`, `combo_discount_for_lines`,
`presentation_package_discount_for_lines` y toda la reconciliación por línea de spec 040.

```python
@dataclass
class AppliedPromotion:
    promotion_id: UUID
    name: str
    amount: Decimal          # descuento agregado de esta promoción en este cobro

@dataclass
class SetDiscountResult:
    total: Decimal = Decimal(0)
    by_line: dict = field(default_factory=dict)      # line_index -> descuento (Decimal)
    applied: list[AppliedPromotion] = field(default_factory=list)

def evaluate_variant_sets(db: Session, promo_lines: list, now: datetime) -> SetDiscountResult
```

Algoritmo (traducción directa de FR-006 a FR-009):

1. `active_variant_set_promotions(db, now)` — `status == "active"`, `ends_at` no vencido,
   `_valid_now(p, now)` verdadero (hora local del tenant, `_valid_now` **sin cambio**, conserva
   A-57), `selectinload(Promotion.variants)`.
2. Para cada promoción `p` (tipo `percent` o `package_price`), con su conjunto
   `S = {pv.product_variant_id}`:
   a. **Unidades elegibles**: de `promo_lines`, toda unidad cuya `product_variant_id ∈ S` **y**
      `_variant_active` (FR-011: una variante desactivada no cuenta) **y** `combo_id is None`
      (defensivo — los combos ya no se crean, pero una línea histórica reprocesada no debe
      contar). Cada unidad lleva `(line_index, unit_price, product_variant_id, line_id)`.
   b. `total_unidades = Σ`; `grupos = total_unidades // p.min_qty` (FR-007: solo grupos
      completos). Si `grupos == 0`: `p` no aplica (FR de "no alcanza el mínimo").
   c. **Consumo codicioso** (`_greedy_units`): ordenar las unidades por `unit_price`
      **descendente**, desempate `product_variant_id` ascendente y luego `line_id` ascendente
      (FR-008). Las primeras `grupos * p.min_qty` unidades entran a los grupos (en bloques
      consecutivos de `min_qty`); el resto es **remanente a precio normal**.
   d. Para cada grupo `g` (bloque de `min_qty` unidades): `normal_g = Σ unit_price` de sus
      unidades.
      - `package_price`: `descuento_g = max(Decimal(0), normal_g - Decimal(p.value))`
        (FR-006, FR-009: si el precio de paquete iguala o supera el normal del grupo, 0).
      - `percent`: `descuento_g = (normal_g * Decimal(p.value) / 100).quantize(Decimal("1"),
        ROUND_HALF_UP)` (FR-006: redondeado a peso).
   e. **Reparto** (`_distribute_group_discount`, FR-008a): repartir `descuento_g` entre las
      líneas que aportan unidades al grupo, repartiendo el **importe cobrado**. Para cada línea
      contribuyente `L`: `aporte_L = Σ unit_price` de sus unidades en `g`;
      `aporte_total = normal_g`. `cobrado_L = floor(aporte_L - descuento_g * aporte_L /
      aporte_total)`. Los pesos que falten para llegar a `normal_g - descuento_g` se suman al
      `cobrado_L` de la línea cuya **variante tiene el id más alto** (desempate: `line_id` más
      alto). `descuento_de_L_en_g = aporte_L - cobrado_L`.
   f. `by_line[line_index] += Σ_g descuento_de_esa_línea_en_g`. `applied` gana/actualiza la
      entrada de `p` con `amount += Σ by_line de esta promo`.
3. `result.total = Σ by_line`, redondeado una sola vez, `ROUND_HALF_UP`
   (`Decimal("0.01")`, mismo patrón que el motor actual, `service.py:354`).

**Por qué esta forma**:
- Es la generalización de `presentation_package_discount_for_lines` (`service.py:576-606`) que
  spec 040 ya construyó y probó: agrupa líneas por una clave (allá `presentation_id`, ahora
  "pertenencia a `S`"), cuenta grupos completos, usa el precio de cada unidad, devuelve desglose
  por línea. Las diferencias respecto de spec 040: (a) la clave de agrupación es el conjunto
  explícito de la promoción, no `presentation_id` de la variante; (b) el precio de referencia
  **no es único** por grupo — el consumo codicioso trabaja con el precio real de cada unidad
  (FR-008, la spec 040 usaba el menor precio de la presentación); (c) `percent` entra al **mismo**
  camino de agrupación (spec 040 lo dejaba en el motor línea-por-línea).
- **No hay reconciliación**: FR-014 garantiza que una variante nunca está en dos promociones
  vigentes a la vez, así que cada línea recibe el descuento de **una sola** promoción sin
  competir. Se elimina el bucle de punto fijo de spec 040 D6 (`combined_discount_detailed`,
  `service.py:632-703`) y `_best_line_match` entero.
- Determinismo (SC-005, CA "otro orden → idéntico"): el orden de consumo es un orden total con
  desempate fijo (`unit_price` desc, `product_variant_id` asc, `line_id` asc) que **no usa la
  posición de la línea**; el residuo del redondeo va siempre a la variante de id más alto. Igual
  criterio que spec 040 (`_unit_sort_key`, `service.py:522-529`).

**Alternativa considerada**: mantener `combined_discount_detailed` orquestando y solo cambiar
`presentation_package_discount_for_lines` — rechazada: sin combos ni targets ni presentación, los
tres mecanismos colapsan en uno; conservar la orquestación sería mantener andamiaje muerto
(Principio V).

---

## Decisión 6 — Reparto del descuento de porcentaje y residuo de redondeo (FR-008a), verificado contra los ejemplos numéricos de la spec

**Decisión**: `_distribute_group_discount(group_units, discount)` implementa FR-008a **para ambos
tipos**. `group_units` es la lista de `(line_index, unit_price, product_variant_id, line_id)` de
un grupo; `discount` ya viene redondeado a peso (FR-006). Devuelve `{line_index: descuento}`.

Pasos, contrastados con `spec.md` §"Ejemplos numéricos que cuadran exacto":

| Ejemplo (spec.md) | Grupo | `discount` | Reparto |
|---|---|---|---|
| **"15% llevando 3 medianos"**, 2×$11.000 + 2×$8.000 | 3 más caras: 2×$11.000 + 1×$8.000 = $30.000 | 15% → $4.500 | línea "con licor" (aporte $22.000): `floor(22000 − 4500×22000/30000)` = `floor(18.700)` = $18.700 → descuento $3.300. línea "sin licor" (aporte $8.000, 1 u en grupo): `floor(8000 − 4500×8000/30000)` = $6.800 → descuento $1.200. Σ = $4.500, sin residuo. ✔ (US2-CA5) |
| **"3 Pequeños sin licor por $16.000"**, 3×$6.000 | los 3, $18.000 | `18000 − 16000` = $2.000 | por unidad `floor(6000 − 2000×6000/18000)` = `floor(5333.33)` = $5.333; falta `16000 − 3×5333` = **$1** → a la unidad de `product_variant_id` más alto (se cobra $5.334). Σ descuentos = $2.000, al peso. ✔ (Assumptions) |
| **"2X Pequeños con licor $12.000"**, 2×$8.000 | $16.000 | $4.000 | `floor(8000 − 4000×8000/16000)` = $6.000 por línea → descuento $2.000 c/u. Σ = $4.000. ✔ (SC-008) |
| **Remanente**, 3×$8.000, min_qty 2 | grupo: 2×$8.000 = $16.000; remanente: 1×$8.000 | grupo $4.000 | grupo: $6.000 por unidad. Remanente a $8.000 normal. Total $20.000. La unidad suelta sale de la variante de id más alto (determinista). ✔ (Edge Cases) |
| **Conjunto mal armado**, 1×$8.000 + 1×$6.000 + 1×$8.000, min_qty 2, paquete $12.000 | codicioso: 2×$8.000 = $16.000 | $4.000 | el $6.000 queda suelto a normal. Total $12.000 + $6.000 = **$18.000** (no $20.000). ✔ (Assumptions) |

**Por qué el residuo a "la variante de id más alto"**: coherente con FR-008 (esa variante es la
última en entrar al grupo por el orden de consumo) y determinista sin depender del orden de las
líneas — mismo criterio que la spec 040 ya usa (`_rule_discount_by_line`, `service.py:558-565`) y
que `combo_discount_for_lines` usa con `min(...)` (`service.py:454-456`). SC-005 se verifica en el
char test y en el script de CI con el caso de división no exacta.

**Alternativa considerada**: repartir el descuento (no el importe cobrado) proporcionalmente y
sumar el residuo a la primera línea — rechazada: depende del orden de las líneas del pedido
(rompe SC-005 y "otro orden → idéntico").

---

## Decisión 7 — Punto de enganche: `checkout.auto_discount` y los ~9 call-sites, sin migrar el resto de `evaluate`

**Decisión**: `checkout.auto_discount(db, lines, now)` cambia su cuerpo para llamar
`promotions.evaluate_variant_sets(db, promo_lines_for(db, lines), now)` y devolver
`(total, promotion_id, applied)` donde `applied: list[dict]` es el snapshot para
`applied_promotions` (FR-021). `promo_lines_for` (`checkout.py:241-267`) **deja de traer**
`product_id` / `category_id` (targets eliminados) y **agrega** la descripción de la línea (para el
snapshot). Los call-sites que hoy llaman `auto_discount` / `combined_discount_detailed` se
actualizan al retorno de tres valores.

**Call-sites** (verificados con `grep -rn "auto_discount\|combined_discount_detailed"`):

| # | Fichero:línea | Camino | Real / preview |
|---|---|---|---|
| 1 | `orders/checkout.py:296` | `pay_order` (orden de mesa legada) | real |
| 2 | `sales/service.py:272` | venta de mostrador + `checkout_and_send` | real |
| 3 | `table_sessions/service.py:187` | `compute_bill` (preview por comensal / split) | preview |
| 4 | `table_sessions/service.py:666` | `release_paid_session` / `_close_unified` | real |
| 5 | `table_sessions/service.py:762` | `_close_split` | real |
| 6 | `orders/tables_advanced.py:154` | `group_bill` (mesas fusionadas) | preview |
| 7 | `orders/checkout.py:136` | `compute_bill` (preview de cuenta por mesa) | preview |
| 8 | `cart/service.py:333` | `serialize_cart` (`GET /cart` del comensal) | preview |
| 9 | `cart/service.py` (`submit_cart` snapshot, ~635) | snapshot de descuento del pedido (spec 038) | preview |
| 10 | `menu/router.py:159` | precio con descuento por variante en el menú público | preview |

Los call-sites 1-7 toman `total` + `promotion_id` + `applied` de `evaluate_variant_sets`. El
**8-9** (carrito del comensal) usa el `by_line` para pintar `discounted_line_total` por ítem y el
`total` para `discounted_total`, y `submit_cart` snapshotea del **mismo** desglose (spec 038
sigue cuadrando: `test_submit_cart_snapshot_de_descuento_coincide_con_el_carrito`). El **10**
(menú, sin carrito) usa `menu_unit_discount` (D9). Los caminos reales (1, 2, 4, 5) además
persisten `applied_promotions` en `Sale`, `Invoice` y `CustomerOrder` (D14).

**Por qué no un helper "combinado" nuevo**: hoy `auto_discount` ya es ese helper de un solo punto
(`checkout.py:270-277`), introducido por la spec 040. Solo cambia a qué función interna delega.
`sales/service.py` y `cart/service.py` llaman a `combined_discount_detailed` directamente
(`sales/service.py:272`, `cart/service.py:333`); se los redirige a `evaluate_variant_sets` o a
`auto_discount` según convenga. No se toca el resto de `evaluate` legacy fuera de promociones
(deuda de spec 012, Principio V).

**Alternativa considerada**: rehacer los caminos de cobro para que el descuento sea un `LineDiscount`
persistido por `sale_item` — rechazada: el desglose por línea de venta está **fuera de alcance**
(FR-021, spec.md §Assumptions); es una spec aparte.

---

## Decisión 8 — Se retira el mecanismo de combo del carrito, la orden y el cobro; las columnas históricas no se tocan

**Decisión**: se borran del código `promotions.expand_combo`, `get_active_combo`,
`combo_discount_for_lines`, `ComboComponent`; la rama `data.combo_id` de `cart.service.add_item`
y `_add_combo` (`cart/service.py:355-411`); la rama de combo de
`orders.consolidation.add_item_to_table`; el campo `combo_id` de `CartItemIn` /
`AddItemToTableIn`. **Se conservan** las columnas `cart_items.combo_id`, `order_items.combo_id`,
`sale_items.combo_id` (nullable, FK `SET NULL`, dejan de escribirse) y **toda línea de venta
histórica marcada con un combo** (FR-024, Principio VII: borrar la columna alteraría la
representación histórica — Principio VIII). El motor filtra `combo_id is None` de forma defensiva
(D5.2a).

**Por qué conservar las columnas**: Principio VII prohíbe "cambiar la representación histórica" de
una venta emitida. `sale_items.combo_id` de ventas viejas es un hecho registrado; `DROP COLUMN`
lo borraría. El costo de conservarlas es nulo (nullable, sin escritura nueva).

**Promociones `combo` vigentes**: pasan a `Finalizada` en la migración (FR-025, D12). **No** se
migran a otra forma — un `combo` es "esta canasta específica de componentes" y el modelo nuevo
solo expresa "N unidades cualesquiera del conjunto"; traducirlo cambiaría el precio en silencio
(clarification 2026-08-31). Cambio de comportamiento **A-61** (D18).

**Frontend**: se borra `modules/tables/components/combo-select.component.ts` y el flujo "agregar
combo" de `product-select.component.ts` / `pos-catalog-drawer.component.ts` (FR-024).

---

## Decisión 9 — Menú QR: `menu_unit_discount` para el precio por variante; `variant_set_condition_text` para el anuncio; `_build_menu` no cambia de firma

**Decisión**:
- `_build_menu` (`menu/router.py:82`) **NO cambia de firma** — sigue devolviendo
  `list[MenuCategoryResponse]`. Es entrada del `"CONGELA comportamiento corregido:"` de
  `test_menu_router.py` (A-08), de `cart_fixtures.py` y del endpoint QR con token; cambiar su
  retorno rompería un test protegido (Principio III). Igual criterio que spec 040 D12.
- El precio con descuento por variante (`menu/router.py:156-164`, hoy `best_line_discount(promos,
  p.id, cat.id, 1, v.price)`) pasa a una función mínima
  `menu_unit_discount(promos, variant_id, unit_price)`: si alguna promoción `active` + vigente
  tiene `variant_id` en su conjunto **y** es `percent` con `min_qty == 1`, devuelve
  `round(unit_price * value / 100)`; en cualquier otro caso `None`. `package_price` y `percent`
  con `min_qty > 1` **no bajan el precio unitario** en el menú (depende de cuántas unidades
  combinadas haya, que en el menú aún no existen — mismo motivo por el que hoy `qty_price` no
  baja precio, `menu/router.py:153-155`).
- `_build_menu_promotions` (`menu/router.py:189-213`) se adapta: el texto de cada anuncio sale
  de `variant_set_condition_text(promo)` —construido de `(tipo, value, min_qty, nº de variantes
  del conjunto)`— en vez de las reglas de presentación. Ej.: `"Llevando 2 de estos 8 sabores
  pagas $12.000"` / `"10% en estos 5 productos"`. Se anuncia solo si `_valid_now(p, now)`
  (vigencia en ese instante, FR-022, SC-007). El endpoint `GET /menu/promotions` y la clave
  `"promotions"` del flujo QR con token se conservan (spec 040 FR-021).

**Por qué se conserva el anuncio de spec 040**: `spec.md` §"Cambios de comportamiento" punto 6:
"el resto de spec 040 —vigencia por día/hora, cruce de medianoche, anuncio en menú QR— se
conserva". Solo cambia la fuente del texto (conjunto de variantes en vez de presentación).

---

## Decisión 10 — Terminal de staff (FR-023): condición siempre visible, descuento efectivo al alcanzar `min_qty`, calculado por el preview del cobro

**Decisión**: `pos-terminal.store.ts` (frontend) pinta, para cada variante que pertenece al
conjunto de una promoción vigente en ese momento, **siempre** su condición en lenguaje llano
(`variant_set_condition_text`, servida por el backend en la respuesta de la promoción y/o en un
endpoint de preview). Cuando el pedido en curso alcanza `min_qty` unidades elegibles del
conjunto, muestra además el **descuento efectivo**, tomado del preview del cobro
(`compute_bill` / el preview de `checkout_and_send`, call-sites 3/6/7 de D7) — **no** de un
cálculo local. La terminal nunca aplica el descuento por su cuenta (FR-023, mismo trato que hoy
`promotion-pricing.util.ts` para `qty_price`).

**Por qué el preview del backend y no el `util`**: el descuento por conjunto depende de cuántas
unidades combinadas de todo el pedido pertenecen al conjunto (varias líneas, varios productos);
`promotion-pricing.util.ts` opera línea por línea y no tiene ese contexto. Replicarlo en
TypeScript reintroduciría el riesgo de divergencia que `registro-de-anomalias.md` A-09/A-10 ya
documenta. El `util` solo se usa para la **insignia** (condición legible) a partir de los datos de
la promoción.

---

## Decisión 11 — Edición de una promoción activa (FR-018): escalares sí, forma no; sin `priority` ni flags de confirmación

**Decisión**: `PromotionUpdate` (escalares, `PATCH /promotions/{id}`) permite `name`,
`description`, `ends_at`, `days_of_week`, `start_time`, `end_time`. **Bloquea** `value` y
`min_qty` cuando `status != "draft"` (422, "duplica la promoción para cambiar el valor o la
cantidad"). `type` y `variant_ids` solo por `PATCH /{id}/shape` y solo en `draft`
(`update_shape` ya exige `draft`, `service.py:1044-1049`). Se van del schema `priority` y los
flags `confirm_precio_no_uniforme` / `confirm_sin_descuento` (spec 040, sin sentido sin
presentación).

**Por qué bloquear `value`/`min_qty` en el servicio y no solo en el schema**: `PromotionUpdate`
no lleva `status`; el bloqueo se evalúa contra el `promo.status` real, igual que hoy
`service.update` valida `percent > 100` y `qty_price min_qty` contra el tipo real
(`service.py:1021-1030`).

**Duplicar** (`duplicate`, `service.py:1129`): la copia nace `draft` con el mismo tipo, valor,
`min_qty`, conjunto de variantes y vigencia, nombre distinto (FR-017). El solape de FR-014 se
revalida al **activar** la copia, no al duplicar (mismo criterio que spec 040,
`service.py:1151-1152`).

---

## Decisión 12 — Migración de datos: `percent` materializa su conjunto (foto fija); el resto va a `Finalizada`

**Decisión**: en la revisión **`063a`** (aditiva, Incremento A), `@for_each_tenant_schema`, con
`promotion_targets` / `promotion_combo_items` / `promotion_presentation_rules` **todavía presentes**
(su `DROP` es exclusivo de la revisión `063b`, Incremento F):

1. **`percent`** (FR-026): por cada promoción `type='percent'`, materializar el conjunto de
   variantes en `promotion_variants` (foto fija al momento de migrar):
   - por cada `promotion_targets` con `product_id`: todas las `product_variants.active = true` de
     ese producto;
   - por cada `promotion_targets` con `category_id`: todas las `product_variants.active = true`
     de los `products` de esa categoría;
   - si la promoción **no tiene targets** (percent global): todas las
     `product_variants.active = true` del tenant.
   `INSERT ... ON CONFLICT DO NOTHING` sobre `UNIQUE(promotion_id, product_variant_id)`.
   `type`, `value`, `status`, `starts_at`, `ends_at`, `days_of_week`, `start_time`, `end_time`,
   `min_qty` **se conservan tal cual**.
2. **`combo` / `fixed` / `qty_price` / `qty_price_presentation`** (FR-025): `UPDATE promotions SET
   status = 'finished', closed_by_refactor_at = now() WHERE type IN (...) AND status <>
   'finished'`. Las ya `finished` no se tocan (`closed_by_refactor_at` queda `NULL` para ellas —
   no las cerró el refactor). El `type` **no cambia** — queda como registro histórico.
3. El `DROP TABLE promotion_targets, promotion_combo_items, promotion_presentation_rules,
   presentations` (en ese orden, respetando FKs) va en la revisión **`063b`** (Incremento F), no
   aquí — porque hasta la Phase 8 hay código que aún las consulta.

**El aviso a recrear** (FR-025): el frontend consulta `GET /promotions?closed_by_refactor=true`
(query param nuevo, filtra `closed_by_refactor_at IS NOT NULL`) y muestra un banner descartable
(una vez por administrador, descarte en `localStorage`) con la lista. No se construye ninguna
tabla de "avisos" ni job — la marca en la promoción es suficiente, auditable (Principio XII) y
legible para siempre ("¿por qué se finalizó esta promo?").

**Por qué `percent` sí y el resto no** (clarification 2026-08-31): el `value` de un `percent` es
un porcentaje, semántica **idéntica** en el modelo nuevo — migrar es seguro. El `value` de un
`fixed` es un **monto de descuento por línea**, el de un `qty_price`/`qty_price_presentation` vive
**por target/regla** con su propio precio, y un `combo` es una **canasta específica**: traducir
cualquiera de ellos a "precio de paquete de N unidades cualesquiera del conjunto" cambiaría el
importe cobrado en silencio (Principio II). El admin recrea a mano lo que siga vigente.

**Alternativa considerada** para el aviso: una tabla `promotion_refactor_notices` poblada por la
migración y un endpoint dedicado — rechazada: una tabla de un solo uso es más peso que una
columna nullable en `promotions` que responde la misma pregunta.

---

## Decisión 13 — Qué de la spec 040 se conserva y qué se revierte

**Se revierte** (parte de modelo de datos, `spec.md` §"Cambios de comportamiento" punto 6):
- La entidad `Presentation` (`presentations`), su módulo de administración
  (`api/v1/presentations/`, `modules/presentations/`), el selector en el formulario de producto,
  la columna `product_variants.presentation_id` **y la integración que la spec 040 añadió a
  `api/v1/catalog/`** (`_resolve_presentation_id` en `catalog/router.py`, `presentation_id` en
  `VariantCreate` / `VariantUpdate` / `VariantResponse`) y en el frontend
  (`modules/products/services/product.service.ts`, `modules/tables/services/diner.service.ts`).
- La tabla `promotion_presentation_rules` y el tipo `qty_price_presentation`.
- Los tests de la spec 040 sin prefijo CONGELA: `test_promotions_presentation_pricing.py`,
  `test_promotions_presentation_rules.py`, `test_presentations_service.py`,
  `presentation_fixtures.py` — se **eliminan** (la propia spec.md lo dice).

**Se conserva** (correcciones y superficies útiles de la spec 040):
- `_valid_now` con la atribución de día al cruzar medianoche (**A-57** — sigue vigente para los
  dos tipos que quedan). El `check()` de esa combinación en el script de CI se mantiene.
- La vigencia por `days_of_week` (conjunto), ventana horaria con cruce de medianoche, evaluación
  en zona horaria del tenant (FR-012, FR-013).
- El anuncio en el menú QR: endpoint `GET /menu/promotions`, clave `"promotions"` del flujo QR
  con token, banner en `public-menu.component.ts` — adaptados a "conjunto de variantes" (D9).
- El patrón "`_build_menu` no se toca" (Principio III, spec 040 D12).

Cambio de comportamiento **A-63** (elimina `Presentation`; revierte la parte de modelo de datos
de spec 040) — D18.

---

## Decisión 14 — Persistencia del resultado (FR-021): columna JSONB `applied_promotions`, no tablas puente

**Decisión**: columna `applied_promotions JSONB NOT NULL DEFAULT '[]'` en `sales`, `invoices` y
`customer_orders`. Contenido: `[{"promotion_id": "<uuid>", "name": "<snapshot>", "amount":
"<decimal>"}]` — una entrada por promoción que descontó **alguna** línea de ese cobro, con su
**monto agregado** (no por línea). `customer_orders` además gana `discount NUMERIC(12,2) NOT NULL
DEFAULT 0` (hoy no tiene ningún campo de descuento). `Sale.discount` / `Invoice.discount` ya
existen y siguen siendo el agregado.

- `build_sale` (`sales/builder.py:79`) acepta `applied_promotions: list[dict]` y lo persiste.
- `issue_for_sale` (`invoices/service.py:67`) copia `sale.applied_promotions` a la factura
  (snapshot inmutable, como `discount=sale.discount`).
- Los caminos de cobro de mesa (`pay_order`, `_close_unified`, `_close_split`) fijan además
  `order.discount` y `order.applied_promotions` en la `CustomerOrder` que se paga.
- `Sale.promotion_id` (FK única) **se conserva** y se sigue poblando cuando **una sola** promoción
  explica todo el descuento (compatibilidad hacia atrás con el detalle de venta y el menú);
  cuando hay dos o más, queda `NULL` como hoy — pero ahora `applied_promotions` lo cubre. **Eso
  resuelve A-29** (hoy, con más de una promoción o combo, no queda registrada ninguna).

**Por qué JSONB y no una tabla puente** (`sale_promotions`, `invoice_promotions`,
`customer_order_promotions`):
- El proyecto ya usa JSONB para snapshots inmutables por fila: `sale_items.options`
  (`sale.py:122-124`), `audit_logs.payload`, `order_payment_attempts`. `applied_promotions` es
  exactamente eso — una foto del resultado al emitir, que **no** se vuelve a consultar
  relacionalmente (el desglose por línea, que sí querría `JOIN`s, está fuera de alcance).
- Tres tablas puente nuevas con FK `SET NULL` a `promotions` (para no romper al borrar una
  promo) más su cascada desde `sales`/`invoices`/`customer_orders` es mucho más esquema para el
  mismo dato. El `name` snapshot en el JSON sobrevive al borrado de la promoción sin FK.
- FR-021 pide "el monto de descuento agregado más la lista de promociones", no una relación
  normalizada.

**Alternativa considerada**: tabla puente `sale_promotions(sale_id, promotion_id, amount)` +
espejos en invoice/order — rechazada por lo anterior; se reconsideraría si una spec futura
necesita el desglose por línea (entonces la tabla llevaría también `sale_item_id`).

**No retroactivo**: `applied_promotions` nace `'[]'`, `customer_orders.discount` nace `0`; sin
backfill. Ninguna venta ni factura emitida antes del despliegue cambia (Principio VII, FR-021).

Cambio de comportamiento **A-64** (persiste `applied_promotions`; resuelve A-29) — D18.

---

## Decisión 15 — La regla "precio uniforme por presentación" se deroga formalmente

**Decisión**: ninguna validación del modelo nuevo asume que dos variantes "del mismo tamaño"
cuestan lo mismo. El motor (D5) trabaja con el `unit_price` real de cada unidad; FR-016 usa el
precio de la variante **más barata** del conjunto; el consumo codicioso ordena por precio real.

**Por qué es un cambio registrable**: la spec 040 introdujo la verificación de uniformidad de
precio (`_check_presentation_rule_prices`, FR-017 de esa spec) como una barrera con confirmación.
`spec.md` §"Cambios de comportamiento" punto 8 la deroga porque "es falsa en el catálogo real (un
Pequeño con licor cuesta $8.000 y uno sin licor $6.000)". Cambio de comportamiento **A-65**
— D18. Se elimina `_check_presentation_rule_prices` y los flags `confirm_*`.

---

## Decisión 16 — FR-016: el guardado se bloquea si el precio de paquete no representa un descuento

**Decisión**: `_guard_package_is_discount(db, promo)` en `promotions/service.py`, invocado desde
`create`, `update_shape` y `change_status → active`. Para `type == "package_price"`: si
`Decimal(promo.value) >= promo.min_qty * (menor product_variants.price entre las variantes del
conjunto, activas o no)` → **409** con el detalle (`value`, `min_qty`, `cheapest_unit_price`,
`variant_id`).

**Por qué "la más barata" y bloqueo duro** (spec.md Edge Cases, FR-016): si el peor caso del
conjunto (`min_qty` unidades de la variante más barata) cuesta ≤ el precio de paquete, existe una
combinación real del conjunto para la que la promoción **encarece o no hace nada** — la spec
exige "bloquear, no basta con advertir". El ejemplo de las Assumptions ("Los seis 2X"): si el
conjunto incluyera variantes **sin licor**, FR-016 bloquearía Medianos/Extra grandes/Baldes/Litros
(2 sin licor cuestan menos que el precio 2X).

**Por qué también en `change_status → active`**: los precios de las variantes pueden bajar
mientras la promo está en `draft`; revalidar al activar evita activar una promo que ya no
descuenta.

---

## Decisión 17 — Entrega incremental (Principio VI)

**Decisión**: 6 incrementos, cada uno verificable por separado antes de seguir:

- **A — Migración aditiva + modelo**: revisión **`063a`** — `promotion_variants`, `PromotionVariant`,
  `closed_by_refactor_at`, columnas `applied_promotions` + `customer_orders.discount`,
  `ck_promotion_min_qty`, `ck_promotion_type` **ampliado** con `package_price`, y el **paso de datos**
  (percent→conjunto; combo/fixed/qty_price/qty_price_presentation→finished). **100% aditivo**: las
  tablas viejas y el motor viejo siguen intactos, la suite existente sigue en verde. Verificable
  contra PostgreSQL real: `alembic upgrade head` / `downgrade -1` / `upgrade head`; comprobar el
  estado y la forma de cada promoción migrada. El borrado de `PromotionTarget` / `PromotionComboItem`
  / `PromotionPresentationRule` / `Presentation` / `priority` / `presentation_id` (código + revisión
  destructiva **`063b`**) se difiere al Incremento F — hasta la Phase 8 `menu/router.py` importa
  `Presentation`, así que no puede borrarse antes sin romper el build.
- **B — CRUD + validaciones**: schemas con `variant_ids`, `_apply_variant_set`,
  `_guard_variant_overlap` (FR-014), `_guard_package_is_discount` (FR-016), edición limitada de
  activas (FR-018), duplicar, máquina de estados, permisos (solo admin del tenant, FR-019).
  Verificable creando/activando/duplicando promociones y comprobando los rechazos, **sin** que el
  cobro las use todavía.
- **C — Motor + persistencia**: `evaluate_variant_sets`, `_greedy_units`,
  `_distribute_group_discount`, rewire de los ~9 call-sites (D7), `applied_promotions` en
  `Sale`/`Invoice`/`CustomerOrder` (FR-021). Verificable con los 10 Acceptance Scenarios de US2 y
  SC-005.
- **D — Menú QR + terminal**: `menu_unit_discount`, `variant_set_condition_text`,
  `_build_menu_promotions` adaptado (FR-022), banner en `public-menu.component.ts`, condición +
  descuento efectivo en la terminal (FR-023).
- **E — Frontend de administración**: reescritura de `promotions-page.component.ts`,
  `promotion.interface.ts`, `promotion-pricing.util.ts`; borrado de `scope-picker.component.ts`,
  `modules/presentations/`, `combo-select.component.ts`; selector de conjunto de variantes con
  filtros; resumen legible (FR-005); banner de FR-025; ajuste de `product-form.component.ts` y la
  navegación.
- **F — Retiro de estructura legada + tests**: revisión destructiva **`063b`** (drop de
  `promotion_targets` / `promotion_combo_items` / `promotion_presentation_rules` / `presentations` /
  `priority` / `presentation_id`; `ck_promotion_type` estrechado con escape `OR status='finished'`);
  borrado de `Presentation` / targets / combos del ORM (`promotion.py`, `product_variant.py`,
  `__init__.py`, `presentation.py`), del paquete `api/v1/presentations/` y de su integración en
  `api/v1/catalog/` (+ en `pos-heladeria` `product.service.ts` / `diner.service.ts`); reescritura de
  los `CONGELA` afectados citando A-58…A-65, reescritura del script de CI, borrado de los tests de la
  spec 040, tests nuevos por historia ([contracts/migracion.md](./contracts/migracion.md)
  §"Inventario de tests").

Ningún incremento mezcla la **migración de datos** (solo A, revisión `063a`) con el cambio del
cálculo de cobro (solo C). El **borrado de estructura** (`063b`) se difiere a F —no por ser un
cambio de comportamiento, sino porque hasta la Phase 8 hay módulos que importan `Presentation`— para
que A quede 100% aditivo y verificable con la suite en verde. `tasks.md` ordena las tareas dentro y
entre incrementos. La entrada de A-58…A-65 en `registro-de-anomalias.md` es **prerrequisito del
incremento A** (Principio II: registrar antes de implementar), igual que spec 040 hizo con A-57 en
su fase Foundational.

---

## Decisión 18 — Ocho entradas en `registro-de-anomalias.md` (A-58 … A-65)

`spec.md` §"Cambios de comportamiento respecto de producción" lista 8 cambios observables sobre
promociones en producción; el Principio II exige una entrada por cada uno, **antes** de
implementar. Última entrada previa: **A-57** (spec 040). Se agregan:

| ID | Qué cambia | RN afectadas / FR |
|---|---|---|
| **A-58** | Se elimina `Promotion.priority` por completo (columna, desempate del motor, orden del listado, campo del formulario). Con el bloqueo de solape real (A-59) no queda conflicto que desempatar. | RN-PROMO-13, RN-PROMO-14, RN-PROMO-43; FR (spec.md §Cambios 1) |
| **A-59** | El solape entre promociones automáticas pasa de **advertencia** (`find_overlaps`) a **bloqueo** (409), con el criterio acotado de FR-014/FR-014a (variante compartida + intersección simultánea de fecha, días y horas; dimensión abierta = todo su dominio). | RN-PROMO-30; FR-014, FR-014a |
| **A-60** | Se elimina el alcance por **categoría** y la regla "marcar una categoría incluye los productos creados después". El alcance se resuelve **solo** por la lista explícita de variantes. | RN-PROMO-06 (parte categoría); FR-003, FR-004, FR-010 |
| **A-61** | Se elimina el tipo `combo` y su mecanismo de selección explícita (`combo_id` en líneas de carrito/orden/venta, expansión al agregarlo). Las promociones `combo` vigentes pasan a `Finalizada`; **no** se migran. Las líneas de venta históricas marcadas con combo **no se tocan**. | FR-024, FR-025 |
| **A-62** | Se eliminan los tipos `qty_price` (producto/categoría), `qty_price_presentation` (presentación) y `fixed` (monto fijo por línea); las instancias vigentes pasan a `Finalizada`. Solo `percent` se migra automáticamente (materializando su conjunto de variantes, foto fija). Los identificadores reservados `buy_x_get_y` y `qty_price` se retiran del enum. | FR-002, FR-025, FR-026 |
| **A-63** | Se elimina la entidad `Presentation` y todo lo que depende de ella (tabla, `product_variants.presentation_id`, módulo de administración, selector en el formulario de producto, `promotion_presentation_rules`). Revierte la parte de **modelo de datos** de la spec 040; se conserva el resto (vigencia día/hora, cruce de medianoche + A-57, anuncio en menú QR). | FR-027 (spec.md §Cambios 6) |
| **A-64** | Se persiste el **descuento agregado + la lista de promociones que lo generaron** (columna JSONB `applied_promotions`) en `sales`, `invoices` (factura) y `customer_orders` (que además gana `discount`), además de calcularse al vuelo. **Resuelve A-29** (hoy, con más de una promoción o combo, `promotion_id` queda `NULL` y no se registra ninguna). No retroactivo. | FR-021; cierra A-29 |
| **A-65** | Se deroga la regla "dentro de una misma presentación el precio unitario es siempre el mismo" (introducida como verificación con confirmación por la spec 040, FR-017 de esa spec). Es falsa en el catálogo real; ninguna validación del modelo nuevo la usa. | spec.md §Cambios 8 |

**Quién y cuándo** (para las 8): propietario del repositorio (leonardogomez306@gmail.com),
2026-08-31, en las Clarifications y en la sección "Decisiones tomadas en la sesión de
clarificación" de
[`specs/063-promociones-por-variante/spec.md`](../063-promociones-por-variante/spec.md).
**No retroactivo** (Principio VII): ninguna `Sale`/`SaleInvoice` ya emitida se recalcula ni cambia
de representación. **Tratamiento**: implementar en el incremento A/B/C de `tasks.md`, con la
reescritura de los tests `CONGELA` afectados en el mismo commit que cita la decisión (Principio
III, FR-028).
