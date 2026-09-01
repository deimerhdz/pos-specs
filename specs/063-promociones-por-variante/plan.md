# Implementation Plan: Refactorización del módulo de promociones — modelo por conjunto explícito de variantes

**Branch**: `refactor/063-promociones-por-variante` | **Date**: 2026-09-01 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/063-promociones-por-variante/spec.md`

> **Nota de esta revisión del plan (2026-09-01)**: la primera versión de este plan (modelo
> "una promoción = una combinación") **ya está construida** en las dos ramas de feature —
> `refactor/063-promociones-por-variante` en `pos-backend` (8 commits, migraciones `063a`/
> `063b` aplicadas) y `feature/063-promociones-por-variante` en `pos-heladeria` (6 commits) —
> pero **ninguna de las dos está mergeada a `main`** (verificado con
> `git merge-base --is-ancestor HEAD main` en ambos repos, 2026-09-01). La sesión de
> `/speckit-clarify` del 2026-09-01 cambió el modelo de datos de `spec.md` para introducir
> `Regla` como entidad hija de `Promoción` (Clarifications, sesión 2026-09-01). Por decisión
> explícita del propietario del repositorio, este plan **construye sobre las ramas de feature
> ya existentes** en vez de revertirlas o esperar a mergearlas: trata el modelo plano ya
> implementado como el "estado actual a reemplazar" y añade **dos migraciones nuevas**
> (`063c` aditiva, `063d` destructiva) encima de las ya aplicadas (`063a`/`063b`), sin tocar
> ninguna de las dos.

## Summary

Las ramas de feature de `pos-backend` y `pos-heladeria` ya implementan el modelo "una
promoción = (tipo, valor, cantidad mínima) + conjunto explícito de variantes" descrito por la
primera versión de `spec.md` (verificado en detalle, ver "Estado real de las ramas de feature"
más abajo): tabla puente `promotion_variants`, motor único `evaluate_variant_sets`, dos tipos
vivos (`percent`/`package_price`), `priority` y `Presentation` ya eliminados, `applied_promotions`
JSONB ya persistido en `sales`/`invoices`/`customer_orders`.

La sesión de clarificación del 2026-09-01 (`spec.md` → Clarifications, FR-001/FR-001a) reemplaza
ese modelo por uno de dos niveles:

1. **`Promoción`** conserva vigencia (fechas, días, horas) y estado (`Borrador`/`Activa`/
   `Pausada`/`Finalizada`). **Pierde** `type`, `value`, `min_qty` — pasan a la nueva entidad.
2. **`Regla`** (nueva, hija de `Promoción`, `promotion_rules`) es la unidad que hoy tiene la
   `Promoción`: tipo (`percent`/`package_price`), valor, cantidad mínima y su propio conjunto de
   variantes (`promotion_variants`, repuntada de `promotion_id` a `promotion_rule_id`). Una
   promoción agrupa **una o más** reglas; dentro de una misma promoción sus conjuntos de
   variantes son disjuntos entre sí (FR-001a).
3. El motor (`evaluate_variant_sets`) pasa de agrupar por **promoción vigente** a agrupar por
   **regla cuya promoción está vigente** — la vigencia se resuelve una vez por promoción y se
   aplica a todas sus reglas; el consumo codicioso, el reparto por importe cobrado y el tope al
   precio normal (FR-006–FR-009) no cambian, solo la unidad sobre la que corren.
4. El bloqueo de solapamiento (FR-014) se extiende: entre reglas de promociones **distintas**
   seguirá exigiendo variante compartida **y** ventanas que se intersectan (sin cambio); entre
   reglas de la **misma** promoción, comparten vigencia por definición, así que compartir
   variante **siempre** bloquea (FR-001a, invariante nuevo, sin equivalente en el modelo plano).
5. La persistencia (FR-021) pasa de "lista de promociones" a "lista de reglas" —cada entrada de
   `applied_promotions` gana `rule_id` junto al `promotion_id` que ya tenía— para que el arqueo
   distinga qué regla (qué tramo de precio) generó cada descuento, no solo qué promoción.
6. El formulario de administración pasa de "una promoción = un tipo/valor/conjunto" a "una
   promoción = vigencia + N reglas capturadas en la misma sesión" — la mejora de UX de creación y
   mantenimiento por lote que motivó la partición (Clarifications, sesión 2026-09-01): pausar,
   activar o extender la vigencia de una promoción con varias reglas es **una sola acción**, sin
   repetirla por cada regla.

Es, en términos de alcance, **un segundo refactor sobre un refactor recién terminado pero aún no
mergeado**: no reabre nada de lo que 063a/063b ya resolvió (migración de `percent`/`combo`/
`fixed`/`qty_price`/`qty_price_presentation`, retiro de `Presentation`, retiro de `priority`) —
eso se hereda intacto — y se acota estrictamente a partir `type`/`value`/`min_qty`/`conjunto` en
una entidad propia.

### Estado real de las ramas de feature (verificado 2026-09-01)

**`pos-backend`** (rama `refactor/063-promociones-por-variante`, working tree limpio, no mergeada
a `main`; head de Alembic **`ba4b6bd573a6`** = revisión `063b`):

- `app/models/promotion.py` (150 líneas): `class Promotion` (43–114) con `type`/`value`/`min_qty`
  como columnas propias, `ck_promotion_type` (`type IN ('percent','package_price') OR
  status='finished'`), `ck_promotion_value_positive`, `ck_promotion_min_qty`,
  `ck_promotion_percent_range`; relación `variants` (90–92, cascade all/delete-orphan). `class
  PromotionVariant` (117–149): `promotion_id` FK CASCADE, `product_variant_id` FK CASCADE,
  `UNIQUE(promotion_id, product_variant_id)`. Sin `priority`, sin `PromotionTarget`/
  `PromotionComboItem`/`PromotionPresentationRule` (borrados en `063b`).
- `app/api/v1/promotions/service.py` (635 líneas): `active_variant_set_promotions` (138),
  `_unit_sort_key` (155), `_greedy_units` (164), `_distribute_group_discount` (175),
  `evaluate_variant_sets` (211), `applied_to_dicts` (272), `menu_unit_discount` (281),
  `variant_set_condition_text` (294), `_guard_variant_overlap` (338),
  `_guard_package_is_discount` (383), `list_query` (416), `_apply_variant_set` (441),
  `_revalidate_type_rules` (475), `create` (488), `update` (506), `update_shape` (530),
  `change_status` (551), `duplicate` (573), `serialize_promotion` (597).
- `app/api/v1/promotions/schemas.py` (217 líneas): `PromotionType` enum = `{percent,
  package_price}` (17); `PromotionCreate` (104) con `type`/`value`/`min_qty`/`variant_ids` a nivel
  de promoción; `PromotionShapeUpdate` (151) igual; `PromotionResponse` (180) con
  `variants: list[PromotionVariantResponse]` (172) directamente.
- Call sites únicos de `evaluate_variant_sets`: `app/api/v1/orders/checkout.py:273` (dentro de
  `auto_discount`, definida en 265; `promo_lines_for` en 241), `app/api/v1/cart/service.py:264` y
  `:588`. `app/api/v1/menu/router.py` importa `active_variant_set_promotions`,
  `menu_unit_discount`, `variant_set_condition_text` (26–27) y las usa en 86/158-159/197-198.
  `app/api/v1/table_sessions/service.py` (187, 666, 772) y `app/api/v1/sales/service.py:254`
  llaman `checkout.auto_discount`, no el motor directamente.
- `sale.py`/`invoice.py`/`customer_order.py` ya tienen `discount` + `applied_promotions` JSONB
  (`server_default='[]'::jsonb`); `app/api/v1/sales/builder.py:88-89/120-122` los persiste;
  `app/api/v1/invoices/service.py:73` los copia al emitir factura.
- Migraciones: `387ef3e638cd_063a_promociones_por_conjunto_aditivo.py` (down_revision
  `e1c455751dbc`) y `ba4b6bd573a6_063b_promociones_retiro_estructura_.py` (down_revision `063a`,
  head actual).
- Tests: `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` (sin
  pytest/conftest.py). `test_promotions_service.py`, `test_promotions_router.py`,
  `test_promotions_rules_admin.py`, `test_promotions_migration.py` ya existen y ya pasan contra
  el modelo plano.

**`pos-heladeria`** (rama `feature/063-promociones-por-variante`, working tree limpio, no
mergeada a `main`):

- `src/app/modules/promotions/interfaces/promotion.interface.ts`: `PromotionType` (13) =
  `'percent' | 'package_price'`; `Promotion` (41) con `type`/`value`/`min_qty`/
  `variants: PromotionVariant[]` (34) a nivel de promoción, no de regla.
- `src/app/modules/promotions/pages/promotions-page.component.ts` (900 líneas): formulario
  **template-driven** (`FormsModule`/`ngModel`, sin `ReactiveFormsModule` ni `FormGroup`/
  `FormArray` — confirmado por grep). Un solo tipo/valor/cantidad mínima/conjunto por promoción
  (316–405).
- `src/app/modules/promotions/services/promotion-pricing.util.ts`: `getPromoDisplay` (107),
  `discountInfo` (73), `effectivePrice` (54) — todas asumen una promoción = una condición.
  `services/promotion.service.ts` (272 líneas): sin `overlapCandidates` ni targets/combos/
  presentación (ya retirados); señales `overlapConflict` (39) y `packageNotDiscount` (41) para
  los 409 de FR-014/FR-016.
- `src/app/modules/tables/services/diner.service.ts:27-35`: `MenuPromotionAnnouncement` **ya**
  tiene la forma `{ promotion_id, promotion_name, rules: {text, variant_count, min_qty, value}[]
  }` — el DTO del menú QR **ya anticipa una lista de "rules" por promoción** aunque hoy el
  backend solo puebla una por promoción (una promoción = una combinación). Esto reduce el riesgo
  de este refactor en la superficie del menú QR: el contrato externo no cambia de forma, solo de
  cardinalidad (ver research.md D-R3).
- `src/app/modules/tables/services/pos-terminal.store.ts`: `combos` (392) es un stub vacío
  permanente (comentario 390-391 cita FR-024); `productDiscountBadge()` (407-419) escanea
  `promotionService.activePromotions()` por variante — hoy a nivel de promoción, debe pasar a
  escanear reglas.
- Sin `presentations`, sin `presentation_id` en productos, sin `combo-select.component.ts`, sin
  `scope-picker.component.ts` — todo ya retirado.

## Technical Context

**Language/Version**: Backend Python 3.12 (imagen Docker) / 3.14 (venv local `pos-backend/env`).
Frontend Angular (`../pos-heladeria`). Esta spec toca **ambos repositorios** (Constitución
§Alcance), sobre las ramas de feature ya existentes (no `main`).

**Primary Dependencies**: FastAPI + SQLAlchemy 2.0 (sync, `Mapped`/`mapped_column`), Alembic
(`@for_each_tenant_schema`, head actual **`ba4b6bd573a6`**), Pydantic v2. Angular + signals,
formulario **template-driven** (`FormsModule`/`ngModel`, sin `ReactiveFormsModule` — se mantiene
el patrón ya usado en `promotions-page.component.ts` en vez de introducir `FormArray`, research.md
D-R4). **Ninguna dependencia nueva** (Principio IX no aplica): una tabla hija más, una columna
repuntada, un campo más en un JSONB ya existente.

**Storage**: PostgreSQL 16, schema por tenant. Dos revisiones nuevas **encima** de `063a`/`063b`
(sin modificarlas):

*Revisión `063c` (aditiva):*
- `promotion_rules` — **tabla nueva** (hija de `promotions`; `promotion_id` FK CASCADE). Columnas:
  `type` (mismo `CHECK` que hoy tiene `promotions.type`, sin el escape `OR status='finished'`
  porque una regla no tiene estado propio), `value` NUMERIC(12,2), `min_qty` INTEGER, mismos
  `CHECK`s que hoy tiene `promotions` (`ck_promotion_value_positive`, `ck_promotion_min_qty`,
  `ck_promotion_percent_range`), renombrados `ck_promotion_rule_*`.
- `promotion_variants` — **columna nueva** `promotion_rule_id` (FK CASCADE a `promotion_rules.id`,
  **nullable** en esta revisión).
- **Paso de datos**: para cada fila de `promotions`, INSERT en `promotion_rules` con su
  `type`/`value`/`min_qty` (una regla por promoción existente — todas las promociones actuales
  tienen exactamente una combinación, así que esta migración es 1:1, sin ambigüedad); UPDATE
  `promotion_variants.promotion_rule_id` = el `id` de esa regla nueva, para todas las filas cuyo
  `promotion_id` coincida.
- `promotions.type`/`value`/`min_qty` y `promotion_variants.promotion_id` **se conservan** en esta
  revisión (patrón aditivo, igual que `063a`): el código de aplicación deja de leerlos/escribirlos
  en el mismo incremento que aplica `063c` (Principio VI, un solo incremento cambia esquema +
  código), pero la revisión en sí no los borra — eso es `063d`.

*Revisión `063d` (destructiva):*
- `promotion_variants` — `promotion_rule_id` pasa a NOT NULL; **se borra** `promotion_id` (+ su FK
  + su índice) y la `UNIQUE(promotion_id, product_variant_id)` vieja; la unicidad nueva es
  `UNIQUE(promotion_rule_id, product_variant_id)`.
- `promotions` — **se borran** `type`, `value`, `min_qty` y sus tres `CHECK`s
  (`ck_promotion_type`, `ck_promotion_value_positive`, `ck_promotion_min_qty`,
  `ck_promotion_percent_range`).
- **Columnas históricas que NO se tocan** (Principio VII): `applied_promotions` en `sales`/
  `invoices`/`customer_orders` conserva las entradas ya escritas por el modelo plano (sin
  `rule_id`, porque no existía) — se leen igual, un lector tolera la ausencia del campo; no hay
  backfill retroactivo (FR-021, "el registro NO es retroactivo").

**Testing**: `unittest` vía `python -m unittest discover -s app/characterization_tests -p
'test_*.py' -v` (sin pytest/conftest.py, confirmado). Los tests ya reescritos para el modelo plano
(`test_promotions_service.py`, `test_promotions_rules_admin.py`, `test_promotions_migration.py`,
`test_promotions_router.py`) se **reescriben otra vez** para el modelo `Promoción`+`Regla` — la
mayoría de sus fixtures (`make_promotion`) ganan un nivel: en vez de crear una promoción con
tipo/valor/conjunto, crean una promoción y le agregan una regla. Los characterization tests de
`cart`/`checkout`/`table_sessions`/`sales`/`menu` que ya pasan contra el modelo plano (ver research.md
D-R5) se ajustan solo donde tocan la forma de `applied_promotions` (gana `rule_id`) o el texto de
condición (`variant_set_condition_text` ahora recibe una regla).

**Target Platform**: Linux server (`pos-backend`) + SPA Angular (`pos-heladeria`), ambos aún en
rama de feature, no en producción.

**Project Type**: Web application (API FastAPI + frontend Angular), dos repos independientes.

**Performance Goals**: Sin objetivo nuevo. `evaluate_variant_sets` sigue corriendo una vez por
cálculo de cobro/preview; el cambio de agrupar por promoción a agrupar por regla no cambia el
orden de magnitud (decenas de reglas activas por tenant, no miles) — una consulta trae
`promotion_rules` con `selectinload` de `variants` y `joinedload` de su `Promotion` padre (para la
vigencia), en vez de `promotions` con `selectinload` de `variants` directo.

**Constraints**:
- **FR-001a (reglas de la misma promoción no comparten variante)**: invariante **nuevo**, sin
  equivalente en el modelo plano. Se aplica en `_guard_variant_overlap`, no en un `CHECK` de base
  de datos (la unicidad cruzada "una variante no puede estar en dos reglas de la misma promoción"
  no es expresable con un `UNIQUE` simple sobre `promotion_variants`, que solo conoce
  `promotion_rule_id`, no la promoción a la que pertenece esa regla — igual que el bloqueo de
  solape entre promociones distintas, FR-014, ya es un guard de aplicación y no un `CHECK`,
  research.md D-R6).
- **FR-020 (no congelar el descuento)**: sin cambio de fondo — sigue sin tabla que guarde montos
  calculados antes de emitir.
- **FR-008/FR-008a/SC-005 (reparto determinista)**: sin cambio de fondo, ahora corre por regla en
  vez de por promoción.
- **FR-014/FR-014a (bloqueo de solape real)**: se extiende (no se reemplaza) a reglas en vez de
  promociones, con el invariante nuevo intra-promoción de FR-001a.
- **Principio VII (datos históricos)**: ninguna `Sale`/`Invoice`/`CustomerOrder` ya emitida se
  recalcula; las entradas ya escritas de `applied_promotions` (sin `rule_id`) no se reescriben.
  `063c` solo agrega estructura y hace un paso de datos sobre `promotions`/`promotion_variants`
  (no sobre ventas); `063d` solo borra estructura ya sin lectores.
- **Compatibilidad hacia atrás**: ninguna, porque ninguna de las dos ramas está en producción —
  no hay clientes de la API vieja que romper fuera de los propios repos en refactor.

**Scale/Scope**: 1 tabla nueva (`promotion_rules`) + 1 columna nueva (`promotion_variants.
promotion_rule_id`) + 4 columnas/constraints movidos de `promotions` a `promotion_rules` + 1
columna borrada (`promotion_variants.promotion_id`), en **2 migraciones `@for_each_tenant_schema`
nuevas** (`063c` aditiva, `063d` destructiva) encima de las 2 ya aplicadas. El motor
(`evaluate_variant_sets` y sus 6 funciones auxiliares) se reescribe para operar sobre reglas; el
CRUD de promociones (`create`/`update_shape`/`duplicate`/`serialize_promotion`) gana un nivel de
anidamiento (promoción → lista de reglas). El formulario de administración
(`promotions-page.component.ts`, ~900 líneas) gana una sub-lista repetible de reglas dentro del
formulario existente (sin cambiar de `FormsModule` a `ReactiveFormsModule`).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` actualizado con la sesión de clarificación 2026-09-01 (3 preguntas resueltas, FR-001/FR-001a nuevos, Key Entities y 8 FR más ajustados), checklist `requirements.md` sigue en 100%. Este plan es posterior. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | Frente a `main` (producción real), el comportamiento observable que cambia sigue siendo exactamente el ya registrado como A-58…A-65 en `registro-de-anomalias.md` (spec.md §"Cambios de comportamiento") — ninguno de esos puntos se reabre. La introducción de `Regla` (ítem 9 de esa misma sección) **no** es un cambio de comportamiento de producción: ni `main` ni las ramas de feature (no mergeadas) exponen hoy el modelo plano a un usuario real. Es una revisión del propio diseño del refactor, documentada como tal. | PASS |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | Los tests ya reescritos para el modelo plano (`test_promotions_*`) se reescriben otra vez citando esta revisión del plan; el comportamiento congelado que sigue vigente (una sola regla por línea, tope al precio normal, remanente a precio normal, vigencia en hora local, cruce de medianoche) se re-congela sin cambio de fondo, solo de la entidad que lo porta. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | La partición `Promoción`/`Regla`, el bloqueo intra-promoción (FR-001a) y la creación/mantenimiento por lote son comportamiento nuevo definido por `spec.md`. El criterio de éxito es conformidad con esos FR/SC, no equivalencia con el modelo plano. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Todo lo que se toca deriva de FR-001/FR-001a o de un FR ya existente que cambia de unidad (promoción→regla). **No** se toca nada de lo que `063a`/`063b` ya resolvió (migración de tipos legados, retiro de `Presentation`/`priority`) — se hereda intacto. **No** se aprovecha para renombrar `applied_promotions`, tocar el índice `ix_promotions_status_ends_at`, ni reescribir código de checkout/cart/sales más allá del cambio de firma que exige operar sobre reglas. | PASS |
| **VI. Evolución Incremental** | Este plan es en sí mismo un incremento sobre los 6 ya construidos (A–F de la primera versión, ya en las ramas de feature). Se entrega en **5 incrementos nuevos** (research.md D-R7), separando explícitamente migración de datos y cambio de cálculo dentro de lo que antes era un solo "Incremento G" (corrección `/speckit-analyze` 2026-09-01, hallazgo F2 — mismo criterio que ya separó A de C en la primera versión): (G1) migración aditiva `063c` + modelo `PromotionRule` — **sin tocar el motor**; la suite existente pasa **sin editar** con el motor viejo aún leyendo `promotions.type/value/min_qty` directo, checkpoint aislado que aísla cualquier fallo de migración de cualquier fallo de cálculo; (G2) motor (`evaluate_variant_sets` y auxiliares) reescrito sobre reglas, verificado aparte; (H) CRUD multi-regla (`create`/`update_shape`/`duplicate`) + `_guard_variant_overlap` con el invariante FR-001a; (I) frontend: interfaces + `promotion.service.ts` + formulario con sub-lista de reglas + `pos-terminal.store.ts`/`diner.service.ts`; (J) retiro de estructura (`063d` destructiva) + reescritura final de tests CONGELA. Ningún incremento mezcla el paso de datos (solo G1) con el cambio de cálculo (solo G2) ni con el cambio de CRUD (solo H). | PASS |
| **VII. Compatibilidad con Datos Históricos** | Ninguna `Sale`/`Invoice`/`CustomerOrder` se recalcula. El paso de datos de `063c` opera sobre `promotions`/`promotion_variants` (configuración, no hechos contables), no sobre ventas. Las entradas de `applied_promotions` ya escritas (sin `rule_id`) no se reescriben ni se les hace backfill — un lector nuevo tolera su ausencia. | PASS |
| **VIII. Evolución del Modelo de Datos** | [data-model.md](./data-model.md) especifica `promotion_rules` (columnas, `CHECK`s, FK, `ondelete`), el repunte de `promotion_variants` (columna nueva aditiva → NOT NULL + borrado de la vieja en la destructiva), el paso de datos 1:1 y el `downgrade` simétrico de cada revisión, con `@for_each_tenant_schema` + guarda `_has_table`/`_has_column`. | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | Ninguna dependencia nueva. Se descarta explícitamente introducir `ReactiveFormsModule`/`FormArray` en el frontend (research.md D-R4): el formulario ya es template-driven, y una lista repetible de reglas es expresable con `*ngFor` + `ngModel` indexado sin cambiar de patrón. | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de `spec.md` tiene "Independent Test"; [quickstart.md](./quickstart.md) los traduce a `unittest` ejecutables + verificación manual del formulario multi-regla, la terminal y el menú QR. Se ejercita el invariante FR-001a (variante repetida entre reglas de la misma promoción → 409), la migración 1:1 de `063c`, y que el reparto/consumo codicioso siga cuadrando al peso por regla. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Las decisiones de negocio están en `spec.md` (Clarifications, sesión 2026-09-01): `Regla` como entidad hija, invariante de disjunción intra-promoción, persistencia a nivel de regla, límite por plan fuera de alcance. Las decisiones técnicas —tabla `promotion_rules` vs. columna JSONB en `promotions`, dos migraciones nuevas vs. modificar `063a`/`063b` in situ, invariante FR-001a como guard de aplicación vs. `CHECK`/índice parcial, formulario template-driven con `*ngFor` vs. `ReactiveFormsModule`— están en [research.md](./research.md), cada una con su alternativa descartada. | PASS |
| **XII. Trazabilidad** | Cadena: `spec.md` (Clarifications 2026-09-01 + FR-001/FR-001a) → este `plan.md`/`research.md`/`data-model.md`/`contracts/` (decisión técnica) → `registro-de-anomalias.md` (sin entrada nueva: no es cambio de comportamiento de producción, ítem 9 de spec.md lo documenta como revisión de diseño) → `tasks.md` (Fase 2, `/speckit-tasks` — reemplaza el `tasks.md` ya existente, escrito para el modelo plano) → implementación sobre las ramas de feature ya existentes → tests reescritos → `quickstart.md`. | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y sus artefactos, nombres de tests nuevos, mensajes de error (variante repetida entre reglas de la misma promoción, conjunto vacío por regla) y mensajes de commit se escriben en español de Colombia. | PASS |

Sin violaciones que justificar en Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/063-promociones-por-variante/
├── plan.md                    # Este fichero (/speckit-plan) — reemplaza la versión del modelo plano
├── research.md                # Fase 0 — decisiones técnicas de la partición Promoción/Regla
├── data-model.md              # Fase 1 — promotion_rules, repunte de promotion_variants, 063c/063d
├── quickstart.md              # Fase 1 — validación ejecutable por historia de usuario
├── contracts/
│   ├── motor-y-persistencia.md      # evaluate_variant_sets sobre reglas, applied_promotions+rule_id
│   ├── administracion-promociones.md # CRUD multi-regla, FR-001a, FR-014, FR-016, FR-018
│   ├── superficies-consumo.md       # menú QR y terminal por regla (FR-022/FR-023)
│   └── migracion.md                 # paso de datos 063c, inventario de tests a reescribir
├── checklists/
│   └── requirements.md        # Ya existente, 100% (revalidado por /speckit-clarify)
└── tasks.md                   # Fase 2 (/speckit-tasks — reemplaza el tasks.md del modelo plano)
```

### Source Code (repositorios sibling de `pos-specs`, sobre las ramas de feature ya existentes)

```text
# ../pos-backend (rama refactor/063-promociones-por-variante)
app/
├── models/
│   └── promotion.py                 # MODIFICADO —
│                                       Incremento G1 (063c): clase nueva `PromotionRule`
│                                       (promotion_id FK CASCADE; type/value/min_qty +
│                                       ck_promotion_rule_type/value/min_qty/percent_range,
│                                       mismas reglas que hoy tiene Promotion.type/value/min_qty
│                                       pero sin el escape `OR status='finished'`); relación
│                                       `Promotion.rules` (cascade all, delete-orphan).
│                                       `PromotionVariant` gana `promotion_rule_id` (FK CASCADE,
│                                       nullable) y relación `rule`.
│                                       Incremento J (063d): `Promotion` pierde `type`/`value`/
│                                       `min_qty` y sus tres CHECK; `PromotionVariant` pierde
│                                       `promotion_id` (+FK+índice+UNIQUE vieja);
│                                       `promotion_rule_id` pasa a NOT NULL; UNIQUE nueva
│                                       `(promotion_rule_id, product_variant_id)`.
│
├── api/v1/promotions/
│   ├── schemas.py                   # MODIFICADO —
│   │                                   `PromotionRuleIn` nuevo: {type, value, min_qty,
│   │                                   variant_ids}. `PromotionCreate` pierde
│   │                                   type/value/min_qty/variant_ids propios, gana
│   │                                   `rules: list[PromotionRuleIn]` (>= 1). `PromotionShapeUpdate`
│   │                                   pasa a reemplazar la lista completa de reglas de una
│   │                                   promoción en Borrador (mismo criterio "todo editable en
│   │                                   Borrador", FR-018). `PromotionResponse` cambia
│   │                                   `variants` por `rules: list[PromotionRuleResponse]`
│   │                                   (cada una con su tipo/valor/min_qty/variantes+precio,
│   │                                   FR-005). `OverlapConflict` gana `rule_id` (propia y en
│   │                                   conflicto) junto a `promotion_id`.
│   ├── service.py                   # REESCRITURA —
│   │                                     `active_variant_set_promotions` → pasa a devolver
│   │                                       reglas activas con `selectinload(rule.variants)` +
│   │                                       `joinedload(rule.promotion)` para la vigencia
│   │                                       (renombrada `active_variant_set_rules`).
│   │                                     `_greedy_units`/`_distribute_group_discount`: sin
│   │                                       cambio de cuerpo, ahora reciben el conjunto de UNA
│   │                                       regla.
│   │                                     `evaluate_variant_sets`: itera reglas en vez de
│   │                                       promociones; `AppliedPromotion` gana `rule_id`.
│   │                                     `variant_set_condition_text(rule)`: recibe una regla
│   │                                       (antes una promoción).
│   │                                     `menu_unit_discount(rules, variant_id, price)`: itera
│   │                                       reglas.
│   │                                     `_guard_variant_overlap(db, promotion, rules,
│   │                                       variant_ids_by_rule)`: **nuevo caso** — bloquea si
│   │                                       una variante se repite entre dos reglas de la
│   │                                       `promotion` que se está guardando (FR-001a), antes
│   │                                       de comparar contra otras promociones (comportamiento
│   │                                       ya existente, sin cambio).
│   │                                     `_guard_package_is_discount(db, rule)`: por regla.
│   │                                     `_apply_variant_set(db, rule, variant_ids)`: por
│   │                                       regla.
│   │                                     `create`/`update_shape`/`duplicate`/
│   │                                       `serialize_promotion`: ganan el nivel de reglas
│   │                                       (crear/serializar N reglas por promoción en la misma
│   │                                       transacción).
│   ├── router.py                    # MODIFICADO — payloads con `rules`; 409 de FR-001a nombra
│   │                                   la promoción y las dos reglas en conflicto.
│
├── api/v1/orders/checkout.py        # SIN CAMBIO DE FIRMA — `auto_discount`/`promo_lines_for`
│                                       llaman al mismo `evaluate_variant_sets`, que ahora
│                                       resuelve reglas internamente; el resultado
│                                       (`total, promotion_id, applied`) se mantiene, `applied`
│                                       gana `rule_id` por entrada.
├── api/v1/cart/service.py           # SIN CAMBIO DE FIRMA — mismos 2 call sites (264, 588).
├── api/v1/table_sessions/service.py # SIN CAMBIO DE FIRMA — mismos 3 call sites (187, 666, 772)
│                                       vía `checkout.auto_discount`.
├── api/v1/sales/service.py          # SIN CAMBIO DE FIRMA — call site 254 vía
│                                       `checkout.auto_discount`.
├── api/v1/sales/builder.py          # MODIFICADO — `applied_promotions: list[dict]` ahora trae
│                                       `rule_id` por entrada; se persiste igual (120-122).
├── api/v1/invoices/service.py       # SIN CAMBIO — copia `sale.applied_promotions` tal cual
│                                       (línea 73), ya con `rule_id` incluido.
├── api/v1/menu/router.py            # MODIFICADO — `_build_menu_promotions` (189) itera reglas
│                                       de cada promoción vigente en vez de una condición por
│                                       promoción; el DTO de salida `MenuPromotionRule`
│                                       (`schemas.py`) ya tenía forma de lista (research.md D-R3),
│                                       ahora se puebla con N > 1 elementos cuando corresponde.
│
├── alembic/versions/
│   ├── ZZZZ_063c_promociones_reglas_aditivo.py       # NUEVO (Incremento G1) — down_revision
│   │                                   "ba4b6bd573a6". @for_each_tenant_schema + guarda
│   │                                   _has_table/_has_column: crea promotion_rules; agrega
│   │                                   promotion_variants.promotion_rule_id (nullable); PASO DE
│   │                                   DATOS 1:1 (una regla por promoción existente, copiando
│   │                                   type/value/min_qty; repunta promotion_variants). downgrade:
│   │                                   drop de lo aditivo (promotions.type/value/min_qty y
│   │                                   promotion_variants.promotion_id siguen intactos).
│   └── WWWW_063d_promociones_reglas_destructivo.py   # NUEVO (Incremento J) — down_revision
│                                       "063c". Borra promotions.type/value/min_qty (+CHECKs) y
│                                       promotion_variants.promotion_id (+FK+índice+UNIQUE
│                                       vieja); promotion_rule_id → NOT NULL; UNIQUE nueva
│                                       (promotion_rule_id, product_variant_id). downgrade
│                                       simétrico (recrea columnas vacías/con foto fija inversa
│                                       no es posible tras el borrado — mismo criterio que 063b:
│                                       downgrade recrea estructura, no datos).
│
└── characterization_tests/
    ├── test_promotions_service.py        # REESCRITURA — motor sobre reglas: los mismos
    │                                        Acceptance Scenarios de US2, ahora con `make_rule`
    │                                        anidada en `make_promotion`; caso nuevo: dos reglas
    │                                        de la misma promoción con variante compartida → 409
    │                                        (FR-001a).
    ├── test_promotions_rules_admin.py    # REESCRITURA — CRUD multi-regla: crear promoción con
    │                                        N reglas en una sesión, editar reglas solo en
    │                                        Borrador, FR-016 por regla, FR-014 entre reglas de
    │                                        promociones distintas, FR-001a entre reglas de la
    │                                        misma promoción, duplicar copia todas las reglas.
    ├── test_promotions_migration.py      # REESCRITURA — 063c: cada promoción existente termina
    │                                        con exactamente una regla que conserva su
    │                                        type/value/min_qty; promotion_variants apunta a esa
    │                                        regla; Sale/Invoice previas intactas.
    ├── test_promotions_router.py         # MODIFICADO — payloads con `rules`; cita esta revisión.
    ├── test_menu_router.py               # MODIFICADO — anuncio con N reglas por promoción.
    ├── test_orders_checkout.py           # MODIFICADO — `applied_promotions` con `rule_id`.
    ├── test_cart_service.py              # MODIFICADO — ídem + `variant_set_condition_text` por
    │                                        regla.
    └── cart_fixtures.py / orders_fixtures.py /
        table_sessions_fixtures.py        # MODIFICADO — `make_promotion` dejar de aceptar
                                             type/value/min_qty directos; se agrega
                                             `add_rule_to_promotion(promotion, **kwargs)`.

# ../pos-heladeria (rama feature/063-promociones-por-variante)
src/app/
├── modules/promotions/
│   ├── interfaces/promotion.interface.ts    # MODIFICADO — `Promotion` pierde
│   │                                            type/value/min_qty/variants propios, gana
│   │                                            `rules: PromotionRule[]`. `PromotionRule` nuevo:
│   │                                            {id, type, value, min_qty,
│   │                                            variants: PromotionVariant[]}. `PromotionForm`
│   │                                            gana `rules: PromotionRuleForm[]`.
│   │                                            `OverlapConflictError` gana `ruleId`.
│   ├── services/promotion.service.ts        # MODIFICADO — payloads con `rules`; sin cambio de
│   │                                            método (create/update/updateShape/duplicate).
│   ├── services/promotion-pricing.util.ts   # MODIFICADO — `getPromoDisplay`/`discountInfo`/
│   │                                            `effectivePrice` reciben una regla, no una
│   │                                            promoción; un componente que muestra "todas las
│   │                                            condiciones de una promoción" itera `rules`.
│   ├── pages/promotions-page.component.ts    # MODIFICADO — sub-lista repetible de reglas
│   │                                            dentro del formulario existente (`*ngFor` sobre
│   │                                            `form.rules` con `[(ngModel)]` indexado —
│   │                                            se mantiene template-driven, research.md D-R4);
│   │                                            botón "agregar regla" / "quitar regla"; resumen
│   │                                            legible por regla (FR-005); vigencia sigue
│   │                                            siendo un único bloque de campos a nivel de
│   │                                            promoción (sin cambio); edición de
│   │                                            Activa/Pausada sigue bloqueando toda la sección
│   │                                            de reglas (FR-018, ya bloqueaba
│   │                                            type/value/min_qty/conjunto, ahora bloquea la
│   │                                            sub-lista completa).
│
├── modules/tables/
│   ├── services/diner.service.ts             # SIN CAMBIO DE TIPOS — `MenuPromotionAnnouncement.
│   │                                            rules[]` ya tiene la forma correcta
│   │                                            (research.md D-R3); solo deja de asumir "como
│   │                                            mucho 1 elemento" en cualquier lugar que lo
│   │                                            renderice.
│   ├── pages/public-menu.component.ts        # MODIFICADO — si itera `promo.rules` asumiendo un
│   │                                            solo elemento, pasa a `*ngFor` sobre todos.
│   └── services/pos-terminal.store.ts        # MODIFICADO — `productDiscountBadge()` (407-419)
│                                                escanea `promo.rules[].variants` en vez de
│                                                `promo.variants` directo.
│
└── core/services/menu.service.ts            # SIN CAMBIO — no tiene campo `promotions`
                                                 (confirmado); las promociones del terminal
                                                 vienen de `PromotionService`, no de este
                                                 servicio.
```

**Structure Decision**: el refactor añade **una** tabla hija (`promotion_rules`) y repunta la
tabla puente existente (`promotion_variants`) de `promotion_id` a `promotion_rule_id`, sin tocar
ninguna otra tabla ni ningún call-site de checkout/cart/sales/table_sessions (su firma externa —
`auto_discount(db, lines, now) -> (total, promotion_id, applied)`— no cambia; solo el contenido de
`applied`). El motor se reescribe para agrupar por regla; el bloqueo de solapamiento gana un caso
nuevo intra-promoción (FR-001a) que no existía en el modelo plano porque una promoción plana no
podía tener dos reglas que se solaparan consigo misma. El frontend evita introducir
`ReactiveFormsModule`: la sub-lista repetible de reglas se implementa con el mismo patrón
template-driven que ya usa el formulario de 900 líneas (research.md D-R4). Dos migraciones nuevas
(`063c` aditiva, `063d` destructiva) se apilan sobre las dos ya aplicadas, siguiendo el mismo
patrón aditivo-luego-destructivo, en vez de reescribir `063a`/`063b` in situ — preserva el trabajo
ya construido y probado en las ramas de feature (decisión del propietario del repositorio,
2026-09-01, Option A).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
