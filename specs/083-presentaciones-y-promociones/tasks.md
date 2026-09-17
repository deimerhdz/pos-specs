---

description: "Task list template for feature implementation"
---

# Tasks: Catálogo de Presentaciones y Rediseño de Promociones

**Input**: Documentos de diseño de `/specs/083-presentaciones-y-promociones/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md) (D1–D10), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: Este proyecto usa *characterization tests* (`app/characterization_tests/`, `python -m unittest`) en `pos-backend` como árbitro de comportamiento (Principio III), no TDD clásico — se generan tareas de test junto a cada endpoint/lógica nueva, no antes. `pos-heladeria` usa `ng test` sobre los `*.spec.ts` ya existentes.

**Organización**: Las tareas se agrupan por historia de usuario (US1 → US2 → US3, mismo orden de prioridad que `spec.md`) para que cada una sea implementable y probable de forma independiente, sobre una única rama de implementación `feat/090-presentaciones-y-promociones` en `../pos-backend` y `../pos-heladeria` (spec.md §Assumptions).

## Formato: `[ID] [P?] [Story] Descripción`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3)
- Cada tarea incluye la ruta exacta del archivo, relativa a `../pos-backend`, `../pos-heladeria` o `pos-specs` (esta carpeta) según corresponda

## Convenciones de ruta

- `../pos-backend/...` — API FastAPI + SQLAlchemy + Alembic
- `../pos-heladeria/...` — SPA Angular 21 (standalone)
- Sin prefijo — archivo dentro de este repo (`pos-specs`), p. ej. `specs/000-reconocimiento/registro-de-anomalias.md`

---

## Phase 1: Setup

**Propósito**: preparar las ramas de trabajo en ambos repositorios de código.

- [X] T001 Crear la rama `feat/090-presentaciones-y-promociones` desde `develop` en `../pos-backend` y en `../pos-heladeria` (convención ya fijada por el equipo, spec.md §Clarifications pregunta 4; independiente del número `083` de esta carpeta de spec)
- [X] T002 Eliminar el `__pycache__` huérfano en `../pos-backend/app/api/v1/presentations/` (único contenido del directorio hoy — resto de `.py` de spec 040/063 ya borrados, research.md D1)

**Checkpoint**: ambos repos listos sobre la rama de esta funcionalidad.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Propósito**: el esquema de base de datos que comparten US1 y US2 — una sola migración aditiva con ambas tablas nuevas (data-model.md), tal como está diseñada; no se puede partir en dos migraciones sin contradecir D8.

**⚠️ CRITICAL**: Ninguna historia de usuario puede implementarse hasta terminar esta fase.

- [X] T003 [P] Crear el modelo `Presentation` en `../pos-backend/app/models/presentation.py` — espejo de `app/models/option_group.py`: `id` (UUID PK), `name` (String(255), `unique=True`, `nullable=False`), `active` (Boolean, default `True`), `created_at`/`updated_at` (`TimestampMixin`), `__table_args__ = {"schema": "tenant"}` (data-model.md §`Presentation`)
- [X] T004 [P] Crear el modelo `CategoryPresentation` en `../pos-backend/app/models/category_presentation.py` — espejo de `app/models/variant_option_group.py`: `id` (UUID PK), `category_id` (FK → `categories.id`, `ondelete="CASCADE"`, indexado), `presentation_id` (FK → `presentations.id`, **sin** `ondelete`, indexado), `UniqueConstraint("category_id", "presentation_id")`, `__table_args__` con `schema="tenant"` (data-model.md §`CategoryPresentation`)
- [X] T005 Agregar la relación `presentations: Mapped[List["Presentation"]] = relationship(secondary="tenant.category_presentations")` a `Category` en `../pos-backend/app/models/category.py` (import de `Presentation` bajo `TYPE_CHECKING`, research.md D2)
- [X] T006 Registrar `from .presentation import Presentation` y `from .category_presentation import CategoryPresentation` en `../pos-backend/app/models/__init__.py`, junto al bloque "Catálogo / menú" (después de `VariantOptionGroup`), para que Alembic autogenere/reconozca ambas tablas nuevas
- [X] T007 Crear la migración Alembic `../pos-backend/alembic/versions/XXXX_083_presentaciones_aditivo.py` con `down_revision = "ef0abdf40889"` (head actual verificado), decorada `@for_each_tenant_schema`: `CREATE TABLE` de `presentations` y `category_presentations` con las columnas/constraints de T003/T004; paso de datos de siembra (FR-019) `INSERT INTO {schema}.presentations (id, name, active, created_at, updated_at) SELECT gen_random_uuid(), v.name, true, now(), now() FROM (SELECT DISTINCT name FROM {schema}.product_variants) AS v`, con guarda `_has_table(schema, "product_variants")` (mismo patrón que `94144eaa60b5`/`387ef3e638cd`); sin ninguna fila en `category_presentations` (FR-019 explícito); `downgrade()` con `DROP TABLE` de ambas tablas, sin intentar revertir el paso de datos (data-model.md §Migración Alembic)
- [X] T008 Crear el characterization test `../pos-backend/app/characterization_tests/test_presentations_migration.py`: factoriza el paso de datos de T007 en una función pura testeable contra SQLite en memoria (`schema_translate_map={"tenant": None}`, mismo patrón que `test_promotions_migration.py`); verifica una `Presentation` activa por cada `product_variants.name` distinto y cero filas en `category_presentations` (FR-019)

**Checkpoint**: esquema de base de datos listo — US1 y US2 pueden implementarse (en paralelo o en el orden de prioridad P1→P2).

---

## Phase 3: User Story 1 - Administrar el catálogo global de presentaciones (Priority: P1) 🎯 MVP

**Goal**: el administrador puede crear, listar, editar, desactivar y reactivar presentaciones del catálogo global de su tenant desde una sección nueva del panel.

**Independent Test**: crear tres presentaciones ("Pequeño", "Mediano", "Grande"), verlas en el listado, editar el nombre de una, desactivarla, reactivarla, y confirmar que un nombre duplicado (al crear o al editar) se rechaza con 409.

### Implementation for User Story 1

- [X] T009 [US1] Crear `PresentationCreate` (`name: str`, 1–255), `PresentationUpdate` (`name`/`active` opcionales) y `PresentationResponse` (`id`, `name`, `active`, `created_at`, `updated_at`) en `../pos-backend/app/api/v1/presentations/schemas.py` (molde de `app/api/v1/categories/schemas.py`, contracts/catalogo-presentaciones.md)
- [X] T010 [US1] Crear el router `../pos-backend/app/api/v1/presentations/router.py` (molde de `categories/router.py`): `GET ""` paginado (`Page[PresentationResponse]`, filtros `active`/`search` por `ilike`, orden por `name`), `GET "/{id}"` (404 si no existe), `POST ""` (nace `active=true`, `ensure_unique` → 409 si `name` duplicado, FR-003), `PATCH "/{id}"` (renombra con `ensure_unique(..., exclude_id=id)`, togglea `active` en cualquier sentido sin validar referencias, FR-002/US1 Escenario 5) — **sin** `DELETE` (research.md D4)
- [X] T011 [US1] Registrar el router nuevo en `../pos-backend/app/main.py`: `from app.api.v1.presentations.router import router as presentations_router` + `app.include_router(presentations_router, prefix="/api/v1")`, junto a `categories_router`
- [X] T012 [P] [US1] Crear el characterization test `../pos-backend/app/characterization_tests/test_presentations_service.py`: CRUD completo, unicidad al crear y al editar (FR-003, 409 en ambos casos), toggle `active` en ambos sentidos (US1 Escenario 5), 404 en `GET`/`PATCH {id}` inexistente
- [X] T013 [P] [US1] Crear las interfaces `Presentation`, `PresentationForm`, `PresentationCreatePayload`, `PresentationUpdatePayload` en `../pos-heladeria/src/app/modules/presentations/interfaces/presentation.interface.ts` (molde de `modules/categories/interfaces/category.interface.ts`)
- [X] T014 [US1] Crear `PresentationService` en `../pos-heladeria/src/app/modules/presentations/services/presentation.service.ts` — molde exacto de `modules/categories/services/category.service.ts`: `injectPagedQuery` paginado + `allQuery` (para el picker de categorías de US2), `createPresentation`, `updatePresentation`, `toggleActive`, invalidación de queries `['presentations']`
- [X] T015 [US1] Crear `PresentationsPageComponent` en `../pos-heladeria/src/app/modules/presentations/pages/presentations-page.component.ts` — listado paginado + formulario crear/editar + toggle activo/inactivo (molde de `modules/categories/pages/categories-page.component.ts` + `components/category-form.component.ts`)
- [X] T016 [US1] Agregar la ruta de "Presentaciones" al router de Angular apuntando a `PresentationsPageComponent` (mismo archivo de rutas donde ya está registrada la ruta de Categorías)
- [X] T017 [P] [US1] Agregar el ítem "Presentaciones" al grupo `CATÁLOGO` en `../pos-heladeria/src/app/core/config/navigation.config.ts`, sin `moduleKey` (research.md D3 — no gatea por plan, igual que Categorías/Productos)

**Checkpoint**: US1 completamente funcional y probable de forma independiente — el catálogo de presentaciones existe, aunque todavía no se asocia a ninguna categoría.

---

## Phase 4: User Story 2 - Asociar presentaciones a una categoría y heredarlas al crear un producto (Priority: P2)

**Goal**: al asociar presentaciones a una categoría, todo producto nuevo creado dentro de ella nace con esas presentaciones ya creadas como variantes (o "Presentación única" si no hay ninguna asociada).

**Independent Test**: asociar "Pequeña"/"Mediana"/"Grande" a la categoría "Ensaladas"; crear un producto nuevo sin enviar `variants` y verificar que nace con esas 3 variantes a precio $0; crear un producto en una categoría sin presentaciones asociadas y verificar que nace con "Presentación única"; cambiar las presentaciones de "Ensaladas" y confirmar que el producto ya creado no cambia (FR-008).

### Implementation for User Story 2

- [X] T018 [US2] Extender `../pos-backend/app/api/v1/categories/schemas.py`: `CategoryCreate`/`CategoryUpdate` ganan `presentation_ids: list[UUID] | None = None`; `CategoryResponse` gana `presentations: list[PresentationSummary]`; agregar `PresentationSummary` (`id: UUID`, `name: str`) (contracts/categoria-herencia-producto.md)
- [X] T019 [US2] Implementar `_replace_category_presentations(db, category, presentation_ids)` en `../pos-backend/app/api/v1/categories/router.py` (o un `service.py` nuevo si se prefiere separar, research.md D5 deja ambas opciones abiertas): reemplazo total de `category_presentations` para esa categoría, 404 si algún id no existe en `presentations` — mismo patrón que `_replace_option_groups` (`catalog/service.py:178`)
- [X] T020 [US2] Llamar `_replace_category_presentations` desde `create_category`/`update_category` en `../pos-backend/app/api/v1/categories/router.py`: en `POST`, si `presentation_ids` viene presente (incluida lista vacía); en `PATCH`, solo si el campo fue enviado (`None` = no tocar la asociación, `[]` = desasociar todas, research.md D5)
- [X] T021 [US2] Registrar la anomalía **A-74** en `specs/000-reconocimiento/registro-de-anomalias.md` (siguiente número tras A-73): renombre del literal por defecto de variante de `"Single"` a `"Presentación única"`, citando spec 083 §Clarifications (pregunta 2, 2026-09-16) como decisión de negocio — **debe mergearse antes que T022** (Principio II/XII, plan.md D7)
- [X] T022 [US2] Renombrar el literal en `ensure_default_variant` (`../pos-backend/app/api/v1/catalog/service.py:94`) de `name="Single"` a `name="Presentación única"`; actualizar `server_default="Single"` de `ProductVariant.name` (`../pos-backend/app/models/product_variant.py:25`) al mismo texto por consistencia documental (research.md D7, cita A-74 de T021)
- [X] T023 [US2] En `ProductService.create_product` (`../pos-backend/app/api/v1/products/service.py:76-82`), rama `else` de `if data.variants:`: resolver las presentaciones activas asociadas a `product.category_id` (JOIN `CategoryPresentation`+`Presentation` filtrando `active=true`, ordenadas por `name`); si hay ≥1, armar una lista sintética `[VariantSaveIn(name=p.name) for p in presentaciones]` y llamar `self._save_variant_tree(db, product, sintetica)` (FR-005, precio queda en 0 por default del schema); si no hay ninguna, seguir llamando `ensure_default_variant(db, product)` sin cambios de firma (FR-006, ahora crea "Presentación única" tras T022)
- [X] T024 [US2] Actualizar el characterization test RN-CAT-05 en `../pos-backend/app/characterization_tests/test_products_service.py:173-187`: la aserción `variants[0].name` pasa de `"Single"` a `"Presentación única"`, citando A-74 (T021) en el commit
- [X] T025 [P] [US2] Crear el characterization test `../pos-backend/app/characterization_tests/test_categories_presentations.py`: reemplazo total de `presentation_ids` al crear/editar una categoría, desasociación con `[]`, 404 con un id inexistente, y no-retroactividad — cambiar `presentation_ids` de una categoría no modifica las `ProductVariant` de productos ya creados en ella (FR-008)
- [X] T026 [P] [US2] Crear el characterization test `../pos-backend/app/characterization_tests/test_products_presentations_inheritance.py`: producto creado en categoría con N presentaciones activas asociadas nace con N variantes a precio $0 (FR-005); producto creado en categoría sin presentaciones asociadas (o sin categoría) nace con "Presentación única" (FR-006); una presentación inactiva asociada a la categoría no se hereda
- [X] T027 [US2] Extender `Category` (campo `presentations: { id: string; name: string }[]`) y `CategoryUpdatePayload`/`CategoryCreatePayload` (campo `presentation_ids?: string[]`) en `../pos-heladeria/src/app/modules/categories/interfaces/category.interface.ts`
- [X] T028 [US2] Extender `CategoryService` (`../pos-heladeria/src/app/modules/categories/services/category.service.ts`) para incluir `presentation_ids` en los payloads de `createCategory`/`updateCategory`
- [X] T029 [US2] Agregar un multi-select de presentaciones activas a `../pos-heladeria/src/app/modules/categories/components/category-form.component.ts` (mismo patrón visual que el checkbox-list de variantes de `promotions-page.component.ts`), consumiendo `PresentationService.allQuery`/`presentations` (T014) filtrado por `active=true`, pre-marcando las ya asociadas a la categoría aunque estén inactivas (contracts/categoria-herencia-producto.md §`CategoryResponse`)

**Checkpoint**: US1 + US2 funcionales de forma independiente — crear una categoría con presentaciones asociadas hace que los productos nuevos hereden variantes automáticamente, sin afectar productos ya existentes.

---

## Phase 5: User Story 3 - Configurar una promoción con vigencia y reglas de precio por presentación (Priority: P3)

**Goal**: las tres pantallas de administración de promociones (listado, creación, configuración) siguen al 100% los prototipos entregados, sin cambiar el motor de cálculo de descuentos ni ningún endpoint de `app/api/v1/promotions/` (research.md D9).

**Independent Test**: crear la promoción "2X Granizados 8oz" (tipo precio de paquete, vigencia lunes a miércoles 14:00–18:00, producto "Granizado de Mora", presentación "8oz", 2 unidades, $12.000), verla en el listado con su vigencia y resumen de regla, y confirmar que una promoción `Activa` bloquea la edición de tipo/valor/unidades/variantes de sus reglas ya guardadas.

### Implementation for User Story 3

Todas las tareas de esta fase son frontend; **ningún archivo de `app/api/v1/promotions/` cambia** (research.md D9).

- [X] T030 [US3] Rediseñar la Pantalla 1 (listado) en `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts` según `listado-promociones.html`: columnas Promoción/Reglas/Vigencia/Estado/Acciones, pestañas por estado (`STATUS_TABS` ya existente), búsqueda por nombre, paginación 20/50/100 (FR-017) — reutiliza `GET /promotions` y `promotion-condition.util.ts` sin cambio de lógica
- [X] T031 [US3] Rediseñar la Pantalla 2 (creación) según `formulario-crear-promocion.html`: campo Nombre + selector de Tipo (`TYPE_OPTIONS` ya existente: `percent`/`package_price`), retenidos en el estado local del formulario sin llamar `POST /promotions` todavía — el tipo elegido se fija como `type` de todas las reglas que se agreguen en la Pantalla 3 y no se vuelve a preguntar (FR-010)
- [X] T032 [US3] Rediseñar el bloque "Configuración de Vigencia y Horarios" de la Pantalla 3 según `configurar-promocion.html`: días de la semana (multi-select, vacío = todos), fecha inicio (requerida)/fin (opcional), hora desde/hasta (opcionales, ambas o ninguna) — sin cambio de la validación ya existente (`_VigenciaMixin._time_window_pair`, FR-011)
- [X] T033 [US3] Rediseñar el bloque "Paso 1/Paso 2" de la Pantalla 3: buscador de producto con filtro por categoría (FR-012, reutiliza el aplanado `catalogVariants` de `promotions-page.component.ts:701`); selector "PRESENTACIÓN/TAMAÑO" que lista **todas** las variantes reales del producto elegido, sin excluir ninguna (FR-013); campos `UNIDADES`/`PRECIO PROMOCIONAL`; botón "Agregar a la lista" con validación local espejo de `_guard_package_is_discount` para feedback inmediato (FR-016, la autoritativa sigue siendo la respuesta 409 del backend); soporte para agregar varias filas con distinta presentación (FR-014)
- [X] T034 [US3] Resolver la etiqueta de cada variante en el selector de T033 (FR-013): extender el `computed<CatalogVariant[]>` (`promotions-page.component.ts:701`) o agregar un mapa auxiliar `category_id -> Set<nombre_presentación_activa>` a partir de `this.categories.allCategories()` (`CategoryService`, ya inyectado y cargado en `promotions-page.component.ts:750` vía `loadAllCategories()`, T027/T018 le agregan `presentations`) — nombre de la presentación si `variant.name` coincide exactamente con una presentación activa asociada a la categoría del producto, `"Presentación única"` si el producto no maneja variaciones, nombre propio de la variante en cualquier otro caso
- [X] T035 [US3] Mostrar en cada fila de regla agregada el precio regular tachado + el ahorro calculado (FR-015), usando `promotion-pricing.util.ts` ya existente sin cambio de lógica
- [X] T036 [US3] Reflejar en la pantalla de configuración el bloqueo de una promoción `Activa` (FR-018, ya vigente en backend spec 063): aviso "Tipo seleccionado... Fijado en creación" sin control editable, campos de cada regla ya guardada de solo lectura; nombre/descripción/fin de vigencia/días/horas siguen editables vía `PATCH /promotions/{id}`
- [X] T037 [P] [US3] Actualizar `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.spec.ts` con los casos de UI nuevos que apliquen (fidelidad de listado/creación/configuración, mapeo de etiqueta de T034), sin tocar ningún caso que congele el motor de cálculo existente

**Checkpoint**: las tres historias de usuario son funcionales de forma independiente — el catálogo de presentaciones, su herencia en producto, y el rediseño de promociones.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Propósito**: verificación final exigida por Principio X y por el checklist de `quickstart.md`.

- [X] T038 Ejecutar `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` en `../pos-backend` y confirmar que ningún test `"CONGELA comportamiento actual:"` de promociones/categorías/productos quedó en rojo (quickstart.md Paso 0, repetido al final)
- [X] T039 [P] Ejecutar `ng test` en `../pos-heladeria` y confirmar que la suite de Categorías/Promociones/Presentaciones sigue en verde
- [X] T040 [P] Probar la migración de T007 en ambos sentidos (`alembic upgrade head` / `alembic downgrade -1`) contra una base con datos reales de al menos un tenant con variantes existentes, verificando la siembra (quickstart.md Paso 1)
- [X] T041 Recorrer manualmente `quickstart.md` completo (Pasos 2 a 4: US1, US2, US3) y marcar su checklist final "Antes de dar por completada esta spec": los 5 Acceptance Scenarios de US1, 5 de US2 y 7 de US3, y las 6 Success Criteria (SC-001 a SC-006)
- [X] T042 [P] Comparar visualmente las tres pantallas de promociones rediseñadas contra los prototipos `~/Escritorio/promociones/*.html` (SC-004, verificable sin leer código)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato.
- **Foundational (Phase 2)**: depende de Setup. **BLOQUEA** US1 y US2 (ambos necesitan las tablas nuevas); no bloquea US3 en términos de esquema, pero US3 solo tiene sentido probarlo después de que exista al menos un producto con variantes (puede ser manual, sin depender de US1/US2 en código).
- **US1 (Phase 3)**: depende de Foundational. Sin dependencia de otra historia.
- **US2 (Phase 4)**: depende de Foundational. Depende de **US1** solo en el frontend (T029 consume `PresentationService` de T014) — el backend de US2 (T018-T026) no depende de los endpoints de US1, solo del modelo `Presentation` ya creado en Foundational.
- **US3 (Phase 5)**: depende de Foundational. La fidelidad visual (T030-T033, T035, T036) es independiente de US1/US2 — puede probarse con productos que ya tengan variantes creadas a mano (plan.md, "Evolución Incremental"). Solo T034 (mapeo de etiqueta FR-013) necesita `CategoryResponse.presentations`, entregado por **US2** (T018/T027) — por eso esta fase se implementa después de US2 en el orden de este documento, aunque T030-T033/T035/T036 podrían adelantarse en paralelo a US2 si hay más de una persona disponible.
- **Polish (Phase 6)**: depende de que US1, US2 y US3 estén completas.

### Parallel Opportunities

- Foundational: T003 y T004 (modelos nuevos, archivos distintos) en paralelo.
- US1: T012 (test backend), T013 y T017 (archivos frontend independientes) en paralelo entre sí y con el resto de la fase.
- US2: T025 y T026 (archivos de test nuevos y distintos) en paralelo. El backend de US2 (T018-T026) puede avanzar en paralelo al frontend de US1 (T013-T017) si hay más de una persona, ya que tocan repos/módulos distintos.
- US3: T030-T033/T035/T036 (mismo archivo, `promotions-page.component.ts`) son secuenciales entre sí; T037 (archivo `.spec.ts` distinto) en paralelo.
- Polish: T039, T040 y T042 en paralelo entre sí.

---

## Parallel Example: Foundational

```bash
# Modelos nuevos, archivos distintos, sin dependencia entre sí:
Task: "Crear el modelo Presentation en ../pos-backend/app/models/presentation.py"
Task: "Crear el modelo CategoryPresentation en ../pos-backend/app/models/category_presentation.py"
```

## Parallel Example: User Story 1

```bash
# Backend (test) y frontend (interfaces + navegación), archivos y repos distintos:
Task: "Test de caracterización test_presentations_service.py"
Task: "Interfaces presentation.interface.ts"
Task: "Ítem 'Presentaciones' en navigation.config.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (bloquea todo lo demás)
3. Completar Phase 3: User Story 1
4. **Parar y validar**: recorrer el Independent Test de US1 (crear/editar/desactivar/reactivar presentaciones)
5. Esto ya es demostrable: el catálogo de presentaciones existe y se administra solo, aunque todavía no cambia el comportamiento de Categorías/Productos/Promociones

### Entrega incremental

1. Setup + Foundational → esquema de base de datos listo
2. + US1 → catálogo de Presentaciones administrable (MVP)
3. + US2 → asociación por categoría y herencia automática al crear producto
4. + US3 → rediseño completo de las pantallas de Promociones, con el mapeo de etiqueta de FR-013 ya alimentado por US2
5. Cada incremento agrega valor sin romper el anterior — ninguno mezcla migración de datos con cambio de UI ni con refactorización no relacionada (plan.md, Principio VI)

### Nota sobre A-74 (T021)

T021 (registrar la anomalía en `registro-de-anomalias.md`) **debe mergearse antes** que T022 (el cambio de código que renombra `"Single"` → `"Presentación única"`) — es un requisito de Principio II/XII, no una sugerencia de orden. Si se implementa en paralelo por más de una persona, T021 no puede quedar pendiente cuando T022 se mergea.

---

## Phase 7: Convergence

**Propósito**: `/speckit-converge` (2026-09-16) evaluó el código real de `pos-backend`/
`pos-heladeria` (rama `feat/090-presentaciones-y-promociones`, sin commitear) contra
`spec.md` tras la sesión de `/speckit-clarify` que agregó FR-017/FR-020–FR-024 (correcciones
post-primera-ronda-de-pruebas). Hallazgo principal: FR-017 y FR-020 a FR-023 ya estaban
completamente implementados (860 tests de `pos-backend` en verde, incluida la nueva
anomalía A-75; suite de `pos-heladeria` en verde para categorías/promociones/presentaciones)
aunque ninguno tenía tarea propia en las Fases 1–6. Solo quedan dos brechas reales:

- [X] T043 Crear `../pos-heladeria/src/app/modules/presentations/pages/presentations-page.component.spec.ts` cubriendo listado paginado, crear, editar/renombrar (incluido el rechazo 409 por nombre duplicado, FR-003) y toggle activar/desactivar en ambos sentidos (US1 Escenario 5) — mismo patrón de cobertura que `categories-page.component.spec.ts`/`category-form.component.spec.ts`, los componentes molde que T015 indicó calcar; hoy `presentations-page.component.ts` (312 líneas, entregable central de US1/MVP) no tiene ningún test propio per Constitution X / US1 Independent Test (missing)
- [X] T044 Reconciliar la altura del contenedor con scroll del grid de productos del Paso 1 (`../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts:545`, implementado hoy como `max-h-[360px]`) con el valor de 320px que declara `spec.md` FR-024 — ajustar el código a 320px, o corregir el valor documentado en FR-024 si 360px es el que se prefiere, para que código y spec dejen de contradecirse per FR-024 (contradicts)

**Checkpoint**: con T043 y T044 resueltas, el código converge por completo con `spec.md` (incluidas las clarificaciones de la sesión post-pruebas) sin dejar hallazgos accionables pendientes.

---

## Phase 8: Convergence (2026-09-17)

**Propósito**: `/speckit-converge` (2026-09-17) evaluó el código real de `pos-backend`/
`pos-heladeria` (rama `feat/090-presentaciones-y-promociones`, sin commitear) contra `spec.md`
tras la sesión de `/speckit-clarify` del mismo día que agregó FR-025 a FR-027 (unidades mínimas,
precio vs. suma de precios regulares, y exclusión de toppings del descuento por porcentaje en
reglas de precio de paquete). Estos tres FR no tienen ninguna tarea propia en las Fases 1–7 ni
están mencionados en `plan.md`/`research.md`/`data-model.md`/`quickstart.md` — son requisitos
posteriores a esos artefactos. T043 y T044 (Fase 7) se reverificaron y siguen resueltos:
`presentations-page.component.spec.ts` existe y el contenedor de tarjetas del Paso 1 usa
`max-h-[320px]`.

- [X] T045 Agregar la validación `minQuantity (min_qty) >= 2` para toda regla de tipo
  `package_price`: backend en `PromotionRuleIn`/`_guard_package_is_discount`
  (`../pos-backend/app/api/v1/promotions/schemas.py`, `service.py`) — cubre creación y edición a
  la vez porque ambas comparten el mismo schema (`create` y `update_shape`); frontend en
  `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`: inicializar
  `pickerQty` en 2 cuando el tipo de promoción es `package_price`, impedir ingresar o decrementar
  por debajo de 2, y mostrar "En promociones por paquete, el mínimo es de 2 unidades" — sin
  aplicar el mínimo a reglas ya guardadas con `min_qty < 2` (no retroactivo) per FR-025 (done:
  validador `_package_price_min_qty` en `schemas.py`; `pickerQty`/`pickerLabel`/`pickerValue`
  pasaron a signals para soportar `minPickerQty()`/`onPickerQtyChange()`; tests nuevos en
  `test_promotions_rules_admin.py` y `promotions-page.component.spec.ts`)
- [X] T046 Corregir el guard de precio de paquete para que compare contra la suma de precios
  regulares de la variante elegida (`precio_regular_de_la_variante × unidades`), no contra el
  precio más barato del conjunto de la regla: ajustar `_guard_package_is_discount`
  (`../pos-backend/app/api/v1/promotions/service.py:619-648`, llamado en `create` y
  `update_shape`) y su espejo en `addRuleRow()`
  (`../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts:1145-1177`)
  para usar exactamente esa fórmula y el mensaje "El precio promocional ($X) debe ser menor a la
  suma del precio regular de los productos seleccionados ($Y)." en ambos lados; en frontend,
  reemplazar la validación puntual al hacer clic en "Agregar a la lista" por un recálculo
  dinámico (`computed`/efecto) que se actualice cada vez que cambien producto, presentación,
  unidades o precio, e incorporar el resultado a `canAddRuleRow()` per FR-026 (done: la fórmula
  ya coincidía para el caso de una sola variante de FR-013 — se ajustó el mensaje/campo
  `regular_price_sum` en ambos lados y se agregaron `packagePriceCheck`/
  `packagePriceExceedsRegularSum` como `computed` consumidos por `canAddRuleRow()`/`addRuleRow()`;
  renombrado `cheapest_unit_price` -> `regular_price_sum` en `PackageNotDiscountError`/
  `promotion.service.ts`)
- [X] T047 Separar el precio base de la variante del precio de los toppings/`OptionGroup` antes de
  aplicar un descuento de tipo porcentaje: en backend, ajustar `compute_line_price`
  (`../pos-backend/app/catalog_engine/core.py:39-46`), `promo_lines_for`
  (`../pos-backend/app/api/v1/orders/checkout.py:313-333`) y `evaluate_variant_sets`
  (`../pos-backend/app/api/v1/promotions/service.py:224-287`) para que, cuando `type == percent`,
  el `%` se calcule solo sobre el precio base de la variante y el precio de cada topping elegido
  se sume íntegro (sin descuento) al total de la línea — sin cambiar el cálculo para
  `package_price`; reflejar el mismo criterio en el preview de frontend
  (`../pos-heladeria/src/app/modules/promotions/services/promotion-pricing.util.ts`,
  `pos-terminal.store.ts`) per FR-027 (done: `evaluate_variant_sets` usa `base_unit_price` para el
  descuento `percent` y `unit_price` completo sin cambios para `package_price`; `base_unit_price`
  se agregó como propiedad de `SaleLine` (`sales/builder.py`) y se propaga desde
  `checkout.promo_lines_for` y `cart.service._cart_promo_lines`; no se tocó `compute_line_price`
  -- ese cálculo sigue siendo el precio TOTAL de la línea, que es lo que necesitan `unit_price`/
  `line_total`, la separación ocurre después, al armar `promo_lines`; no se tocó
  `promotion-pricing.util.ts`/`pos-terminal.store.ts`: ya delegan el 100% del cálculo de
  descuento al backend (`discounted_unit_price`/`discounted_line_total`), sin ninguna réplica
  local de la fórmula porcentual que ajustar; tests nuevos: `test_promotions_service.py`
  (`evaluate_variant_sets` puro) y `test_promotions_toppings_base_price.py` (`SaleLine`,
  `promo_lines_for`, `_cart_promo_lines`))

**Checkpoint**: con T045–T047 resueltas, el código converge con `spec.md` incluida la sesión de
Clarifications 2026-09-17 (FR-025 a FR-027), sin dejar hallazgos accionables pendientes.
