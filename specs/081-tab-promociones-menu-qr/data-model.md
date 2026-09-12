# Data Model: Pestaña dedicada de Promociones en el menú QR

Sin cambios de modelo de datos persistido: cero tablas, columnas o migraciones nuevas en
`pos-backend` (research.md D1, D6). Este documento describe únicamente las estructuras de
**frontend** (`pos-heladeria`) que la funcionalidad extiende, todas derivadas de datos que el
backend ya sirve hoy.

## Entidades reutilizadas sin cambio (contexto)

Documentadas por specs previas — solo se leen, no se modifican:

- **`MenuVariantPromotion`** (`pos-backend/app/api/v1/menu/schemas.py`, spec 066): `type`,
  `min_qty`, `value`, `condition_text`, `display_text`, etc. Ya viaja en
  `MenuVariantResponse.promotion` dentro de `GET /menu`.
- **`Promotion` / `PromotionRule` / `PromotionVariant`** (`pos-backend/app/models/promotion.py`,
  spec 063): vigencia, tipo, `min_qty`, conjunto de variantes. Fuente de `MenuVariantPromotion`,
  nunca consultada directamente por esta funcionalidad (el frontend solo lee el `promotion` ya
  resuelto por variante).
- **`CartItem`** (`pos-backend/app/models/cart_item.py`): `quantity` (`CheckConstraint("quantity
  > 0")`), sin tope ni regla de múltiplo — sin cambios.

## Estructuras de frontend que cambian

### `MenuProduct` / `MenuVariant` (`product.interface.ts`)

Sin cambios de forma. Se documentan aquí porque son la fuente de la nueva derivación:

- **Producto en promoción** (concepto nuevo, no persistido): `product` tal que
  `product.variants.some(v => v.promotion != null)` — exactamente el criterio que ya usa
  `hasPromotion()` para la insignia (spec 066). La pestaña "Promociones" reutiliza este mismo
  criterio para decidir qué productos listar (research.md D2); no se introduce un campo
  `has_promotion` nuevo (decisión ya tomada en spec 066, D1).

### `CartLine` (`dining-cart.service.ts`) — gana dos campos derivados

```text
export interface CartLine {
  id: string;
  productName: string;
  variantName: string;
  optionNames: string[];
  quantity: number;
  notes: string | null;
  unitPrice: number;
  lineTotal: number;
  // NUEVO — derivados de `CartResponse.items[]`, ya presentes en la respuesta del
  // backend pero descartados hoy al construir la línea (research.md D3):
  productVariantId: string;
  optionKey: string;   // ids de opción elegidas, ordenados y unidos ("id1,id2")
}
```

- **Origen**: `apply()` ya recibe `it.product_variant_id` y `it.options` de `CartResponse`; solo
  deja de descartarlos.
- **Uso**: `cart.component.ts` los usa para pedirle a `DiningCartService` el paso de cantidad de
  esa línea (ver siguiente entidad), sin tocar el backend.
- **Compatibilidad**: campos aditivos — ningún consumidor existente de `CartLine` se rompe por
  ganarlos.

### `DiningCartService.stepByKey` (nuevo, privado, solo en memoria)

```text
private stepByKey: Map<string, number> = new Map();
// clave: `${productVariantId}::${optionIds.slice().sort().join(',')}`
// valor: la `min_qty` de la regla vigente que cubría la variante en el momento
//        del agregado desde "Promociones" (1 = paso libre, valor por defecto
//        cuando la clave no está presente)
```

- **Ciclo de vida**: se escribe únicamente cuando `DiningCartService.add(...)` recibe el nuevo
  parámetro opcional `stepQuantity` (pasado solo desde el flujo de "Promociones", D4); nunca se
  lee ni se escribe desde `GET /cart` — no sobrevive a una recarga de página (research.md D3,
  limitación documentada y aceptada).
- **Método de lectura**: `stepFor(line: CartLine): number` — devuelve
  `this.stepByKey.get(key(line)) ?? 1`.
- **No persistido**: no existe columna, tabla ni campo de `CartResponse` equivalente; es estado
  de interacción del navegador, no un dato de negocio (research.md D3, D6).

### `product-select.component.ts` — dos nuevos `@Input()`

```text
@Input() fromPromotions = false;
// true solo cuando public-menu.component.ts abrió el modal con
// activeCategoryId() === PROMOTIONS_TAB_ID (research.md D4).

@Input() existingQtyFor: (variantId: string, optionKey: string) => number = () => 0;
// Cierre provisto por public-menu.component.ts, ligado a cart.lines() en el momento de
// abrir el modal (research.md D5) — permite recalcular `existingQty` para CUALQUIER
// variante que el comensal seleccione dentro del modal, no solo la inicial.
```

**Revisión 2026-09-12 (`/speckit-analyze`, hallazgo C1)**: la primera versión de este documento
pasaba un único `initialQuantityOverride: number | null` precalculado para la variante
inicialmente seleccionada. Eso dejaba sin definir qué pasaba si el comensal cambiaba de variante
**dentro** del modal después de abrirlo: el paso de `increment()/decrement()` se recalculaba en
vivo (`variant.promotion?.min_qty`) pero `quantity` no, pudiendo quedar en un valor que ya no era
múltiplo del nuevo `min_qty`. Se reemplaza por los dos inputs de arriba, más reactivos:

```text
minQty = computed(() => fromPromotions ? (selectedVariant()?.promotion?.min_qty ?? 1) : 1)
existingQty = computed(() => existingQtyFor(selectedVariant()?.id ?? '', currentOptionKey()))

effect(): cada vez que minQty() o existingQty() cambian (lo que incluye un cambio de
  variante, porque ambos dependen de selectedVariant()):
    quantity.set(
      minQty() === 1
        ? 1
        : (existingQty() % minQty() === 0 ? minQty() : minQty() - (existingQty() % minQty()))
    )

increment(): quantity.update(q => q + minQty())
decrement(): quantity.update(q => Math.max(minQty(), q - minQty()))
```

- **Sin `fromPromotions`** (todo lo demás fuera de "Promociones"): `minQty()` siempre da `1` —
  comportamiento idéntico al actual (FR-010).
- **Con `fromPromotions`**: `quantity` se mantiene alineado al `min_qty` de la variante
  **actualmente** seleccionada en todo momento, incluido un cambio de variante a mitad de sesión
  del modal (FR-011) — no solo en el instante en que se abrió.

## Sin estado de transición ni ciclo de vida adicional

No hay una máquina de estados nueva: la "elegibilidad" de un producto y el "paso" de una línea
son valores derivados, recalculados en cada render a partir de datos que ya existen (`promotion`
por variante, carrito actual) — no hay una entidad con estados propios que persista entre
sesiones más allá de lo que el carrito ya persiste hoy en el backend.
