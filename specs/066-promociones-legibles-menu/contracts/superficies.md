# Contrato: qué pinta cada superficie (FR-013 a FR-018)

**Normativo.** Define el reparto de responsabilidades en el frontend. La regla que gobierna todo
el documento: **el frontend pinta lo que llega y compara importes; no recalcula ninguno**
(FR-007).

---

## 0. Mapa de superficies

| Superficie | Fichero | Origen de los datos | Qué gana |
|---|---|---|---|
| Cartel del menú QR | `public-menu.component.ts:356-368` | `promotions[].rules[].text` | Texto por nombres (automático: el backend ya cambió). |
| Tarjeta del menú QR | `public-menu.component.ts:376-411` | `variants[].promotion` | Insignia genérica `🎉 Promo` (FR-013). |
| Modal de presentaciones | `product-select.component.ts` (**compartido**) | `variants[].promotion` + `discounted_price` | Línea de promoción por presentación (FR-007). |
| Terminal — catálogo | `pos-terminal.store.ts:410-424` | `activePromotions()` | **Nada. No cambia** (FR-016). |
| Terminal — modal | `product-select.component.ts` (**el mismo**) | `variants[].promotion` | Condición + equivalente (FR-016), **sin importes** (FR-017). |
| Administración — listado | `promotions-page.component.ts:228` | `r.condition_text` | Texto por nombres (automático). |
| Administración — vista previa | `promotions-page.component.ts:1070` | Cálculo local | Texto por nombres seleccionados (FR-018). |

---

## 1. Interfaces del frontend

```typescript
// src/app/modules/products/interfaces/product.interface.ts

export interface MenuVariantPromotion {
  condition_text: string;
  short_condition: string;
  unit_equivalent: number;
  unit_equivalent_approx: boolean;
  unit_equivalent_text: string;
  display_text: string;
  type: 'percent' | 'package_price';
  min_qty: number;
  value: number;
}

export interface MenuVariant {
  ...
  discounted_price?: number | null;
  discount_kind?: string | null;
  /** Info de la regla vigente que cubre esta presentación (FR-007), o `null`.
   *  La calcula el backend: aquí solo se pinta. */
  promotion?: MenuVariantPromotion | null;
}
```

### Mapeo por transporte — **la diferencia importa**

| Transporte | Superficie | `promotion` | `discounted_price` / `discount_kind` |
|---|---|---|---|
| `diner.service.ts:352-362` | Comensal (QR) | **mapear** | mapear (ya lo hace) |
| `menu.service.ts:88-110` | Terminal (staff) | **mapear** | **NO mapear** (sigue como hoy) |

`MenuService` hoy **descarta** `discounted_price` — por eso el modal de la terminal muestra precio
normal. Se mantiene descartándolo: mapearlo haría que `effectivePrice` empezara a mostrar precios
con descuento en la terminal, lo que choca con FR-017 y con la spec 063 FR-023 (el importe de la
terminal lo resuelve el **preview del cobro**, `discounted_unit_price` de la línea) y sería un
cambio de comportamiento sin decisión de negocio registrada. Ver
[research.md D-10](../research.md).

**Test de no-regresión obligatorio**: tras el cambio, el modal de la terminal sobre una variante
con `package_price min_qty 1` vigente sigue mostrando el **precio normal** y el total del botón
sigue siendo `variant.price`.

---

## 2. Modal de presentaciones — `ProductSelectComponent` (FR-007, FR-016)

Componente **compartido** por `public-menu.component.ts:498` (comensal),
`manual-order-page.component.ts:308` y `pos-catalog-drawer.component.ts:84` (cajero). Se cambia
**una vez, sin ramas por superficie**: SC-005 se cumple por construcción.

### 2.1 Fila de presentación

Estructura de cada fila (`:63-93`), en orden:

1. Nombre de la presentación (sin cambio).
2. Precio, con esta prioridad:
   - `v.available === false` → `Agotado` (sin cambio, gana a todo);
   - `discountFor(v)` no nulo → **precio normal tachado + precio vigente**;
   - en cualquier otro caso → `variantPrice(v)`.
3. **Nuevo**: si `v.promotion`, una línea secundaria bajo el precio con
   `v.promotion.display_text`, en tipografía menor y tono discreto (no compite con el precio).

### 2.2 La insignia de porcentaje se acota a `percent`

Hoy `:80-86` pinta `-{{ disc.percent }}%` para cualquier `kind` distinto de `'fixed'`. Se añade la
guarda `discount_kind === 'percent'`.

**Por qué**: con `package_price min_qty 1` (`$8.000 → $6.000`), el porcentaje mostrado sería un
`-25%` **fabricado por el frontend** a partir de dos importes — un número que la regla nunca
enuncia y que FR-007 prohíbe. Ese caso muestra tachado + precio vigente + su línea de promoción,
sin porcentaje.

El valor `'fixed'` que la plantilla todavía contempla pertenece a un tipo retirado por la spec 063
y no llega nunca; se deja como está para no mezclar limpieza no relacionada (Principio V).

### 2.3 Tachado: una comparación, no un recálculo (FR-010, FR-015)

`discountFor(v)` → `discountInfo(v.price, v.discounted_price, v.discount_kind)`
(`promotion-pricing.util.ts:82-100`) **no se toca**. Ya devuelve `null` cuando
`discounted >= original`, que es exactamente FR-015:

| `discounted_price` vs `price` | `discountInfo` | Lo que se ve |
|---|---|---|
| menor | objeto | Tachado + precio vigente + (si `percent`) insignia |
| mayor o igual | `null` | **Solo** el precio vigente (= valor de la regla), sin tachado ni señal de descuento |
| `null` | `null` | Precio normal |

En la tercera fila, `variantPrice(v)` → `effectivePrice(price, discounted_price)` → el valor de la
regla. Es lo que FR-010 pide: **lo mostrado coincide con lo cobrado**, sin anunciar un aumento
como ahorro.

Comparar dos importes que ya vinieron calculados no es recalcularlos
([research.md D-7](../research.md)).

### 2.4 Total del botón de agregar

`:310-311` ya usa `effectivePrice(variant.price, variant.discounted_price)`. Con FR-010 poblado,
el total del botón pasa a usar el valor de la regla **automáticamente** en el menú QR, y **no**
cambia en la terminal (no mapea `discounted_price`). Sin cambio de código.

### 2.5 Condición completa para la terminal (FR-016)

Bajo la lista de presentaciones, cuando la presentación seleccionada tenga `promotion`, se muestra
`promotion.condition_text` completo. Se pinta en las dos superficies: al comensal le refuerza el
cartel, y al cajero le da lo que FR-016 exige (*"la condición en lenguaje llano … con el mismo
texto que ve el comensal"*) sin abrir una rama por superficie.

---

## 3. Tarjeta del menú QR (FR-013 a FR-015)

Solo `public-menu.component.ts`. **No aplica a la terminal.**

### 3.1 Insignia genérica

```typescript
/** FR-013 (A-67): un producto tiene promoción si alguna de sus presentaciones
 *  trae información de promoción del backend. No evalúa reglas ni vigencia:
 *  eso ya lo hizo el backend al poblar `promotion`. */
hasPromotion(product: MenuProduct): boolean {
  return product.variants.some((v) => v.promotion != null);
}
```

En `:382-389`, el bloque `@if (productDiscount(product); as disc)` que hoy pinta
`🏷️ -{{ amountOff }}` / `🏷️ -{{ percent }}%` se **reemplaza** por:

```html
@if (hasPromotion(product)) {
  <span class="absolute top-2 right-2 …">🎉 Promo</span>
}
```

Una sola insignia, igual para porcentaje y para paquete, sin distinguir tipo (FR-013). Aparece
aunque no haya precio unitario con descuento que mostrar (FR-014) — que es el caso que hoy no
produce ninguna señal.

**No se añade `has_promotion` al DTO del producto**: se deriva de un campo que ya viaja
([research.md D-11](../research.md)).

### 3.2 El precio tachado se conserva (FR-015)

El bloque de `:402-406` (`productDiscount` → tachado + precio con descuento) **no se toca**.
`productDiscount()` (`:775`) tampoco: sigue eligiendo la presentación de menor precio efectivo y
delegando en `discountInfo`, con la misma consecuencia de §2.3 — si el valor de la regla no es
menor que el precio normal, la tarjeta muestra el precio vigente sin tachado y la insignia
genérica queda como única señal.

Lo que cambia es el reparto: `productDiscount` deja de gobernar **si hay insignia** y solo
gobierna **si hay tachado**.

### 3.3 `priceLabel` (`:752-756`)

Sin cambios de código. Como ya usa `effectivePrice`, con FR-010 poblado el "Desde $X" de un
producto con `package_price min_qty 1` vigente pasa a reflejar el valor de la regla —
automáticamente, y es lo correcto: es el importe que se cobra.

---

## 4. Terminal de staff (FR-016, FR-017)

| Elemento | Cambio |
|---|---|
| Insignia por producto (`productDiscountBadge`, `pos-terminal.store.ts:410-424`) | **Ninguno.** Conserva `-10%` / `Paquete $12.000`. La insignia genérica de FR-013 es exclusiva del menú QR. |
| Modal de configuración | Gana la condición y el equivalente vía `ProductSelectComponent` (§2), porque `MenuService` mapea `promotion`. |
| Importes | **Ninguno.** `MenuService` no mapea `discounted_price`; `addDraftFromSelection` (`:1289`) sigue usando `sel.variant.price`; el descuento efectivo sigue llegando del preview del cobro (`itemUnitPrice`, `:399-403`). |

FR-017 se cumple estructuralmente: la terminal no recibe ningún importe con descuento del menú, así
que no tiene con qué calcular uno.

---

## 5. Administración (FR-018)

### 5.1 Listado

`promotions-page.component.ts:228` ya pinta `{{ r.condition_text || '—' }}`. **Sin cambios de
código**: recibe el texto nuevo cuando el backend cambia. Es lo que hace que SC-005 se cumpla entre
menú, terminal y listado sin coordinación adicional.

### 5.2 Vista previa del formulario

`ruleConditionPreview` se reescribe sobre `promotion-condition.util.ts`
([texto-condicion.md §6](./texto-condicion.md)):

```typescript
ruleConditionPreview(ruleIndex: number): string {
  const rule = this.form.rules[ruleIndex];
  const names = this.selectedVariantsForRule(ruleIndex)
    .map((v) => (v.variantName?.trim() || v.productName?.trim() || ''))
    .filter((n) => n !== '');
  return conditionText(rule, names, rule.variantIds.length);
}
```

- Se calcula sobre las variantes **seleccionadas en ese momento**, no sobre las guardadas (FR-018).
- La plantilla pasa de `ruleConditionPreview(rule)` (`:554`) a `ruleConditionPreview($index)`;
  `$index` ya está en ese alcance (`:556` lo usa).
- La lista completa de variantes con su precio normal que el resumen ya muestra
  (`:556-559`, spec 063 FR-005) **no cambia**.

---

## 6. Resumen de lo que NO cambia en el frontend

- `promotion-pricing.util.ts`: `inTimeWindow`, `isPromoActiveNow`, `effectivePrice`,
  `discountInfo`, `getPromoDisplay` — ninguna se toca.
- `pos-terminal.store.ts`: `productDiscountBadge`, `itemUnitPrice`, `addDraftFromSelection`.
- `cart.component.ts` y el carrito del comensal: los importes los sigue resolviendo el backend.
- `MenuPromotionAnnouncement` y su render en el cartel: la forma del DTO no cambia, solo el
  contenido de `text`.
