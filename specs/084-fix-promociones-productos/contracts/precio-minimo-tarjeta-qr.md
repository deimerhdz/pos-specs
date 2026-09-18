# Contrato: Precio mínimo visible en la tarjeta de promoción del menú QR (bug 1)

Cubre spec.md FR-012 a FR-015. **Sin cambio de backend** (research.md D6) — el dato ya viaja en
`MenuVariantResponse.promotion` (`app/api/v1/menu/schemas.py`, clase `MenuVariantPromotion`),
poblado por `menu_variant_promotion` (`app/api/v1/promotions/service.py:357-411`) para cualquier
`min_qty`, servido desde `app/api/v1/menu/router.py:184`.

## Forma del dato ya disponible (referencia, sin cambio)

```
MenuVariantResponse:
  ...
  promotion: MenuVariantPromotion | null
    display_text: str          # p. ej. "2 x $15.000 · $7.500 c/u"
    unit_equivalent: Decimal   # p. ej. 7500 — precio equivalente por unidad, normalizado
    short_condition: str
    ...
```

`MenuProductResponse.variants: list[MenuVariantResponse]` — cada variante trae su propia
`promotion` (o `null` si ninguna regla vigente la cubre).

## Frontend — nuevo helper (`promotion-pricing.util.ts`)

```ts
function minPromoPriceForProduct(variants: MenuVariantResponse[]): MinPromoPrice | null {
  const covered = variants.filter(v => v.promotion != null);
  if (covered.length === 0) return null;
  const cheapest = covered.reduce((a, b) =>
    a.promotion!.unit_equivalent <= b.promotion!.unit_equivalent ? a : b
  );
  return {
    displayText: cheapest.promotion!.display_text,
    isMinimum: covered.length > 1,   // más de una variante cubierta → mostrar "Desde $X"
  };
}
```

No toca `effectivePrice`/`discountInfo` existentes (siguen cubriendo el caso de precio unitario con
descuento de spec 066 FR-015 / spec.md FR-014, sin cambio).

## Frontend — tarjeta (`public-menu.component.ts`)

- `hasPromotion(product)` (línea ~928-930): sin cambio — sigue controlando la insignia "🎉 Promo".
- `productDiscount()` (línea ~906-914): cuando no hay un precio unitario con descuento (caso hoy
  cubierto por `discountInfo`, `min_qty == 1`), se consulta `minPromoPriceForProduct(product.variants)`
  y, si devuelve un resultado, se muestra junto a la insignia — con el prefijo "Desde " cuando
  `isMinimum` es `true` (más de una variante cubierta con precios distintos), sin prefijo cuando es
  una sola variante cubierta.
- Rama `@else` (línea ~504-506, hoy solo precio normal sin ningún indicio de descuento): pasa a
  mostrar el resultado de `minPromoPriceForProduct` cuando exista, antes de caer al precio normal
  sin más.
- Producto sin ninguna variante con `promotion != null`: sin cambio — ni insignia ni precio de
  promoción (spec.md FR-015).

## Casos de aceptación cubiertos

- Única variante cubierta por "2 x $15.000" → tarjeta muestra "2 x $15.000 · $7.500 c/u" (sin
  prefijo "Desde", una sola opción).
- Dos variantes del mismo producto cubiertas por reglas de la misma promoción con precios
  equivalentes por unidad distintos → tarjeta muestra el más bajo, con "Desde $X".
- Variante cubierta por porcentaje con `min_qty=1` (caso spec 066 FR-015) → sin cambio, precio
  tachado + precio vigente como hoy (no pasa por `minPromoPriceForProduct`).
- Producto sin ninguna variante cubierta → sin insignia, sin precio de promoción.
