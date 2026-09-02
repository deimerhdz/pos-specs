---

description: "Task list for Orden personalizado de categorías en el filtro del menú QR"
---

# Tasks: Orden personalizado de categorías en el filtro del menú QR

**Input**: Design documents from `/specs/067-orden-categorias-menu-qr/` (plan.md, spec.md,
research.md, data-model.md, contracts/, quickstart.md)

**Tests**: incluidos — `plan.md` (Technical Context) y `quickstart.md` fijan de antemano qué
fichero de test crea o extiende cada historia (Constitución, Principio X: Verificación
Obligatoria), así que no son opcionales para esta spec. Ningún characterization test existente se
modifica (Principio III no aplica — no hay ninguno que documente orden de categorías hoy).

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
  `pos-backend/alembic/versions/` que `a96852d7be6a` sigue siendo la revisión `head` real
  (research.md); si otra spec agregó una migración más reciente entre el 2026-09-01 y ahora, usar
  esa nueva revisión como `down_revision` en T002 en su lugar.

**Checkpoint**: entornos listos, línea base verde confirmada, `down_revision` de T002 confirmado.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: hacer existir la columna `display_order` de punta a punta (BD → modelo → schemas →
asignación al crear/editar) sin tocar todavía ningún punto de lectura ni ninguna UI — prerrequisito
real y compartido por las 3 historias (ninguna puede construirse sin que el campo exista y ya se
asigne correctamente).

**⚠️ CRITICAL**: ninguna historia puede empezar hasta que esta fase esté completa.

- [X] T002 Crear la migración
  `pos-backend/alembic/versions/<rev>_categories_display_order.py`, `down_revision='a96852d7be6a'`
  (T001), siguiendo el esqueleto `@for_each_tenant_schema` ya usado por migraciones previas (spec
  042, `c8ff3a5551cb_product_variants_display_order.py`): en `upgrade()`, por cada schema de
  tenant — (a) `op.add_column("categories", sa.Column("display_order", sa.Integer(), nullable=True),
  schema=schema)`; (b) backfill con
  `UPDATE categories c SET display_order = sub.rn FROM (SELECT id, ROW_NUMBER() OVER (ORDER BY
  name DESC) AS rn FROM categories) sub WHERE c.id = sub.id` (vía `op.execute`, con el nombre de
  tabla/schema calificado); (c) `op.alter_column("categories", "display_order", nullable=False,
  schema=schema)`; (d) `op.create_check_constraint("ck_category_display_order_non_negative",
  "categories", "display_order >= 0", schema=schema)`. En `downgrade()`: `op.drop_constraint(...)`
  seguido de `op.drop_column("categories", "display_order", schema=schema)` (data-model.md,
  research.md Decisión 4/5)
- [X] T003 [P] En `pos-backend/app/models/category.py`, agregar
  `display_order: Mapped[int] = mapped_column(Integer, nullable=False)` junto a `active` (línea
  ~18) — sin default de aplicación: toda ruta que construya `Category(...)` debe asignarlo
  explícitamente (T005); agregar el import de `Integer`/`CheckConstraint` desde `sqlalchemy` y
  cambiar `__table_args__` (hoy `({"schema": "tenant"},)`, línea 22) para incluir
  `CheckConstraint("display_order >= 0", name="ck_category_display_order_non_negative")` junto al
  diccionario de schema (research.md Decisión 5)
- [X] T004 [P] En `pos-backend/app/api/v1/categories/schemas.py`, agregar
  `display_order: int | None = Field(None, ge=0, description="Posición en el filtro del menú QR;
  si se omite, se asigna automáticamente al final de la lista actual.", examples=[10])` a
  `CategoryCreate` y a `CategoryUpdate`; agregar `display_order: int = Field(...,
  description="Posición en el filtro del menú QR.", examples=[10])` a `CategoryResponse`
  (contracts/categorias-orden.md)
- [X] T005 En `pos-backend/app/api/v1/categories/router.py`: en `create_category`, cuando
  `body.display_order is None`, calcular
  `select(func.max(Category.display_order))` (importar `func` de `sqlalchemy`) y asignar
  `(current_max or 0) + 1`; cuando venga explícito, asignar `body.display_order` tal cual (validado
  por T004). En `update_category`, si `body.display_order is not None`, asignar
  `category.display_order = body.display_order` (mismo patrón que ya usa para `description`) —
  depende de T003, T004 (data-model.md, tabla de asignación; FR-001 a FR-004)

**Checkpoint**: `display_order` existe de punta a punta y se asigna correctamente al crear
(explícito o `MAX+1` automático) y al editar una categoría — sin que ningún punto de lectura lo use
todavía para ordenar, y sin que ninguna UI lo muestre o lo edite.

---

## Phase 3: User Story 1 - Definir el orden al crear o editar una categoría (Priority: P1) 🎯 MVP

**Goal**: que un administrador pueda escribir un valor de orden numérico en el formulario de
creación o edición de categorías, que quede guardado y validado (spec FR-001 a FR-004,
Clarifications 2026-09-01).

**Independent Test**: crear una categoría especificando un valor de orden y verificar que queda
guardado; editar una categoría existente cambiando su valor y verificar que se reemplaza; crear una
categoría sin especificar orden y verificar que el sistema le asigna uno automáticamente sin
bloquear el guardado; intentar guardar con un valor negativo o no numérico y verificar que se
rechaza.

### Tests para User Story 1 (backend)

- [X] T006 [P] [US1] Crear `pos-backend/app/characterization_tests/test_category_display_order.py`
  con casos: (a) `POST /categories` con `display_order` explícito lo persiste tal cual; (b)
  `POST /categories` sin `display_order` (u omitiéndolo) asigna `MAX(display_order) + 1` —
  verificable creando dos categorías seguidas sin el campo y comprobando que la segunda recibe un
  valor mayor que la primera; (c) `PATCH /categories/{id}` con un nuevo `display_order` lo
  reemplaza; (d) `PATCH /categories/{id}` sin el campo en el body no modifica el valor existente;
  (e) `POST`/`PATCH` con `display_order` negativo o no numérico responde `422` sin
  crear/modificar la categoría — depende de T005

### Implementación frontend para User Story 1

- [X] T007 [P] [US1] En `pos-heladeria/src/app/modules/categories/interfaces/category.interface.ts`,
  agregar `display_order: number` a `Category`; `display_order: number | null` a `CategoryForm`;
  `display_order: number | null` a `CategoryCreatePayload` y a `CategoryUpdatePayload`
  (contracts/categorias-orden.md)
- [X] T008 [P] [US1] En
  `pos-heladeria/src/app/modules/categories/components/category-form.component.ts`: agregar un
  `FormControl<number | null>('display_order', ...)` al `form` (junto a `name`/`description`, línea
  ~113-119); agregar un campo numérico en el template (junto a "Descripción", línea ~64-75) con
  etiqueta "Orden" y texto de ayuda indicando que, si se deja vacío, la categoría se ubicará primero
  en el filtro del Menú QR; en `ngOnChanges` (línea ~125-137), pre-llenar el control con
  `this.category.display_order` al editar; en `onSubmit` (línea ~149-167), incluir
  `display_order: this.form.controls.display_order.value` en el objeto `CategoryForm` armado —
  depende de T007
- [X] T009 [P] [US1] En
  `pos-heladeria/src/app/modules/categories/services/category.service.ts`, en `createCategory` y
  `updateCategory` (líneas ~106-145), agregar `display_order: data.display_order` al
  `CategoryCreatePayload`/`CategoryUpdatePayload` armado — depende de T007

### Tests para User Story 1 (frontend)

- [X] T010 [P] [US1] En
  `pos-heladeria/src/app/modules/categories/components/category-form.component.spec.ts` (créalo si
  aún no existe), agregar casos: al editar una categoría, el campo de orden se pre-llena con su
  `display_order` actual; guardar con un valor de orden lo incluye en el payload enviado; guardar
  dejando el campo vacío envía `null` (el backend asigna el valor por defecto, T005); el formulario
  no bloquea el guardado por dejar el campo vacío — depende de T008, T009

**Checkpoint**: la Historia 1 es funcional y verificable de forma independiente — crear y editar
categorías con un valor de orden explícito, o dejándolo vacío, funciona de punta a punta.

---

## Phase 4: User Story 2 - Ver el filtro del menú QR ordenado de mayor a menor (Priority: P1)

**Goal**: que el filtro de categorías del Menú QR muestre las categorías activas ordenadas por su
`display_order`, de mayor a menor, con desempate alfabético (spec FR-005 a FR-007, FR-009).

**Independent Test**: configurar varias categorías activas con distintos valores de orden y
verificar que el filtro del Menú QR las lista de mayor a menor; verificar que dos categorías con el
mismo valor se ordenan entre sí alfabéticamente; verificar que una categoría inactiva no aparece sin
importar su valor de orden; verificar que el listado de administración de categorías sigue ordenado
por nombre, sin cambios.

### Implementación para User Story 2

- [X] T011 [US2] En `pos-backend/app/api/v1/menu/router.py`, dentro de `_build_menu` (línea ~96-98),
  cambiar `select(Category).where(Category.active.is_(True)).order_by(Category.name)` por
  `select(Category).where(Category.active.is_(True)).order_by(Category.display_order.desc(),
  Category.name)` — depende de T003 (data-model.md, "Menú QR — consulta modificada")

### Tests para User Story 2

- [X] T012 [P] [US2] En `test_category_display_order.py`, agregar un caso: con tres categorías
  activas de `display_order` 10, 5 y 1, el resultado de `_build_menu` (o `GET /menu`) las devuelve
  en esa misma secuencia (10, 5, 1) — depende de T011
- [X] T013 [P] [US2] En el mismo fichero, agregar dos casos: (a) dos categorías activas con el
  mismo `display_order` aparecen ordenadas entre sí alfabéticamente por `name`; (b) una categoría
  inactiva con un `display_order` alto no aparece en el resultado de `_build_menu` — depende de T011
- [X] T014 [P] [US2] En el mismo fichero, agregar un caso de regresión: `GET /categories` (listado
  de administración) sigue devolviendo las categorías ordenadas por `name`, no por `display_order`,
  tras este cambio — confirma que `categories/router.py:40` no se vio afectado (data-model.md,
  "Listado de administración — SIN CAMBIO DE ORDEN") — depende de T011
- [X] T015 [P] [US2] En el mismo fichero, agregar un caso de regresión de migración: para
  categorías creadas antes de este feature (backfill de T002), el orden que devuelve `_build_menu`
  tras aplicar la migración es idéntico al que tenían por `ORDER BY name ASC` antes de esta
  funcionalidad — verifica FR-009/SC-003 (research.md Decisión 4) — depende de T002, T011

**Checkpoint**: la Historia 2 es funcional y verificable de forma independiente — el filtro del
Menú QR ya refleja el orden configurado, sin haber tocado la UI del comensal
(`public-menu.component.ts` ya itera el arreglo tal como lo recibe, research.md).

---

## Phase 5: User Story 3 - Ver el orden configurado desde la administración de categorías (Priority: P3)

**Goal**: que el listado de administración de categorías muestre el valor de orden de cada
categoría, sin cambiar el orden en que la tabla ya las lista (spec FR — User Story 3, data-model.md).

**Independent Test**: abrir el listado de administración de categorías y verificar que cada fila
muestra su valor de orden actual.

### Implementación para User Story 3

- [X] T016 [US3] En
  `pos-heladeria/src/app/modules/categories/pages/categories-page.component.ts`, agregar una
  columna "Orden" a la tabla (`<th>` junto a "Nombre", línea ~85; `<td>` junto a la celda de
  nombre en el `@for`, línea ~92-101) mostrando `cat.display_order` — sin tocar el `order_by` de
  `categoryService.categories()` (que sigue viniendo de `GET /categories` sin cambios) — depende de
  T004 (Foundational, `CategoryResponse` ya expone el campo)

### Tests para User Story 3

- [X] T017 [P] [US3] En
  `pos-heladeria/src/app/modules/categories/pages/categories-page.component.spec.ts` (créalo si
  aún no existe), agregar un caso: la tabla renderiza el `display_order` de cada categoría en la
  nueva columna — depende de T016

**Checkpoint**: todas las historias son funcionales de forma independiente — el orden se define en
el formulario (US1), se refleja en el Menú QR (US2), y se puede revisar desde la administración
(US3).

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: validación de extremo a extremo y verificación de no-regresión sobre el resto del
catálogo.

- [X] T018 Ejecutar `quickstart.md` completo (Historias 1 a 3, más la verificación de migración)
  contra `pos-backend`/`pos-heladeria` con la migración de T002 ya aplicada
- [X] T019 [P] Correr `python -m unittest discover app/characterization_tests -v` completo en
  `pos-backend` y confirmar que toda la suite existente sigue en verde sin modificación
- [X] T020 [P] Correr la suite completa de Vitest en `pos-heladeria` y confirmar que ningún otro
  spec de `categories-page.component.spec.ts`, `category-form.component.spec.ts` o de módulos
  relacionados quedó roto

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciar de inmediato.
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA las 3 historias de usuario.
- **User Stories (Phase 3-5)**: todas dependen de Foundational. US1 y US2 son P1 y pueden avanzar
  en paralelo si hay más de una persona (US2 solo necesita T003 de Foundational para ordenar por la
  columna, no depende de la UI de US1); US3 (P3) solo depende de T004 (Foundational, el campo ya
  expuesto en `CategoryResponse`), no de US1 ni de US2.
- **Polish (Phase 6)**: depende de que las historias que se quieran entregar estén completas.

### User Story Dependencies

- **User Story 1 (P1, MVP)**: depende solo de Foundational. Es la única historia que agrega UI de
  edición del campo — sin ella, `display_order` solo se puede fijar llamando la API directamente.
- **User Story 2 (P1)**: depende de Foundational (T003, la columna debe existir para poder
  ordenar por ella). No depende de US1: puede probarse sembrando valores de `display_order`
  directamente vía API (T005 ya lo permite) sin que exista todavía el campo en el formulario.
- **User Story 3 (P3)**: depende de Foundational (T004, `CategoryResponse` ya expone el campo). No
  depende de US1 ni de US2 — solo necesita que el valor exista y viaje en la respuesta de
  `GET /categories`, que Foundational ya garantiza.

### Parallel Opportunities

- T003 y T004 (Foundational) pueden ejecutarse en paralelo (ficheros distintos).
- T007, T008, T009 (US1) pueden ejecutarse en paralelo una vez completado Foundational (ficheros
  distintos; T008 y T009 dependen de T007 pero no entre sí).
- T012, T013, T014, T015 (US2, todos tests) son paralelos entre sí una vez completado T011.
- T019 y T020 (Polish) son paralelos entre sí.
- Una vez completada Foundational, US1, US2 y US3 pueden avanzar en paralelo entre sí (equipos
  distintos), ya que ninguna depende de las otras dos.

---

## Parallel Example: Foundational + User Story 1

```bash
# Foundational, en paralelo tras T002:
Task: "Agregar display_order + CheckConstraint a Category en app/models/category.py"
Task: "Agregar display_order a CategoryCreate/CategoryUpdate/CategoryResponse en app/api/v1/categories/schemas.py"

# User Story 1, en paralelo tras Foundational:
Task: "Agregar display_order a las interfaces en category.interface.ts"
Task: "Agregar el campo de orden al formulario en category-form.component.ts"
Task: "Incluir display_order en los payloads de category.service.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (CRÍTICO — bloquea todas las historias)
3. Completar Phase 3: User Story 1
4. **DETENER y VALIDAR**: probar la Historia 1 de forma independiente (quickstart.md)
5. Desplegar/demostrar si está listo — el administrador ya puede definir el orden, aunque el Menú
   QR todavía no lo respete visualmente hasta sumar la Historia 2

### Incremental Delivery

1. Setup + Foundational → base lista (columna existe, se asigna correctamente al crear/editar)
2. Agregar Historia 1 → probar de forma independiente → demo (definir el orden desde el formulario)
3. Agregar Historia 2 → probar de forma independiente → demo (el Menú QR ya respeta ese orden)
4. Agregar Historia 3 → probar de forma independiente → demo (el admin lo revisa en el listado)
5. Cada historia agrega valor sin romper las anteriores

---

## Notes

- [P] = ficheros distintos, sin dependencias entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- Ningún characterization test existente se modifica — solo se agrega
  `test_category_display_order.py` (nuevo, no existía ningún characterization test de `categories`)
- Verificar que los tests fallan antes de implementar, cuando aplique (T006/T012-T015 antes de
  T005/T011 si se sigue TDD estricto; el orden aquí lista implementación primero por claridad de
  dependencias, pero ambos órdenes son válidos)
- Confirmar en T001 que `a96852d7be6a` sigue siendo el `head` real antes de fijar el
  `down_revision` de T002
