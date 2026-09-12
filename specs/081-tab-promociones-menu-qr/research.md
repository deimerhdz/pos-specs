# Research: Pestaña dedicada de Promociones en el menú QR

Insumo para [plan.md](./plan.md). Todas las decisiones parten del código real leído en
`pos-backend` y `pos-heladeria` (reconocimiento hecho durante esta sesión de planeación, no
supuestos) — cada una cita el archivo/línea que la sustenta.

## D1 — Sin cambios de backend: filtrado y descuento ya viajan en `GET /menu`

**Decisión**: no se agrega ningún endpoint, campo de respuesta ni migración en `pos-backend`.

**Evidencia**: `app/api/v1/menu/schemas.py` — `MenuVariantResponse.promotion:
MenuVariantPromotion | None`, y `MenuVariantPromotion` (spec 066) ya trae `min_qty`, `type`,
`value`, `condition_text`, `display_text`, todos calculados por el backend. En
`pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts:880`,
`hasPromotion(product)` ya deriva "¿este producto tiene alguna variante en promoción vigente?"
de ese mismo campo (`product.variants.some(v => v.promotion != null)`), sin evaluar reglas ni
vigencia en el cliente — esa evaluación ya la hizo el backend.

**Alternativas consideradas**:
- Endpoint nuevo `GET /menu/promociones` con el catálogo pre-filtrado — rechazado: duplicaría en
  el backend un filtro que el cliente ya puede aplicar sobre datos que de todos modos tiene que
  cargar completos para el resto del menú; el catálogo de una heladería (decenas de productos) no
  justifica paginar ni pre-filtrar en el servidor.
- Campo nuevo `has_promotion: bool` en `MenuProductResponse` — rechazado: sería superficie de
  contrato derivable de un campo que ya viaja (`variants[].promotion`), con riesgo de
  desincronizarse; el propio comentario de `hasPromotion()` en el código ya documenta esta
  decisión para el caso de la insignia (research.md D-11 de spec 066), y aplica igual aquí.

## D2 — Pestaña "Promociones" como categoría virtual, no como sistema paralelo

**Decisión**: se reutiliza el mecanismo de pestañas ya existente (`activeCategoryId` signal,
`selectCategory(id)`, `visibleProducts()` computed — `public-menu.component.ts:729-762`)
agregando una constante reservada (p. ej. `PROMOTIONS_TAB_ID = '__promociones__'`, que nunca
colisiona con un UUID real de categoría) como valor posible de `activeCategoryId`.
`visibleProducts()` gana una rama: cuando `activeCategoryId() === PROMOTIONS_TAB_ID`, en vez de
`activeCategory()?.products`, recorre `categories().flatMap(c => c.products)` y filtra con
`hasPromotion()` (el mismo helper que ya pinta la insignia, sin duplicar el criterio de
elegibilidad).

**Evidencia**: el patrón de "recorrer todas las categorías cuando la vista no es por categoría"
ya existe para la búsqueda (`visibleProducts()`, líneas 754-762: `if (searchOpen() && q) { return
this.categories().flatMap(...) }`) — la pestaña "Promociones" sigue exactamente ese mismo patrón,
con el criterio de filtro cambiado.

**Alternativas consideradas**:
- Un signal/computed independiente (`showPromotions`, con su propio `visiblePromoProducts()`) —
  rechazado: obligaría a duplicar la lógica de "qué grilla se pinta" en la plantilla (dos `@if`
  distintos) en vez de que `visibleProducts()` siga siendo la única fuente de la grilla.

## D3 — Paso de cantidad rastreado por variante+opciones, solo en memoria del navegador

**Decisión**: `DiningCartService` gana un mapa privado `stepByKey: Map<string, number>`, con
clave `` `${productVariantId}::${optionIds.sort().join(',')}` ``, poblado únicamente cuando
`add()` se invoca con un nuevo parámetro opcional `stepQuantity` (pasado solo desde el flujo que
abre `product-select` desde la pestaña "Promociones"). `CartLine` (hoy sin `productVariantId` ni
información de opciones resuelta — `dining-cart.service.ts:12-22`) gana esos dos campos
derivados en `apply()`, para que `cart.component.ts` pueda resolver `cart.stepFor(line)` con la
misma clave al pintar los botones +/-.

**Evidencia**: `dining-cart.service.ts:137-161` (`apply()`) ya tiene `it.product_variant_id` y
`it.options` disponibles en la respuesta cruda del backend (`CartResponse`) pero los descarta al
construir `CartLine` — hoy solo sirven para resolver `productName`/`variantName`/`optionNames`
vía el índice. Extender `apply()` para conservarlos es un cambio aditivo sobre datos que ya
llegan, no una petición nueva.

**Por qué por variante+opciones y no por id de línea de carrito**: el id de una línea nueva
(`cart_items.id`) solo se conoce **después** de que el backend responde al `POST /cart/items`;
identificar cuál de las líneas devueltas es la recién creada requeriría comparar el carrito
antes/después. Usar la clave variante+opciones evita ese diffing: es la misma clave con la que el
backend ya fusiona (o no) líneas repetidas, así que coincide sin importar cómo resuelva el
backend el merge.

**Por qué solo en memoria (no persistido)**: al recargar la página, `DiningCartService.load()`
reconstruye `lines()` desde `GET /cart`, que no sabe nada de "esta línea nació en la pestaña
Promociones" (no es un dato de negocio, es una preferencia de interacción). El mapa se pierde y
la línea vuelve al paso libre de 1 en 1 — el valor por defecto ya existente, sin ningún efecto
sobre el descuento cobrado (D6). Se documenta como limitación conocida, no como pendiente: no
hay ninguna necesidad de negocio que exija sobrevivir a un F5 para una preferencia puramente de
qué tan rápido saltan los botones +/-.

**Alternativas consideradas**:
- Persistir el origen en el backend (`cart_items.added_from_promotions_tab` o similar) —
  rechazado: exige migración y cambio de contrato para un dato que el motor de descuentos no
  necesita (D6); violaría el Principio IX/VIII sin una justificación de negocio detrás.
- Guardar el mapa en `localStorage` para sobrevivir a un F5 — rechazado por desproporcionado: la
  sesión de un comensal en la mesa rara vez implica recargar la página, y si lo hace, degradar a
  cantidad libre (el comportamiento de siempre) no es una regresión perceptible ni afecta el
  cobro.

## D4 — Cantidad inicial y paso del modal de selección (revisada, ver nota 2026-09-12)

**Decisión**: en `product-select.component.ts`, `quantity` (signal, línea 370, hoy siempre
arranca en 1) pasa a mantenerse **reactivamente** alineada al `min_qty` de la variante
**actualmente** seleccionada, no solo a la que estaba seleccionada cuando se abrió el modal —
ver [contracts/pestana-promociones-comportamiento.md §2](./contracts/pestana-promociones-comportamiento.md#2-paso-de-cantidad-al-agregar-por-primera-vez-fr-004-fr-005-fr-011-fr-012--cubre-us2)
para el algoritmo completo (`minQty`/`existingQty` como `computed()`, un `effect()` que
resincroniza `quantity` cada vez que cualquiera de los dos cambia). `increment()`/`decrement()`
(líneas ~795/799, hoy `q => q + 1` / `Math.max(1, q - 1)`) pasan a sumar/restar `minQty()` en vez
de `1`, con el piso ajustado a ese mismo paso (mismo patrón que ya usa el botón, línea 318:
`[disabled]="quantity() === 1"`, comparando contra `minQty()`).

**Evidencia**: `product-select.component.ts:484-509` (`lineTotal`, `packagePrice()`) ya lee
`variant.promotion.min_qty` para calcular el precio de línea en tiempo real mientras el comensal
mueve la cantidad — el dato ya está disponible exactamente donde se necesita el nuevo
comportamiento, sin ninguna petición adicional.

**Cómo sabe el modal que se abrió desde "Promociones"**: `public-menu.component.ts:917`
(`openProduct(product)`) es el único punto que llama `this.selectedProduct.set(product)`; gana un
`@Input() fromPromotions` derivado de `activeCategoryId() === PROMOTIONS_TAB_ID` en el momento
del tap, pasado a `<app-product-select>` (línea 575-579).

**Revisión 2026-09-12 (`/speckit-analyze`, hallazgo C1)**: la decisión original de este documento
pasaba un único número precalculado (`initialQuantityOverride`) para la variante con la que se
abrió el modal, sin cubrir qué pasaba si el comensal cambiaba de variante **dentro** del modal —
el paso de `increment()/decrement()` sí se recalculaba en vivo, pero `quantity` no, pudiendo
quedar en un valor que ya no era múltiplo del nuevo `min_qty` (violaba FR-011 en ese caso). Se
corrige pasando, en cambio, `fromPromotions: boolean` + un closure `existingQtyFor(variantId,
optionKey)`, y recalculando `quantity` con un `effect()` atado a `selectedVariant()` — ver D5 y el
contrato §2 actualizados.

## D5 — Ajuste al múltiplo más cercano (FR-012), reactivo a la variante seleccionada (revisada)

**Decisión**: `product-select.component.ts` recibe `existingQtyFor(variantId, optionKey): number`
— un closure que `public-menu.component.ts` construye a partir de `cart.lines()` en el momento de
abrir el modal (consultando por `productVariantId`/`optionKey`, ver D3) — en vez de un número ya
calculado. Dentro del modal, `existingQty = computed(() =>
existingQtyFor(selectedVariant()?.id ?? '', currentOptionKey()))` se reevalúa cada vez que cambia
la variante u opciones seleccionadas, y el `effect()` de D4 recalcula `quantity` como
`siguienteMúltiplo(existingQty(), minQty()) − existingQty()` — la cantidad que, sumada a lo que ya
hay en el carrito para esa variante+opciones, completa el múltiplo más cercano hacia arriba.
Confirmado el modal, esa cantidad se agrega (no reemplaza) a la línea existente.

**Ejemplo**: variante con 1 unidad ya en el carrito (agregada antes desde su categoría), regla
vigente `min_qty = 2`. Al abrir el modal desde "Promociones" sobre esa variante, `quantity` arranca
en `2 − 1 = 1` (no `2`): al confirmar, el total queda en `2` (alineado), no en `3`. Si el comensal
cambia a otra variante del mismo producto sin unidades previas y con el mismo `min_qty = 2`,
`quantity` se reajusta de inmediato a `2` (no se queda en `1`).

**Alternativas consideradas**: ver Clarifications de spec.md (sesión 2026-09-12, pregunta 4) —
la alternativa "dejar la cantidad existente sin cambios hasta la siguiente pulsación" fue
evaluada y descartada explícitamente por el dueño del producto en esa sesión. Para el cambio de
variante dentro del modal (hallazgo C1 de `/speckit-analyze`), la alternativa "dejar `quantity`
congelada hasta el próximo `increment()/decrement()`" se descartó por la misma razón: dejaría una
cantidad que no es múltiplo del `min_qty` real de la variante ya seleccionada, exactamente el
defecto que esta spec busca eliminar.

## D6 — Ningún cambio de validación en `pos-backend`

**Decisión**: `CartItemIn.quantity: int = Field(1, ge=1)` y `CartItemUpdate.quantity: int | None
= Field(None, ge=1)` (`app/api/v1/cart/schemas.py`) se dejan exactamente igual. No se agrega
ninguna validación de "múltiplo de `min_qty`" en el backend.

**Evidencia**: `evaluate_variant_sets` (motor de descuentos, spec 063) agrupa las unidades
elegibles en bloques completos de `min_qty` **sea cual sea** la cantidad total que reciba —
`_greedy_units` en `app/api/v1/promotions/service.py` no asume que el cliente ya envió múltiplos;
si el frontend manda una cantidad ya alineada, el descuento simplemente cubre el 100% de las
unidades desde el primer cálculo (FR-007), sin que el backend necesite saberlo ni validarlo.

**Por qué no validar también en el backend**: FR-010 exige que agregar desde una categoría normal
siga aceptando **cualquier** cantidad en el mismo endpoint — una validación de múltiplo en el
backend rompería ese requisito, porque el endpoint es el mismo para ambos flujos y el backend no
tiene (ni necesita tener) noción de "desde qué pestaña" se originó la llamada.

## D7 — Botón de navegación: ubicación y estado vacío

**Decisión**: el botón "Promociones" se agrega en `public-menu.component.ts` dentro del mismo
`<nav>` de pestañas de categoría (línea ~244-258), **antes** de la primera categoría del `@for`,
reutilizando las mismas clases Tailwind de pestaña activa/inactiva. Siempre visible (FR-008); si
`visibleProducts()` queda vacío con esa pestaña activa, se reutiliza el mismo patrón de estado
vacío que ya existe para "sin resultados de búsqueda" (línea ~435,
`@else if (visibleProducts().length === 0)`), con el texto ajustado a "No hay promociones activas
en este momento."

**Evidencia**: el `<nav>` de pestañas y el estado vacío de la grilla ya son puntos de extensión
existentes — no se crea ningún contenedor ni patrón visual nuevo.
