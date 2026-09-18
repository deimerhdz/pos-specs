# Implementation Plan: Corrección de Bugs en Promociones y Productos

**Branch**: `084-fix-promociones-productos` (carpeta de spec) — implementación en
`pos-backend`/`pos-heladeria` sobre una rama nueva desde `develop` (convención ya usada por spec
083, `feat/090-...`; el número de rama de implementación es independiente del número de esta
carpeta) | **Date**: 2026-09-17 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/084-fix-promociones-productos/spec.md`

## Summary

Seis correcciones independientes sobre funcionalidad ya mergeada a `develop` en ambos repos
(spec 083, PR #75 backend / PR #83 frontend, y specs 063/066/081 antes de esa): (1) asociar cada
`ProductVariant` con una `Presentation` del catálogo mediante una columna nueva y sincronizar su
nombre — único cambio de modelo de datos de esta spec; (2) deshabilitar el botón "Configurar" del
listado de promociones para toda promoción con `status=active`; (3) mostrar en la tarjeta de
producto de la pestaña "Promociones" del menú QR el precio (o precio mínimo) de la regla vigente,
reusando un dato que el backend ya calcula por variante y que hoy el frontend descarta; (4)
reemplazar la regla combinada que `addRuleRow()` genera hoy (una sola regla compartida entre
varios productos) por N reglas independientes — una por producto ya seleccionado que comparte la
presentación, con exclusión por casilla; (5) una guarda de servicio nueva que impide seleccionar
un producto en una promoción si ya tiene alguna variante cubierta por otra promoción `active`; y
(6) reemplazar el `<div>` posicionado en absoluto del menú de acciones ⋮ de `promotions-page` por
Angular CDK Overlay, para que no se recorte contra el `overflow-x: auto` de la tabla.

Verificado contra el código real de ambos repos (`develop`, 2026-09-17, investigación de
research.md D0): ningún archivo de `app/api/v1/promotions/router.py`/`schemas.py` cambia de forma;
solo `service.py` gana dos guardas nuevas (`update()` para bug 4, `_guard_product_overlap` para la
exclusividad de producto del bug 3) — la primera rompe dos tests de characterization protegidos que
se actualizan explícitamente (research.md D9). El motor `menu_variant_promotion` (backend) **ya**
calcula el precio para cualquier `min_qty`, incluido `>= 2` — el bug 1 es 100% frontend, sin tocar
`app/api/v1/menu/`. Tres cambios de esta spec son decisiones de negocio sobre comportamiento ya en
producción y exigen anomalía nueva antes de implementar (Principio II): **A-76** (bloqueo total de
"Configurar" en `active`, reemplaza la edición parcial que spec 083 FR-018 dejó vigente),
**A-77** (romper la regla combinada de `addRuleRow()` en N reglas independientes) y **A-78**
(exclusividad de producto entre promociones `active`, restricción nueva que puede rechazar
selecciones hoy válidas) — ver research.md D7.

## Technical Context

**Language/Version**: Backend Python 3.12 (imagen Docker) / 3.14 (venv local `pos-backend/env`).
Frontend Angular **21** standalone (signals, sin `NgModule`). Esta spec toca **ambos
repositorios** (Constitución §Alcance), sobre `develop` como base — ambos repos están al día con
`origin/develop` y spec 083 ya mergeada (backend PR #75, frontend PR #83).

**Primary Dependencies**: FastAPI + SQLAlchemy 2.0 (`Mapped`/`mapped_column`), Alembic
(`@for_each_tenant_schema`, head actual verificado **`da7581f7bb18`**), Pydantic v2 (backend).
Angular con signals + TanStack Query (`injectPagedQuery`), formularios `FormsModule`/`ngModel`
(frontend). **Una dependencia ya presente que empieza a usarse por primera vez**: `@angular/cdk`
(`^21.2.14`, ya en `package.json` de `pos-heladeria`, pero sin ningún uso de `OverlayModule`/
`CdkMenu` hoy en el repo) para el menú de acciones del bug 5 (research.md D5) — no es una
dependencia nueva en el sentido del Principio IX (ya está instalada y justificada desde que se
agregó Angular Material/CDK como base del proyecto), solo un módulo de esa librería sin uso previo.

**Storage**: PostgreSQL 16, schema por tenant. Una migración nueva, 100% aditiva: columna
`product_variants.presentation_id` (FK nullable a `presentations.id`, `ON DELETE SET NULL`), sin
tocar ninguna fila existente (todas las variantes actuales nacen con `presentation_id = NULL`).
Ver [data-model.md](./data-model.md).

**Testing**: `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`
(backend). `ng test` (`@angular/build:unit-test`, frontend, sin Karma/Jasmine).

**Target Platform**: Linux server (`pos-backend`) + SPA Angular (`pos-heladeria`), ambos en
producción vía `develop`.

**Project Type**: Web application (API FastAPI + frontend Angular), dos repos independientes.

**Performance Goals**: Sin objetivo nuevo. La guarda de exclusividad de producto (FR-021) agrega,
como mucho, una consulta adicional por creación/edición de promoción (no es un hot path de venta);
el cálculo de precio mínimo de la tarjeta del menú QR (bug 1) es una reducción en memoria sobre un
arreglo ya presente en la respuesta (`product.variants`), sin consulta adicional.

**Constraints**:
- **FR-007 (no retroactivo, bug 2)**: la columna nueva nace `NULL` en toda variante existente; el
  sistema nunca fuerza una asociación — coincide con Principio VII.
- **FR-024 (exclusividad solo entre promociones distintas)**: la guarda nueva (FR-021) no debe
  bloquear variantes distintas del mismo producto dentro de la **misma** promoción — debe
  excluirse a sí misma de la comparación al editar.
- **FR-025 / no retroactivo (bug 3/5)**: la guarda nueva de exclusividad de producto no se evalúa
  contra promociones ya `active` hoy que compartan un producto bajo el criterio anterior
  (`_guard_variant_overlap`, por variante) — solo aplica hacia adelante, al crear/guardar/activar.
- **Principio VII (datos históricos)**: ninguna `Sale`/`Invoice`/`CustomerOrder` se toca — spec
  100% de catálogo, configuración de promociones y UI.
- **D7 (cambios de comportamiento a registrar)**: **A-76**, **A-77**, **A-78** (research.md D7)
  deben existir en `registro-de-anomalias.md` **antes** de mergear los commits que implementan
  bug 4, bug 3 y la exclusividad de producto respectivamente (Principio II).

**Scale/Scope**: 1 columna nueva + 1 migración aditiva (backend); extensión de
`VariantSaveIn`/`VariantResponse` con `presentation_id` y de `PresentationUpdate` con cascada de
renombre (`app/api/v1/catalog/service.py`, `app/api/v1/presentations/router.py`); 1 función de
guarda nueva en `app/api/v1/promotions/service.py` (`_guard_product_overlap` o equivalente),
invocada desde `create()` y `update_shape()`; cero endpoints nuevos, cero tablas nuevas. Frontend:
selector de presentación en `product-form.component.ts` (~1 fila de UI reusando
`presentation.service.ts`); `[disabled]` + guarda en `promotions-page.component.ts` (bug 4);
refactor de `addRuleRow()`/`resolvedVariantIdsForLabel` a N reglas + lista de checkboxes (bug 3);
reemplazo del `<div>` absoluto del menú ⋮ por `CdkOverlay`/`CdkMenu` (bug 5, mismo archivo);
extensión de `promotion-pricing.util.ts` y `public-menu.component.ts` para leer
`variant.promotion.unit_equivalent`/`display_text` y calcular el mínimo por producto (bug 1). Cero
cambios en `app/api/v1/menu/`, `app/api/v1/cart/`, checkout/table\_sessions, y cero cambios de
forma en `promotions/router.py`/`schemas.py`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` completo: 6 historias de usuario con Acceptance Scenarios, Edge Cases, 28 FR, Key Entities, 6 Success Criteria, Assumptions, y una sesión de Clarifications de 7 preguntas (más 5 adicionales en `/speckit-clarify`) que redefinió con precisión el alcance real de los bugs 2 y 3 contra el código en producción. Este plan es posterior y no reabre nada de eso. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | Tres cambios de comportamiento observable sobre código en producción: bloqueo total de "Configurar" en `active` (bug 4, reemplaza la edición parcial de spec 083 FR-018), reemplazo de la regla combinada de `addRuleRow()` por N reglas independientes (bug 3, comportamiento vigente desde el merge de spec 083), y la nueva exclusividad de producto entre promociones `active` (puede rechazar selecciones hoy válidas). Los tres están documentados en `spec.md` §Clarifications como decisiones de negocio explícitas del propietario del producto, y este plan exige registrarlos como anomalías **A-76**, **A-77**, **A-78** en `registro-de-anomalias.md` **antes** de implementar (research.md D7) — pendiente de ejecutar, no de decidir. El resto (columna `presentation_id` nueva y opcional, precio visible en tarjeta del menú QR, menú de acciones sin recorte) es funcionalidad aditiva o corrección de un vacío, sin comportamiento previo que proteger. | PASS (con 3 acciones pendientes registradas) |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | La regla combinada de `addRuleRow()` (bug 3) es lógica de frontend, sin equivalente en characterization tests de backend — no hay test protegido que editar por ese cambio. La exclusividad de producto (bug 3/US5) es guarda nueva, sin comportamiento previo que proteger. **Bug 4 sí rompe dos tests protegidos**, confirmado por lectura directa del código (research.md D9): `test_ca1_editar_escalares_de_una_activa` y `test_editar_vigencia_de_promocion_multi_regla_afecta_a_todas_con_una_accion` (`test_promotions_rules_admin.py`) llaman hoy `service.update()` sobre una promoción `active` y esperan éxito — la guarda nueva de FR-008/FR-011 los rompe tal cual están. Ambos se actualizan explícitamente (no en silencio) citando **A-76**, con evidencia en el mismo archivo de que el resto de sus tests (`test_ca2_cambiar_reglas_de_una_activa_bloquea`, `test_ca4_duplicar_copia_borrador_con_las_mismas_reglas`, los de `TestUS5MantenimientoPorLote` sobre pausar/reactivar) no dependen de `service.update()` en `active` y siguen en verde sin tocarse. | PASS (2 tests protegidos, actualización explícita planificada) |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | La asociación variante↔presentación, el precio visible en la tarjeta del menú QR, y el menú de acciones sin recorte son comportamiento enteramente nuevo definido por FR-001 a FR-006, FR-012 a FR-015 y FR-026 a FR-028. Los tres cambios de comportamiento (bug 3, bug 4, exclusividad) están autorizados por decisión de negocio explícita en Clarifications, no solo por este plan. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Se descartó introducir `ReactiveFormsModule` en `product-form.component.ts` para el selector de presentación (sigue `FormsModule`/`ngModel`, mismo patrón que el resto del formulario, research.md D-frontend) y se descartó modificar `_guard_variant_overlap` existente para que también cubra producto completo (research.md D3) — ambos habrían sido refactorizaciones no exigidas por ningún FR. Cada archivo tocado deriva directamente de un FR: `catalog/service.py` (FR-002/003/004), `promotions/service.py` (FR-021, guarda nueva y separada), `promotions-page.component.ts` (FR-008, FR-016/017, FR-026/027/028 — mismo archivo para tres bugs porque las tres historias viven en esa página, no por conveniencia de refactor). | PASS |
| **VI. Evolución Incremental** | Las seis correcciones son separables y se implementan como incrementos distintos, en el mismo orden de prioridad que `spec.md`: (A) columna `presentation_id` + migración + selector en formulario de producto (US1, sin dependencias); (B) `[disabled]` de "Configurar" (US2, sin dependencias); (C) precio en tarjeta del menú QR (US3, sin dependencias — no requiere que exista la exclusividad de producto para funcionar, solo la asume como supuesto documentado); (D) refactor de `addRuleRow()` a N reglas + checkboxes (US4, sin dependencias); (E) guarda de exclusividad de producto (US5, sin dependencias — es aditiva, no requiere D); (F) `CdkOverlay` en el menú ⋮ (US6, sin dependencias). Ningún incremento mezcla la migración de datos (A) con cambio de UI de otro bug, ni con refactorización no relacionada. | PASS |
| **VII. Compatibilidad con Datos Históricos** | Ninguna `Sale`/`Invoice`/`CustomerOrder`/`CustomerOrderLine` se toca. La migración de la columna nueva solo agrega una columna `NULL` por defecto — no reescribe ninguna fila existente de `product_variants` ni de ninguna otra tabla. | PASS |
| **VIII. Evolución del Modelo de Datos** | [data-model.md](./data-model.md) especifica la columna nueva (tipo, FK, `ondelete`, default), el `UniqueConstraint` que la acompaña (FR-006), el paso de sincronización de nombre (cascada en `PATCH /presentations/{id}`) y el `downgrade` (DROP de la columna, sin pérdida de datos porque es aditiva). Compatibilidad con datos existentes: explícita (toda variante existente nace con `presentation_id = NULL`, comportamiento idéntico a hoy). | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No aplica una dependencia nueva en el sentido estricto — `@angular/cdk` ya está en `package.json` de `pos-heladeria`; research.md D5 documenta por qué se empieza a usar su módulo `Overlay`/`Menu` (ya resuelve exactamente el recorte y reposicionamiento que causa el bug 5) en vez de seguir con posicionamiento CSS manual. | PASS (no aplica dependencia nueva) |
| **X. Verificación Obligatoria** | Cada historia de `spec.md` tiene "Independent Test"; [quickstart.md](./quickstart.md) los traduce en pasos ejecutables (curl + verificación de UI) por historia, más el paso 0 (suite existente en verde). Se ejercitan los escenarios más sensibles: unicidad de presentación por producto (FR-006), no bloqueo de "Configurar" fuera de `active` (FR-009), exclusividad de producto solo entre promociones distintas (FR-024), y que el motor de cálculo de descuentos no cambió (`test_promotions_service.py` sigue en verde tal cual). | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Decisiones de negocio en `spec.md` §Clarifications (12 preguntas entre `/speckit-specify` y `/speckit-clarify`: alcance de la aplicación masiva, nivel de bloqueo de "Configurar", mecánica de exclusión por casilla, redefinición completa del bug 2, sincronización de nombre, exclusividad de producto y su granularidad). Decisiones técnicas en [research.md](./research.md) (D1-D9: modelo de datos de la columna nueva, estrategia de sincronización de nombre, guarda de exclusividad como función separada, refactor de `addRuleRow`, CDK Overlay vs. CSS manual, cálculo de precio mínimo en frontend vs. backend), cada una con alternativa descartada y su porqué. | PASS |
| **XII. Trazabilidad** | Cadena: `spec.md` (Clarifications + 28 FR) → este `plan.md`/`research.md`/`data-model.md`/`contracts/` → `registro-de-anomalias.md` (entradas A-76/A-77/A-78 pendientes, citando research.md D7) → `tasks.md` (Fase 2, `/speckit-tasks`) → implementación → tests (nuevos, citando las anomalías) → [quickstart.md](./quickstart.md). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research, data-model, contracts, quickstart), los nombres de tests nuevos, las entradas A-76/A-77/A-78 y los mensajes de commit de esta funcionalidad se escriben en español de Colombia, igual que el resto del repositorio. | PASS |

Sin violaciones que justificar en Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/084-fix-promociones-productos/
├── plan.md                    # Este fichero (/speckit-plan)
├── research.md                # Fase 0 — D0 (verificación de código real) + D1-D9
├── data-model.md              # Fase 1 — product_variants.presentation_id, migración y rollback
├── contracts/
│   ├── variante-presentacion.md            # FR-001–007 (bug 2)
│   ├── configurar-bloqueada-activa.md      # FR-008–011 (bug 4)
│   ├── precio-minimo-tarjeta-qr.md         # FR-012–015 (bug 1)
│   ├── aplicacion-masiva-regla.md          # FR-016–020 (bug 3)
│   ├── exclusividad-producto-promociones.md # FR-021–025 (nuevo, bug 3)
│   └── menu-acciones-sin-scroll.md         # FR-026–028 (bug 5)
├── quickstart.md               # Fase 1 — validación ejecutable por historia de usuario
└── tasks.md                    # Fase 2 (/speckit-tasks — no generado por este comando)
```

### Source Code (repositorios sibling de `pos-specs`, sobre rama nueva desde `develop`)

```text
# ../pos-backend
app/
├── models/
│   └── product_variant.py             # MODIFICADO — nueva columna `presentation_id`
│                                         (FK nullable a presentations.id, ondelete SET NULL),
│                                         nuevo UniqueConstraint(product_id, presentation_id),
│                                         relationship `presentation` (sin back_populates de
│                                         lista, es 1:1 por variante)
│
├── api/v1/catalog/
│   ├── schemas.py                     # MODIFICADO — VariantSaveIn/VariantResponse ganan
│   │                                     `presentation_id` opcional
│   └── service.py                     # MODIFICADO — _save_variant_entry: al recibir
│                                         presentation_id, valida que la presentación esté
│                                         activa y no usada por otra variante del mismo
│                                         producto (FR-006), y toma `name` de
│                                         `Presentation.name` (FR-002/003) en vez del `name`
│                                         del payload cuando presentation_id no es null
│
├── api/v1/presentations/
│   └── router.py                      # MODIFICADO — PATCH /{id}: al renombrar, cascada
│                                         UPDATE product_variants SET name = :nuevo_nombre
│                                         WHERE presentation_id = :id (FR-004), misma
│                                         transacción que el UPDATE de Presentation
│
├── api/v1/promotions/
│   └── service.py                     # MODIFICADO — dos cambios independientes:
│                                         (bug 4, US2) guarda `if promo.status == "active":
│                                         raise 409` al inicio de `update()` (línea ~762,
│                                         respalda `PATCH /promotions/{id}`, router.py:81) —
│                                         NO reutiliza la condición de `update_shape`, que
│                                         también bloquearía `finished` (FR-009, research.md
│                                         D9);
│                                         (bug 3/US5) nueva función `_guard_product_overlap`
│                                         (junto a `_guard_variant_overlap`, línea ~579),
│                                         invocada desde create() (~742) y update_shape()
│                                         (~781); compara product_id de las variantes contra
│                                         promotion_variants de OTRAS promociones con
│                                         status="active" (no draft/paused, FR-021/FR-024)
│
├── alembic/versions/
│   └── XXXX_084_variante_presentacion.py   # NUEVO — down_revision "da7581f7bb18".
│                                              @for_each_tenant_schema: ALTER TABLE
│                                              product_variants ADD COLUMN presentation_id
│                                              (FK nullable, ON DELETE SET NULL) + UNIQUE
│                                              constraint. downgrade: DROP COLUMN.
│
└── characterization_tests/
    ├── test_products_variant_presentation.py   # NUEVO — asociar/reasociar/quitar
    │                                              presentación (FR-001–003/005), unicidad
    │                                              por producto (FR-006), cascada de renombre
    │                                              al editar una Presentación (FR-004),
    │                                              incluido el 409 de colisión de nombre
    │                                              contra una variante sin presentación
    │                                              (edge case agregado tras /speckit-analyze)
    ├── test_promotions_product_overlap.py       # NUEVO — exclusividad de producto entre
    │                                              promociones active (FR-021–025), cita A-78
    └── test_promotions_rules_admin.py           # MODIFICADO — `test_ca1_editar_escalares_
                                                    de_una_activa` y
                                                    `test_editar_vigencia_de_promocion_multi_
                                                    regla_afecta_a_todas_con_una_accion`
                                                    (ambos en `service.update()` sobre una
                                                    promoción `active`) pasan de esperar éxito
                                                    a esperar 409, citando A-76 (research.md
                                                    D9). El resto del archivo (reemplazo de la
                                                    regla combinada por N reglas es lógica de
                                                    frontend, sin equivalente aquí) sigue en
                                                    verde sin tocarse — confirmar en
                                                    verificación final.

# ../pos-heladeria
src/app/
├── modules/products/
│   ├── pages/product-form.component.ts        # MODIFICADO — cada fila de variante gana un
│   │                                             <select> de presentación (reusa
│   │                                             PresentationService.allPresentations,
│   │                                             filtrado a active), que al elegirse
│   │                                             autocompleta y bloquea el <input> de name
│   │                                             (FR-002/005)
│   └── interfaces/product.interface.ts          # MODIFICADO — VariantDraft y
│                                                   VariantSavePayload ganan
│                                                   `presentationId: string | null`
│
├── modules/promotions/
│   └── pages/promotions-page.component.ts       # MODIFICADO — tres bugs en el mismo
│       │                                          archivo:
│       │                                          (bug 4) botón "Configurar":
│       │                                            [disabled]="p.status === 'active'",
│       │                                            mismo criterio que canDelete();
│       │                                          (bug 3) addRuleRow()/
│       │                                            resolvedVariantIdsForLabel: refactor a
│       │                                            generar N PromotionRuleForm (una por
│       │                                            producto coincidente) + lista de
│       │                                            checkboxes premarcados antes de
│       │                                            confirmar (FR-016–019), validaciones de
│       │                                            FR-020 aplicadas por fila;
│       │                                          (bug 3, nuevo) Paso 1: excluir/deshabilitar
│       │                                            productos ya cubiertos por otra
│       │                                            promoción active, con el mensaje de
│       │                                            FR-022 (requiere que el backend exponga
│       │                                            qué promoción activa cubre cada
│       │                                            producto candidato — ver
│       │                                            contracts/exclusividad-producto-
│       │                                            promociones.md para la forma exacta);
│       │                                          (bug 5) menú ⋮ (líneas ~284-354):
│       │                                            reemplazar el <div> absoluto por
│       │                                            CdkOverlay/CdkMenu (import
│       │                                            OverlayModule/MenuModule de
│       │                                            @angular/cdk), toggleActionsMenu/
│       │                                            closeActionsMenu adaptados al ciclo de
│       │                                            vida del overlay (FR-026–028)
│
├── modules/promotions/services/
│   └── promotion-pricing.util.ts                # MODIFICADO — nuevo helper
│                                                    `minPromoPriceForProduct(variants)` que
│                                                    itera `variant.promotion.unit_equivalent`
│                                                    y devuelve el más bajo con su
│                                                    `display_text` (FR-012/013), sin tocar
│                                                    `effectivePrice`/`discountInfo`
│                                                    existentes (siguen cubriendo el caso de
│                                                    spec 066 FR-015, FR-014 de esta spec)
│
└── modules/tables/pages/
    └── public-menu.component.ts                 # MODIFICADO — productDiscount()/tarjeta
                                                     (líneas ~483-509): usa
                                                     minPromoPriceForProduct() en vez de
                                                     depender solo de discounted_price, para
                                                     mostrar precio también cuando min_qty >= 2
                                                     (FR-012–015)
```

**Structure Decision**: la asociación variante↔presentación se implementa como una columna nueva
en `ProductVariant` (relación 1:1 por variante, no una tabla puente — research.md D1), reusando el
`UniqueConstraint` existente de `(product_id, name)` como precedente para el nuevo
`(product_id, presentation_id)`. La sincronización de nombre se resuelve con una cascada de
`UPDATE` en el `PATCH` de `Presentation` en vez de derivar `name` en tiempo de lectura
(research.md D2) — mantiene intacto todo el código que hoy lee `variant.name` como columna simple
(recibos, promociones, exportes). La exclusividad de producto entre promociones vive en una
función de guarda **nueva y separada** de `_guard_variant_overlap` (research.md D3), porque su
criterio de estado es distinto (solo `active`, no `draft/paused` con cruce de vigencia) y no debe
alterar la guarda que spec 063 FR-014 sigue exigiendo tal cual. El menú de acciones migra a
Angular CDK Overlay (research.md D5) en vez de un fix CSS manual, porque ya es la dependencia
estándar del proyecto para este problema exacto y no hay ningún uso previo que reutilizar ni
romper. El precio mínimo de la tarjeta del menú QR se calcula 100% en frontend (research.md D6),
reusando un dato que `menu_variant_promotion` (backend) ya calcula por variante para cualquier
`min_qty` — cero cambios en `app/api/v1/menu/`.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
