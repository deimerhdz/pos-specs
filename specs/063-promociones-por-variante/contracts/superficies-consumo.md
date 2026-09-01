# Contrato: superficies de consumo (menú QR, terminal de staff, formulario)

> **Reemplaza** la versión de este contrato escrita para el modelo plano — la que describe con
> exactitud lo ya implementado en `menu/router.py`, `promotion-pricing.util.ts` y
> `promotions-page.component.ts` de las ramas de feature (verificado 2026-09-01). Describe el
> delta: cada superficie pasa de leer **una condición por promoción** a leer **una condición por
> regla**. El cálculo vinculante sigue siendo el del cobro
> (`evaluate_variant_sets`, [motor-y-persistencia.md](./motor-y-persistencia.md)).

---

## 1. Menú QR público (FR-022)

### `GET /menu` — precio con descuento por variante

`_build_menu` sigue sin cambiar de firma. El bloque que calcula `discounted_price`
(`menu/router.py:156-164`) pasa de `menu_unit_discount(promos, v.id, v.price)` a:

```python
disc = menu_unit_discount(rules, v.id, v.price)   # rules = active_variant_set_rules(db, now)
```

`menu_unit_discount(rules, variant_id, unit_price) -> Decimal | None`: **mismo criterio**, ahora
sobre reglas — si alguna `rule` vigente (de una promoción activa y válida ahora) tiene
`variant_id` en su conjunto **y** es `percent` con `min_qty == 1` → descuento unitario; en
cualquier otro caso (`package_price`, o `percent` con `min_qty > 1`) → `None`, sin cambio de
motivo (el descuento depende de cuántas unidades combinadas hay, que en el menú sin carrito no
existen).

### `GET /menu/promotions` y clave `"promotions"` del flujo QR con token — anuncio

`_build_menu_promotions(db, now)` (`menu/router.py:189`) **se conserva**. La diferencia es de
cardinalidad, no de forma — el DTO de salida ya modelaba una lista:

```
MenuPromotionAnnouncement:
  promotion_id, promotion_name,
  rules: [ MenuPromotionRule(                        # UNA por cada PromotionRule de esta promoción
             text = variant_set_condition_text(rule),  # "Llevando 2 de estos 8 sabores pagas $12.000"
             variant_count = len(rule.variants),
             min_qty = rule.min_qty, value = rule.value ) ]
```

Antes de este refactor, `rules` tenía **siempre 1 elemento** (una promoción = una regla implícita).
Ahora tiene **N elementos**, uno por cada `PromotionRule` de la promoción — el frontend
(`diner.service.ts`, `MenuPromotionAnnouncement.rules[]`) **ya tiene el tipo correcto** (research.md
D-R3): no hace falta ningún cambio de interfaz, solo dejar de asumir cardinalidad 1 en cualquier
lugar que renderice `rules[0]` en vez de iterar todo el arreglo (a verificar puntualmente en
`public-menu.component.ts` durante la implementación).

Se incluye una regla en el anuncio solo si su promoción está `_valid_now` en ese instante — sin
cambio de criterio (FR-022, SC-007).

---

## 2. Terminal de staff (FR-023)

### Condición siempre visible — ahora por regla

Para cada variante que pertenece al conjunto de **una regla** cuya promoción está vigente en ese
momento, la terminal muestra siempre `variant_set_condition_text(rule)` (p. ej. "Llevando 2 pagas
$12.000"), aunque el pedido en curso no alcance el `min_qty` de esa regla. Si una variante
perteneciera al conjunto de más de una regla vigente al mismo tiempo, eso ya sería un estado
imposible por diseño (FR-014/FR-001a lo bloquean al guardar) — la terminal no necesita resolver
ese caso.

### Descuento efectivo al alcanzar `min_qty` de la regla

**Sin cambio de criterio**: viene del preview del cobro (`evaluate_variant_sets`), nunca de un
cálculo local en el cliente (FR-023).

### `promotion-pricing.util.ts` (frontend)

**Cambia el parámetro** de las funciones que arman la insignia — reciben una `PromotionRule`, no
una `Promotion`:
- `getPromoDisplay(rule)` → insignia para 2 tipos, **mismo criterio** que antes (`percent`
  previsualizable a nivel de precio unitario si `rule.min_qty === 1`; `package_price` fuera de la
  previsualización unitaria).
- Un componente que muestra "todas las condiciones vigentes de una promoción" itera
  `promotion.rules` y llama `getPromoDisplay` por cada una.

### `pos-terminal.store.ts`

`productDiscountBadge()` (`pos-terminal.store.ts:407-419`, verificado 2026-09-01) escaneaba
`promotionService.activePromotions()` cruzando `promo.variants` con las variantes del producto.
Pasa a cruzar **`promo.rules[].variants`** — la variante puede coincidir con el conjunto de
cualquiera de las reglas de cualquiera de las promociones activas, no solo con un conjunto único
por promoción.

---

## 3. Formulario de administración (FR-001, FR-004, FR-005) — frontend

### Sub-lista repetible de reglas

**Nuevo respecto del modelo plano**: el formulario deja de tener un único bloque
tipo/valor/cantidad mínima/conjunto y pasa a tener:
- Un bloque de **vigencia** a nivel de promoción (nombre, descripción, fechas, días, horario) —
  sin cambio respecto de hoy.
- Una **lista repetible de reglas**, cada una con su propio tipo/valor/cantidad mínima/selector de
  conjunto (research.md D-R4: `*ngFor` + `ngModel` indexado sobre `form.rules`, sin introducir
  `ReactiveFormsModule`). Botones "agregar regla" / "quitar regla". Al menos una regla es
  obligatoria para guardar (FR-001).

### Selector de conjunto — sin cambio de principio, ahora por regla

Multi-select de variantes del catálogo, uno **por cada fila de regla**. Filtros solo para poblar
esa fila: por producto, por categoría, por texto. Lo que se guarda es `variant_ids` de esa regla —
la lista concreta visible/marcada, no el filtro (FR-004). Una variante creada después en una
categoría usada como filtro no entra sola en ninguna regla existente.

### Validación en el cliente (antes de enviar)

Antes de llamar al backend, el formulario valida que ninguna variante se repita entre dos filas de
regla del mismo formulario (mismo criterio que `_guard_variant_overlap` chequeo 1, FR-001a) — da
feedback inmediato sin esperar el 409; el backend **sigue validando lo mismo** del lado del
servidor (nunca confiar solo en el cliente).

### Resumen antes de confirmar (FR-005)

**Por cada regla**: el tipo, la condición en lenguaje llano (`condition_text` de esa regla), la
lista de variantes de su conjunto con su precio normal vigente. La vigencia (días, horario, fechas)
se muestra **una vez**, a nivel de promoción — no se repite por regla.

### Diálogos de error

| Origen | Contenido |
|---|---|
| 409 FR-001a (variante repetida entre reglas de la misma promoción) | "La variante {nombre} está en más de una regla de esta promoción — cada variante solo puede pertenecer a una." Señala las dos filas de regla en conflicto. |
| 409 FR-014 (solape entre promociones distintas) | "Otra promoción activa ({promotion_name}) ya cubre esta(s) variante(s) en un horario que se cruza" — sin cambio respecto del modelo plano. |
| 409 FR-016 (regla sin descuento) | "Con este precio de paquete, {min_qty} unidades de {variante más barata} costarían {cheapest×min_qty} — esta regla no representa un descuento." Señala la fila de regla. |
| 409 FR-018 (editar reglas de una promoción activa) | "Las reglas de una promoción activa no se pueden cambiar. Duplícala para crear una versión nueva." |

### Edición de `Activa`/`Pausada` (FR-018)

Toda la sección de reglas queda **de solo lectura** (no solo los campos tipo/valor/min_qty/
conjunto de una regla existente — tampoco se pueden agregar ni quitar filas). Los únicos campos
editables son los de vigencia (nombre, descripción, fin de vigencia, días, horario), igual que en
el modelo plano — el bloqueo ya existía, ahora aplica a un bloque más grande.

### Banner de migración (FR-025) — sin cambio

`GET /promotions?closed_by_refactor=true` y el banner descartable siguen igual; las promociones
que quedaron `Finalizada` por la migración del modelo plano (`063a`) siguen listadas ahí sin
cambio — la migración `063c` no vuelve a tocar `closed_by_refactor_at`.
