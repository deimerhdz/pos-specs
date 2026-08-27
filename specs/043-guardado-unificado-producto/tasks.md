---

description: "Task list for Guardado Unificado de Producto (Crear y Actualizar)"
---

# Tasks: Guardado Unificado de Producto (Crear y Actualizar)

**Input**: Design documents from `/specs/043-guardado-unificado-producto/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/product-save-endpoints.md](./contracts/product-save-endpoints.md),
[quickstart.md](./quickstart.md)

**Tests**: Incluidos — la Constitución (Principio X, Verificación Obligatoria) exige verificar toda
funcionalidad nueva con characterization tests, y las 4 historias de `spec.md` ya definen su propio
"Independent Test". Backend: `unittest` (`python -m unittest`, sin `pytest` en el repo). Frontend:
Vitest (`@angular/build:unit-test`).

**Organization**: Tareas agrupadas por historia de usuario (spec.md) para que cada una sea
implementable y verificable de forma independiente. Rutas relativas a los repos sibling de
`pos-specs`: `../pos-backend` y `../pos-heladeria`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de una tarea incompleta)
- **[Story]**: Historia de usuario a la que pertenece (US1, US2, US3, US4)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

- Backend: `../pos-backend/app/...`
- Frontend: `../pos-heladeria/src/app/...`

---

## Phase 1: Setup

**Purpose**: Confirmar la línea base antes de tocar código — esta spec no crea proyecto nuevo ni
agrega dependencias (plan.md Technical Context).

- [X] T001 [P] Confirmar que no hay ninguna migración de Alembic pendiente en `../pos-backend`
      (`alembic current` == `alembic heads`) — esta spec no cambia el esquema (data-model.md)
- [X] T002 [P] Confirmar que `../pos-backend/requirements.txt` y
      `../pos-heladeria/package.json` no requieren ninguna dependencia nueva para esta spec
      (plan.md Technical Context — Principio IX no aplica)

**Checkpoint**: línea base confirmada, sin bloqueos de entorno.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: DTOs y funciones de servicio compartidas que tanto crear (US1) como actualizar (US2)
necesitan — sin esto ninguna historia puede implementarse.

**⚠️ CRITICAL**: Ninguna historia de usuario empieza hasta completar esta fase.

- [X] T003 [P] Crear `VariantSaveIn` en `../pos-backend/app/api/v1/catalog/schemas.py`
      (`id: UUID | None`, `name`, `price`, `sku`, `active: bool = True`,
      `recipe: list[RecipeItemIn] = []`, `option_groups: list[VariantOptionGroupIn] = []` —
      reutiliza `RecipeItemIn`/`VariantOptionGroupIn` ya existentes; data-model.md)
- [X] T004 [P] Crear `VariantSaveOut` (extiende `VariantResponse` + `display_order`, `recipe`,
      `option_groups`) y `ProductSaveResponse` (extiende `ProductResponse` + `variants:
      list[VariantSaveOut]`) en `../pos-backend/app/api/v1/products/schemas.py` (data-model.md)
- [X] T005 Crear `_save_variant_entry(db, product, entry: VariantSaveIn, existing_by_id: dict) ->
      ProductVariant` en `../pos-backend/app/api/v1/catalog/service.py`: si `entry.id` es `None`
      crea la presentación (reutiliza `variante_duplicada`, `_unique_sku`, `_slug`, mismas
      validaciones que `create_variant` hoy); si `entry.id` está presente, actualiza esa fila
      (reutiliza `variante_duplicada` con `exclude_id`, `ensure_unique` para SKU, mismas
      validaciones que `update_variant` hoy); en ambos casos reemplaza `recipe`/`option_groups`
      por completo (mismo patrón que `set_recipe`/`set_variant_option_groups`); agrega
      `variant_index` al `detail` de cualquier `HTTPException` que levante (research.md Decisión 5,
      depende de T003)
- [X] T006 Crear `_assign_display_orders(variants: list[ProductVariant], ordered_ids: list[UUID])
      -> None` en `../pos-backend/app/api/v1/catalog/service.py`: aplica el patrón de dos pasadas
      de `reorder_variants` (negativos → definitivos) sobre la lista ya resuelta por T005, en el
      orden recibido (research.md Decisión 2, mismo archivo que T005 — secuencial tras T005)
- [X] T007 [P] Crear `VariantSavePayload` (`id?`, `name`, `price`, `sku?`, `active?`, `recipe`,
      `option_groups`) y extender `ProductCreatePayload`/`ProductUpdatePayload` con `variants?:
      VariantSavePayload[]` en `../pos-heladeria/src/app/modules/products/interfaces/product.interface.ts`
      (data-model.md — mapeable directo desde `VariantDraft`, líneas 249-258)

**Checkpoint**: DTOs y helpers compartidos listos — US1 y US2 pueden implementarse.

---

## Phase 3: User Story 1 - Crear un producto completo en un solo guardado (Priority: P1) 🎯 MVP

**Goal**: `POST /products` acepta el árbol completo (`variants` con receta y grupos de opciones) y
lo persiste en una sola transacción, devolviendo el estado final completo.

**Independent Test**: crear un producto con al menos dos presentaciones (cada una con receta y un
grupo de opciones) dispara una sola petición de escritura y el producto resultante trae todo
completo en la respuesta (spec.md US1 Independent Test, quickstart.md Historia 1).

### Tests for User Story 1 ⚠️

- [X] T008 [US1] Test unittest: `POST /products` con `variants` (2 presentaciones, cada una con
      `recipe` y `option_groups`) crea producto + presentaciones + receta + grupos en una sola
      transacción — nuevo caso en `../pos-backend/app/characterization_tests/test_products_service.py`
- [X] T009 [US1] Test unittest: `POST /products` sin `variants` (u omitido) sigue creando la
      presentación `"Single"` a precio 0 automática — back-compat, `RN-CAT-05` — nuevo caso en
      `../pos-backend/app/characterization_tests/test_products_service.py` (mismo archivo que T008)
- [X] T010 [US1] Test unittest: la respuesta `201` de `POST /products` incluye `variants` con
      `recipe`/`option_groups` ya resueltos, sin necesitar ninguna lectura adicional (FR-006) —
      nuevo caso en `../pos-backend/app/characterization_tests/test_products_service.py` (mismo
      archivo que T008/T009)

### Implementation for User Story 1

- [X] T011 [US1] Extender `ProductCreate` con `variants: list[VariantSaveIn] = []` en
      `../pos-backend/app/api/v1/products/schemas.py` (depende de T004)
- [X] T012 [US1] Implementar el árbol completo en `create_product`
      (`../pos-backend/app/api/v1/products/service.py`): si `variants` viene vacío, mantener
      `ensure_default_variant` sin cambios; si trae entradas, iterar cada una con
      `_save_variant_entry` (T005) y asignar orden con `_assign_display_orders` (T006), todo dentro
      de la misma `Session`, un solo `db.commit()` al final (depende de T005, T006, T011)
- [X] T013 [US1] Cambiar el `response_model` de `POST /products` a `ProductSaveResponse` en
      `../pos-backend/app/api/v1/products/router.py` (depende de T004, T012)
- [X] T014 [P] [US1] Reescribir `saveNewProduct` en
      `../pos-heladeria/src/app/modules/products/services/product.service.ts` para hacer una sola
      llamada `POST /products` con el `draft` mapeado a `VariantSavePayload[]`, eliminando el bucle
      de `postVariant`/`patchVariant` + `saveVariantConfig` (depende de T007,
      contracts/product-save-endpoints.md)
- [X] T015 [US1] Test Vitest: crear un producto con dos presentaciones desde el formulario dispara
      exactamente una llamada de red — nuevo caso en
      `../pos-heladeria/src/app/modules/products/services/product.service.spec.ts` (o
      `product-form.component.spec.ts` si el mock de `HttpClient` vive ahí; depende de T014)

**Checkpoint**: US1 completo y verificable de forma independiente (quickstart.md Historia 1) — MVP
alcanzado.

---

## Phase 4: User Story 2 - Editar un producto existente combinando varios cambios en un solo guardado (Priority: P1)

**Goal**: `PATCH`/`PUT /products/{id}` acepta el árbol completo y reconcilia el conjunto de
presentaciones activas (crea, actualiza, desactiva las no listadas) en una sola transacción.

**Independent Test**: editar un producto con al menos tres presentaciones, tocando datos del
producto y de al menos dos presentaciones (receta, grupos, orden) en una sola llamada (spec.md US2
Independent Test, quickstart.md Historia 2).

### Tests for User Story 2 ⚠️

- [X] T016 [US2] Test unittest: `PATCH /products/{id}` con `variants` mezclando entradas con `id`
      (editar), sin `id` (crear) y una presentación activa existente no listada (desactivar) —
      verifica creación/edición/desactivación y `display_order` según la posición en la lista —
      nuevo caso en `../pos-backend/app/characterization_tests/test_products_service.py`
- [X] T017 [US2] Test unittest: `PATCH /products/{id}` con una entrada cuyo `id` es el de una
      presentación desactivada (sin `active: false`) la reactiva, conservando receta/grupos previos
      si no se reenvían — nuevo caso en
      `../pos-backend/app/characterization_tests/test_products_service.py` (mismo archivo que T016)
- [X] T018 [US2] Test unittest: `PATCH /products/{id}` sin `variants` en el body no toca ninguna
      presentación (back-compat con cualquier llamador que solo actualice campos del producto) —
      nuevo caso en `../pos-backend/app/characterization_tests/test_products_service.py` (mismo
      archivo que T016/T017)

### Implementation for User Story 2

- [X] T019 [US2] Extender `ProductUpdate` con `variants: list[VariantSaveIn] | None = None` en
      `../pos-backend/app/api/v1/products/schemas.py`, usando `model_fields_set` para distinguir
      "ausente" de "`[]`" (depende de T004, mismo archivo que T011 — secuencial tras T011)
- [X] T020 [US2] Implementar la reconciliación en `update_product`
      (`../pos-backend/app/api/v1/products/service.py`): calcular presentaciones activas existentes
      vs. `variants` recibido, desactivar (`active=False`) las no listadas, resolver el resto con
      `_save_variant_entry` (T005) y `_assign_display_orders` (T006), un solo `db.commit()` al
      final (depende de T005, T006, T019; data-model.md tabla de reconciliación)
- [X] T021 [US2] Cambiar el `response_model` de `PATCH`/`PUT /products/{id}` a
      `ProductSaveResponse` en `../pos-backend/app/api/v1/products/router.py` (depende de T004,
      T020, mismo archivo que T013 — secuencial tras T013)
- [X] T022 [US2] Reescribir `saveExistingProduct` en
      `../pos-heladeria/src/app/modules/products/services/product.service.ts` para una sola llamada
      `PATCH /products/{id}` con el `draft` mapeado, eliminando el bucle de
      `patchVariant`/`postVariant` + `saveVariantConfig` + los `DELETE` por presentación quitada +
      `reorderVariants` (depende de T007, T014, contracts/; mismo archivo que T014 — secuencial)
- [X] T023 [P] [US2] Reescribir `restoreVariant` en
      `../pos-heladeria/src/app/modules/products/pages/product-form.component.ts`: mover la
      presentación de `draft().deactivated` a `draft().variants` en memoria (con su `id` real, su
      receta y sus grupos ya cargados) en vez de llamar `PATCH /variants/{id}` de inmediato
      (research.md Decisión 4)
- [X] T024 [US2] Test Vitest: editar producto (nombre + precio de una presentación + receta de otra
      + reordenar) dispara exactamente una llamada de red; "Restaurar" no dispara ninguna — nuevo
      caso en `product.service.spec.ts`/`product-form.component.spec.ts` (depende de T022, T023)

**Checkpoint**: US1 + US2 completos, ambos verificables de forma independiente (quickstart.md
Historia 2).

---

## Phase 5: User Story 3 - Un error en cualquier parte del guardado no deja el producto a medias (Priority: P1)

**Goal**: cualquier fallo de validación en cualquier presentación/receta/grupo del payload aborta
el guardado completo sin persistir nada, con un error que identifica qué parte falló.

**Independent Test**: guardar varias presentaciones válidas junto con una inválida (nombre
duplicado, precio negativo, receta/grupo inválido) no persiste nada y el error identifica la parte
que falló (spec.md US3 Independent Test, quickstart.md Historia 3).

### Tests for User Story 3 ⚠️

- [X] T025 [US3] Test unittest: `POST /products` con 3 presentaciones válidas + 1 con nombre
      duplicado no persiste ninguna de las cuatro (verificar ausencia de filas en
      `product_variants` tras la llamada fallida) — nuevo caso en
      `../pos-backend/app/characterization_tests/test_products_service.py`
- [X] T026 [US3] Test unittest: `PATCH /products/{id}` con un insumo repetido en `recipe` o un
      grupo de opciones inactivo/repetido en `option_groups` de cualquier presentación del payload
      no persiste ningún cambio del guardado — nuevo caso en
      `../pos-backend/app/characterization_tests/test_products_service.py` (mismo archivo que T025)
- [X] T027 [US3] Test unittest: el error devuelto en ambos casos anteriores identifica
      `variant_index` de la entrada que falló, sin perder los campos existentes (`error`,
      `variant_id`, `active`) — nuevo caso en
      `../pos-backend/app/characterization_tests/test_products_service.py` (mismo archivo que
      T025/T026; contracts/product-save-endpoints.md)

### Implementation for User Story 3

- [X] T028 [US3] Envolver `create_product`/`update_product`
      (`../pos-backend/app/api/v1/products/service.py`) en `try/except` con `db.rollback()`
      explícito ante cualquier excepción antes del `commit()` final, confirmando que ningún
      `db.flush()` intermedio dentro de `_save_variant_entry`/`_assign_display_orders` deja filas
      visibles a otra transacción (depende de T012, T020, mismo archivo — secuencial)
- [X] T029 [P] [US3] Confirmar que `product.service.ts`
      (`../pos-heladeria/src/app/modules/products/services/product.service.ts`) muestra el error
      consolidado (mensaje + `variant_index`) en el banner existente (`otherError`/
      `lastVariantConflict`) sin ninguna lógica de reintento parcial — ajustar `extractError`/
      `toNameConflict` si `variant_index` debe reflejarse en la UI (depende de T022). Verificado:
      `extractError` ya lee `detail.error` sin importar si trae `variant_index` (código sin
      cambios); `toNameConflict` sigue funcionando por `variant_id`. Se agregó `variant_index?`
      a la interfaz `VariantNameConflict` (product.interface.ts) solo para que el tipo documente
      el campo nuevo del contrato — sin lógica nueva.

**Checkpoint**: US1 + US2 + US3 completos (quickstart.md Historia 3) — el guardado consolidado es
funcional y atómico de punta a punta.

---

## Phase 6: User Story 4 - Retiro de los endpoints separados una vez migrado el formulario (Priority: P2)

**Goal**: una vez el formulario usa solo los dos endpoints consolidados, se retiran los cinco
endpoints separados que ya no tienen llamador, previa verificación de que ningún otro consumidor
los necesita.

**Independent Test**: ninguna de las cinco rutas retiradas sigue registrada ni alcanzable, y el
formulario de productos sigue funcionando sin regresión (spec.md US4 Independent Test, quickstart.md
Historia 4).

### Implementation for User Story 4

- [X] T030 [US4] Auditar consumidores de las 5 rutas a retirar: buscar en `../pos-heladeria/src`
      cualquier llamada a `POST /products/{id}/variants`, `PATCH /variants/{id}`, `PUT
      /variants/{id}/recipe`, `PUT /variants/{id}/option-groups`, `PATCH
      /products/{id}/variants/reorder` fuera de lo ya reescrito en T014/T022/T023, y confirmar que
      ningún otro flujo del sistema (Menú QR, venta de mostrador) las usa (quickstart.md Historia 4;
      depende de T014, T022, T023). **Resultado**: en `pos-heladeria`, ningún otro consumidor (los
      únicos matches son los métodos privados ya muertos de `product.service.ts`, ver T034). En
      `pos-backend`, sí se encontró un consumidor real: `app/scripts/test_variantes_duplicadas.py`
      (script citado por spec 002 para `RN-CAT-08`/`RN-CAT-09`) importa y llama `create_variant` y
      `update_variant` **directamente en proceso** (sin HTTP). Por FR-007, esos dos endpoints
      quedan **excluidos del retiro** — documentado en spec.md (Edge Cases) y
      [`registro-de-anomalias.md` A-55](../000-reconocimiento/registro-de-anomalias.md). Los otros
      tres (`PUT recipe`, `PUT option-groups`, `PATCH reorder`) no tienen ningún otro consumidor y
      sí se retiran (T031).
- [X] T031 [US4] Retirar de `../pos-backend/app/api/v1/catalog/router.py` las 3 rutas sin
      consumidor (`PUT /variants/{id}/recipe`, `PUT /variants/{id}/option-groups`,
      `PATCH /products/{id}/variants/reorder`). `POST /products/{id}/variants` y
      `PATCH /variants/{id}` quedan excluidas del retiro (excepción documentada en T030) (depende
      de T030)
- [X] T032 [US4] Eliminar de `../pos-backend/app/api/v1/catalog/service.py`/`schemas.py` las
      funciones/schemas que quedan sin ningún llamador tras T031: `reorder_variants`,
      `VariantReorderError` (service.py); `RecipeSet`, `VariantOptionGroupSet`,
      `VariantReorderRequest`, `VariantOrderEntry`, `VariantReorderResponse` (schemas.py) —
      verificado que `_assign_display_orders`/`_save_variant_entry` (T005/T006) no dependen de
      ninguna (depende de T031)
- [X] T033 [P] [US4] Reescribir los characterization tests que ejercitan directamente las rutas
      retiradas: `../pos-backend/app/characterization_tests/test_product_variant_reorder.py`
      (clase `ReorderVariantsTests`, que probaba `reorder_variants`/`reorder_product_variants`
      directamente) — se retira esa clase y sus imports, citando esta spec y
      [`registro-de-anomalias.md` A-55](../000-reconocimiento/registro-de-anomalias.md); el resto
      del archivo (`NextDisplayOrderTests`, `EnsureDefaultVariantOrderTests`,
      `CreateVariantOrderTests`, `DeleteReactivateOrderTests`) sigue vigente sin cambios, porque
      ejercita funciones que no se retiraron (Principio III — no se elimina en silencio, se
      documenta por qué) (depende de T031)
- [X] T034 [P] [US4] Eliminar de `../pos-heladeria/src/app/modules/products/services/product.service.ts`
      los métodos privados que ya no tienen llamador tras T014/T022 y que además envuelven rutas
      retiradas (`postVariant`, `patchVariant`, `saveVariantConfig`, `setRecipe`,
      `setVariantOptionGroups`, `reorderVariants`). `createVariant`/`updateVariant`/`deleteVariant`/
      `restoreVariant` (públicos) quedan fuera de este retiro: envuelven rutas que siguen vivas
      (`POST`/`PATCH`/`DELETE /variants`, excepción de T030) — sin llamador hoy, pero preexistente a
      esta spec y no forman parte de su alcance (Principio V) (depende de T030)

**Checkpoint**: las 4 historias completas y verificables (quickstart.md Historias 1-4).

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de no regresión y de los criterios de éxito medibles del spec.

- [X] T035 [P] Ejecutar `python -m unittest discover app/characterization_tests -v` en
      `../pos-backend` y confirmar cero regresiones en la suite completa (quickstart.md
      "Verificación de no-regresión"). **Resultado**: 452/452 en verde (era 461/461 antes de T033;
      los 9 tests de `ReorderVariantsTests` se retiraron junto con el endpoint que probaban, no son
      una regresión). Frontend: `ng test` sin nuevas fallas — los mismos 5 archivos/12 tests que ya
      fallaban en `develop` antes de esta spec (confirmado con `git stash`) siguen fallando igual,
      sin relación con `products/` (`app.spec.ts`, `auth.service.spec.ts`,
      `sidebar.component.spec.ts`, `pos-checkout-panel.component.spec.ts`,
      `session-bill-panel.component.spec.ts`).
- [ ] T036 Medir el tiempo entre presionar "Guardar" y la confirmación para un producto típico
      (≤10 presentaciones) antes/después del cambio y confirmar SC-002 (reducción ≥50%, y en todo
      caso menor a 2 segundos). **No ejecutado**: este entorno no tiene `pos-backend` y
      `pos-heladeria` corriendo juntos contra una base de datos con latencia de red real, así que no
      hay forma honesta de medir un tiempo de reloj de pared aquí. Verificado en su lugar, por
      construcción y por test (T008/T015/T022/T024): el guardado consolidado hace exactamente 1
      petición de escritura donde antes hacía `1 + 1 + 3·N + D + 1` (SC-001, ya cumplido) — con eso
      la meta relativa de SC-002 (≥50% menos tiempo) es prácticamente inevitable para cualquier N
      ≥ 1, pero el número absoluto (<2s) queda pendiente de medir en un entorno real (staging o con
      ambos servidores corriendo) antes de dar el spec por completamente verificado.
- [X] T037 [P] Revisar que `contracts/product-save-endpoints.md` siga reflejando exactamente la
      forma final del `detail` de error (`variant_index` y campos existentes) implementada en T005/
      T028 — actualizar si divergió durante la implementación

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA todas las historias.
- **US1 (Phase 3)**: depende de Foundational. Sin dependencia de otras historias — es el MVP.
- **US2 (Phase 4)**: depende de Foundational. Reutiliza los mismos archivos que US1 tocó en backend
  (`products/schemas.py`, `products/service.py`, `products/router.py`) y frontend
  (`product.service.ts`) — por eso sus tareas de implementación son secuenciales respecto a las de
  US1 en esos archivos (T019 tras T011, T020 tras T012, T021 tras T013, T022 tras T014), aunque la
  historia en sí es independientemente verificable una vez integrada.
- **US3 (Phase 5)**: depende de Foundational y de que US1/US2 ya envuelvan la escritura en la misma
  transacción (T012, T020) — verifica y endurece la atomicidad que esas dos ya establecen.
- **US4 (Phase 6)**: depende de que US1, US2 y US3 estén completas y el formulario ya use solo los
  endpoints consolidados (T014, T022, T023) — el retiro no puede ocurrir antes.
- **Polish (Phase 7)**: depende de las 4 historias completas.

### Dentro de cada historia

- Tests (si se incluyen) se escriben primero y deben fallar antes de implementar.
- Schemas antes que servicio; servicio antes que router; backend antes que el mapeo del frontend
  correspondiente (el frontend depende del contrato ya implementado).

### Oportunidades de paralelismo

- T001/T002 (Setup) en paralelo.
- T003/T004/T007 (Foundational, archivos distintos) en paralelo entre sí; T005 depende de T003, T006
  depende de T005 (mismo archivo, secuencial).
- T014 (US1, frontend) puede avanzar en paralelo con T011-T013 (US1, backend) una vez T007 está
  listo, coordinando contra el contrato ya fijado en `contracts/product-save-endpoints.md` en vez de
  esperar a que el backend esté desplegado.
- T023 (US2) en paralelo con T019-T022 (archivo distinto).
- T029 (US3, frontend) en paralelo con T028 (US3, backend).
- T033/T034 (US4) en paralelo entre sí tras T030-T032.
- T035/T037 (Polish) en paralelo; T036 depende de que T035 ya confirmó cero regresiones.

---

## Parallel Example: Foundational

```bash
# Backend, archivos distintos — en paralelo:
Task: "Crear VariantSaveIn en app/api/v1/catalog/schemas.py"          # T003
Task: "Crear VariantSaveOut y ProductSaveResponse en app/api/v1/products/schemas.py"  # T004

# Frontend, en paralelo con las dos anteriores:
Task: "Crear VariantSavePayload en product.interface.ts"              # T007
```

---

## Implementation Strategy

### MVP primero (User Story 1 solamente)

1. Completar Fase 1: Setup
2. Completar Fase 2: Foundational (CRÍTICO — bloquea todas las historias)
3. Completar Fase 3: User Story 1
4. **DETENER y VALIDAR**: probar `POST /products` con árbol anidado de forma independiente
   (quickstart.md Historia 1)
5. Desplegar/demostrar si está listo — ya resuelve el caso de creación reportado por el usuario

### Entrega incremental

1. Setup + Foundational → base lista
2. + US1 → validar independientemente → Demo (MVP: crear ya es una sola petición)
3. + US2 → validar independientemente → Demo (editar también es una sola petición)
4. + US3 → validar independientemente → Demo (atomicidad garantizada y verificada)
5. + US4 → validar independientemente → Demo (endpoints viejos retirados, limpieza cerrada)
6. Cada historia agrega valor sin romper las anteriores (spec.md FR-005: ninguna regla de negocio
   existente cambia).

### Nota sobre US4

A diferencia de US1-US3 (que pueden desarrollarse casi en paralelo tras Foundational), US4 tiene una
dependencia dura hacia adelante: no puede ejecutarse hasta que el formulario de administración de
productos deje de llamar las 5 rutas viejas (T014, T022, T023 ya integradas y verificadas). Intentar
adelantar el retiro rompería US1/US2 mientras el frontend todavía las usa.

---

## Notes

- `[P]` = archivo distinto, sin dependencia de una tarea incompleta.
- `[Story]` mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- T005/T006 (Foundational) concentran la lógica que FR-004 (todo o nada) y FR-005 (reglas de negocio
  sin cambios) exigen — por eso son bloqueantes para US1, US2 y US3 por igual.
- El retiro de endpoints (US4, FR-007) es condicional: si T030 encuentra un consumidor no
  identificado hoy, esa ruta puntual se excluye del retiro (T031) y se documenta como excepción —
  no se fuerza su eliminación.
- Commitear después de cada tarea o grupo lógico; detenerse en cada Checkpoint para validar la
  historia de forma independiente antes de continuar.
