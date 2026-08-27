---

description: "Task list for Orden de Presentaciones de un Producto"
---

# Tasks: Orden de Presentaciones de un Producto

**Input**: Design documents from `/specs/042-orden-presentaciones-producto/` (plan.md, spec.md,
research.md, data-model.md, contracts/, quickstart.md)

**Tests**: incluidos — `plan.md` (Technical Context) y `quickstart.md` fijan de antemano qué
ficheros de test crea o extiende cada historia (Constitución, Principio X: Verificación
Obligatoria), así que no son opcionales para esta spec. Ningún characterization test existente se
modifica (Principio III no aplica — no hay ninguno que documente orden de presentaciones hoy).

**Organization**: tareas agrupadas por historia de usuario (US1-US3, prioridades de `spec.md`) para
que cada una sea implementable y verificable de forma independiente, per `quickstart.md`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (ficheros distintos, sin dependencia de una tarea sin
  terminar)
- **[Story]**: historia de usuario a la que pertenece (US1, US2, US3)
- Cada tarea incluye la ruta de fichero exacta, relativa a la raíz del repo sibling que corresponda
  (`pos-backend` o `pos-heladeria`)

## Path Conventions

Dos repositorios sibling de `pos-specs` (Constitución §Alcance, plan.md §Project Structure):

- Backend: `pos-backend/app/...` y `pos-backend/alembic/...` (rutas ya incluyen el prefijo
  `pos-backend/`)
- Frontend: `pos-heladeria/src/app/...` (rutas ya incluyen el prefijo `pos-heladeria/`)

---

## Phase 1: Setup

**Purpose**: confirmar que el entorno está listo y que hay una línea base verde antes de tocar
nada.

- [X] T001 Confirmar entorno: `pos-backend` con el venv activado (Python 3.14) y `pos-heladeria`
  con `npm install` ya corrido; correr `python -m unittest discover app/characterization_tests -v`
  en `pos-backend` y `npx vitest run` (o `npx ng test --watch=false`) en `pos-heladeria` como línea
  base, confirmando que ambas suites pasan antes de empezar. Reverificar contra
  `pos-backend/alembic/versions/` que `187e491e597a` sigue siendo la revisión `head` (research.md);
  si otra spec agregó una migración más reciente entre el 2026-08-27 y ahora, usar esa nueva
  revisión como `down_revision` en T002 en su lugar.

**Checkpoint**: entornos listos, línea base verde confirmada, `down_revision` de T002 confirmado.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: hacer existir la columna `display_order` de punta a punta (BD → modelo → orden de
lectura → asignación al crear) sin ningún endpoint ni UI nuevos todavía — prerrequisito real y
compartido por las 3 historias (ninguna puede construirse sin que el campo exista y ya se lea/asigne
correctamente).

**⚠️ CRITICAL**: ninguna historia puede empezar hasta que esta fase esté completa.

- [X] T002 Crear la migración
  `pos-backend/alembic/versions/<rev>_product_variants_display_order.py`,
  `down_revision='187e491e597a'` (T001), siguiendo el esqueleto `@for_each_tenant_schema` ya usado
  por migraciones previas (spec 027): en `upgrade()`, por cada schema de tenant — (a)
  `op.add_column("product_variants", sa.Column("display_order", sa.Integer(), nullable=True),
  schema=schema)`; (b) backfill con
  `UPDATE product_variants pv SET display_order = sub.rn FROM (SELECT id, ROW_NUMBER() OVER
  (PARTITION BY product_id ORDER BY id) AS rn FROM product_variants) sub WHERE pv.id = sub.id`
  (vía `op.execute`, con el nombre de tabla/schema calificado); (c)
  `op.alter_column("product_variants", "display_order", nullable=False, schema=schema)`; (d)
  `op.create_unique_constraint("uq__product_variants__product_id__display_order",
  "product_variants", ["product_id", "display_order"], schema=schema,
  deferrable=True, initially="DEFERRED")`. En `downgrade()`: `op.drop_constraint(...)` seguido de
  `op.drop_column("product_variants", "display_order", schema=schema)` (data-model.md, research.md
  Decisión 3/5)
- [X] T003 [P] En `pos-backend/app/models/product_variant.py`, agregar
  `display_order: Mapped[int] = mapped_column(Integer, nullable=False)` junto a `active` (línea
  ~33) — sin default de aplicación: toda ruta que construya `ProductVariant(...)` debe asignarlo
  explícitamente (T005), igual que ya hace con otros campos obligatorios (research.md Decisión 1)
- [X] T004 [P] En `pos-backend/app/models/product.py`, agregar `order_by="ProductVariant.display_order"`
  a la relación `variants` (línea 52) — un solo cambio que corrige el orden en todo lugar que haga
  `product.variants`, incluidos el `GET /products/{id}` del formulario y `menu/router.py:110-114`
  del Menú QR (research.md Decisión 7)
- [X] T005 En `pos-backend/app/api/v1/catalog/service.py`, en cada punto que construye una
  `ProductVariant` nueva (la función de creación usada por `POST /products/{id}/variants`, y
  `ensure_default_variant` de spec 002 para la variante inicial `"Single"`), asignar
  `display_order = (MAX(display_order) entre todas las presentaciones —activas e inactivas— del
  mismo product_id) + 1`, o `1` si el producto todavía no tiene ninguna — depende de T003
  (data-model.md, tabla de asignación; FR-005, FR-009)

**Checkpoint**: `display_order` existe de punta a punta, se asigna correctamente al crear cualquier
presentación (nueva o la "Single" por defecto), y toda lectura de `product.variants` ya viene
ordenada — sin que exista todavía ninguna forma de cambiar ese orden desde la UI ni desde la API.

---

## Phase 3: User Story 1 - Reordenar presentaciones arrastrándolas en el formulario (Priority: P1) 🎯 MVP

**Goal**: que un administrador pueda arrastrar una presentación a otra posición dentro del
formulario, viendo el número de orden actualizarse de inmediato como vista previa, y que ese orden
se persista únicamente al presionar "Guardar" — nunca al soltar la fila (spec FR-001/FR-002/FR-003,
Clarifications 2026-08-27, research.md Decisión 2/3b/6).

**Independent Test**: abrir el formulario de un producto con al menos tres presentaciones,
arrastrar una fila a otra posición, verificar que el número de cada fila cambia de inmediato sin
ninguna llamada de red todavía; presionar "Guardar" y confirmar (recargando el formulario) que el
nuevo orden persiste; y verificar que recargar o salir **sin** guardar revierte al último orden
guardado.

### Implementación backend para User Story 1

- [X] T006 [P] [US1] En `pos-backend/app/api/v1/catalog/schemas.py`, agregar
  `VariantReorderRequest` (`variant_ids: list[UUID]`) y `VariantReorderResponse`
  (`variants: list[{id: UUID, display_order: int}]`) (contracts/product-variants-reorder.md)
- [X] T007 [US1] En `pos-backend/app/api/v1/catalog/service.py`, agregar
  `reorder_variants(db: Session, product_id: UUID, variant_ids: list[UUID]) ->
  list[ProductVariant]`: valida que `variant_ids` sea exactamente el conjunto de IDs de
  presentaciones **activas** de `product_id` (sin duplicados, sin faltantes, sin IDs ajenos ni de
  presentaciones desactivadas — `422` estructurado si no), y en una sola transacción asigna
  `display_order = 1..N` según la posición de cada ID en la lista recibida, devolviendo las
  presentaciones actualizadas — depende de T003, T004, T006 (contracts/product-variants-reorder.md,
  research.md Decisión 2/3)
- [X] T008 [US1] En `pos-backend/app/api/v1/catalog/router.py`, agregar el endpoint
  `PATCH /products/{product_id}/variants/reorder` que llama a `reorder_variants` y devuelve
  `VariantReorderResponse` (`404` si `product_id` no existe) — depende de T007

### Tests para User Story 1 (backend)

- [X] T009 [P] [US1] Crear `pos-backend/app/characterization_tests/test_product_variant_reorder.py`
  con casos: (a) `PATCH .../reorder` con una lista válida en orden distinto asigna
  `display_order=1..N` según esa lista; (b) la misma llamada con un ID ajeno al producto, un ID
  repetido, o un ID de una presentación desactivada responde `422` sin modificar ninguna fila —
  depende de T008

### Implementación frontend para User Story 1

- [X] T010 [US1] En `pos-heladeria/src/app/modules/products/pages/product-form.component.ts`:
  importar `DragDropModule` (`@angular/cdk/drag-drop`) como standalone import; envolver el `@for (v
  of draft().variants; track v.localId)` de la línea 181 en `cdkDropList` con
  `(cdkDropListDropped)="onVariantDrop($event)"`, cada fila como `cdkDrag`; agregar
  `onVariantDrop(event: CdkDragDrop<VariantDraft[]>)` que aplica `moveItemInArray` sobre una copia
  de `draft().variants` y actualiza el signal vía `this.draft.update(d => ({...d, variants:
  reordered}))` — puramente local, sin ninguna llamada HTTP (research.md Decisión 3b/6). El `@for`
  de la línea 202 (`draft().deactivated`) NO se toca — depende de T004 (Foundational)
- [X] T011 [P] [US1] En `pos-heladeria/src/app/modules/products/services/product.service.ts`,
  agregar `reorderVariants(productId: string, orderedIds: string[])` que llama
  `PATCH /products/{productId}/variants/reorder` — depende de T008
- [X] T012 [US1] En `product.service.ts`, dentro de `saveExistingProduct` (líneas ~478-513), agregar
  un `await this.reorderVariants(productId, draft.variants.map(v => v.id))` como paso adicional
  después del loop existente de `patch/post/delete` por variante — solo si el orden de
  `draft.variants` cambió respecto al orden con el que se cargó el formulario, para no disparar una
  llamada innecesaria cuando nada se arrastró — depende de T010, T011 (research.md Decisión 2)

### Tests para User Story 1 (frontend)

- [X] T013 [P] [US1] En `pos-heladeria/src/app/modules/products/pages/product-form.component.spec.ts`
  (créalo si aún no existe), agregar casos: simular un `CdkDragDrop` reordena `draft().variants` y
  renumera de inmediato sin ninguna llamada HTTP; llamar a `save()`/`saveExistingProduct()` sobre un
  producto con el orden ya modificado dispara exactamente una llamada a `reorderVariants` con el
  nuevo orden; recargar el `draft` sin haber guardado (releer del producto original) revierte al
  orden previo — depende de T012

**Checkpoint**: la Historia 1 es funcional y verificable de forma independiente — arrastrar
reordena y renumera de inmediato en el formulario, y el nuevo orden solo se persiste al guardar.

---

## Phase 4: User Story 2 - El orden definido se refleja en el detalle del producto en el Menú QR (Priority: P1)

**Goal**: que el detalle de un producto en el Menú QR liste sus presentaciones en el mismo orden
guardado desde el formulario (spec FR-004, research.md Decisión 7).

**Independent Test**: reordenar y guardar las presentaciones de un producto en el formulario, y
verificar que una lectura de ese producto a través de la misma relación que usa el Menú QR
(`product.variants`) devuelve las presentaciones en el nuevo orden.

### Tests para User Story 2

- [X] T014 [P] [US2] En `test_product_variant_reorder.py`, agregar un caso: tras llamar
  `reorder_variants` (T007) sobre un producto, leer `product.variants` (la misma relación que usa
  `pos-backend/app/api/v1/menu/router.py:110-114`) y confirmar que el orden devuelto coincide
  exactamente con `variant_ids` enviado — depende de T007, T004
- [X] T015 [P] [US2] En el mismo fichero, agregar un caso de regresión: para un producto que nunca
  fue reordenado tras aplicar la migración (T002), `product.variants` devuelve el mismo orden que
  tenía antes de esta funcionalidad (orden de creación, vía backfill `ROW_NUMBER()`) — verifica
  FR-009/SC-004 — depende de T002

**Checkpoint**: la Historia 2 es funcional y verificable de forma independiente — el Menú QR (a
través de la misma relación ORM) ya refleja cualquier reordenamiento guardado, sin haber tocado
`menu/router.py` directamente.

---

## Phase 5: User Story 3 - Reordenar convive con crear, editar y eliminar presentaciones (Priority: P2)

**Goal**: que crear, editar (nombre/precio) y eliminar (soft-delete)/reactivar presentaciones siga
funcionando exactamente igual sobre un producto ya reordenado, sin dejar huecos ni alterar
posiciones que no correspondan (spec FR-005/FR-006/FR-007/FR-008, research.md Decisión 4).

**Independent Test**: sobre un producto ya reordenado, agregar una presentación nueva (debe ir al
final), eliminar una intermedia (las demás conservan su orden relativo, sin huecos visibles),
editar nombre/precio de una existente (su posición no cambia), y reactivar una desactivada
(recupera el orden que tenía).

### Tests para User Story 3

- [X] T016 [P] [US3] En `test_product_variant_reorder.py`, agregar un caso: sobre un producto ya
  reordenado, `POST /products/{id}/variants` asigna a la nueva presentación
  `display_order = MAX(...) + 1`, sin alterar el `display_order` de las demás — depende de T005
- [X] T017 [P] [US3] En el mismo fichero, agregar un caso: `DELETE /variants/{id}` (soft-delete)
  sobre una presentación intermedia de un producto reordenado no modifica su `display_order` ni el
  de ninguna otra presentación (activa o inactiva) del mismo producto — sin ningún `422` de la
  constraint `UNIQUE` — depende de T002, T003
- [X] T018 [P] [US3] Ejecutar `pos-backend/app/characterization_tests/test_variantes_duplicadas.py`
  sin modificarlo, y agregar (en `test_product_variant_reorder.py`) un caso adicional: reactivar
  (`PATCH /variants/{id} {"active": true}`) una presentación previamente desactivada conserva
  exactamente el `display_order` que tenía antes de desactivarse — depende de T002, T003
- [X] T019 [P] [US3] En el mismo fichero, agregar un caso: `PATCH /variants/{id}` con solo
  `name`/`price` en el body no incluye ni modifica `display_order` — depende de T003
- [X] T020 [P] [US3] En `product-form.component.spec.ts`, agregar casos: agregar una presentación
  nueva en el `draft` la coloca al final de `draft().variants` sin reordenar las demás; eliminar
  (soft-delete) una presentación intermedia la mueve de `draft().variants` a `draft().deactivated`
  sin alterar el orden relativo de las que quedan en `variants` — depende de T010

**Checkpoint**: todas las historias son funcionales de forma independiente — reordenar convive sin
fricción con el resto del ciclo de vida de una presentación.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: validación de extremo a extremo y verificación de no-regresión sobre el resto del
catálogo.

- [X] T021 Ejecutar `quickstart.md` completo (Historias 1 a 3) contra `pos-backend`/`pos-heladeria`
  con la migración de T002 ya aplicada
- [X] T022 [P] Correr `python -m unittest discover app/characterization_tests -v` completo en
  `pos-backend` y confirmar que toda la suite existente (specs 002, 003, 004, 027, entre otras)
  sigue en verde sin modificación
- [X] T023 [P] Correr la suite completa de Vitest en `pos-heladeria` y confirmar que ningún otro
  spec de `product-form.component.spec.ts` o de módulos relacionados quedó roto

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA las 3 historias de usuario.
- **User Stories (Phase 3-5)**: todas dependen de Foundational. US1 y US2 son P1 y pueden avanzar en
  paralelo si hay más de una persona (US2 solo necesita T004 y T007 de US1 para tener algo que
  probar reordenado — ver nota abajo); US3 depende conceptualmente de que exista *algo* que
  reordenar, pero sus tests no dependen de que la UI de arrastre (T010-T013) esté terminada, solo de
  la capa de datos/backend (T002-T005, T007).
- **Polish (Phase 6)**: depende de que las historias que se quieran entregar estén completas.

### User Story Dependencies

- **User Story 1 (P1, MVP)**: depende solo de Foundational. Es la única historia que agrega UI y el
  endpoint de reordenamiento — las otras dos dependen de que este endpoint exista para tener un
  orden no trivial que verificar.
- **User Story 2 (P1)**: depende de Foundational (T004) y de que exista una forma de producir un
  orden no trivial para probar contra él — en la práctica, T014 depende de T007 (US1) aunque el
  *comportamiento* que verifica (US2) es completamente independiente y no requiere ningún código
  nuevo propio.
- **User Story 3 (P2)**: depende de Foundational (T002, T003, T005) y, para sus casos de arrastre en
  UI (T020), de US1 (T010).

### Parallel Opportunities

- T003 y T004 (Foundational) pueden ejecutarse en paralelo (ficheros distintos).
- T006 (schemas) puede empezar en paralelo con el resto de Foundational, ya que no depende de T002-T005.
- T010 y T011 (US1, frontend) son paralelos entre sí; T009, T014, T015, T016-T020 (todos tests) son
  paralelos entre sí una vez sus prerrequisitos de implementación están listos.
- T022 y T023 (Polish) son paralelos entre sí.

---

## Parallel Example: Foundational + User Story 1

```bash
# Foundational, en paralelo tras T002:
Task: "Agregar display_order a ProductVariant en app/models/product_variant.py"
Task: "Agregar order_by a Product.variants en app/models/product.py"

# User Story 1, en paralelo tras Foundational:
Task: "Agregar VariantReorderRequest/Response en app/api/v1/catalog/schemas.py"
Task: "Envolver el @for de presentaciones activas en CdkDropList/CdkDrag en product-form.component.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (CRÍTICO — bloquea todas las historias)
3. Completar Phase 3: User Story 1
4. **DETENER y VALIDAR**: probar la Historia 1 de forma independiente (quickstart.md)
5. Desplegar/demostrar si está listo — ya resuelve el problema central reportado por el usuario

### Incremental Delivery

1. Setup + Foundational → base lista (columna existe, se asigna y se lee ordenada en todos lados)
2. Agregar Historia 1 → probar de forma independiente → demo (MVP: arrastrar y guardar)
3. Agregar Historia 2 → probar de forma independiente → demo (el Menú QR ya refleja el orden)
4. Agregar Historia 3 → probar de forma independiente → demo (crear/editar/eliminar siguen intactos)
5. Cada historia agrega valor sin romper las anteriores

---

## Notes

- [P] = ficheros distintos, sin dependencias entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- Ningún characterization test existente se modifica — solo se agrega
  `test_product_variant_reorder.py` (nuevo) y se ejecuta (sin tocar)
  `test_variantes_duplicadas.py` como verificación de no-regresión (T018)
- Verificar que los tests fallan antes de implementar, cuando aplique (T009 antes de T007/T008 si se
  sigue TDD estricto; el orden aquí lista implementación primero por claridad de dependencias, pero
  ambos órdenes son válidos)
- Confirmar en T001 que `187e491e597a` sigue siendo el `head` real antes de fijar el
  `down_revision` de T002
