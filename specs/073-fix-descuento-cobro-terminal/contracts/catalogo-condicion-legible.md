# Contrato: Condición legible de la promoción en el catálogo de la Terminal

**Cubre**: FR-016, FR-017 — User Story 6 (spec.md).
**Research**: [research.md](../research.md) D9.

Contrato **solo de frontend** — sin cambios de API. `MenuVariantPromotion` (spec 066, FR-007/
FR-008) ya viaja completa hasta el catálogo de la Terminal; este contrato fija cómo se consume
ahí, reemplazando el cálculo local actual.

## Origen del dato (sin cambios)

```
GET /menu/... (o el endpoint que puebla store.categories())
  └─ MenuVariantResponse.promotion?: MenuVariantPromotion
       { condition_text, short_condition, unit_equivalent, unit_equivalent_approx,
         unit_equivalent_text, display_text, type, min_qty, value }
  └─ MenuService (menu.service.ts:37-46, 120-129) — ya lo mapea sin filtrar
  └─ PosTerminalStore.categories() / catalogProducts() — ya lo trae completo
```

## Qué se elimina

`PosTerminalStore.productDiscountBadge()` / `productDiscountBadges()`
(`pos-terminal.store.ts:404-441`): recalculan localmente, recorriendo
`promotionService.activePromotions()` y las reglas de cada una, un texto suelto tipo `-50%` /
`Paquete $20.000` — sin la condición completa (`2 x -50%`) ni el equivalente por unidad
(`≈ $4.000 c/u`) que el backend ya resuelve.

## Qué se agrega — a nivel de tarjeta/grilla (no del modal, que ya usa `condition_text` completo vía `ProductSelectComponent`)

En `pos-catalog-drawer.component.ts:69-70` y `manual-order-page.component.ts:87-89` (los dos
consumidores actuales de `productDiscountBadges()`): por cada producto, buscar la primera
variante con `.promotion` no nulo entre `product.variants`, y pintar su `short_condition` (o
`display_text` si hay espacio) en vez del badge calculado.

```ts
// Reemplaza productDiscountBadges(): computed<Map<string, string>>
private cardPromotionText(variants: MenuVariant[]): string | null {
  const withPromo = variants.find((v) => v.promotion != null);
  return withPromo?.promotion?.short_condition ?? null;
}
```

## FR-017 — sin precio unitario descontado suelto cuando la regla exige más de una unidad

**Ya resuelto por los datos, sin lógica nueva en el frontend**: `MenuVariantPromotion` distingue
`min_qty === 1` (donde sí corresponde mostrar un precio por unidad) de `min_qty > 1` (donde
`unit_equivalent`/`unit_equivalent_text` ya vienen marcados como equivalente, con
`unit_equivalent_approx` cuando no cae exacto en pesos) — la misma resolución que ya usa el menú
QR (spec 066). El catálogo de la Terminal solo necesita pintar `display_text`/`short_condition`
tal cual llegan; no hay ninguna rama `if min_qty === 1` que escribir en este componente porque el
backend ya la resolvió al construir el texto.

## Acceptance Scenarios cubiertos (spec.md, Historia 6)

1. Promoción `2 x -50%` sobre un cono de $8.000 → tarjeta muestra `short_condition`/
   `display_text` con el equivalente (`2 x -50% · ≈ $4.000 c/u`).
2. Regla de paquete `3 x $20.000` → tarjeta muestra esa condición, no un precio unitario suelto.
3. Producto sin promoción vigente → `variant.promotion` es `null` para todas sus variantes →
   `cardPromotionText()` devuelve `null` → tarjeta igual que hoy.
4. Regla con `min_qty = 1` → el propio `unit_equivalent_text` del backend ya no lleva `≈` (no es
   equivalente, es el precio real de una unidad) — se pinta igual, sin rama especial en el
   frontend.
