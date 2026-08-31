# Contrato: superficies de consumo (menú QR, terminal de staff, formulario)

Tres superficies replican el criterio de elegibilidad y de condición legible; **el cálculo
vinculante es el del cobro** (`evaluate_variant_sets`, [motor-y-persistencia.md](./motor-y-persistencia.md)).
Ver [research.md](../research.md) D9/D10 y `spec.md` FR-005/FR-022/FR-023.

---

## 1. Menú QR público (FR-022)

### `GET /menu` — precio con descuento por variante

`_build_menu` **no cambia de firma** (sigue devolviendo `list[MenuCategoryResponse]` — CONGELA de
`test_menu_router.py`, A-08). El bloque que hoy calcula `discounted_price`
(`menu/router.py:156-164`) pasa a:

```python
disc = menu_unit_discount(promos, v.id, v.price)   # promos = active_variant_set_promotions(db, now)
if disc is not None:
    discounted_price = (v.price - disc).quantize(Decimal("0.01"), ROUND_HALF_UP)
    discount_kind = <tipo de la promo que aplicó>
```

`menu_unit_discount(promos, variant_id, unit_price) -> Decimal | None`:
- Si alguna `promo` vigente tiene `variant_id` en su conjunto **y** es `percent` con
  `min_qty == 1` → `round(unit_price * value / 100, ROUND_HALF_UP)`.
- En cualquier otro caso (`package_price`, `percent` con `min_qty > 1`) → `None`. Igual que hoy
  `qty_price` no baja el precio unitario en el menú: el descuento depende de cuántas unidades
  combinadas hay, que en el menú sin carrito no existen.

### `GET /menu/promotions` y clave `"promotions"` del flujo QR con token — anuncio

`_build_menu_promotions(db, now)` (`menu/router.py:189`) **se conserva** (spec 040 FR-021), pero
el texto de cada anuncio sale del **conjunto de variantes**:

```
MenuPromotionAnnouncement:
  promotion_id, promotion_name,
  rules: [ MenuPromotionRule(
             text = variant_set_condition_text(promo),   # "Llevando 2 de estos 8 sabores pagas $12.000"
             variant_count = len(promo.variants),
             min_qty, value ) ]
```

Se incluye solo si `_valid_now(promo, now)` es verdadero — **vigencia en ese instante** (ventana
de día/hora, zona del tenant), no solo `status == "active"` (FR-022, SC-007). Fuera de la ventana
no se anuncia. `_build_menu_promotions` no arrastra A-08: `now` viene aware
(`datetime.now(timezone.utc)`).

`MenuPromotionRule` deja de describir una presentación; describe el conjunto ("estas N
variantes"). El frontend (`public-menu.component.ts`) renderiza el banner con esos textos,
visible con el carrito vacío (US4-CA1); cuando el carrito alcanza `min_qty` unidades elegibles,
el `discounted_total` del carrito ya refleja el precio efectivo (vía `serialize_cart` →
`evaluate_variant_sets`, US4-CA3).

El cliente del menú (`pos-heladeria/src/app/modules/tables/services/diner.service.ts`) parsea
`promotions[].rules[]` del payload del QR: su interfaz `MenuPromotionAnnouncement.rules` pierde
`presentation_name` / `pack_price` y adopta `{ text, variant_count, min_qty, value }` (mismo shape
que `MenuPromotionRule`). El parser (`mapCategory` / el mapeo de `promotions`) deja de leer
`presentation_name` y `pack_price`.

---

## 2. Terminal de staff (FR-023)

### Condición siempre visible

Para cada variante que pertenece al conjunto de una promoción **vigente en ese momento**
(`status == "active"` + `_valid_now`), la terminal muestra **siempre**
`variant_set_condition_text(promo)` (p. ej. "Llevando 2 pagas $12.000"), aunque el pedido en
curso no alcance `min_qty` unidades elegibles. El texto viene del backend
(`PromotionResponse.condition_text` y/o el preview del cobro), no se arma en el `util`.

### Descuento efectivo al alcanzar `min_qty`

Cuando el pedido en curso alcanza `min_qty` unidades elegibles del conjunto, la terminal muestra
además el **descuento efectivo**, tomado del **preview del cobro** (`compute_bill` /
el preview de `checkout_and_send`, que corren `evaluate_variant_sets`) — no de un cálculo local.
La terminal **nunca** aplica el descuento por su cuenta: el cálculo vinculante es el del cobro
(FR-023, US2-CA10).

### `promotion-pricing.util.ts` (frontend)

- `getPromoDisplay(promo)` → insignia para 2 tipos:
  - `percent` → "N% de descuento" (previsualizable a nivel de precio unitario **si**
    `min_qty === 1`).
  - `package_price` → "Llevando {min_qty} pagas {value}" — **fuera** de la previsualización de
    precio unitario (como `qty_price` hoy, `promotion-pricing.util.ts:127`).
- Se eliminan las ramas de `combo`, `qty_price`, `qty_price_presentation`.
- El `util` no calcula el descuento de paquete: depende del pedido completo, lo calcula el
  backend (research.md D10, evita la divergencia de A-09/A-10).

---

## 3. Formulario de administración (FR-004, FR-005) — frontend

### Selector de conjunto de variantes

- Un multi-select de variantes del catálogo. Filtros **solo para poblar** la vista del selector:
  por **producto**, por **categoría**, por **texto** (nombre de variante / producto). Lo que se
  guarda es `variant_ids` — la lista concreta visible/marcada, **no** el filtro (FR-004).
- Una variante creada después en una categoría que se usó como filtro **no** entra sola
  (US1-CA2): el frontend no re-consulta el filtro al guardar.

### Resumen antes de confirmar (FR-005)

Muestra, tomándolo de `PromotionResponse` (draft) o del estado del formulario:
- el **tipo** ("Descuento %" / "Precio de paquete");
- la **condición en lenguaje llano** (`condition_text`): "Llevando 2 de cualquiera de estas 8
  variantes pagas $12.000";
- la **lista de variantes** del conjunto con su **precio normal vigente** (`variants[].unit_price`).

### Diálogos de error

| Origen | Contenido |
|---|---|
| 409 FR-014 (solape) | "Otra promoción activa ya cubre esta(s) variante(s) en un horario que se cruza": nombra `conflicts[].promotion_name` y las variantes compartidas. |
| 409 FR-016 (sin descuento) | "Con este precio de paquete, {min_qty} unidades de {variante más barata} costarían {cheapest×min_qty} — la promoción no representa un descuento." |
| 422 FR-018 (editar activa) | "El valor y la cantidad de una promoción activa no se pueden cambiar. Duplícala para crear una versión nueva." |

### Banner de migración (FR-025)

Al entrar al módulo de promociones, si `GET /promotions?closed_by_refactor=true` devuelve
resultados, se muestra un banner **descartable** (una vez por administrador, descarte en
`localStorage`) con la lista de promociones que la migración de la spec 063 pasó a `Finalizada`,
invitando a recrearlas si siguen vigentes.

### Se elimina del formulario

Selector de tipo `combo` / `qty_price` / `qty_price_presentation`; el formulario de combo
(`combo_items`); el formulario de precio por target (`scope-picker.component.ts`); el formulario
de reglas por presentación; el input de `priority`; el panel de `overlaps` (advertencia).
