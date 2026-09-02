# Implementation Plan: Orden personalizado de categorías en el filtro del menú QR

**Branch**: `067-orden-categorias-menu-qr` | **Date**: 2026-09-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/067-orden-categorias-menu-qr/spec.md`

## Summary

Hoy `categories` no tiene ningún campo de orden — confirmado que tanto `GET /categories`
(`app/api/v1/categories/router.py:40`, admin) como el armado del Menú QR
(`app/api/v1/menu/router.py:97`, `_build_menu`) ordenan por `Category.name` (alfabético ascendente),
y el frontend del comensal (`public-menu.component.ts:208`) simplemente itera el arreglo que el
backend ya devuelve, sin reordenar nada del lado del cliente. Este plan agrega una columna
`display_order` (`Integer NOT NULL`, sin restricción de unicidad — spec.md, Assumptions) a
`categories`, y modifica **únicamente** la consulta que arma el Menú QR
(`menu/router.py:97`) para ordenar por `Category.display_order DESC, Category.name ASC` (FR-005,
FR-006); la lista de administración (`categories/router.py:40`) sigue ordenada por nombre —
fuera de alcance según spec.md, Assumptions. El campo se agrega directo a
`CategoryCreate`/`CategoryUpdate`/`CategoryResponse` (Pydantic, `Field(None, ge=0)`) porque es un
valor numérico que el administrador escribe directamente, sin arrastrar-soltar ni endpoint de
reordenamiento dedicado (a diferencia de spec 042, `product_variants.display_order`, que sí
necesitó ambos por ser 1..N sin huecos y con unicidad por producto). Si el administrador no
especifica el valor al crear, el backend asigna `MAX(display_order) + 1` entre todas las
categorías (mismo patrón que `catalog/service.py`, función que calcula el siguiente
`display_order` de una presentación) para que la categoría nueva aparezca primera (FR-004,
clarification 2026-09-01). La migración hace backfill con
`display_order = ROW_NUMBER() OVER (ORDER BY name DESC)`, que al ordenar luego por
`display_order DESC` reproduce exactamente el orden alfabético ascendente que el Menú QR ya
muestra hoy — así desplegar el feature no reordena nada por sí solo (FR-009, clarification
2026-09-01, SC-003).

## Technical Context

**Language/Version**: Backend — Python 3.14 (`pos-backend`, mismo entorno que specs previas, p.
ej. 042). Frontend — TypeScript 5.9.2 (Angular 21.1.x, standalone components + signals, sin
NgModules).

**Primary Dependencies**:
- Backend: FastAPI 0.136.3, SQLAlchemy 2.0 (sync), Pydantic 2, Alembic 1.18.4. Ninguna
  dependencia nueva.
- Frontend: Angular 21 (standalone + signals), Reactive Forms, Tailwind CSS v4. Ninguna
  dependencia nueva — a diferencia de spec 042, este feature no necesita `@angular/cdk` porque no
  hay arrastrar-soltar (el orden se escribe como número en un campo del formulario).

**Storage**: PostgreSQL 16, schema-per-tenant. **Con migración** — nueva columna
`categories.display_order` (`Integer NOT NULL` tras backfill) más
`CheckConstraint("display_order >= 0", name="ck_category_display_order_non_negative")`, agregada
vía `@for_each_tenant_schema` (mismo patrón que `c8ff3a5551cb_product_variants_display_order.py` y
`f5a6b7c8d9e0_availability_change_partial_count.py`). `down_revision` = `a96852d7be6a`
(`a96852d7be6a_option_group_selection_mode_and_item_option_quantity.py`), confirmado como la única
revisión `head` real de `pos-backend/alembic/versions/` (47 migraciones, resuelto siguiendo
`down_revision` — el `alembic heads` de la CLI no corre hoy por un `NameError` de módulo en
`94b7e35f5e5e_063d_promociones_reglas_destructivo.py`, preexistente y sin relación con este
feature). Reverificar que sigue siendo el head antes de implementar, por si otra spec agregó una
migración más reciente entre tanto.

**Testing**: Backend — `unittest` vía `python -m unittest discover -s app/characterization_tests
-p 'test_*.py'` (sin pytest en el repo). Se crea `app/characterization_tests/test_category_display_order.py`
(NUEVO — funcionalidad nueva, no modifica ningún comportamiento congelado existente; no hay
ningún characterization test de `categories` hoy). Frontend — Vitest vía `@angular/build:unit-test`;
se crea o extiende `category-form.component.spec.ts` y `categories-page.component.spec.ts` (a
verificar si ya existen al implementar) con los casos del campo de orden.

**Target Platform**: Linux server (API `pos-backend`) + navegador (SPA `pos-heladeria`) — el
formulario de categorías lo usan administradores de catálogo; el orden resultante lo ve
cualquier comensal que abra el Menú QR (solo lectura, sin ningún control de orden de su lado).

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de este repo `pos-specs`).

**Performance Goals**: sin objetivo de throughput nuevo. `MAX(display_order)` es un solo
`SELECT` adicional dentro de `create_category` (una tabla con, típicamente, unas pocas decenas de
filas por tenant); el `ORDER BY` añadido en `_build_menu` no cambia su complejidad porque ya
ordenaba por una columna indexable de la misma tabla.

**Constraints**:
- El campo `display_order` NO tiene restricción de unicidad (spec.md, Assumptions) — a
  diferencia de `product_variants.display_order` (spec 042), que sí la exige por producto. Esto
  simplifica la migración: no hace falta `UNIQUE ... DEFERRABLE` ni una estrategia de dos pasadas
  para editar el valor de una categoría existente.
- El orden por defecto (`MAX(display_order) + 1`) solo se calcula al **crear** una categoría sin
  valor explícito; editar una categoría existente sin tocar el campo de orden NO recalcula nada
  (FR-002, el valor solo cambia si el administrador lo cambia explícitamente).
- El listado de administración (`GET /categories`, `categories/router.py:40`) sigue ordenado por
  `Category.name` — este feature NO reordena esa tabla, solo el filtro del Menú QR (spec.md,
  Assumptions, alcance limitado explícitamente).

**Scale/Scope**: 1 tabla modificada (`categories`, +1 columna +1 `CheckConstraint`), 1 migración
nueva, 0 endpoints nuevos (se reutilizan `POST`/`PATCH /categories` ya existentes, +1 campo cada
uno), 1 consulta modificada (`_build_menu`, `menu/router.py:97`, +1 `order_by`); en
`pos-heladeria`, 2 componentes modificados (`category-form.component.ts`: +1 campo numérico;
`categories-page.component.ts`: +1 columna en la tabla, User Story 3) + su servicio
(`category.service.ts`, payloads de create/update) + su interfaz (`category.interface.ts`), sin
componentes nuevos.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 3 historias priorizadas y 9 FRs, sesión de clarificación completada (`/speckit-clarify`, 2026-09-01) sin `[NEEDS CLARIFICATION]` pendientes, antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El único comportamiento que cambia es el orden del filtro del Menú QR (hoy implícito/alfabético) por uno explícito y editable — documentado como funcionalidad nueva en spec.md. Ninguna categoría existente cambia su posición visible hasta que un administrador reordene explícitamente (FR-009, clarification 2026-09-01, migración con backfill que reproduce el orden alfabético actual). El listado de administración no se toca (fuera de alcance explícito). | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | No existe ningún characterization test de `categories` hoy — nada que modificar. Los characterization tests de `menu` (si existen) no fijan un orden específico de categorías más allá de "activas primero"; se verificará al implementar que ninguno asuma orden alfabético como comportamiento congelado antes de tocar `_build_menu`. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El orden editable de categorías y su reflejo en el Menú QR es comportamiento enteramente nuevo (no existía ningún concepto de orden antes); no se exige equivalencia con el pasado más allá del backfill inicial (FR-009). | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Se reutilizan los endpoints `POST`/`PATCH /categories` existentes y el patrón `MAX(...)+1` ya usado en `catalog/service.py` para presentaciones, en vez de introducir un endpoint o patrón nuevo. No se toca el listado de administración (`categories/router.py:40`) ni ningún otro comportamiento de `category-form.component.ts`/`categories-page.component.ts` fuera del campo de orden. | PASS |
| **VI. Evolución Incremental** | El alcance se divide en las mismas unidades que las historias del spec (US1 definir el orden en el formulario → US2 reflejo en el filtro del Menú QR → US3 verlo en el listado de administración), cada una verificable por separado (quickstart.md). No se mezcla con ninguna refactorización de `category-form.component.ts` más allá de agregar el campo. | PASS |
| **VII. Compatibilidad con Datos Históricos** | No se toca ninguna venta, factura, pago ni movimiento de inventario ya generado — la migración solo agrega una columna y un `CheckConstraint` a `categories`. | PASS (no aplica directamente) |
| **VIII. Evolución del Modelo de Datos** | Migración especificada completa en data-model.md/research.md: columna nueva, tipo, nulabilidad, estrategia de backfill (`ROW_NUMBER() OVER (ORDER BY name DESC)`, resuelta en la sesión de clarificación 2026-09-01) y estrategia de rollback (`ALTER TABLE ... DROP CONSTRAINT` + `DROP COLUMN`, reversible sin afectar ninguna otra columna). | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se agrega ninguna dependencia nueva en ningún repositorio. | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a comandos `unittest`/Vitest ejecutables. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Que el orden se pueda definir en el formulario, que el filtro del Menú QR lo respete de mayor a menor, y los valores por defecto (categorías nuevas y preexistentes) son decisiones de negocio (spec.md + Clarifications). Modelar `display_order` como columna simple sin unicidad, reutilizar los endpoints existentes en vez de uno de reordenamiento dedicado, y la fórmula exacta del backfill son las decisiones técnicas correspondientes, documentadas en research.md sin mezclarse con las de negocio. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec, con sesión de clarificación) → este `plan.md`/`research.md`/`data-model.md`/`contracts/` (Decisión técnica, verificada contra el código real de `pos-backend`/`pos-heladeria`) → `tasks.md` (Fase 2, no generada por este comando) → implementación → tests nuevos → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/067-orden-categorias-menu-qr/
├── plan.md                          # Este fichero (/speckit-plan)
├── research.md                      # Fase 0 — decisiones técnicas y hechos verificados
├── data-model.md                    # Fase 1 — columna nueva, migración, asignación de orden
├── quickstart.md                    # Fase 1 — validación ejecutable por historia de usuario
├── contracts/                       # Fase 1 — contrato de los endpoints modificados
│   └── categorias-orden.md
└── tasks.md                         # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
alembic/versions/
└── <rev>_categories_display_order.py    # NUEVO — down_revision='a96852d7be6a' (head
                                           confirmado, research.md); op.add_column + backfill
                                           ROW_NUMBER() OVER (ORDER BY name DESC) + CheckConstraint
                                           display_order >= 0, por @for_each_tenant_schema
                                           (data-model.md)

app/models/
└── category.py                          # MODIFICADO — nueva columna display_order:
                                           Mapped[int] = mapped_column(Integer, nullable=False),
                                           junto a `active`; nuevo CheckConstraint en
                                           __table_args__ (hoy es una tupla sin constraints,
                                           solo {"schema": "tenant"})

app/api/v1/categories/
├── schemas.py                           # MODIFICADO — CategoryCreate/CategoryUpdate ganan
                                           display_order: int | None = Field(None, ge=0);
                                           CategoryResponse gana display_order: int
└── router.py                            # MODIFICADO — create_category calcula
                                           MAX(display_order)+1 cuando body.display_order es
                                           None (nueva función next_display_order() en
                                           service.py o inline, a decidir en research.md);
                                           update_category asigna body.display_order si viene.
                                           list_categories (línea 40) NO cambia su order_by.

app/api/v1/menu/
└── router.py                            # MODIFICADO — _build_menu (línea 97) cambia
                                           .order_by(Category.name) por
                                           .order_by(Category.display_order.desc(), Category.name)

app/characterization_tests/
└── test_category_display_order.py       # NUEVO — cubre FR-001 a FR-009 (quickstart.md)

# pos-heladeria
src/app/modules/categories/interfaces/
└── category.interface.ts                # MODIFICADO — Category, CategoryForm,
                                           CategoryCreatePayload, CategoryUpdatePayload ganan
                                           display_order

src/app/modules/categories/services/
└── category.service.ts                  # MODIFICADO — createCategory/updateCategory incluyen
                                           display_order en el payload

src/app/modules/categories/components/
├── category-form.component.ts           # MODIFICADO — nuevo campo numérico "Orden" en el
                                           formulario (creación y edición), con validación
                                           entero >= 0; placeholder indicando el valor por
                                           defecto si se deja vacío
└── category-form.component.spec.ts      # MODIFICADO (o NUEVO si aún no existe al
                                           implementar) — casos: guardar con orden explícito,
                                           guardar sin orden (backend asigna por defecto),
                                           rechazo de valores negativos/no numéricos

src/app/modules/categories/pages/
├── categories-page.component.ts         # MODIFICADO — nueva columna "Orden" en la tabla
                                           (User Story 3); no cambia el order_by de la consulta
                                           paginada
└── categories-page.component.spec.ts    # MODIFICADO (o NUEVO si aún no existe) — verifica
                                           que la columna de orden se muestra
```

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
