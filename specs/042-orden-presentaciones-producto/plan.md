# Implementation Plan: Orden de Presentaciones de un Producto

**Branch**: `042-orden-presentaciones-producto` | **Date**: 2026-08-27 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/042-orden-presentaciones-producto/spec.md`

## Summary

Hoy `product_variants` no tiene ningún campo de orden — confirmado que ninguna consulta que lista
presentaciones de un producto (ni la que alimenta el formulario, ni `menu/router.py:110-114` del
Menú QR) tiene `.order_by()` propio; el orden visible es el que devuelve la base de datos
(efectivamente inserción/PK). Este plan agrega una columna `display_order` (Integer, con
`UNIQUE (product_id, display_order) DEFERRABLE INITIALLY DEFERRED`, research.md Decisión 3) y un
`order_by` en la relación `Product.variants` (`app/models/product.py:52`, research.md Decisión 7)
que hace que **todo** punto que recorra `product.variants` —incluido el Menú QR— reciba la lista ya
ordenada, sin tocar cada endpoint por separado. El reordenamiento en el formulario se implementa con
`@angular/cdk/drag-drop` (`CdkDropList`/`CdkDrag`/`moveItemInArray`), que ya está declarado en
`package.json` (`^21.2.14`) pero sin ningún consumidor hoy en `src/` — esta es su primera vez en uso
real, sin agregar dependencia nueva (Principio IX no aplica). El arrastre solo actualiza
`draft().variants` en memoria (feedback inmediato de la numeración, FR-003); la persistencia ocurre
mediante un endpoint nuevo y dedicado, `PATCH /products/{id}/variants/reorder`, invocado como un
paso `await` más dentro de `saveExistingProduct` (`product.service.ts:478-513`) — se descartó
repartir `display_order` entre los `PATCH`/`POST` que ya hace esa función para cada variante porque
esa secuencia es HTTP-por-HTTP sin transacción compartida (confirmado leyendo el código), y la
unicidad `(product_id, display_order)` necesita que el reordenamiento completo sea atómico
(research.md Decisión 2). Desactivar o reactivar una presentación no toca `display_order` en
absoluto (research.md Decisión 4): la lista de presentaciones desactivadas es una franja aparte en
el formulario (`draft().deactivated`, línea 202, distinta de `draft().variants`, línea 181), así que
FR-007/FR-008 quedan satisfechos sin ningún código nuevo de renumeración. La migración hace backfill
por producto con `ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY id)`, reproduciendo el orden
de creación que el sistema ya muestra hoy, para que desplegar esta funcionalidad no reordene nada
por sí sola (FR-009, SC-004).

## Technical Context

**Language/Version**: Backend — Python 3.14 (`pos-backend`, mismo entorno de specs previas).
Frontend — TypeScript 5.9.2 (Angular 21.1.x, standalone components + signals, sin NgModules).

**Primary Dependencies**:
- Backend: FastAPI, SQLAlchemy 2.0 (sync), Pydantic 2, Alembic. Ninguna dependencia nueva.
- Frontend: Angular 21 (standalone + signals), `@angular/cdk` `^21.2.14` (ya declarado en
  `package.json`, confirmado sin ningún import en `src/` hoy — primer consumidor real vía
  `@angular/cdk/drag-drop`, research.md Decisión 6), Tailwind CSS v4. Ninguna dependencia nueva
  (Principio IX no aplica).

**Storage**: PostgreSQL 16, schema-per-tenant. **Con migración** — nueva columna
`product_variants.display_order` (`Integer NOT NULL` tras backfill) y constraint
`UNIQUE (product_id, display_order) DEFERRABLE INITIALLY DEFERRED`
(`uq__product_variants__product_id__display_order`), agregada vía `@for_each_tenant_schema`
siguiendo el patrón ya usado por migraciones previas (spec 027, research.md Decisión 4 de esa spec).
`down_revision='187e491e597a'` (`187e491e597a_order_item_discount_snapshot.py`), confirmado como la
revisión `head` actual de `pos-backend/alembic/versions/` al planear esta spec (2026-08-27,
research.md) — reverificar si otra spec agregó una migración más reciente antes de implementar.

**Testing**: Backend — `unittest` vía `python -m unittest` (sin pytest en el repo). Se crea
`app/characterization_tests/test_product_variant_reorder.py` (NUEVO — funcionalidad nueva, no
modifica ningún comportamiento congelado existente). Frontend — Vitest vía
`@angular/build:unit-test`; se extiende `product-form.component.spec.ts` (creado por la spec 027;
si para cuando se implemente esta spec aún no existe, se crea aquí) con los casos de arrastre.

**Target Platform**: Linux server (API `pos-backend`) + navegador (SPA `pos-heladeria`) — el
formulario lo usan administradores de catálogo; el orden resultante lo ve cualquier comensal que
abra el Menú QR (solo lectura, sin ningún control de orden de su lado).

**Project Type**: Web application (backend FastAPI + frontend Angular, dos repositorios
independientes, siblings de este repo `pos-specs`).

**Performance Goals**: sin objetivo de throughput nuevo. El endpoint de reordenamiento hace un
`UPDATE` por presentación del producto dentro de una sola transacción — el volumen esperado por
producto (SC-001: hasta 10) hace este costo despreciable frente al resto de la operación de guardar
un producto, que ya hace varias llamadas secuenciales por presentación (research.md Decisión 2).

**Constraints**:
- El reordenamiento completo de un producto DEBE ejecutarse en una sola transacción de base de
  datos — la constraint `UNIQUE (product_id, display_order)` es `DEFERRABLE INITIALLY DEFERRED`
  precisamente para permitirlo sin un estado intermedio inválido (research.md Decisión 3).
- La persistencia del nuevo orden NO DEBE dispararse en peticiones HTTP separadas por presentación —
  DEBE ser una sola llamada atómica al endpoint de reordenamiento (research.md Decisión 2).
- `display_order` NO DEBE recalcularse en ningún flujo de `active`/soft-delete (crear, desactivar,
  reactivar salvo la asignación inicial al crear) — research.md Decisión 4.

**Scale/Scope**: 1 tabla modificada (`product_variants`, +1 columna +1 constraint), 1 migración
nueva, 1 endpoint nuevo (`PATCH /products/{id}/variants/reorder`), 1 relación ORM modificada
(`Product.variants`, +`order_by`), 0 cambios de *shape* en ningún schema Pydantic ni interfaz
TypeScript existente; en `pos-heladeria`, 1 componente modificado (`product-form.component.ts`,
envolver la franja de presentaciones activas en `CdkDropList`/`CdkDrag`) + su servicio
(`product.service.ts`, +1 método), sin componentes nuevos.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 3 historias priorizadas y 10 FRs, sin `[NEEDS CLARIFICATION]` pendientes (sesión `/speckit-specify`, 2026-08-27), antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El único comportamiento que cambia es el orden de despliegue de presentaciones (hoy implícito/no especificado) por uno explícito y editable — documentado en `spec.md` ("Naturaleza de esta spec") como funcionalidad nueva, no como corrección de algo protegido. Ningún producto existente cambia su orden visible hasta que un administrador reordene explícitamente (FR-009, SC-004, migración con backfill idéntico al orden actual). | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | Ningún characterization test existente (spec 002, 003, 004, 027) referencia orden de presentaciones — no hay ninguno que modificar. `test_variantes_duplicadas.py` se ejecuta sin cambios como verificación de que reactivar sigue intacto (quickstart.md, Historia 3). | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El orden editable y su reflejo en el Menú QR es comportamiento enteramente nuevo (no existía ningún concepto de orden antes); no se exige equivalencia con el pasado más allá del backfill inicial (FR-009). | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Se reutiliza `@angular/cdk` ya declarado (research.md Decisión 6) y el patrón de reemplazo total ya usado por `setRecipe`/`setVariantOptionGroups` (research.md Decisión 2) en vez de introducir una librería o un patrón nuevo. No se toca ningún otro comportamiento de `product-form.component.ts` fuera de la franja de presentaciones activas. | PASS |
| **VI. Evolución Incremental** | El alcance se divide en las mismas unidades que las historias del spec (US1 arrastrar y persistir → US2 reflejo en Menú QR → US3 convivencia con crear/editar/eliminar), cada una verificable por separado (quickstart.md). No se mezcla con ninguna refactorización de `saveExistingProduct` más allá de agregar un paso, ni con el registro de riesgos R21 (`@angular/cdk` sin uso) — usarlo aquí no pretende "corregir" ese registro, solo resolver esta spec. | PASS |
| **VII. Compatibilidad con Datos Históricos** | No se toca ninguna venta, factura, pago ni movimiento de inventario ya generado — la migración solo agrega una columna y una constraint a `product_variants`. | PASS (no aplica directamente) |
| **VIII. Evolución del Modelo de Datos** | Migración especificada completa en data-model.md/research.md Decisión 5: columna nueva, tipo, nulabilidad, estrategia de backfill (`ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY id)`) y estrategia de rollback (`ALTER TABLE ... DROP CONSTRAINT` + `DROP COLUMN`, reversible sin destruir otro dato — perder `display_order` no afecta ninguna otra columna). | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se agrega ninguna dependencia nueva en ningún repositorio — `@angular/cdk` ya está declarado en `package.json` desde antes de esta spec (research.md Decisión 6). | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de usuario tiene su "Independent Test" en `spec.md`; quickstart.md los traduce a comandos `unittest`/Vitest ejecutables, más la suite completa de characterization tests como red de no-regresión. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | La existencia del arrastre, que el orden se refleje en el Menú QR, y que crear/editar/eliminar sigan funcionando igual son las decisiones de negocio (spec.md). Cómo modelar la columna, dónde imponer la unicidad, cuándo persistir (al guardar, no al soltar) y dónde ordenar (la relación ORM, no cada consulta) son las decisiones técnicas correspondientes, documentadas en research.md sin mezclarse con las de negocio. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec) → este `plan.md`/`research.md`/`data-model.md`/`contracts/` (Decisión técnica, incluidos los hechos verificados contra el código real por una sesión hermana) → `tasks.md` (Fase 2, no generada por este comando) → implementación → tests nuevos + suite de characterization tests sin regresión → `quickstart.md` (Verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/042-orden-presentaciones-producto/
├── plan.md                                    # Este fichero (/speckit-plan)
├── research.md                                # Fase 0 — decisiones técnicas y hechos verificados
├── data-model.md                              # Fase 1 — columna nueva, migración, asignación de orden
├── quickstart.md                               # Fase 1 — validación ejecutable por historia de usuario
├── contracts/                                  # Fase 1 — contrato del endpoint nuevo
│   └── product-variants-reorder.md
└── tasks.md                                    # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` y
`../pos-heladeria` (Constitución §Alcance). Rutas relativas a la raíz de cada repo.

```text
# pos-backend
alembic/versions/
└── <rev>_product_variants_display_order.py   # NUEVO — down_revision='187e491e597a' (head
                                                 confirmado, research.md); op.add_column +
                                                 backfill ROW_NUMBER() + constraint UNIQUE
                                                 DEFERRABLE, por @for_each_tenant_schema
                                                 (data-model.md, research.md Decisión 5)

app/models/
├── product_variant.py                        # MODIFICADO — nueva columna display_order:
                                                 Mapped[int] = mapped_column(Integer,
                                                 nullable=False), junto a `active` (línea ~33)
└── product.py                                 # MODIFICADO — relationship `variants` (línea 52)
                                                 gana order_by="ProductVariant.display_order"
                                                 (research.md Decisión 7)

app/api/v1/catalog/
├── schemas.py                                 # MODIFICADO — sin cambio de forma en los schemas
                                                 existentes; se agrega VariantReorderRequest/
                                                 VariantReorderResponse (contracts/
                                                 product-variants-reorder.md)
├── router.py                                  # MODIFICADO — nuevo endpoint
                                                 `PATCH /products/{id}/variants/reorder`
└── service.py                                 # MODIFICADO — nueva función reorder_variants(db,
                                                 product_id, variant_ids) -> list[ProductVariant]
                                                 que valida el conjunto de IDs (activos, sin
                                                 duplicados/faltantes) y asigna display_order=1..N
                                                 en una transacción; create_variant asigna
                                                 display_order = MAX(...)+1 (data-model.md, tabla
                                                 de asignación)

app/characterization_tests/
└── test_product_variant_reorder.py           # NUEVO — cubre FR-001 a FR-003, FR-005, FR-007,
                                                 FR-008, FR-010 (quickstart.md)

# pos-heladeria
src/app/modules/products/interfaces/
└── product.interface.ts                       # MODIFICADO — sin cambio de forma pública; tipo
                                                 interno para la respuesta de reorder si el
                                                 servicio lo necesita tipado

src/app/modules/products/services/
└── product.service.ts                         # MODIFICADO — nuevo método
                                                 reorderVariants(productId, orderedIds), invocado
                                                 dentro de saveExistingProduct (líneas ~478-513,
                                                 después del loop existente de patch/post/delete
                                                 por variante, research.md Decisión 2)

src/app/modules/products/pages/
├── product-form.component.ts                  # MODIFICADO — import de DragDropModule
                                                 (`@angular/cdk/drag-drop`); el `@for` de la línea
                                                 181 (draft().variants) se envuelve en
                                                 CdkDropList/CdkDrag; handler
                                                 onVariantDrop(event) llama moveItemInArray sobre
                                                 draft().variants vía this.draft.update(...)
                                                 (research.md Decisión 3b/6). El `@for` de la
                                                 línea 202 (draft().deactivated) no cambia.
└── product-form.component.spec.ts             # MODIFICADO (o NUEVO si aún no existe al
                                                 implementar) — casos nuevos: arrastrar reordena
                                                 localmente y renumera de inmediato; guardar
                                                 dispara reorderVariants(); crear/eliminar/editar
                                                 no disparan reorder innecesario
```

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
