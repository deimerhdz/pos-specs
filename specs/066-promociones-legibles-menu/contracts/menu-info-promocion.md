# Contrato: información de promoción por variante en el menú (FR-007 a FR-012)

**Normativo.** Define el bloque que el backend entrega **ya calculado** por variante en la
respuesta del menú, y la extensión de `discounted_price` que corrige el defecto de importe
(FR-010, A-68).

---

## 1. El DTO

```python
# app/api/v1/menu/schemas.py  (nuevo)

class MenuVariantPromotion(BaseModel):
    """Información de la ÚNICA regla vigente que cubre esta variante en este
    instante (FR-012: la spec 063 FR-014 impide que haya dos). Derivada, no
    persistida. El importe vinculante lo sigue calculando el cobro (FR-011)."""
    condition_text: str          # FR-004 completo — idéntico al cartel y a administración (SC-005)
    short_condition: str         # "2 x $12.000" | "3 x -15%"            (FR-008)
    unit_equivalent: Decimal     # ya redondeado al peso                  (FR-009)
    unit_equivalent_approx: bool # el exacto no era entero en pesos       (FR-009)
    unit_equivalent_text: str    # "$6.000 c/u" | "≈ $4.333 c/u"          (FR-009)
    display_text: str            # "2 x $12.000 · $6.000 c/u"             (FR-008)
    type: PromotionType
    min_qty: int
    value: Decimal
```

```python
# app/api/v1/menu/schemas.py  (modificado, ADITIVO)

class MenuVariantResponse(BaseModel):
    ...
    discounted_price: Decimal | None = None   # significado extendido, ver §4
    discount_kind: PromotionType | None = None
    promotion: MenuVariantPromotion | None = None   # NUEVO
```

`promotion` es opcional con default `None`: un cliente sin desplegar ignora el campo y sigue
funcionando igual que hoy.

`MenuProductResponse` **no** gana ningún campo: la insignia de la tarjeta se deriva de
`variants[].promotion` en el frontend (FR-013, [research.md D-11](../research.md)).

---

## 2. Cálculo

```python
# app/api/v1/promotions/service.py  (nuevo)

def menu_variant_promotion(
    rules: list[PromotionRule],
    variant_id: UUID,
    unit_price: Decimal,
    names: Mapping[UUID, str],
) -> dict | None:
    """Bloque de FR-007 para una variante, o `None` si ninguna regla vigente la
    cubre. `rules` son las de `active_variant_set_rules(db, now)` — la vigencia
    ya está resuelta. Primera coincidencia sin ordenar: FR-012 garantiza que
    hay a lo sumo una."""
```

### 2.1 Selección de la regla (FR-012)

Recorrer `rules` y quedarse con la primera cuyo conjunto contenga `variant_id`. **No hay criterio
de desempate y no debe inventarse uno**: `_guard_variant_overlap` (spec 063 FR-014) impide que dos
reglas vigentes compartan una variante en el mismo instante.

Una regla cuyo `variant_set_condition_text` devuelve `None` (tipo retirado) **no** produce bloque:
no llega aquí porque `active_variant_set_rules` ya filtra por `LIVE_TYPES`, pero la guarda se
mantiene por si el filtro cambia.

### 2.2 Equivalente por unidad (FR-008)

| Tipo | Exacto | Base |
|---|---|---|
| `package_price` | `Decimal(value) / Decimal(min_qty)` | El precio del paquete es **único**: el mismo equivalente para todas las variantes del conjunto, aunque mezclen precios normales distintos. |
| `percent` | `Decimal(unit_price) * (100 - Decimal(value)) / 100` | El precio normal **de esa variante**. |

### 2.3 Redondeo y marca de aproximado (FR-009)

```python
exacto = ...                                  # §2.2, sin redondear
aprox  = exacto % 1 != 0
valor  = exacto.quantize(Decimal("1"), rounding=ROUND_HALF_UP)
```

- **Peso más cercano, medio hacia arriba** — el mismo `ROUND_HALF_UP` que el motor de cobro aplica
  a su descuento de grupo (`service.py:260-262`).
- La marca `≈` aparece **siempre que el exacto no sea entero en pesos**, en los dos tipos. En
  `package_price` porque el valor puede no dividirse exacto; en `percent` porque el importe
  vinculante lo calcula el cobro sobre el total del grupo y luego lo reparte
  (`_distribute_group_discount`), de modo que el equivalente por variante puede diferir en un peso.
- Un precio de catálogo llega como `Decimal("8000.00")`; `Decimal("8000.00") % 1` es
  `Decimal("0.00")`, que compara igual a `0`. No hace falta normalizar antes.

### 2.4 Textos

Con `_money()` (el mismo formateador de `$12.000`) y `pct` con el mismo despojo de ceros que
[texto-condicion.md §3](./texto-condicion.md):

```text
short_condition       package_price  →  "{min_qty} x {_money(value)}"
                      percent        →  "{min_qty} x -{pct}%"

unit_equivalent_text  aprox          →  "≈ {_money(unit_equivalent)} c/u"
                      exacto         →  "{_money(unit_equivalent)} c/u"

display_text                         →  "{short_condition} · {unit_equivalent_text}"
```

El separador es `" · "` (espacio, punto medio U+00B7, espacio), tal como lo escriben la
Clarification y FR-008.

### 2.5 Ejemplos verificables

| Regla | Precio normal | `short_condition` | `unit_equivalent_text` | `display_text` |
|---|---|---|---|---|
| paquete, `min_qty` 2, `$12.000` | `$8.000` | `2 x $12.000` | `$6.000 c/u` | `2 x $12.000 · $6.000 c/u` |
| paquete, `min_qty` 3, `$13.000` | `$5.000` | `3 x $13.000` | `≈ $4.333 c/u` | `3 x $13.000 · ≈ $4.333 c/u` |
| 15%, `min_qty` 3 | `$11.000` | `3 x -15%` | `$9.350 c/u` | `3 x -15% · $9.350 c/u` |
| 10%, `min_qty` 1 | `$8.000` | `1 x -10%` | `$7.200 c/u` | `1 x -10% · $7.200 c/u` |

---

## 3. Dónde se puebla

`menu/router.py`, `_build_menu` — dentro del bucle de variantes que ya existe (`:158-171`):

```python
rules = active_variant_set_rules(db, now)                 # ya existía
names = variant_display_names(db, {v.product_variant_id   # NUEVO, una consulta
                                   for r in rules for v in r.variants})
...
promotion = menu_variant_promotion(rules, v.id, v.price, names) if rules else None
```

`names` se resuelve **una vez por llamada**, fuera del bucle. Nunca dentro: sería un N+1 por
variante.

`_build_menu_promotions` hace su propia resolución de nombres para el cartel
([texto-condicion.md §1](./texto-condicion.md)). El flujo del QR con token llama a las dos
funciones, así que son dos consultas constantes por resolución de QR — no se comparten a propósito
([research.md D-12](../research.md)).

---

## 4. `discounted_price`: la corrección de importe (FR-010, A-68)

### 4.1 Qué cambia

`menu_unit_discount` hoy (`service.py:300-310`) devuelve descuento **solo** para `percent` con
`min_qty == 1`. Pasa a cubrir también `package_price` con `min_qty == 1`, que es un **precio
unitario especial**:

| Regla vigente que cubre la variante | `discounted_price` | `discount_kind` |
|---|---|---|
| `percent`, `min_qty == 1` | `price - price × value/100`, `quantize(0.01, HALF_UP)` (sin cambio) | `"percent"` |
| `package_price`, `min_qty == 1` | **`rule.value`**, tal cual | `"package_price"` |
| `percent`, `min_qty > 1` | `None` (sin cambio) | `None` |
| `package_price`, `min_qty > 1` | `None` (sin cambio) | `None` |
| Ninguna | `None` | `None` |

### 4.2 El invariante que hace falta leer despacio

Para `package_price` con `min_qty == 1`, `discounted_price` es el valor de la regla **siempre**,
incluso cuando resulta **mayor o igual** que `price`. No se recorta, no se descarta, no se
sustituye por `price`.

**Por qué**: es el importe que el cobro aplica. Devolver `None` cuando el valor es mayor
reintroduciría exactamente el defecto que A-68 corrige (mostrado ≠ cobrado), solo que en la
dirección contraria. Lo que **no** se muestra en ese caso es la **señal de ahorro**: FR-015 exige
mostrar el valor sin tachado y sin insignia. Esa decisión la toma el frontend comparando dos
importes que ya le llegaron — ver [superficies.md §2](./superficies.md).

Ese caso es alcanzable en producción pese a `_guard_package_is_discount`: la guarda corre en
`create`, `update_shape` y `change_status`, y **no** cuando el catálogo cambia un precio
([research.md D-6](../research.md)). Debe tener test.

### 4.3 Lo que NO cambia

- El equivalente por unidad de `promotion` es **informativo** y no altera el precio unitario con el
  que la unidad entra al carrito, ni el subtotal, ni el cobro (FR-011).
- `evaluate_variant_sets` no se toca. `discounted_price` no alimenta ningún cálculo del backend:
  es salida de presentación.
- El criterio de vigencia no cambia (`active_variant_set_rules` / `_valid_now`, A-57 intacto).

---

## 5. Nulidad: la tabla completa

Para una variante en una llamada al menú:

| Situación | `promotion` | `discounted_price` |
|---|---|---|
| Sin regla vigente que la cubra | `None` | `None` |
| Promoción activa pero fuera de su ventana en ese instante | `None` | `None` |
| Regla de tipo retirado (histórica) | `None` | `None` |
| Conjunto sin variantes elegibles (todas desactivadas) | `None` para las que no llegan al menú | `None` |
| `percent`, `min_qty > 1` | poblado | `None` |
| `package_price`, `min_qty > 1` | poblado | `None` |
| `percent`, `min_qty == 1` | poblado | poblado |
| `package_price`, `min_qty == 1` | poblado | poblado (= `rule.value`) |

La fila que más importa: **`promotion` poblado con `discounted_price` en `None`** es el caso normal
de una promoción de paquete, y es justo el que hoy no produce ninguna señal en el menú.
