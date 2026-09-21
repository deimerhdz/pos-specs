---

description: "Task list template for feature implementation"
---

# Tasks: Corrección de Bugs en Promociones y Productos

**Input**: Documentos de diseño de `/specs/084-fix-promociones-productos/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md) (D0–D12), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: Este proyecto usa *characterization tests* (`app/characterization_tests/`, `python -m unittest`) en `pos-backend` como árbitro de comportamiento (Principio III), no TDD clásico — las tareas de test se generan junto a cada endpoint/lógica nueva, no antes. `pos-heladeria` usa `ng test` sobre los `*.spec.ts` ya existentes (o nuevos cuando el archivo tocado no tenía spec).

**Organización**: Las tareas se agrupan por historia de usuario (US1 → US6, mismo orden de prioridad que `spec.md`) para que cada una sea implementable y probable de forma independiente, sobre una rama nueva desde `develop` en `../pos-backend` y `../pos-heladeria` (spec.md §Assumptions, plan.md §Branch).

## Formato: `[ID] [P?] [Story] Descripción`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: A qué historia de usuario pertenece (US1–US6)
- Cada tarea incluye la ruta exacta del archivo, relativa a `../pos-backend`, `../pos-heladeria` o `pos-specs` (esta carpeta) según corresponda

## Convenciones de ruta

- `../pos-backend/...` — API FastAPI + SQLAlchemy + Alembic
- `../pos-heladeria/...` — SPA Angular 21 (standalone)
- Sin prefijo — archivo dentro de este repo (`pos-specs`), p. ej. `specs/000-reconocimiento/registro-de-anomalias.md`

**⚠️ Advertencia de concurrencia**: US2, US4, US5 y US6 modifican el **mismo archivo**
`../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts` (y su spec). Son
independientes en cuanto a comportamiento y se pueden implementar en cualquier orden, pero **no en
paralelo entre sí** sin coordinar merges — implementarlas de forma secuencial (o con cuidado de
conflictos) evita pisarse el archivo. US1 y US3 no comparten archivo con ninguna otra historia y sí
pueden avanzar en paralelo con cualquiera de las demás.

---

## Phase 1: Setup

**Purpose**: Preparar ambos repos para el trabajo de esta spec.

- [X] T001 Crear una rama nueva desde `develop` en `../pos-backend` para esta funcionalidad (nombre según convención del equipo, independiente del número de esta carpeta — mismo criterio que spec 083) — `fix/084-promociones-productos`
- [X] T002 [P] Crear una rama nueva desde `develop` en `../pos-heladeria` para esta funcionalidad — `fix/084-promociones-productos`
- [X] T003 [P] Ejecutar la suite de characterization tests en `../pos-backend` (`python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`) y confirmar que está en verde antes de empezar (quickstart.md Paso 0) — 870 tests, verde
- [X] T004 [P] Ejecutar `ng test` en `../pos-heladeria` y confirmar que está en verde antes de empezar (quickstart.md Paso 0) — **no estaba verde**: `promotions-page.component.spec.ts` tenía un error de compilación preexistente (`screen.set('form')`, valor removido por spec 083) que bloqueaba correr *cualquier* test del proyecto; corregido en un commit aparte (`7657243`, fuera del alcance de spec 084). Tras corregirlo, quedan **19 tests fallando en 6 archivos no relacionados** con esta spec (auth, tenant/super-admin, checkout/order panel, terminal de menú) — deuda preexistente de `develop`, documentada aquí, no se toca. Ninguno de los archivos que tocan las historias de esta spec (products, promotions, presentations, public-menu) tenía fallas propias de esta spec en la línea base.

---

## Phase 2: Foundational

**Purpose**: Verificaciones que condicionan el diseño de más de una historia, sin bloquear su implementación en sí (las seis historias de esta spec son independientes entre sí, plan.md §Constitution Check, Principio VI).

- [X] T005 [P] Confirmar que la cabeza de Alembic en `../pos-backend` sigue siendo `da7581f7bb18` (`alembic heads`) antes de escribir la migración de US1 — si cambió, ajustar `down_revision` en T007 — confirmado, sin cambios
- [X] T006 [P] Confirmar en `../pos-heladeria/package.json` que `@angular/cdk` sigue disponible (`^21.2.14` o superior) y sin uso previo de `Overlay`/`CdkMenu` en el repo, antes de US6 (research.md D5) — confirmado (`^21.2.14`, `@angular/cdk/drag-drop` en uso, `@angular/cdk/overlay` sin uso previo)

**Checkpoint**: Verificaciones listas — cualquier historia de usuario puede empezar, en cualquier orden.

---

## Phase 3: User Story 1 - Asociar cada variante con una presentación del catálogo (Priority: P1) 🎯 MVP

**Goal**: El formulario de crear/editar producto permite asociar cada variante con una presentación del catálogo, con el nombre sincronizado (spec.md FR-001–007).

**Independent Test**: crear o editar un producto, agregar una variante, seleccionar una presentación activa del catálogo, verificar que el nombre se autocompleta; renombrar esa presentación desde "Presentaciones" y verificar que el nombre de la variante se actualiza también.

### Implementación para User Story 1

- [X] T007 [US1] Crear migración Alembic `<rev>_084_variante_presentacion.py` en `../pos-backend/app/alembic/versions/` (`down_revision='da7581f7bb18'`): `ADD COLUMN presentation_id` (FK nullable a `presentations.id`, `ON DELETE SET NULL`) + `UNIQUE (product_id, presentation_id)` por tenant (`@for_each_tenant_schema`), con `downgrade` que hace `DROP CONSTRAINT`/`DROP COLUMN` — ver data-model.md — `a9f7d0310f6b` (ruta real: `alembic/versions/`, sin prefijo `app/` — corregido respecto al plan)
- [X] T008 [US1] Agregar columna `presentation_id`, relationship `presentation` y el `UniqueConstraint` nuevo a `ProductVariant` en `../pos-backend/app/models/product_variant.py` (depende de T007)
- [X] T009 [P] [US1] Agregar `presentation_id: UUID | None` a `VariantSaveIn` y `VariantResponse` en `../pos-backend/app/api/v1/catalog/schemas.py`
- [X] T010 [US1] Implementar en `_save_variant_entry` (`../pos-backend/app/api/v1/catalog/service.py`): validar que la presentación exista y esté activa, rechazar con 409 si otra variante del mismo producto ya la usa (FR-006), y tomar `name` de `Presentation.name` cuando `presentation_id` no es nulo (FR-002/003) — depende de T008, T009
- [X] T011 [US1] Implementar en `PATCH /presentations/{id}` (`../pos-backend/app/api/v1/presentations/router.py`): antes de la cascada, validar que ningún producto con una variante asociada a esta presentación tenga ya otra variante (sin esta presentación) con `name` igual al nuevo nombre — si la hay, rechazar con 409 sin ejecutar ningún `UPDATE` (spec.md FR-004, edge case); si no la hay, ejecutar la cascada `UPDATE product_variants SET name = :nuevo_nombre WHERE presentation_id = :id`, misma transacción que el `UPDATE` de `Presentation` — depende de T008 (ver data-model.md §Efecto en cascada)
- [X] T012 [P] [US1] Agregar `app/characterization_tests/test_products_variant_presentation.py` en `../pos-backend` cubriendo: asociar/reasociar/quitar presentación (FR-001–003/005), unicidad por producto (409, FR-006), cascada de renombre exitosa (FR-004), y el 409 de colisión de nombre contra una variante sin presentación del mismo producto (FR-004, edge case) — depende de T010, T011 — 10 tests, verde; 880/880 en la suite completa del backend
- [X] T013 [P] [US1] Agregar `presentationId: string | null` a `VariantDraft` y `VariantSavePayload` en `../pos-heladeria/src/app/modules/products/interfaces/product.interface.ts` — también `Variant.presentation_id` y `DeactivatedVariant.presentationId`, no previstos en el plan pero necesarios para que el ciclo completo (cargar/restaurar/guardar) compile
- [X] T014 [US1] Agregar `<select>` de presentación por fila de variante en `../pos-heladeria/src/app/modules/products/pages/product-form.component.ts` (reusa `PresentationService.allPresentations` filtrado a `active`; al elegir una, el `<input>` de nombre pasa a solo lectura y se autocompleta; en "Sin presentación" vuelve a ser editable) — depende de T013. Alcance acotado al modo "con tamaños" (grid); el modo "Único" (una sola variante, sin tabla) no muestra el selector, mismo criterio que ya usa esa pantalla para ocultar el nombre — decisión de UX tomada durante la implementación, no bloqueante para ningún FR
- [X] T015 [P] [US1] Actualizar `../pos-heladeria/src/app/modules/products/pages/product-form.component.spec.ts` cubriendo selección/cambio/limpieza de presentación y el bloqueo del campo nombre — depende de T014 — 7 tests nuevos + 3 tests preexistentes ajustados (`product.service.spec.ts` y este archivo) por el campo nuevo en el payload; 41/41 verde; suite completa del frontend: mismas 19 fallas preexistentes de T004, cero nuevas

**Checkpoint**: US1 funcional y verificable de forma independiente (quickstart.md, sección US1).

---

## Phase 4: User Story 2 - Una promoción activa no se puede reconfigurar sin pausarla primero (Priority: P1)

**Goal**: El botón "Configurar" del listado queda deshabilitado para toda promoción con `status=active`, y el backend rechaza editarla por esa vía (spec.md FR-008–011).

**Independent Test**: activar una promoción de prueba, confirmar que "Configurar" queda deshabilitado; pausarla; confirmar que vuelve a estar disponible con todos los campos editables.

### Implementación para User Story 2

- [X] T016 [US2] Registrar la anomalía **A-76** en `specs/000-reconocimiento/registro-de-anomalias.md` (bloqueo total de "Configurar"/`service.update()` en `active`, reemplaza la edición parcial de spec 083 FR-018 — citar spec.md §Clarifications y research.md D9) — debe existir antes de mergear T017/T018
- [X] T017 [US2] Agregar `if promo.status == "active": raise HTTPException(409, "Pausa la promoción antes de modificarla")` al inicio de `update()` en `../pos-backend/app/api/v1/promotions/service.py` (línea ~762, respalda `PATCH /promotions/{promotion_id}`, `router.py:81`) — **no** reutilizar la condición de `update_shape()` (`status not in ("draft", "paused")`), porque también bloquearía `finished` y violaría FR-009 (research.md D9) — depende de T016
- [X] T018 [US2] Actualizar `test_ca1_editar_escalares_de_una_activa` y `test_editar_vigencia_de_promocion_multi_regla_afecta_a_todas_con_una_accion` en `../pos-backend/app/characterization_tests/test_promotions_rules_admin.py` para esperar `HTTPException` 409 en vez de éxito al llamar `service.update()` sobre una promoción `active`, citando **A-76**; confirmar que el resto del archivo (`test_ca2_cambiar_reglas_de_una_activa_bloquea`, `test_ca4_duplicar_copia_borrador_con_las_mismas_reglas`, los de `TestUS5MantenimientoPorLote`) sigue en verde sin tocarse (Principio III) — depende de T017 — agregado además `test_ca1b_editar_escalares_de_una_pausada_sigue_permitido` (cobertura positiva de FR-009); 881/881 en la suite completa del backend
- [X] T019 [US2] Agregar `[disabled]="p.status === 'active'"` al botón "Configurar" y una guarda equivalente al inicio de `openEdit(p)` en `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts` (mismo criterio que `canDelete(p)`) — depende de T016 — método `canConfigure(p)` nuevo, mismo patrón que `canDelete(p)`
- [X] T020 [P] [US2] Actualizar `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.spec.ts` cubriendo: deshabilitado en `active` (incluida vigencia vencida/"Fuera de horario"), habilitado en `draft`/`paused`/`finished`, y habilitado de nuevo tras pausar sin recargar — depende de T019 — 3 tests nuevos; 25/26 verde (1 skip preexistente, spec082)

**Checkpoint**: US2 funcional y verificable de forma independiente (quickstart.md, sección US2).

---

## Phase 5: User Story 3 - La tarjeta de un producto en promoción muestra un precio (Priority: P2)

**Goal**: La tarjeta de producto de la pestaña "Promociones" del menú QR muestra el precio (o precio mínimo) de la regla vigente, sin tocar el backend (spec.md FR-012–015).

**Independent Test**: con una regla de precio de paquete "2 x $15.000" vigente, abrir la pestaña "Promociones" y verificar que la tarjeta muestra ese precio sin abrir el producto.

### Implementación para User Story 3

- [X] T021 [US3] Agregar el helper `minPromoPriceForProduct(variants)` en `../pos-heladeria/src/app/modules/promotions/services/promotion-pricing.util.ts` (toma el `unit_equivalent` más bajo entre `variant.promotion` no nulos, marca `isMinimum` cuando hay más de una variante cubierta) — ver contracts/precio-minimo-tarjeta-qr.md
- [X] T022 [US3] Usar `minPromoPriceForProduct()` en `productDiscount()`/la rama `@else` de la tarjeta en `../pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts`, mostrando "Desde $X" cuando `isMinimum` es verdadero — depende de T021 — nuevo método `minPromoPrice(product)`, rama `@else if` agregada al template
- [X] T023 [P] [US3] Actualizar `../pos-heladeria/src/app/modules/promotions/services/promotion-pricing.util.spec.ts` cubriendo `minPromoPriceForProduct` (una variante cubierta, varias con precios distintos, ninguna cubierta) — depende de T021 — 5 tests nuevos, 12/12 verde
- [X] T024 [P] [US3] Agregar/actualizar `../pos-heladeria/src/app/modules/tables/pages/public-menu.component.spec.ts` (crear el archivo si no existe) cubriendo la tarjeta con `min_qty >= 2` y el caso ya cubierto de `min_qty = 1` sin cambio — depende de T022 — **el archivo ya existía** (550 líneas, contradice la premisa de la tarea); un primer intento con Write lo sobrescribió por error, restaurado en un commit aparte (`f61ced6`); se agregaron 4 tests reusando `carta()`/`promocion()` ya existentes, 28/28 verde

**Checkpoint**: US3 funcional y verificable de forma independiente (quickstart.md, sección US3).

---

## Phase 6: User Story 4 - Definir una regla la aplica de una vez a todos los productos ya seleccionados (Priority: P2)

**Goal**: `addRuleRow()` genera N reglas independientes (una por producto coincidente, con exclusión por casilla) en vez de una regla combinada (spec.md FR-016–020).

**Independent Test**: seleccionar 3 productos que comparten una variante, definir la regla una vez, desmarcar uno en la lista de confirmación, verificar que se generan 2 filas independientes.

### Implementación para User Story 4

- [X] T025 [US4] Registrar la anomalía **A-77** en `specs/000-reconocimiento/registro-de-anomalias.md` (reemplazo de la regla combinada de `addRuleRow()` — comportamiento en producción desde spec 083 — por N reglas independientes; citar spec.md §Clarifications) — debe existir antes de mergear T026/T027 — registrada junto con A-76/A-78 en el commit `9cef914` de este repo
- [X] T026 [US4] Refactorizar `addRuleRow()`/`resolvedVariantIdsForLabel` en `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts` para mostrar una lista de productos coincidentes con casilla premarcada y generar, al confirmar, una `PromotionRuleForm` independiente por cada producto marcado (sin fila para los no coincidentes ni para los desmarcados) — depende de T025 — `matchingProductsForLabel()`, `pendingBulkApply`, `confirmBulkApply()`/`cancelBulkApply()`/`toggleBulkApplyCandidate()` nuevos
- [X] T027 [US4] Confirmar que las validaciones por fila ya existentes (mínimo 2 unidades, precio < suma de precios regulares, spec 083 FR-025/026) se evalúan de forma individual sobre cada fila generada, usando el precio regular propio de la variante de cada producto — mismo archivo, depende de T026 — confirmado: `packagePriceCheck`/`packagePriceExceedsRegularSum` ya usan el precio más barato entre los coincidentes (peor caso), así que si el precio pasa el chequeo pasa para todos los N productos; no requirió cambios
- [X] T028 [P] [US4] Actualizar `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.spec.ts` cubriendo: generación de N filas, exclusión por checkbox, edición/eliminación individual sin afectar las demás — depende de T026, T027 — 1 test reescrito (comportamiento viejo de regla combinada) + 4 tests nuevos; 29/30 verde (1 skip preexistente)

**Checkpoint**: US4 funcional y verificable de forma independiente (quickstart.md, sección US4).

---

## Phase 7: User Story 5 - Un producto no puede quedar cubierto por dos promociones vigentes a la vez (Priority: P1)

**Goal**: Nueva guarda de servicio que impide seleccionar, en una promoción distinta, un producto que ya tiene alguna variante cubierta por otra promoción `active` (spec.md FR-021–025).

**Independent Test**: con una promoción `active` que cubre la variante "Grande" de un producto, intentar seleccionar ese mismo producto (por su variante "Pequeña") en el Paso 1 de una promoción distinta y verificar que el sistema lo impide.

### Implementación para User Story 5

- [X] T029 [US5] Registrar la anomalía **A-78** en `specs/000-reconocimiento/registro-de-anomalias.md` (exclusividad de producto entre promociones `active`, restricción nueva no retroactiva; citar spec.md §Clarifications) — debe existir antes de mergear T030/T032 — registrada junto con A-76/A-77 en el commit `9cef914` de este repo
- [X] T030 [US5] Implementar `_guard_product_overlap` en `../pos-backend/app/api/v1/promotions/service.py`, junto a `_guard_variant_overlap`: compara `product_id` de las variantes de la promoción en curso contra las de otras promociones con `status="active"` (excluyendo la promoción en edición), rechaza con 409 y el nombre de la promoción que ya lo cubre (FR-021/022/024) — depende de T029
- [X] T031 [US5] Invocar `_guard_product_overlap` desde `create()` y `update_shape()` en el mismo archivo, inmediatamente después de `_guard_variant_overlap` (FR-023) — depende de T030 — **también invocada desde `change_status()` al activar** (tercer call site que ya usa `_guard_variant_overlap`, no contemplado explícitamente en la descripción de esta tarea pero necesario para la condición de carrera de FR-023: dos promociones en `draft` sin conflicto entre sí que se activan casi al mismo tiempo)
- [X] T032 [P] [US5] Agregar `app/characterization_tests/test_promotions_product_overlap.py` en `../pos-backend` cubriendo: bloqueo por producto completo, exención dentro de la misma promoción (FR-024), rechazo en condición de carrera al guardar, y no-retroactividad frente a promociones `active` ya existentes (FR-025) — depende de T031 — 8 tests nuevos; además se actualizaron 2 tests preexistentes en `test_promotions_rules_admin.py` cuyos escenarios la nueva guarda ahora rechaza legítimamente (ventanas horarias disjuntas sobre el mismo producto, y reusar una variante distinta del mismo producto que ya cubre una promoción activa) — 891/891 en la suite completa del backend
- [X] T033 [US5] Agregar el parámetro `exclude_active_promotions`/anotación `blocked_by_promotion` al endpoint que alimenta el buscador de productos del Paso 1 en `../pos-backend/app/api/v1/products/` (o `promotions/`, según contracts/exclusividad-producto-promociones.md) — depende de T030 — **implementado sin cambios de backend**: el Paso 1 ya arma su catálogo desde `MenuService`/`GET /menu`, no `GET /products`; en cambio se reusa `PromotionService.activePromotions` (ya cargada para el POS, `status=active`, tope 100, spec 063) y se cruza en el cliente contra `catalogVariants()` — la alternativa que contracts/exclusividad-producto-promociones.md ya dejaba prevista como equivalente
- [X] T034 [US5] En el Paso 1 de `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`, excluir/deshabilitar productos con `blocked_by_promotion` y mostrar la razón (nombre de la promoción activa que ya lo cubre) — depende de T033 — `blockedByPromotion` computed + `blockedReason()`, tarjeta deshabilitada con tooltip, guarda en `toggleProductCandidate()`
- [X] T035 [P] [US5] Actualizar `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.spec.ts` cubriendo la exclusión de productos bloqueados en el Paso 1 y el mensaje explicativo — depende de T034 — 4 tests nuevos; 33/34 verde (1 skip preexistente, spec082)

**Checkpoint**: US5 funcional y verificable de forma independiente (quickstart.md, sección US5).

---

## Phase 8: User Story 6 - Abrir el menú de acciones de una fila no desplaza ni recorta la tabla (Priority: P3)

**Goal**: El menú ⋮ de `promotions-page.component.ts` migra a Angular CDK Overlay, sin quedar sujeto al `overflow` de la tabla (spec.md FR-026–028).

**Independent Test**: con suficientes filas para que haya scroll, abrir el menú de una fila cercana al borde del contenedor y verificar que se muestra completo, sin recorte, y se cierra al hacer scroll.

### Implementación para User Story 6

- [X] T036 [US6] Reemplazar el `<div>` absoluto del menú de acciones por `CdkOverlayOrigin`/`cdkConnectedOverlay` (`@angular/cdk/overlay`) en `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`, con `scrollStrategy: close()` y cierre en `backdropClick()`/resize — ver contracts/menu-acciones-sin-scroll.md — **sin backdrop**: el cierre por clic afuera ya lo cubría el `@HostListener('document:click')` existente (con `stopPropagation()` en el botón y el panel), agregar un backdrop de CDK habría sido redundante; se agregó `@HostListener('window:resize')` en su lugar
- [X] T037 [US6] Adaptar `toggleActionsMenu`/`closeActionsMenu` al ciclo de vida del overlay (abrir/cerrar/backdrop) en el mismo archivo — depende de T036 — `(detach)="closeActionsMenu()"` sincroniza `openActionsId` cuando la estrategia `close()` desconecta el overlay por scroll
- [X] T038 [P] [US6] Actualizar `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.spec.ts` cubriendo: menú visible sin recorte cerca del borde, cierre al hacer scroll/clic fuera/resize — depende de T037 — 3 tests nuevos (overlay fuera del contenedor con scroll, cierre limpio, cierre en resize); 36/37 verde (1 skip preexistente, spec082)

**Checkpoint**: US6 funcional y verificable de forma independiente (quickstart.md, sección US6).

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final de conjunto (Principio X).

- [X] T039 [P] Ejecutar la suite completa de characterization tests en `../pos-backend` y confirmar que `test_promotions_service.py` y `test_promotions_router.py` siguen en verde sin edición, y que `test_promotions_rules_admin.py` solo tiene los dos cambios explícitos de T018 (motor de cálculo intacto, research.md D0/D9) — 891/891, confirmado sin ediciones fuera de las citadas
- [X] T040 [P] Ejecutar `ng test` completo en `../pos-heladeria` y confirmar cero regresiones — 934/954 (19 fallas preexistentes de `develop`, idénticas antes/después, confirmado en 3 corridas consecutivas)
- [X] T041 Ejecutar el checklist final de [quickstart.md](./quickstart.md) de punta a punta para las seis historias — checklist actualizado con el estado real (`test_promotions_rules_admin.py` sí requirió edición explícita, a diferencia de lo previsto originalmente)
- [X] T042 Confirmar que las anomalías **A-76**, **A-77** y **A-78** están registradas en `specs/000-reconocimiento/registro-de-anomalias.md` antes de abrir los PR correspondientes (Principio II) — confirmado, commit `9cef914`

---

## Phase 10: Enmienda 2026-09-20 — User Story 7 (nombre derivado de la presentación) y User Story 8 (orden de la tarjeta)

**Contexto**: las Fases 1–9 están completas (T001–T042). Esta fase se agrega tras una revisión del
formulario ya implementado; sustituye parte del comportamiento de US1 (spec.md §Enmienda, A-79).
Documentación de diseño ya escrita: spec.md FR-029–FR-039, [data-model.md](./data-model.md)
§Enmienda, [contracts/variante-sin-nombre.md](./contracts/variante-sin-nombre.md),
[contracts/formulario-tamanos-orden.md](./contracts/formulario-tamanos-orden.md), research.md D10–D12.

**Orden de despliegue** (research.md D10, contrato no retrocompatible): **Paso A** backend aditivo →
**Paso B** frontend → **Paso C** backend destructivo. Las tareas de abajo están agrupadas por paso;
en desarrollo pueden escribirse en la misma rama, pero cada paso se despliega por separado.

> **Estado 2026-09-20**: se implementó directamente el estado final (Paso C) en las ramas de
> desarrollo `fix/084-promociones-productos` de ambos repos, para poder ver el resultado completo;
> los cambios no están commiteados. Antes de **desplegar** a producción hay que decidir si se
> reintroduce la escalera A→B→C (mantener `name` en las respuestas un ciclo de despliegue) o si se
> acepta un corte con el menú QR en caché mostrando variantes sin nombre hasta que se actualice el
> service worker. La migración se aplicó a la BD de desarrollo.

**Independent Test (US7)**: aplicar la migración sobre un tenant con variantes de nombre libre y
sin presentación; abrir un producto y verificar que no hay campo "Nombre", que cada tamaño muestra
su presentación y que los nombres visibles en menú QR/carrito/promociones no cambiaron; renombrar
una presentación y ver el cambio reflejado sin otra acción.
**Independent Test (US8)**: con "Maneja inventario" apagado, la tabla de tamaños se ve completa
justo bajo el encabezado y el interruptor queda debajo de ella.

### Preparación

- [X] T043 [US7] Confirmar la cabeza de Alembic en `../pos-backend` (`alembic heads`; hoy `a9f7d0310f6b`, o la que corresponda si la rama de spec 084 ya se fusionó) y **repetir** el inventario de lecturas de nombre de variante con `grep` en `../pos-backend/app` (excluyendo `characterization_tests`/`scripts`) y `../pos-heladeria/src` — completar la tabla de contracts/variante-sin-nombre.md con cualquier lectura no listada — cabeza confirmada: `a9f7d0310f6b`; inventario de lecturas repetido con `grep` (coincide con la tabla del contrato)
- [X] T044 [P] [US7] Registrar la anomalía **A-79** en `specs/000-reconocimiento/registro-de-anomalias.md` (Principio II) — registrada el 2026-09-20 junto con esta documentación; debe existir antes de mergear T052/T057

### Paso A — backend aditivo (sin migración)

- [X] T045 [US7] Agregar la propiedad `ProductVariant.presentation_name` y declarar `presentation` con `lazy="joined"` en `../pos-backend/app/models/product_variant.py`; agregar `presentation_name` a `VariantResponse`/`VariantSaveOut` (`catalog/schemas.py`, `products/schemas.py`) y `presentation_name`+`presentation_id` a `MenuVariantResponse` (`menu/schemas.py`), **manteniendo `name`**; poblarlos en `products/service.py::to_save_response` y `menu/router.py` — sin cambio de esquema de BD — **implementado directo en el estado final**: `name` ya no se devuelve (sin el paso A aditivo, ver nota de despliegue abajo)
- [X] T046 [US7] En `../pos-backend/app/api/v1/catalog/service.py`: helper `resolve_default_presentation(db)` (get-or-create de "Presentación única" por nombre exacto sin filtrar `active`, re-consulta ante `IntegrityError`); hacer `name` opcional en `VariantSaveIn`/`VariantCreate`; regla de compatibilidad `presentation_id` nulo **y** `name` ausente ⇒ "Presentación única" (con `name` presente rige el comportamiento actual); `ensure_default_variant` y la herencia de categoría (`products/service.py::_apply_inherited_or_default_variant`, `VariantSaveIn(presentation_id=p.id)`) crean con presentación asociada — depende de T045 — sin la regla de compatibilidad `name` presente/ausente (mismo motivo); `default_presentation()` usa `begin_nested()` (SAVEPOINT) para la carrera del get-or-create
- [X] T047 [P] [US7] Agregar `../pos-backend/app/characterization_tests/test_variant_sin_nombre.py` (parte A): atajo `null` ⇒ "Presentación única" y su get-or-create (incluida la presentación existente desactivada), dos filas nulas del mismo producto ⇒ 409, variantes heredadas de categoría quedan asociadas, las respuestas traen `presentation_name` — depende de T046 — los tests viven en `test_products_variant_presentation.py` (reescrito), no en `test_variant_sin_nombre.py`; 18 tests

**Checkpoint A**: backend desplegable sin romper al frontend anterior.

### Paso B — frontend

- [X] T048 [P] [US7] En `../pos-heladeria/src/app/modules/products/interfaces/product.interface.ts` y `services/product.service.ts`: quitar `name` de `VariantDraft`/`VariantSavePayload`, agregar `presentationName` a `VariantDraft`/`Variant`/`DeactivatedVariant`, leer `presentation_name` en los mapeos (~l.299/404/423/623) y dejar de enviar `name` (~l.540) — depende de T045 — 942 tests de frontend en verde salvo las 19 fallas preexistentes
- [X] T049 [US7] En `../pos-heladeria/src/app/modules/products/pages/product-form.component.ts` (contracts/formulario-tamanos-orden.md §Tabla de tamaños): eliminar la columna y el `<input>` "Nombre" (grid `[28px_28px_1fr_140px_88px]`), quitar la opción "Sin presentación", excluir del `<select>` las presentaciones ya elegidas en otras filas, agregar a `canSave()` la exigencia de presentación por fila con el texto "Elige una presentación", ajustar `setVariantPresentation`/`addVariant`/`toggleHasSizes` (sin literales `'Grande'`/`'Único'`), la lista de desactivadas (`dv.presentationName`) y el `av.name` de la línea ~463 — depende de T048 — **desvío respecto del contrato original**: al encender tamaños se preseleccionan Grande/Mediana/Pequeña del catálogo cuando existen (antes se sembraban por nombre) en vez de dejar las 3 filas vacías; ver contracts/formulario-tamanos-orden.md
- [X] T050 [P] [US7] Leer `presentation_name` en lugar de `name` de variante en `../pos-heladeria/src/app/core/services/menu.service.ts:133`, `modules/tables/services/menu-lookup.ts:50`, `modules/tables/services/dining-cart.service.ts:85`, `modules/tables/components/product-select.component.ts:131` y `modules/promotions/pages/promotions-page.component.ts:1423`; en `promotions-page.component.ts:1801` (aplicación masiva, US4) emparejar la variante de cada producto por `presentation_id` en vez de por nombre — depende de T045 — `menu.service.ts` mapea `presentation_name` al `name` del modelo interno del cliente (`MenuVariant`), por eso menu-lookup, dining-cart, product-select y promotions-page no cambiaron; la aplicación masiva sigue emparejando por la etiqueta (nombre de la presentación, único en el catálogo), no por `presentation_id`
- [X] T051 [P] [US7] Actualizar los `*.spec.ts` que construyen o aserta `name` de variante (`product-form.component.spec.ts` — reemplazar los tests de US1 sobre nombre de solo lectura y "Sin presentación" —, `product.service.spec.ts`, `public-menu.component.spec.ts`, `promotions-page.component.spec.ts` y los specs del carrito de mesa) — depende de T049, T050 — product-form.spec (US1 reemplazado + 3 de orden US8), product.service.spec, menu.service.spec (+1)

**Checkpoint B**: frontend desplegable contra el backend del paso A (y contra el del paso C).

### Paso C — backend destructivo

- [X] T052 [US7] Crear la migración `<rev>_084_variante_sin_nombre.py` en `../pos-backend/alembic/versions/` (`down_revision` = cabeza confirmada en T043), `@for_each_tenant_schema` e idempotente: crear en el catálogo las presentaciones faltantes para nombres libres, enlazar, verificar 0 nulos (aborta si no), `presentation_id NOT NULL`, FK `ON DELETE RESTRICT`, quitar `uq__product_variants__product_id__name` y la columna `name`; `downgrade` sin pérdida (data-model.md §Migración) — depende de T043, T044 — `c7e2b91a4d35`; `seed_free_names_sql`/`link_free_names_sql` como funciones puras
- [X] T053 [US7] Ajustar `../pos-backend/app/models/product_variant.py`: quitar `name` y su `UniqueConstraint`; `presentation_id` no nulo, FK `RESTRICT`; `presentation: Mapped["Presentation"]` — depende de T052
- [X] T054 [US7] Reescribir las lecturas/escrituras de nombre de la tabla de contracts/variante-sin-nombre.md: `variante_duplicada` por `presentation_id`, `_save_variant_entry` (retirar la rama de renombre, SKU con `Presentation.name`), `catalog/router.py` (`PATCH /variants/{id}` y la consulta de grupos en uso, l.~235), `catalog_engine/consumption.py:160`, `sales/service.py:228`, `orders/checkout.py:294,447`, `promotions/service.py:464,949` — depende de T053
- [X] T055 [US7] En `../pos-backend/app/api/v1/presentations/router.py`: eliminar `_rename_conflict`, la cascada `UPDATE product_variants SET name` y el `409` de colisión (y su descripción en el decorador); queda solo la unicidad de `Presentation.name` (FR-031) — depende de T053
- [X] T056 [US7] Quitar `name` de `VariantCreate`/`VariantUpdate`/`VariantSaveIn`/`VariantResponse`/`VariantSaveOut`/`MenuVariantResponse` y retirar la rama de compatibilidad de T046; `presentation_id` en `VariantResponse` deja de ser nullable — depende de T054
- [X] T057 [US7] Actualizar los characterization tests y fixtures que construyen o aserta `ProductVariant.name` (`fixtures.py`, `orders_fixtures.py`, `cart_fixtures.py`, `table_sessions_fixtures.py`, `test_products_service.py`, `test_products_variant_presentation.py` — los casos de cascada de FR-004 se reemplazan por "renombrar se refleja sin escritura", `test_products_presentations_inheritance.py`, `test_categories_presentations.py`, `test_presentations_migration.py`, `test_promotions_migration.py`, `test_product_variant_reorder.py`, `test_plan_gating_inventory_fields.py`, y los que resulten de T043), **citando A-79** (Principio III); los scripts de `app/scripts/` que construyen `ProductVariant(name=…)` se actualizan o se marcan como obsoletos — depende de T053, T056 — golden master, fixtures (`make_variant(name=…)` sigue aceptando `name` como atajo y lo traduce a una presentación) y 7 archivos de test; **pendiente**: los 5 scripts manuales de `app/scripts/` que construyen `ProductVariant(name=…)` (`e2e_qr_flow`, `test_split_blindaje`, `test_cancel_inventory`, `test_receta_obligatoria`, `test_variant_option_groups`) — no corren en la suite
- [X] T058 [P] [US7] Agregar el test de migración en `../pos-backend/app/characterization_tests/` (mismo patrón que `test_presentations_migration.py`): nombres libres → filas del catálogo, presentación desactivada con el mismo nombre reutilizada, mismo nombre libre en dos productos comparte fila, variantes ya asociadas intactas, 0 nulos tras `upgrade`, `downgrade` re-deriva los nombres — depende de T052 — `test_variant_sin_nombre_migration.py`, 7 tests sobre SQLite
- [X] T059 [P] [US7] Agregar a `test_variant_sin_nombre.py` (parte C): renombrar una `Presentation` se refleja en menú, respuesta de producto y descripción de promoción sin ningún `UPDATE` sobre `product_variants` (SC-008); `409` de presentación en uso por variante desactivada; eliminar físicamente una presentación con variantes falla por `RESTRICT` (US7 escenario 7) — depende de T055 — cubierto en `test_products_variant_presentation.py` (renombre se refleja en todas las variantes; ya no choca con otra variante; 409 por variante desactivada). El `RESTRICT` de FK no se prueba en SQLite; verificado en PostgreSQL (T064)

**Checkpoint C**: US7 completa y verificable (quickstart.md, sección US7).

### User Story 8 — orden de la tarjeta

- [X] T060 [US8] Mover el bloque "Maneja inventario" (interruptor + aviso `showsInventoryWarning()`) en `../pos-heladeria/src/app/modules/products/pages/product-form.component.ts` a después de la tabla de tamaños y de la lista de "Presentaciones desactivadas", y antes del detalle del tamaño activo; reajustar el separador `border-t` para que no quede doble (contracts/formulario-tamanos-orden.md) — mismo archivo que T049: hacerlo en el mismo cambio o a continuación, no en paralelo — hecho en el mismo cambio que T049
- [X] T061 [P] [US8] Agregar a `product-form.component.spec.ts` tests de orden DOM: con tamaños, la tabla precede al interruptor "Maneja inventario"; sin tamaños, el interruptor queda bajo el encabezado; con `tracks_inventory=false` la tabla sigue visible y editable — depende de T060

**Checkpoint**: US8 verificable de forma independiente (quickstart.md, sección US8).

### Verificación final de la enmienda

- [X] T062 [P] Ejecutar la suite completa de characterization tests de `../pos-backend` y confirmar que solo cambiaron los tests citados en T057 (línea base previa: 891/891) — 907/907
- [X] T063 [P] Ejecutar `ng test` completo en `../pos-heladeria` y confirmar cero regresiones respecto de la línea base (934/954, con las 19 fallas preexistentes de T004) — 942 pasan, 19 fallan (los mismos 6 archivos preexistentes, idénticos con y sin estos cambios; `menu.service.spec.ts` incluido, falla ya en la línea base)
- [X] T064 Ensayar la migración de T052 sobre una copia de datos reales: contar variantes y nombres visibles antes y después (SC-007), confirmar 0 variantes sin presentación y ejecutar `downgrade` + `upgrade` una vez — BD de desarrollo (`heladeria3`): 14 variantes, 0 sin presentación, 0 nombres cambiados, `downgrade` + `upgrade` sin errores; el esquema quedó con `presentation_id NOT NULL`, FK `RESTRICT`, sin `name` ni `uq…__name`
- [ ] T065 Ejecutar la sección US7/US8 y el checklist de [quickstart.md](./quickstart.md) de punta a punta

---

## Phase 11: Enmienda 2026-09-20 (2) — User Story 4 (selector de presentación independiente de los productos)

**Contexto**: spec.md FR-040–FR-044, research.md D13, anomalía **A-80**. Solo `pos-heladeria`.

**Independent Test**: en una promoción nueva con el Paso 1 vacío, el selector del Paso 2 lista todas
las presentaciones activas del catálogo; al seleccionar productos, la lista no cambia y el aviso
indica a cuántos se aplicaría.

- [X] T066 [P] Registrar la anomalía **A-80** en `specs/000-reconocimiento/registro-de-anomalias.md` (Principio II)
- [X] T067 Propagar `presentation_id` del menú al modelo del cliente: `MenuVariant.presentation_id?` (`../pos-heladeria/src/app/modules/products/interfaces/product.interface.ts`) y su mapeo en `core/services/menu.service.ts`; opcional porque otros armadores del menú (modo diner) no lo necesitan
- [X] T068 En `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`: `availableLabels()` → `availablePresentations()` (catálogo activo, ordenado), `pickerLabel` → `pickerPresentationId`, emparejamiento por `presentationId` (`matchingProductsForPresentation`), `variantLabel` sin la etiqueta "Presentación única", no reiniciar la elección al cambiar productos, `pickerMatchSummary` y su aviso bajo el selector, `loadAllPresentations()` en `ngOnInit` — depende de T067
- [X] T069 [P] Actualizar `promotions-page.component.spec.ts` (fake de `PresentationService`, variantes con `presentation_id`, tests de lista independiente, emparejamiento por id, producto de una variante, resumen, no reinicio y aviso sin coincidencias) y `menu.service.spec.ts` — depende de T068 — 946 pasan, mismas 19 fallas preexistentes
- [ ] T070 Verificar a mano en la pantalla de promociones (quickstart.md, US4 enmendada)
- [X] T071 Mover "Guardar y sincronizar" al final del formulario de configuración y habilitarlo solo con cambios válidos (spec.md FR-045/FR-046): `hasChanges()` compara una foto normalizada del formulario tomada al abrir la configuración (`openEdit`/`continueToConfigure`) con el estado actual, en `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`; 6 tests nuevos en su spec — 952 pasan, mismas 19 fallas preexistentes

---

## Phase 12: Enmienda 2026-09-21 — Reglas por presentación en el Paso 2 (A-81)

**Contexto**: spec.md FR-047–FR-052, research.md D14, anomalía **A-81**. Solo `pos-heladeria`.

**Independent Test**: definir «8 onzas × 2 = $12.000» y «12 onzas × 2 = $17.000», cambiar los
productos del Paso 1 y verificar que la lista sigue teniendo dos reglas y que al guardar se
generan las reglas de backend correctas para los productos seleccionados.

- [X] T072 [P] Registrar la anomalía **A-81** en `specs/000-reconocimiento/registro-de-anomalias.md` (Principio II)
- [X] T073 En `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`: estado `presentationRules` + `rebuildRules()` (expansión a una regla de backend por producto seleccionado), `hydrateFromRules()` (agrupa por presentación/valor/unidades y recupera los productos; espera al menú si no cargó, con un `effect`), `addRuleRow()` sin exigir productos y con una sola regla por presentación, `removeRuleRow()` sobre la lista por presentación, `unresolvedRuleCount()` y su aviso, tablas del Paso 2 y de solo lectura sobre `presentationRules()`, `hasChanges()` incluye reglas y productos; se elimina el panel de confirmación con casilla por producto (`pendingBulkApply`, `confirmBulkApply`, `cancelBulkApply`, `toggleBulkApplyCandidate`)
- [X] T074 [P] Actualizar `promotions-page.component.spec.ts`: reemplazar los tests de aplicación masiva por los de regla por presentación (una regla por presentación, cambiar productos no rehace reglas, una regla por presentación, regla sin productos, quitar regla, hidratación al abrir y diferida) — 955 pasan, mismas 19 fallas preexistentes — depende de T073
- [ ] T075 Verificar a mano en la pantalla de promociones (quickstart.md, US4 enmendada por A-81)
- [X] T079 Duplicar con un nombre ya usado reemplaza a la anterior (spec.md FR-056/FR-057, A-83): `replace_existing` en `POST /promotions/{id}/duplicate` + `duplicate_replacing` (atómico, nunca sobre una `active`, auditoría) y el diálogo «Reemplazar y duplicar» con aviso previo — backend 917 en verde (7 tests nuevos), frontend 962 pasan con las mismas 19 fallas preexistentes
- [X] T078 Formato del texto de condición con un solo nombre («Llevando 8 onzas x 2 pagas $12.000») en backend y réplica frontend, y tarjeta del menú QR sin «· $X c/u» (spec.md FR-054/FR-055, A-82): `variant_set_condition_text`, `conditionText`, `minPromoPriceForProduct` y sus tests; el script de CI `test_promotions_rules.py` actualizado — backend 910 en verde, frontend 957 pasan con las mismas 19 fallas preexistentes
- [X] T077 Menú QR: `_build_menu_promotions` fusiona las reglas de una misma promoción con el mismo texto (spec.md FR-053): sin la fusión, la expansión de FR-049 anunciaba «Llevando 2 8 onzas pagas $12.000» una vez por producto; 3 tests nuevos en `test_menu_router.py` — 910 en verde
- [X] T076 Corregir el bloqueo falso de «variante repetida entre dos reglas»: `sharedVariantConflict` era un `computed` que leía `form.rules` (objeto plano, no signal) y arrastraba el conflicto de una promoción abierta antes, deshabilitando «Guardar y sincronizar» en la siguiente; pasa a método, con test de regresión — 956 pasan, mismas 19 fallas preexistentes

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato.
- **Foundational (Phase 2)**: depende de Setup; son verificaciones rápidas, no bloquean trabajo real de ninguna historia más allá de confirmar un supuesto de diseño (cabeza de Alembic para US1, CDK disponible para US6).
- **User Stories (Phase 3–8)**: todas dependen solo de Foundational — **ninguna depende de otra** (plan.md §Constitution Check, Principio VI). Pueden implementarse en cualquier orden o en paralelo, con la salvedad de concurrencia de archivo señalada arriba (US2/US4/US5/US6 comparten `promotions-page.component.ts`).
- **Polish (Phase 9)**: depende de que las historias que se vayan a entregar en este incremento estén completas.

### Dentro de cada historia

- Migración → modelo → schemas → servicio → cascada/guardas → tests de backend → interfaces/UI de frontend → tests de frontend, en ese orden relativo dentro de cada fase.
- Los tests de characterization/`*.spec.ts` se agregan **junto a** la lógica que cubren, no antes (ver nota de Tests arriba) — se listan después de la implementación correspondiente en cada historia.
- **Excepción explícita en US2**: T018 (actualizar los dos tests de `test_promotions_rules_admin.py` rotos por T017) no es una tarea de cobertura nueva — es la actualización obligatoria de un test protegido existente que el Principio III exige antes de poder mergear T017 (research.md D9).

### Oportunidades de paralelismo

- Dentro de Setup: T002, T003, T004 en paralelo con T001.
- Dentro de Foundational: T005 y T006 en paralelo entre sí.
- US1 completa puede avanzar en paralelo con cualquier otra historia (no comparte archivo con ninguna).
- US3 completa puede avanzar en paralelo con cualquier otra historia (no comparte archivo con ninguna).
- US2, US4, US5, US6 pueden implementarse en cualquier orden entre sí, pero secuencialmente (o coordinando merges) por compartir `promotions-page.component.ts`.
- **Fase 10 (enmienda 2026-09-20)**: T043 → Paso A (T045–T047) → Paso B (T048–T051) → Paso C
  (T052–T059) es el orden de **despliegue**; T060 (US8) comparte archivo con T049 y va después. T044
  (A-79) ya está hecha y debe existir antes de mergear T052/T057. El Paso C no puede desplegarse
  hasta que el Paso B lleve al menos un ciclo de despliegue en producción (research.md D10, PWA en
  caché).
- Los registros de anomalía (T016, T025, T029) no tienen dependencia de código y pueden adelantarse en cualquier momento antes de la tarea de implementación correspondiente.

---

## Parallel Example: User Story 1

```bash
# Tras completar T007/T008 (migración + modelo), en paralelo:
Task: "Agregar presentation_id a VariantSaveIn/VariantResponse en ../pos-backend/app/api/v1/catalog/schemas.py"
Task: "Agregar presentationId a VariantDraft/VariantSavePayload en ../pos-heladeria/.../product.interface.ts"
```

---

## Implementation Strategy

### MVP primero (User Story 1 + User Story 2)

1. Completar Fase 1 (Setup) y Fase 2 (Foundational).
2. Completar Fase 3 (US1) — el catálogo de Presentaciones queda realmente conectado a los
   productos.
3. Completar Fase 4 (US2) — cierra el riesgo de negocio más directo (editar una promoción en
   curso), incluida la actualización explícita de los dos tests protegidos que rompe (T018).
4. **Detener y validar**: correr quickstart.md US1 + US2 de forma independiente.
5. Desplegar/demo si está listo — ya resuelve 2 de los 5 bugs reportados sin depender del resto.

### Entrega incremental

1. Setup + Foundational → base lista.
2. US1 → validar independientemente → desplegar (MVP parcial).
3. US2 → validar independientemente → desplegar.
4. US3 → validar independientemente → desplegar.
5. US4 → validar independientemente → desplegar.
6. US5 → validar independientemente → desplegar.
7. US6 → validar independientemente → desplegar.
8. Cada historia agrega valor sin romper las anteriores — ninguna depende de otra para funcionar.

### Estrategia de equipo en paralelo

Con más de una persona: una puede tomar US1 y otra US3 en paralelo desde el inicio (no comparten
archivo con nada). US2, US4, US5 y US6 conviene repartirlas secuencialmente entre quien(es) trabaje
en `promotions-page.component.ts`, para no generar conflictos de merge sobre el mismo archivo.

---

## Notes

- [P] = archivo distinto, sin dependencia de una tarea sin terminar.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (spec.md, Principio XII).
- Las anomalías A-76/A-77/A-78 deben existir en `registro-de-anomalias.md` antes de mergear el
  commit que implementa el comportamiento correspondiente (Principio II) — no antes de empezar a
  codificar, pero sí antes de integrar a `develop`.
- Único test de characterization existente que se edita en esta spec: los dos casos de
  `test_promotions_rules_admin.py` que T018 actualiza explícitamente (citando A-76, Principio III).
  Ningún otro (`test_promotions_service.py`, `test_promotions_router.py`, `test_products_service.py`,
  etc.) se edita — solo se agregan archivos nuevos.
- Detenerse en cualquier checkpoint para validar una historia de forma independiente antes de
  seguir con la siguiente.
