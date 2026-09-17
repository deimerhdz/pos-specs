# Implementation Plan: Catálogo de Presentaciones y Rediseño de Promociones

**Branch**: `083-presentaciones-y-promociones` (carpeta de spec) — implementación en
`feat/090-...` en `pos-backend`/`pos-heladeria` (convención ya fijada por el equipo, spec.md
§Clarifications, independiente del número de esta carpeta) | **Date**: 2026-09-16 | **Spec**:
[spec.md](./spec.md)

**Input**: Feature specification from `/specs/083-presentaciones-y-promociones/spec.md`

## Summary

Dos entregables independientes pero secuenciales: (1) un catálogo nuevo `Presentation` por
tenant, con asociación N:N a `Category`, que hace que crear un producto dentro de una categoría
con presentaciones asociadas le genere automáticamente una `ProductVariant` por cada una (o
"Presentación única" si la categoría no tiene ninguna) — cero cambios al modelo de `Promotion`/
`PromotionRule`/`PromotionVariant` de spec 063, que sigue intacto; y (2) el rediseño visual y de
flujo de las tres pantallas de administración de promociones (listado, creación, configuración)
siguiendo al 100% los prototipos entregados, usando el catálogo de presentaciones únicamente
como ayuda de etiquetado en el selector de variantes de cada regla — nunca como mecanismo de
alcance (spec.md §Contexto, A-65 sin reabrir).

Verificado contra el código real de ambos repos (`develop`, 2026-09-16): `Presentation` de spec
040 fue eliminada por completo en `063b` (`ba4b6bd573a6`) y no queda nada que reactivar; el
modelo `Promotion`/`PromotionRule`/`PromotionVariant` de spec 063 ya está en `develop` y no
necesita ningún cambio de esquema, servicio ni endpoint para esta spec (research.md D9). El único
cambio de comportamiento observable sobre código existente es renombrar el literal por defecto de
variante de `"Single"` a `"Presentación única"` en `ensure_default_variant`
(`app/api/v1/catalog/service.py`) — ya autorizado por la Clarification de esta spec, pendiente de
registrarse como anomalía **A-74** antes de implementar (research.md D7).

## Technical Context

**Language/Version**: Backend Python 3.12 (imagen Docker) / 3.14 (venv local `pos-backend/env`).
Frontend Angular **21.1** (standalone-only, sin `NgModule` en el repo). Esta spec toca **ambos
repositorios** (Constitución §Alcance), sobre `develop` como base (`main` de ambos repos está
desactualizado frente a `develop`; el resto de specs recientes del proyecto —066, 073, 079, 080—
ya conviven en `develop`).

**Primary Dependencies**: FastAPI + SQLAlchemy 2.0 (sync, `Mapped`/`mapped_column`), Alembic
(`@for_each_tenant_schema`, head actual verificado **`ef0abdf40889`**), Pydantic v2 (backend).
Angular con signals + TanStack Query (`@tanstack/angular-query-experimental` vía el wrapper
`injectPagedQuery`), formularios `FormsModule`/`ngModel` (template-driven, sin
`ReactiveFormsModule`) para el módulo de Promociones existente (frontend). **Ninguna dependencia
nueva** (Principio IX no aplica): una tabla de catálogo más, una tabla puente más, un módulo
Angular más siguiendo patrones ya usados por Categorías/Productos/Promociones.

**Storage**: PostgreSQL 16, schema por tenant. Una migración nueva, 100% aditiva (sin
contraparte destructiva — a diferencia de spec 063, esta spec no retira estructura existente):
tabla `presentations`, tabla puente `category_presentations`, más el paso de datos de siembra
(FR-019). Ver [data-model.md](./data-model.md).

**Testing**: `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` (sin
pytest/conftest.py, backend). `ng test` (`@angular/build:unit-test`, frontend) — sin
Karma/Jasmine/Vitest configurado aparte, es el builder nativo de Angular 21.

**Target Platform**: Linux server (`pos-backend`) + SPA Angular (`pos-heladeria`), ambos en
producción vía `develop`.

**Project Type**: Web application (API FastAPI + frontend Angular), dos repos independientes.

**Performance Goals**: Sin objetivo nuevo. El catálogo de Presentaciones es del mismo orden de
magnitud que Categorías/`OptionGroup` (decenas de filas por tenant, no miles); la resolución de
presentaciones al crear un producto es una consulta con `JOIN` sobre `category_presentations` +
`presentations`, ejecutada una sola vez por creación de producto (no en un hot path de venta).

**Constraints**:
- **FR-009 (fidelidad + motor intacto)**: el rediseño de promociones no puede tocar
  `evaluate_variant_sets` ni ningún call site de checkout/cart/sales/table_sessions/menu —
  confirmado en research.md D9 que ningún archivo de `app/api/v1/promotions/` necesita cambiar.
- **FR-002/D4 (sin borrado físico)**: `Presentation` nunca expone `DELETE`; el único ciclo de vida
  es `POST`/`PATCH` con toggle de `active` en ambos sentidos.
- **FR-008 (sin retroactividad)**: cambiar `category_presentations` no dispara ninguna
  actualización sobre `ProductVariant` ya existentes — la resolución de presentaciones ocurre
  solo dentro de `ProductService.create_product`, nunca en `update_product` ni en un job aparte.
- **Principio VII (datos históricos)**: ninguna `Sale`/`Invoice`/`CustomerOrder` se toca — esta
  spec es 100% catálogo/configuración.
- **D7 (cambio de comportamiento a registrar)**: el renombre "Single"→"Presentación única" exige
  una entrada nueva en `registro-de-anomalias.md` (**A-74**, siguiente número tras A-73) citando
  esta Clarification como decisión de negocio, **antes** de mergear el código que lo implementa
  (Principio II, Flujo de Trabajo de Evolución Funcional).

**Scale/Scope**: 2 tablas nuevas (`presentations`, `category_presentations`) en 1 migración
aditiva con paso de datos por tenant; 1 router nuevo (`app/api/v1/presentations/`, hoy vacío) con
4 endpoints (`GET` lista, `GET {id}`, `POST`, `PATCH`); extensión de `CategoryCreate`/
`CategoryUpdate`/`CategoryResponse` con `presentation_ids`/`presentations`; ~15-20 líneas nuevas
en `ProductService.create_product` + 1 línea cambiada en `ensure_default_variant`; 1 módulo
Angular nuevo (`presentaciones`, calcado de `categories/`); extensión del formulario de
`category-form.component.ts` con un multi-select; rediseño visual de
`promotions-page.component.ts` (~1200 líneas) sin cambio de sus servicios de dominio salvo el
mapeo de etiqueta de FR-013. Cero cambios en `Promotion`/`PromotionRule`/`PromotionVariant`
(modelo, servicio, schemas, router) y cero cambios en checkout/cart/sales/table_sessions/menu.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` completo: problema, 3 historias con Acceptance Scenarios, Edge Cases, 19 FR, Key Entities, 6 Success Criteria, Assumptions, y una sesión de Clarifications de 7 preguntas que resuelve la relación con specs 040/063/081 y el numeramiento de carpetas. Este plan es posterior y no reabre nada de eso. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | Un solo cambio de comportamiento observable sobre código en producción: el literal por defecto de variante ("Single"→"Presentación única", D7). Está documentado en `spec.md` §Clarifications (pregunta 2) como decisión de negocio explícita ("es un renombre en la interfaz... no cambia el comportamiento de creación ni el precio inicial"), y este plan exige registrarlo como anomalía **A-74** en `registro-de-anomalias.md` **antes** de implementar (research.md D7) — pendiente de ejecutar, no de decidir. Todo lo demás (creación de catálogo nuevo, asociación nueva, herencia en producto **nuevo**) es funcionalidad aditiva sin comportamiento previo que proteger. | PASS (con acción pendiente registrada) |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | El único test de caracterización que toca el comportamiento que cambia es el de RN-CAT-05 (`app/characterization_tests/test_products_service.py:173`, cita textual "RN-CAT-05: sin `variants`... el comportamiento no cambia"). Ese test se actualiza explícitamente para esperar `"Presentación única"` en vez de `"Single"`, citando A-74 en el mismo commit — no se toca en silencio. Ningún otro test `"CONGELA comportamiento actual:"` de promociones/categorías/productos necesita cambiar (research.md D9 confirma cero cambios al motor de promociones). | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El catálogo de Presentaciones, la asociación por categoría y la herencia automática son comportamiento enteramente nuevo definido por FR-001 a FR-008. El rediseño de promociones (FR-009 a FR-018) es comportamiento de UI nuevo sobre un motor de cálculo que se mantiene sin cambios — el criterio de éxito es conformidad con estos FR, no equivalencia con las pantallas viejas. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Se descartó explícitamente introducir `ReactiveFormsModule` en `promotions-page.component.ts` (research.md D10) y un sub-router dedicado para la asociación categoría↔presentación (D5) — ambos habrían sido refactorizaciones/patrones nuevos no exigidos por ningún FR. Cada archivo tocado deriva directamente de un FR: `categories/` (FR-004), `products/service.py` (FR-005/FR-006), `promotions/pages/` (FR-009 a FR-018, solo UI). No se toca `promotions/service.py`/`schemas.py`/`router.py`. | PASS |
| **VI. Evolución Incremental** | Los dos entregables del Summary son separables y se implementan como incrementos distintos: (A) modelo + migración + endpoints de `Presentation` (US1, sin dependencias de las otras dos historias); (B) asociación categoría↔presentación + herencia en `create_product` (US2, depende de A); (C) rediseño de las 3 pantallas de promociones (US3, depende de A para el mapeo de etiqueta de FR-013, pero es independiente de B — puede probarse con productos que ya tengan variantes creadas a mano, spec.md "Why this priority" de US3). Ningún incremento mezcla migración de datos con cambio de UI ni con refactorización no relacionada. | PASS |
| **VII. Compatibilidad con Datos Históricos** | Ninguna `Sale`/`Invoice`/`CustomerOrder` se toca. El paso de datos de la migración (FR-019) solo lee `product_variants.name` (sin modificarlo) e inserta filas nuevas en una tabla nueva (`presentations`) — no altera ninguna fila existente de ninguna tabla. | PASS |
| **VIII. Evolución del Modelo de Datos** | [data-model.md](./data-model.md) especifica ambas tablas nuevas (columnas, constraints, FKs, `ondelete`), el paso de datos exacto (`INSERT ... SELECT DISTINCT`, comparación exacta de texto) y el `downgrade` (DROP de lo aditivo, sin intento de revertir el paso de datos, mismo criterio que `063a`/`94144eaa60b5`). Compatibilidad con datos existentes: explícita (FR-019 no modifica `product_variants` ni crea asociaciones). | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No aplica — ninguna dependencia nueva en ningún repo (research.md, todas las decisiones D1-D10 reutilizan patrones/librerías ya presentes). | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de `spec.md` tiene "Independent Test"; [quickstart.md](./quickstart.md) los traduce en pasos ejecutables (curl + verificación de UI) por historia, más el paso 0 (suite existente en verde) y un checklist final de Principio X explícito. Se ejercitan los 3 Acceptance Scenarios más sensibles: unicidad al editar (FR-003), no-retroactividad (FR-008), y bloqueo de ahorro insuficiente en promociones (FR-016, sin cambio de criterio). | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Decisiones de negocio en `spec.md` §Clarifications (7 preguntas: catálogo como etiqueta no como alcance, "Presentación única" = renombre de "Single", numeración de carpeta vs. rama, comportamiento del selector con variantes sin presentación equivalente, siembra automática, reactivación, unicidad al editar). Decisiones técnicas en [research.md](./research.md) (D1-D10: modelo de datos, patrón de endpoints, dónde resolver la etiqueta de FR-013, formulario template-driven vs. reactivo), cada una con alternativa descartada y su porqué. | PASS |
| **XII. Trazabilidad** | Cadena: `spec.md` (Clarifications 2026-09-16 + 19 FR) → este `plan.md`/`research.md`/`data-model.md`/`contracts/` (decisión técnica, este documento) → `registro-de-anomalias.md` (entrada A-74 pendiente, citando D7) → `tasks.md` (Fase 2, `/speckit-tasks`) → implementación en `feat/090-...` → tests (characterization actualizado + nuevos) → [quickstart.md](./quickstart.md). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research, data-model, contracts, quickstart), los nombres de tests nuevos, la entrada A-74 y los mensajes de commit de esta funcionalidad se escriben en español de Colombia, igual que el resto del repositorio. | PASS |

Sin violaciones que justificar en Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/083-presentaciones-y-promociones/
├── plan.md                    # Este fichero (/speckit-plan)
├── research.md                # Fase 0 — D1-D10, cada decisión técnica con alternativa descartada
├── data-model.md              # Fase 1 — presentations, category_presentations, migración y rollback
├── contracts/
│   ├── catalogo-presentaciones.md          # CRUD /presentations (FR-001–003, FR-019)
│   ├── categoria-herencia-producto.md      # presentation_ids en /categories + herencia en /products (FR-004–008)
│   └── promociones-rediseno-ui.md          # Contrato de UI de las 3 pantallas, cero cambio de API (FR-009–018)
├── quickstart.md               # Fase 1 — validación ejecutable por historia de usuario
└── tasks.md                    # Fase 2 (/speckit-tasks — no generado por este comando)
```

### Source Code (repositorios sibling de `pos-specs`, sobre rama `feat/090-...` nueva desde `develop`)

```text
# ../pos-backend
app/
├── models/
│   ├── presentation.py                # NUEVO — class Presentation (espejo de option_group.py)
│   └── category_presentation.py       # NUEVO — class CategoryPresentation (tabla puente, espejo
│                                         de variant_option_group.py sin columnas de negocio)
│
├── api/v1/presentations/              # NUEVO (directorio ya existe vacío, solo __pycache__
│   │                                     huérfano de spec 040/063 — limpiar antes de escribir)
│   ├── __init__.py
│   ├── router.py                      # GET "", GET "/{id}", POST "", PATCH "/{id}" — sin DELETE
│   └── schemas.py                     # PresentationCreate, PresentationUpdate, PresentationResponse
│
├── api/v1/categories/
│   ├── schemas.py                     # MODIFICADO — CategoryCreate/CategoryUpdate ganan
│   │                                     `presentation_ids`; CategoryResponse gana `presentations`
│   │                                     (PresentationSummary embebido)
│   ├── router.py                      # MODIFICADO — create/update_category resuelven el
│   │                                     reemplazo total de category_presentations
│   └── service.py                     # NUEVO (opcional, decisión de detalle en tasks) — o inline
│                                         en router.py: _replace_category_presentations()
│
├── api/v1/products/
│   └── service.py                     # MODIFICADO — create_product: rama `else` de
│                                         `if data.variants` resuelve presentaciones activas de
│                                         la categoría antes de decidir entre variantes heredadas
│                                         (FR-005) y ensure_default_variant (FR-006)
│
├── api/v1/catalog/
│   └── service.py                     # MODIFICADO — ensure_default_variant: "Single" →
│                                         "Presentación única" (D7, cita A-74)
│
├── models/product_variant.py          # MODIFICADO — server_default de `name`: "Single" →
│                                         "Presentación única" (documental, D7)
│
├── main.py                            # MODIFICADO — registrar el router nuevo de presentations
│
├── alembic/versions/
│   └── XXXX_083_presentaciones_aditivo.py   # NUEVO — down_revision "ef0abdf40889".
│                                               @for_each_tenant_schema: crea presentations +
│                                               category_presentations; paso de datos de siembra
│                                               (FR-019, INSERT...SELECT DISTINCT). downgrade:
│                                               DROP de ambas tablas.
│
└── characterization_tests/
    ├── test_presentations_service.py          # NUEVO — CRUD, unicidad al crear y editar
    │                                             (FR-003), toggle en ambos sentidos (US1 Esc. 5)
    ├── test_presentations_migration.py        # NUEVO — paso de datos 1:N por nombre distinto,
    │                                             sin crear asociaciones (FR-019)
    ├── test_categories_presentations.py       # NUEVO — reemplazo total de presentation_ids,
    │                                             no-retroactividad (FR-008)
    ├── test_products_service.py               # MODIFICADO — RN-CAT-05 actualizado a
    │                                             "Presentación única" (cita A-74); casos nuevos
    │                                             de herencia por categoría (FR-005)
    └── test_promotions_*.py                   # SIN CAMBIO (research.md D9 — confirmar en
                                                  verificación final que siguen en verde tal cual)

# ../pos-heladeria
src/app/
├── modules/presentations/              # NUEVO módulo, calcado de modules/categories/
│   ├── pages/presentations-page.component.ts     # Listado + crear/editar + toggle activo
│   ├── services/presentation.service.ts          # injectPagedQuery + create/update, mismo
│   │                                                patrón que category.service.ts
│   └── interfaces/presentation.interface.ts      # Presentation, PresentationForm
│
├── modules/categories/
│   ├── components/category-form.component.ts     # MODIFICADO — multi-select de presentaciones
│   │                                                 activas (mismo patrón que el checkbox-list
│   │                                                 de variantes en promotions-page.component.ts)
│   ├── interfaces/category.interface.ts           # MODIFICADO — Category gana `presentations`;
│   │                                                 CategoryUpdatePayload gana `presentation_ids`
│   └── services/category.service.ts               # MODIFICADO — payload con presentation_ids
│
├── modules/products/
│   └── services/product.service.ts                # SIN CAMBIO DE CONTRATO — sigue sin enviar
│                                                      `variants` cuando el producto nace con
│                                                      herencia automática (el backend decide);
│                                                      revisar `emptyDraft()`/`toggleHasSizes()`
│                                                      solo si la UI de creación de producto debe
│                                                      reflejar las presentaciones heredadas antes
│                                                      de guardar (decisión de detalle en tasks,
│                                                      no exigida por ningún FR — FR-005 solo pide
│                                                      que el producto YA CREADO tenga las
│                                                      variantes, no que el formulario las prevea)
│
├── modules/promotions/
│   └── pages/promotions-page.component.ts         # REDISEÑO — fidelidad visual a los 3
│                                                      prototipos (listado/creación/configuración);
│                                                      gana el mapeo de etiqueta de FR-013 usando
│                                                      `category.presentations`; sin cambio de
│                                                      `promotion.service.ts` ni de las interfaces
│                                                      de Promotion/PromotionRule
│
└── core/config/navigation.config.ts               # MODIFICADO — nuevo ítem "Presentaciones" en
                                                       el grupo CATÁLOGO, sin moduleKey (D3)
```

**Structure Decision**: el catálogo de Presentaciones se implementa como par
entidad-catálogo/tabla-puente calcado de `OptionGroup`/`VariantOptionGroup` (research.md D2), sin
reactivar ni heredar nada del `Presentation` de spec 040 (eliminado por completo en `063b`, D1).
La asociación Categoría↔Presentación se embebe en el payload existente de `Category`
(`presentation_ids`/`presentations`) en vez de crear un sub-recurso nuevo (D5), y la herencia al
crear producto reutiliza el guardado consolidado de variantes ya existente (`_save_variant_entry`/
`_assign_display_orders`, spec 043) en vez de duplicar esa lógica (D6). El rediseño de Promociones
es exclusivamente frontend: cero archivos de `app/api/v1/promotions/` cambian (D9), y el frontend
mantiene su patrón `FormsModule`/`ngModel` ya usado en el componente de 1200 líneas en vez de
introducir `ReactiveFormsModule` (D10, mismo criterio que spec 063 ya aplicó al mismo componente).
El único cambio de comportamiento sobre código en producción — el literal "Single"→"Presentación
única" — queda acotado a una función (`ensure_default_variant`) y su test de caracterización, con
su anomalía (A-74) como prerrequisito de implementación, no de diseño.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
