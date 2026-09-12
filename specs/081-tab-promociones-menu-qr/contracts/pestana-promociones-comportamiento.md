# Contrato: comportamiento de la pestaña "Promociones" (frontend)

Sin contrato de API nuevo — `GET /menu`, `POST /cart/items` y `PATCH /cart/items/{id}` no cambian
de forma (research.md D1, D6). Este documento es el contrato **interno** entre las piezas de
`pos-heladeria` que esta spec modifica, para que `/speckit-tasks` pueda descomponerlo sin volver
a leer el código fuente.

## 1. Elegibilidad de producto (FR-002, FR-008) — cubre US1

**Entrada**: `categories: MenuCategory[]` (ya cargado por `GET /menu`), `activeCategoryId`.

**Regla**:

```text
si activeCategoryId == PROMOTIONS_TAB_ID:
    productosVisibles = categories.flatMap(c => c.products)
                                   .filter(p => hasPromotion(p))   // ya existe, spec 066
    si productosVisibles.length == 0:
        mostrar estado vacío: "No hay promociones activas en este momento."
si no:
    productosVisibles = comportamiento actual (sin cambios, FR-009)
```

- `hasPromotion(p)` **no cambia** (`p.variants.some(v => v.promotion != null)`).
- El orden de `productosVisibles` es el orden natural de recorrido de categorías → productos
  (no se re-ordena ni se agrupa por promoción).
- La pestaña "Promociones" es siempre visible en la navegación, incluso con
  `productosVisibles.length == 0` (FR-008) — nunca se oculta ni se deshabilita.

**Salida esperada**: la grilla de productos (`public-menu.component.ts`, template) no cambia de
estructura — solo cambia qué lista de productos recibe.

## 2. Paso de cantidad al agregar por primera vez (FR-004, FR-005, FR-011, FR-012) — cubre US2

**Entrada**: `fromPromotions: boolean` (el modal se abrió con `activeCategoryId() ===
PROMOTIONS_TAB_ID`); `existingQtyFor(variantId, optionKey): number` (closure hacia `cart.lines()`
en el momento de abrir el modal, `0` si no hay ninguna unidad todavía).

**Regla — reactiva a la variante seleccionada dentro del modal, no solo a la inicial**
(`/speckit-analyze` 2026-09-12, hallazgo C1 — corrige la versión anterior de este contrato, que
solo calculaba la cantidad una vez para la variante con la que se abrió el modal):

```text
minQty = computed(() => fromPromotions ? (selectedVariant()?.promotion?.min_qty ?? 1) : 1)
existingQty = computed(() => existingQtyFor(selectedVariant()?.id ?? '', currentOptionKey()))

// Recalcula `quantity` cada vez que `minQty()` o `existingQty()` cambian — lo que incluye
// cualquier cambio de variante u opción dentro del modal, porque ambos dependen de
// `selectedVariant()`/`currentOptionKey()`:
effect(): quantity.set(
    minQty() == 1
        ? 1                                                          // FR-005, FR-010
        : (existingQty() % minQty() == 0
            ? minQty()
            : minQty() - (existingQty() % minQty()))                 // FR-012
)

increment(): quantity.update(q => q + minQty())
decrement(): quantity.update(q => Math.max(minQty(), q - minQty()))  // nunca baja de un paso completo
```

- **FR-011 (variantes mixtas dentro de un mismo producto)**: como `minQty`/`existingQty` se
  recalculan a partir de `selectedVariant()`, cambiar de variante dentro del modal (p. ej. de una
  cubierta por la regla a una que no lo está) reajusta `quantity` de inmediato al nuevo `minQty`
  — nunca queda "arrastrando" el valor calculado para la variante anterior.
- Con `fromPromotions == false` (todo lo que no sea esta pestaña), `minQty()` siempre es `1`:
  comportamiento idéntico al actual, sin ninguna rama nueva (FR-010).

**Al confirmar** (`added.emit(...)` → `DiningCartService.add(...)`):

```text
DiningCartService.add(product, variant, options, quantity, notes, stepQuantity = fromPromotions ? minQty() : undefined)
```

- Si `stepQuantity` viene definido, `DiningCartService` registra
  `stepByKey.set(key(variant.id, options), stepQuantity)` **después** de que `POST /cart/items`
  responda con éxito (para no registrar un paso sobre un agregado que el backend rechazó, p. ej.
  por falta de stock).
- El total resultante en el carrito, tras confirmar, siempre es múltiplo de `minQty` (SC-002),
  salvo que ya hubiera unidades no múltiplo agregadas *después* del cálculo (condición de carrera
  fuera de alcance — ver quickstart.md, no es un flujo que la UI permita en un solo dispositivo).

## 3. Paso de cantidad en el carrito ya creado (FR-004, FR-006) — cubre US2, US3

**Entrada**: `line: CartLine` (con `productVariantId`/`optionKey`, data-model.md), botón "+"/"−"
presionado en `cart.component.ts`.

**Regla**:

```text
paso = DiningCartService.stepFor(line)   // stepByKey.get(key(line)) ?? 1

"+"  → DiningCartService.setQuantity(line.id, line.quantity + paso)
"−"  → DiningCartService.setQuantity(line.id, line.quantity - paso)
       // setQuantity ya retira la línea si el resultado es <= 0 (dining-cart.service.ts:102-105,
       // sin cambio) — con paso == line.quantity (línea justo en el mínimo) esto cubre FR-006
       // sin lógica adicional.
```

- Líneas sin paso registrado (`stepFor(line) == 1`, incluye todo lo agregado desde una categoría
  normal — FR-010, y todo lo que sobrevivió a una recarga de página — research.md D3) se
  comportan exactamente igual que hoy: +1/−1.
- No existe un tope superior de cantidad distinto del que ya aplique el backend (stock,
  disponibilidad) — esta funcionalidad no introduce ninguno.

## 4. No objetivos explícitos (FR-013, alcance de Clarifications)

- **No** se combina cantidad entre productos/variantes distintos de un mismo grupo de regla para
  completar `min_qty` — cada tarjeta/línea se resuelve de forma aislada, mirando únicamente su
  propia variante+opciones (FR-013). Esa combinación sigue funcionando, sin cambios, agregando
  cada producto desde su categoría original.
- **No** se agrega ninguna validación de cantidad en `pos-backend` (research.md D6) — todo el
  contrato de esta sección es responsabilidad exclusiva del frontend.
- **No** se persiste el origen "agregado desde Promociones" más allá de la sesión del navegador
  (research.md D3) — es intencional, no un pendiente.
