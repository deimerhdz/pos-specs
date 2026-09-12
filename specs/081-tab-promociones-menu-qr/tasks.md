---

description: "Task list for spec 081 — pestaña dedicada de Promociones en el menú QR"
---

# Tasks: Pestaña dedicada de Promociones en el menú QR

**Input**: Design documents from `/specs/081-tab-promociones-menu-qr/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: incluidos — el plan y el contrato ya comprometen ficheros de test concretos
(Constitución, Principio X, Verificación Obligatoria). No son opcionales en este proyecto.

**Organización**: esta funcionalidad es **100% frontend** (`pos-heladeria`); `pos-backend` no se
toca (plan.md, Constitution Check §III/§VIII/§IX). US1 (P1) es literalmente la base de la que
depende el resto — spec.md lo dice explícito ("Sin esta pestaña, el resto de la funcionalidad no
tiene dónde vivir") — por eso no hay una Fase 2 "Foundational" separada: US1 cumple ese papel.
US2 (P1) construye el paso de cantidad encima de la pestaña que entrega US1. US3 (P2) es
puramente de regresión: confirma que nada de lo anterior cambió el comportamiento fuera de
"Promociones".

## Format: `[ID] [P?] [Story] Description`

- **[P]**: se puede hacer en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: US1, US2 o US3

## Path Conventions

Un solo repo tocado, sibling de `pos-specs`: `../pos-heladeria` (Angular 21). Las rutas de cada
tarea son relativas a la raíz de `pos-heladeria`. Los números de línea citados son los
verificados durante `/speckit-plan` (research.md); confirmar contra el fichero real antes de
editar si el código avanzó.

---

## Phase 1: Setup

**Purpose**: fijar la línea base antes de tocar código.

- [X] T001 Levantar `pos-heladeria` (`npm start`) y confirmar que `npm test` pasa en verde como
      línea base, siguiendo [quickstart.md](./quickstart.md) (prerrequisitos). Anotar el número
      de specs verdes para comparar en Polish.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: N/A para esta spec — no existe infraestructura compartida previa a las historias
más allá de lo que la propia US1 entrega (ver nota de Organización arriba). Se documenta esta
fase vacía a propósito, en vez de forzar una tarea artificial, porque no hay ningún fichero que
ambas historias necesiten tocar **antes** de que exista la pestaña misma.

**Checkpoint**: nada que verificar aquí — continuar directo a la Fase 3.

---

## Phase 3: User Story 1 - Encontrar todos los productos en promoción sin buscar por categoría (Priority: P1) 🎯 MVP

**Goal**: una pestaña "Promociones" en la navegación del menú QR que, al seleccionarla, muestra
únicamente los productos con al menos una variante en promoción vigente, sin importar su
categoría — reutilizando el mismo criterio que ya pinta la insignia "🎉 Promo"
([contracts/pestana-promociones-comportamiento.md §1](./contracts/pestana-promociones-comportamiento.md#1-elegibilidad-de-producto-fr-002-fr-008--cubre-us1)).

**Independent Test**: con al menos una promoción vigente sobre productos de distintas
categorías, abrir el menú QR, presionar "Promociones" y verificar que solo aparecen esos
productos; pausar todas las promociones y verificar el aviso de "no hay promociones activas".

### Tests for User Story 1

> Escribir estos tests primero; deben fallar antes de implementar.

- [X] T002 [P] [US1] Escribir tests en
      `src/app/modules/tables/pages/public-menu.component.spec.ts` para: (a) la pestaña
      "Promociones" aparece siempre en la navegación aunque no haya ninguna promoción vigente
      (FR-001, FR-008); (b) seleccionarla filtra `visibleProducts()` a solo los productos con
      `variants.some(v => v.promotion != null)`, mezclando productos de dos o más categorías
      distintas (FR-002); (c) con cero productos elegibles muestra el aviso "No hay promociones
      activas en este momento." en vez de una grilla vacía sin explicación (FR-008); (d)
      seleccionar una categoría normal después sigue mostrando todos sus productos sin filtrar
      por promoción, igual que antes de esta spec (FR-009); (e) un producto con promoción
      renderizado dentro de "Promociones" sigue mostrando la insignia "🎉 Promo" y su condición,
      igual que en su categoría original (FR-003, `/speckit-analyze` hallazgo C3) — deben fallar
      antes de implementar

### Implementation for User Story 1

- [X] T003 [US1] Agregar la constante `PROMOTIONS_TAB_ID` (sentinel que nunca colisiona con un
      UUID de categoría real, p. ej. `'__promociones__'`) en
      `src/app/modules/tables/pages/public-menu.component.ts`, y extender el computed
      `visibleProducts()` (líneas ~754-762) con una rama: cuando
      `activeCategoryId() === PROMOTIONS_TAB_ID`, devolver
      `categories().flatMap(c => c.products).filter(p => hasPromotion(p))` en vez de
      `activeCategory()?.products` — reutilizando `hasPromotion()` (línea ~880) sin modificarlo
      ([research.md D2](./research.md#d2--pestaña-promociones-como-categoría-virtual-no-como-sistema-paralelo)) —
      depende de T002
- [X] T004 [US1] Agregar el botón "Promociones" en el `<nav>` de pestañas de categoría
      (`public-menu.component.ts`, líneas ~244-258), **antes** del `@for` de categorías,
      reutilizando las clases Tailwind de pestaña activa/inactiva ya existentes y llamando a
      `selectCategory(PROMOTIONS_TAB_ID)` — implementa FR-001 ([research.md D7](./research.md#d7--botón-de-navegación-ubicación-y-estado-vacío)) —
      depende de T003
- [X] T005 [US1] Agregar la rama de estado vacío para la pestaña "Promociones" en la plantilla
      (junto al `@else if (visibleProducts().length === 0)` existente de la búsqueda, línea
      ~435), con el texto "No hay promociones activas en este momento." cuando
      `activeCategoryId() === PROMOTIONS_TAB_ID` — implementa FR-008 — depende de T003

### Ajustes tras probar en un entorno real (2026-09-12)

- [X] T019 [P] [US1] Escribir tests en `public-menu.component.spec.ts` para: (a) con
      `activeCategoryId() === PROMOTIONS_TAB_ID`, `activeCategory()` MUST devolver `null` (no la
      primera categoría) — corrige que ambas pestañas aparecieran resaltadas a la vez; (b) al
      cargar el menú con al menos un producto en promoción vigente, `activeCategoryId()` MUST
      quedar en `PROMOTIONS_TAB_ID` sin que el comensal presione nada (FR-014); (c) al cargar el
      menú sin ninguna promoción vigente, `activeCategoryId()` MUST quedar en `null` (primera
      categoría, comportamiento sin cambio) — deben fallar antes de implementar
- [X] T020 [US1] En `public-menu.component.ts`: (1) `activeCategory()` (líneas ~766-773) retorna
      `null` de una vez cuando `activeCategoryId() === PROMOTIONS_TAB_ID`, antes de intentar
      `find()`/caer a `categories()[0]` — evita que la primera categoría se pinte también como
      activa; (2) en `ngOnInit`, justo después de `this.categories.set(categories)` (línea ~839),
      si `categories.some(c => c.products.some(p => this.hasPromotion(p)))` MUST llamar
      `this.activeCategoryId.set(PROMOTIONS_TAB_ID)` — implementa FR-014 — depende de T019

**Checkpoint**: US1 funciona de forma independiente y es verificable por separado — la pestaña
lista solo productos en promoción, con el mismo comportamiento de siempre en el resto del menú, y
es la vista inicial cuando hay algo en promoción al entrar.

---

## Phase 4: User Story 2 - Agregar al carrito respetando las unidades mínimas de la promoción (Priority: P1)

**Goal**: agregar o ajustar un producto desde la pestaña "Promociones" siempre mueve la cantidad
en pasos completos de la cantidad mínima de su regla vigente (2 en 2 para "2 x $17.000"), con el
descuento ya aplicado desde el primer agregado
([contracts/pestana-promociones-comportamiento.md §2-3](./contracts/pestana-promociones-comportamiento.md#2-paso-de-cantidad-al-agregar-por-primera-vez-fr-004-fr-005-fr-012--cubre-us2)).

**Independent Test**: desde "Promociones", agregar un producto cubierto por una regla de precio
de paquete "2 x $17.000" y verificar que el carrito queda en cantidad 2 (no 1) con el precio de
paquete aplicado; subir con "+" salta a 4; bajar con "−" desde 2 retira la línea.

### Tests for User Story 2

> Escribir estos tests primero; deben fallar antes de implementar.

- [X] T006 [P] [US2] Escribir tests en
      `src/app/modules/tables/services/dining-cart.service.spec.ts` para: `CartLine` expone
      `productVariantId`/`optionKey` derivados de `CartResponse.items[]` (data-model.md); `add()`
      acepta un `stepQuantity` opcional y, tras una respuesta exitosa, `stepFor(line)` devuelve
      ese valor para la línea correspondiente (por clave variante+opciones, no por id de línea);
      `stepFor(line)` devuelve `1` para cualquier línea sin `stepQuantity` registrado — deben
      fallar antes de implementar
- [X] T007 [P] [US2] Escribir tests en
      `src/app/modules/tables/components/product-select.component.spec.ts` para (contrato §2,
      algoritmo reactivo revisado tras `/speckit-analyze` hallazgo C1): (a) con
      `fromPromotions == false` (valor por defecto), `quantity` arranca en 1 y el paso es 1, sin
      importar `variant.promotion` (FR-005/FR-010, sin cambio); (b) con `fromPromotions == true`
      y la variante inicial cubierta por `min_qty == 2` y `existingQtyFor` devolviendo `0`,
      `quantity` arranca en 2 y `increment()/decrement()` avanzan de 2 en 2, sin bajar de un paso
      completo (FR-004); (c) con `fromPromotions == true` y `min_qty == 1`, `quantity` arranca en
      1 (FR-005, aunque `fromPromotions` sea `true`); (d) con `fromPromotions == true`,
      `min_qty == 2` y `existingQtyFor` devolviendo `1` para la variante inicial, `quantity`
      arranca en `1` (múltiplo más cercano hacia arriba: `2 − 1`), no en `2` (FR-012, hallazgo
      C2); (e) **cambiando `selectedVariant()` dentro del modal** de una variante sin promoción a
      una con `min_qty == 2` (o viceversa), `quantity` se reajusta de inmediato al nuevo `minQty`
      vía el `effect()`, sin esperar a `increment()/decrement()` (FR-011, hallazgo C1) — deben
      fallar antes de implementar
- [X] T008 [P] [US2] Crear `src/app/modules/tables/components/cart.component.spec.ts` (hoy no
      existe ningún test para este componente) probando: el botón "+" emite
      `line.quantity + cart.stepFor(line)`; el botón "−" emite
      `line.quantity - cart.stepFor(line)`; con `stepFor(line) === line.quantity` (línea
      justamente en su cantidad mínima), el resultado emitido es `0` y `DiningCartService`
      retira la línea (reutilizando la lógica ya existente de `setQuantity`, sin cambiarla) —
      deben fallar antes de implementar

### Implementation for User Story 2

- [X] T009 [US2] Extender `CartLine` y `apply()` en
      `src/app/modules/tables/services/dining-cart.service.ts` (líneas 12-22 y 137-161) para
      conservar `productVariantId: it.product_variant_id` y
      `optionKey: it.options.map(o => o.option_id).sort().join(',')` — hoy se descartan al
      construir la línea ([data-model.md](./data-model.md#cartline-dining-cartservicets--gana-dos-campos-derivados)) —
      depende de T006
- [X] T010 [US2] Agregar el mapa privado `stepByKey: Map<string, number>` y el método
      `stepFor(line: CartLine): number` (`stepByKey.get(key(line)) ?? 1`) en
      `dining-cart.service.ts`; extender `add()` con un parámetro opcional `stepQuantity` que,
      tras una respuesta exitosa de `api.addItem`, registra
      `stepByKey.set(key(variantId, options), stepQuantity)`
      ([research.md D3](./research.md#d3--paso-de-cantidad-rastreado-por-varianteopciones-solo-en-memoria-del-navegador)) —
      depende de T009
- [X] T011 [US2] En `src/app/modules/tables/components/product-select.component.ts`: reemplazar
      el diseño de un solo valor precalculado por dos `@Input()` — `fromPromotions = false` y
      `existingQtyFor: (variantId: string, optionKey: string) => number = () => 0` — más dos
      `computed()` (`minQty`, `existingQty`, ambos dependientes de `selectedVariant()`) y un
      `effect()` que resincroniza el signal `quantity` (línea 370) cada vez que `minQty()` o
      `existingQty()` cambian, siguiendo el algoritmo de
      [contracts/pestana-promociones-comportamiento.md §2](./contracts/pestana-promociones-comportamiento.md#2-paso-de-cantidad-al-agregar-por-primera-vez-fr-004-fr-005-fr-011-fr-012--cubre-us2);
      `increment()`/`decrement()` (líneas ~795/799) pasan a sumar/restar `minQty()` en vez de `1`,
      con el piso del botón "−" ajustado a `minQty()` (mismo patrón que
      `[disabled]="quantity() === 1"`, línea 318); el payload de `added.emit(...)` (interfaz
      `ProductSelection`) gana `stepQuantity?: number`, poblado con `minQty()` solo cuando
      `fromPromotions && minQty() > 1` — implementa FR-004/FR-005/FR-011/FR-012
      ([research.md D4/D5](./research.md#d4--cantidad-inicial-y-paso-del-modal-de-selección-revisada-ver-nota-2026-09-12),
      corregido por `/speckit-analyze` hallazgo C1) — depende de T007
- [X] T012 [US2] En `src/app/modules/tables/pages/public-menu.component.ts`, extender
      `openProduct(product)` (línea ~917) para pasar dos nuevos inputs a `<app-product-select>`
      (línea 575-579): `[fromPromotions]="activeCategoryId() === PROMOTIONS_TAB_ID"` y
      `[existingQtyFor]` con un closure construido sobre `cart.lines()` en ese instante, que
      busca por `productVariantId`/`optionKey` (T009) y devuelve `0` si no encuentra línea — sin
      cálculo de múltiplo aquí (ese cálculo vive ahora en T011, dentro del propio modal) —
      depende de T009, T011
- [X] T013 [US2] Actualizar `onProductAdded()` (línea ~975) para reenviar
      `selection.stepQuantity` (T011) tal cual a `cart.add(...)` — implementa FR-004/FR-007 —
      depende de T010, T011, T012
- [X] T014 [US2] Actualizar los dos botones +/- de
      `src/app/modules/tables/components/cart.component.ts` (líneas ~43 y ~52) para emitir
      `line.quantity + cart.stepFor(line)` / `line.quantity - cart.stepFor(line)` en vez del
      `+ 1`/`- 1` fijo actual — implementa FR-004/FR-006 (contrato §3) — depende de T010

**Checkpoint**: US1 y US2 funcionan juntas y por separado — agregar desde "Promociones" siempre
mueve la cantidad en pasos completos de `min_qty`, con el descuento aplicado desde el primer
agregado.

### Ajustes tras probar en un entorno real (2026-09-12, segunda ronda)

- [X] T021 [P] [US2] Escribir tests en `product-select.component.spec.ts` para: (a) con
      `fromPromotions == true` y una promoción vigente cubriendo la variante seleccionada, el
      precio del encabezado (`data-testid="header-price"`) MUST ser igual a `lineTotal()` (el
      total con descuento de la cantidad ya configurada), no `startingPrice()` (precio de lista
      de una unidad); (b) con `fromPromotions == false`, el encabezado MUST seguir mostrando
      `startingPrice()`, sin cambio (regresión) — deben fallar antes de implementar
- [X] T022 [US2] En `product-select.component.ts`: agregar `data-testid="header-price"` al precio
      del encabezado (línea ~90) y cambiar su binding a
      `(fromPromotions ? lineTotal() : startingPrice()) | money`; ocultar la insignia "Desde"
      (línea ~88) cuando `fromPromotions` (deja de ser un precio "desde", ya es el total exacto a
      cobrar) — implementa FR-015 — depende de T021

**Checkpoint**: el modal abierto desde "Promociones" muestra un único precio consistente entre el
encabezado y el botón "Agregar", para cualquier producto de una o varias presentaciones.

- [X] T023 [P] [US2] Escribir tests en `product-select.component.spec.ts` para: (a) con
      `fromPromotions == true` y una presentación cubierta por una regla con `min_qty > 1`, la
      fila de esa presentación MUST mostrar `v.price` tachado junto al precio del paquete
      completo (`min_qty` unidades), y el texto de condición de esa fila MUST mostrar
      `short_condition` ("2 x $7.000"), no `display_text` (sin el "· $3.500 c/u"); (b) con
      `fromPromotions == false` (Terminal, orden manual, o esta misma pestaña de "Promociones"
      antes de esta revisión), la fila MUST seguir mostrando `display_text` completo (con
      "c/u") y el precio de lista sin tachar, sin cambio (regresión) — deben fallar antes de
      implementar
- [X] T024 [US2] En `product-select.component.ts`: (1) el texto de condición de cada fila de
      presentación (línea ~132-136) muestra `fromPromotions ? v.promotion.short_condition :
      v.promotion.display_text`; (2) se agrega una rama nueva antes del `@else` final del precio
      de la fila (línea ~163), para `fromPromotions && v.promotion` (y `discountFor(v)` es
      `null`, el caso de `min_qty > 1`): tacha `v.price` y muestra en negrita el precio de un
      paquete completo, calculado con el mismo `packagePrice()` privado que ya usa `lineTotal()`
      (pasando `promo.min_qty` como cantidad, para obtener el total exacto de un solo paquete
      sin importar la cantidad configurada) — implementa FR-016 — depende de T023

**Checkpoint**: la fila de presentación, dentro de "Promociones", muestra el mismo lenguaje de
precio (tachado + total) en toda la funcionalidad, sin la cifra por unidad; el resto de
superficies que comparten este componente no cambian.

### Corrección de FR-015/FR-016 (2026-09-12, tercera ronda de prueba)

**Motivo**: T022/T024 gatillaban el precio con descuento con `fromPromotions`, pero el dueño
encontró el producto por el buscador (`fromPromotions == false`) y subió la cantidad a mano hasta
el mínimo — el encabezado y la fila deberían haber mostrado el descuento igual, sin importar por
dónde se llegó al modal. `fromPromotions` sigue existiendo y sigue gobernando **solo** el paso
forzado de cantidad (T011); estas dos tareas reemplazan su rol en la parte de **precio mostrado**
por una condición basada en si `quantity()` ya satisface `min_qty` de la variante elegida.

- [X] T025 [P] [US2] Reescribir en `product-select.component.spec.ts` los tests de T021(b)/T023(b)
      para que, en vez de fijar `fromPromotions: false` sin más, abran el modal sin
      `fromPromotions` y luego llamen `component.inc()` manualmente hasta que `quantity()`
      alcance `min_qty` — deben confirmar que **ahí sí** aparece el descuento (encabezado =
      `lineTotal()`, fila con tachado + `short_condition`), y que con `quantity() < min_qty`
      (sin incrementar) sigue el comportamiento de lista, sin descuento — deben fallar antes de
      implementar
- [X] T026 [US2] En `product-select.component.ts`: agregar `readonly packageDealActive =
      computed(() => { const promo = this.selectedVariant()?.promotion; return !!promo &&
      this.quantity() >= promo.min_qty; })` y el método `rowPackagePromo(v: MenuVariant):
      MenuVariantPromotion | null` (`v.id === this.variantId() && this.quantity() >=
      (v.promotion?.min_qty ?? Infinity) ? v.promotion : null`); reemplazar toda referencia a
      `fromPromotions` en el binding del precio del encabezado (T022), la insignia "Desde" (T022)
      y las dos ramas de T024 (condición de la fila y texto de la insignia) por
      `packageDealActive()`/`rowPackagePromo(v)` respectivamente — `fromPromotions` deja de
      tener ningún efecto sobre qué precio se muestra, solo sigue controlando el paso de
      cantidad — depende de T025

**Checkpoint**: el precio con descuento (encabezado + fila) aparece exactamente cuando la
cantidad configurada satisface la regla vigente, sin importar si el modal se abrió desde
"Promociones", una categoría o el buscador — el paso forzado de cantidad sigue exclusivo de
"Promociones".

---

## Phase 5: User Story 3 - Las categorías normales del menú no cambian su comportamiento (Priority: P2)

**Goal**: confirmar que agregar o ajustar la cantidad de un producto desde su categoría original
(no desde "Promociones") sigue siendo libre, unidad por unidad, exactamente como antes de esta
spec — sin que el paso por cantidad mínima de US2 se filtre a ese flujo.

**Independent Test**: agregar un producto con una regla de precio de paquete de cantidad mínima
2 desde su categoría (no desde "Promociones"); confirmar que queda en cantidad 1 y que los
botones +/- del carrito suman/restan de a 1, no de a 2.

- [X] T015 [P] [US3] Escribir un test de regresión en
      `src/app/modules/tables/pages/public-menu.component.spec.ts` que confirme que
      `openProduct(product)` pasa `fromPromotions == false` a `<app-product-select>` cuando
      `activeCategoryId() !== PROMOTIONS_TAB_ID` (FR-010) — depende de T012
- [X] T016 [P] [US3] Escribir un test de regresión en
      `src/app/modules/tables/components/cart.component.spec.ts` que confirme que
      `cart.stepFor(line)` devuelve `1` para una línea agregada sin `stepQuantity` (flujo de
      categoría normal), y que los botones +/- de esa línea suman/restan de a 1 (FR-009/FR-010)
      — depende de T014

**Checkpoint**: las tres historias funcionan juntas y por separado — sin regresión en el
comportamiento de cantidad libre fuera de "Promociones" (SC-004).

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T017 [P] Correr la batería completa: `python -m unittest discover -s
      app/characterization_tests -p 'test_*.py' -v` en `../pos-backend` (debe seguir exactamente
      igual que la línea base — cero archivos tocados ahí) y `npm test` en `pos-heladeria`
      (comparar contra el conteo anotado en T001) — [quickstart.md, Verificación cruzada
      final](./quickstart.md#verificación-cruzada-final)
- [ ] T018 Ejecutar el recorrido manual de [quickstart.md](./quickstart.md) por las tres
      historias (Historia 1, 2 y 3) en un entorno real, confirmando SC-001 a SC-004, el caso de
      ajuste al múltiplo de FR-012 (Historia 2, paso 6) y el cambio de variante dentro del modal
      de FR-011 (Historia 2, paso 7)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: vacía — no bloquea nada (ver nota de Organización).
- **US1 (Phase 3)**: depende de Phase 1. Es el único prerrequisito real de US2 (la pestaña debe
  existir para que `openProduct()` sepa que se abrió "desde Promociones").
- **US2 (Phase 4)**: depende de Phase 3 (T003, T012 usa `PROMOTIONS_TAB_ID`) para el flujo
  completo de extremo a extremo (abrir desde la pestaña real). A nivel de unidad, T009-T011 y
  T014 son autónomos y se prueban en aislamiento sin la UI de US1 (T007 ejercita
  `product-select.component.ts` pasándole `fromPromotions`/`existingQtyFor` directamente, sin
  pasar por `public-menu.component.ts`) — ver plan.md, Constitution Check §V, para esta misma
  distinción entre "componente en aislamiento" y "historia completa".
- **US3 (Phase 5)**: depende de Phase 4 (T012, T014) — son tests de regresión sobre el código que
  US2 introduce; no agrega implementación propia.
- **Polish (Phase 6)**: depende de que las tres historias deseadas estén completas.

### Within Each User Story

- Tests se escriben y deben fallar antes de implementar (Principio X).
- Servicio (`DiningCartService`) antes que componentes que lo consumen (`cart.component.ts`,
  `product-select.component.ts` vía `public-menu.component.ts`).
- Historia completa y verificada antes de pasar a la siguiente prioridad.

### Parallel Opportunities

- T002 (test de US1) puede escribirse mientras se prepara el entorno de T001.
- T006, T007 y T008 (los tres bloques de test de US2) son archivos distintos y pueden escribirse
  en paralelo entre sí.
- T015 y T016 (regresión de US3) son archivos distintos y pueden escribirse en paralelo entre sí.
- T017 (backend) y la mitad de T017 (frontend) pueden lanzarse en paralelo al ser suites
  independientes, aunque viven en la misma tarea por simplicidad de reporte.

---

## Parallel Example: User Story 2

```bash
# Lanzar los tres bloques de test de la Historia 2 en paralelo:
Task: "Tests de CartLine/stepFor en dining-cart.service.spec.ts (T006)"
Task: "Tests de quantity/paso en product-select.component.spec.ts (T007)"
Task: "Tests de +/- en cart.component.spec.ts (T008, archivo nuevo)"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup.
2. Completar Phase 3: User Story 1 (T002-T005).
3. **Detenerse y validar**: la pestaña "Promociones" lista correctamente los productos elegibles
   y respeta el estado vacío — ya es demostrable al dueño, aunque agregar todavía sea libre.

### Incremental Delivery

1. Setup → Phase 3 (US1) → validar independientemente → demo (MVP: se puede *encontrar* la
   promoción, aunque agregar aún sea de a 1).
2. Phase 4 (US2) → validar independientemente → demo (agregar ya respeta la cantidad mínima).
3. Phase 5 (US3) → confirma que nada de lo anterior rompió el resto del menú.
4. Phase 6 (Polish) → batería completa + recorrido manual → listo para cerrar la spec.
