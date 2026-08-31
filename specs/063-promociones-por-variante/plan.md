# Implementation Plan: Refactorización del módulo de promociones — modelo por conjunto explícito de variantes

**Branch**: `refactor/063-promociones-por-variante` | **Date**: 2026-08-31 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/063-promociones-por-variante/spec.md`

## Summary

Hoy el módulo de promociones (`../pos-backend` FastAPI + `../pos-heladeria` Angular) corre en
producción con cinco tipos (`percent`, `fixed`, `combo`, `qty_price`, `qty_price_presentation`),
alcance por **producto / categoría** (`PromotionTarget`), un desempate por **`priority`**, el
solape como **advertencia** y —desde spec 040— una entidad de catálogo `Presentation` con su
tabla hija `promotion_presentation_rules`. El motor
(`promotions/service.py::combined_discount_detailed`) orquesta tres mecanismos
(línea-por-línea + combos + paquete por presentación) y reconcilia por línea.

Esta spec **reemplaza** ese modelo por uno más simple y expresivo (spec.md §"Estado actual"):

1. **Una promoción = (tipo, valor, cantidad mínima) + un conjunto explícito de variantes.** Los
   tipos quedan en **dos**: `percent` (0 < `value` ≤ 100) y `package_price` (`value` = precio
   total de `min_qty` unidades). El conjunto es una tabla puente nueva `promotion_variants`
   (`promotion_id`, `product_variant_id`) **sin ningún atributo de precio** (FR-001, FR-002,
   FR-003, clarification 2026-08-31).

2. **Un único motor por conjunto.** Una función nueva `evaluate_variant_sets(db, promo_lines,
   now)` reúne, por promoción vigente, todas las unidades del pedido cuyas variantes pertenecen
   al conjunto; arma `total_unidades // min_qty` **grupos completos** por **consumo codicioso
   descendente de precio** (desempate: `product_variant_id` asc, luego `line_id` asc); descuenta
   solo esos grupos y reparte el descuento entre las líneas contribuyentes repartiendo el
   **importe cobrado** (FR-006, FR-007, FR-008, FR-008a, FR-009). Reemplaza `evaluate` /
   `evaluate_detailed` / `combined_discount_detailed` / `combo_discount_for_lines` /
   `presentation_package_discount_for_lines` y toda la reconciliación por línea:
   con el bloqueo de solape real de FR-014 **una variante nunca está en dos promociones vigentes
   a la vez**, así que no queda ningún conflicto que desempatar (por eso se elimina `priority`).

3. **El solape pasa de advertencia a bloqueo.** `_guard_variant_overlap` rechaza (409) crear o
   activar una promoción cuyo conjunto comparta ≥1 variante con otra en `draft`/`active`/`paused`
   **si además** sus rangos de fecha **y** conjuntos de días **y** ventanas horarias se
   intersectan (dimensión no definida = cubre todo su dominio) (FR-014, FR-014a). Reemplaza
   `find_overlaps` (`OverlapResponse`, `PromotionWithOverlaps`, campo `overlaps`).

4. **Migración predecible de lo existente** (FR-024–FR-027), en **dos revisiones Alembic** (Principio
   VI): **`063a` aditiva** (tabla puente, columnas nuevas, `ck_promotion_min_qty`, `ck_promotion_type`
   ampliado con `package_price`, y el **paso de datos**: cada `percent` materializa su conjunto de
   variantes —foto fija—; cada `combo` / `fixed` / `qty_price` / `qty_price_presentation` pasa a
   `Finalizada` con un aviso) — se aplica en el Incremento A y deja la suite existente en verde.
   **`063b` destructiva** (Incremento F): borra `promotion_targets` / `promotion_combo_items` /
   `promotion_presentation_rules` / `presentations` / `promotions.priority` /
   `product_variants.presentation_id`, `ck_promotion_qty_price_pack`, y estrecha `ck_promotion_type`
   **con escape** a `type IN ('percent','package_price') OR status = 'finished'` (las promociones
   `finished` de tipo viejo conservan su `type` histórico) — se aplica solo cuando ningún módulo
   referencia ya esas estructuras (hasta la Phase 8 `menu/router.py` importa `Presentation`). La entidad `Presentation`
   y todo lo suyo (tabla, `product_variants.presentation_id` —también en `api/v1/catalog/` y en
   `pos-heladeria` `product.service.ts` / `diner.service.ts`—, módulo de administración, selector en
   el formulario de producto, tests de la spec 040) se **elimina** en el Incremento F.
   El mecanismo de selección explícita de combos (`combo_id` en carrito/orden/venta, expansión)
   se retira del código; **las columnas `combo_id` históricas no se tocan** (Principio VII).

5. **Persistencia del resultado (FR-021, resuelve A-29).** Al emitir una venta se guarda el
   **monto de descuento agregado** (ya vive en `Sale.discount` / `Invoice.discount`) más la
   **lista de promociones que lo generaron** —columna JSONB nueva `applied_promotions`
   `[{promotion_id, name, amount}]`— en `sales`, `invoices` y `customer_orders` (esta última
   además gana `discount`). El desglose **por línea de venta** queda fuera de alcance.

**Naturaleza**: refactorización de comportamiento en producción (Principios I, II, III, VI, VIII).
Ocho cambios de comportamiento observable, cada uno con su entrada en
[`registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md) (**A-58…A-65**,
[research.md](./research.md) D18). No retroactivo: ninguna `Sale` / `Invoice` ya emitida cambia
de importe ni de representación (Principio VII).

## Technical Context

**Language/Version**: Backend Python 3.12 (imagen Docker) / 3.14 (venv local `pos-backend/env`).
Frontend Angular (`../pos-heladeria`, "pos-frontend" para el negocio). Esta spec toca **ambos
repositorios** (Constitución §Alcance).

**Primary Dependencies**: FastAPI + SQLAlchemy 2.0 (sync, `Mapped`/`mapped_column`, sentencias
Core `select`), Alembic (`@for_each_tenant_schema`, head actual **`e1c455751dbc`**
—`e1c455751dbc_merge_domicilio_presentations.py`, verificado con el grafo de revisiones—),
Pydantic v2. Angular + signals, `promotion-pricing.util.ts` (port parcial del motor). **Ninguna
dependencia nueva** (Principio IX no aplica): una tabla puente, el borrado de cuatro tablas y dos
columnas (en la revisión `063b`), un tipo de promoción menos, una función de agrupación más simple
que la que ya existe y tres columnas JSONB se construyen con lo que el proyecto ya usa (`JSONB` ya lo
usan `sale_items.options`, `audit_logs.payload`).

**Storage**: PostgreSQL 16, **schema por tenant** (`{"schema": "tenant"}`). Cambios de esquema,
todos en `tenant`, **repartidos en dos revisiones** ([data-model.md](./data-model.md)):

*Revisión `063a` (aditiva, Incremento A):*
- `promotion_variants` — **tabla nueva** (hija de `promotions`; `promotion_id` FK CASCADE,
  `product_variant_id` FK CASCADE, `UNIQUE(promotion_id, product_variant_id)`).
- `promotions` — agrega `CHECK min_qty >= 1`; agrega `closed_by_refactor_at TIMESTAMP NULL` (marca
  de "finalizada por la migración", FR-025); `ck_promotion_type` **ampliado** a
  `IN ('percent','fixed','combo','qty_price','qty_price_presentation','package_price')`.
- `sales`, `invoices` — **columna nueva** `applied_promotions JSONB NOT NULL DEFAULT '[]'`.
- `customer_orders` — **columnas nuevas** `discount NUMERIC(12,2) NOT NULL DEFAULT 0` +
  `applied_promotions JSONB NOT NULL DEFAULT '[]'`.
- **Paso de datos** (con las tablas viejas aún presentes).

*Revisión `063b` (destructiva, Incremento F):*
- `promotions` — **borra** `priority`; `ck_promotion_type` **estrechado con escape** a
  `type IN ('percent','package_price') OR status = 'finished'`; **borra** `ck_promotion_qty_price_pack`.
- `product_variants` — **borra** `presentation_id` (+ su FK + su índice).
- **Se borran** `promotion_targets`, `promotion_combo_items`, `promotion_presentation_rules`,
  `presentations`.
- **Columnas históricas que NO se tocan** (Principio VII): `sales.promotion_id`,
  `sale_items.combo_id`, `order_items.combo_id` / `.promotion_id`, `cart_items.combo_id` /
  `.promotion_id` — quedan nullable, dejan de escribirse, siguen legibles.
- **Migración de datos** (en `063a`): `percent` → filas en `promotion_variants` (foto fija, FR-026);
  `combo`/`fixed`/`qty_price`/`qty_price_presentation` no terminales → `status='finished'`,
  `closed_by_refactor_at=now()` (FR-025).
- **Integración de `Presentation` fuera de `api/v1/presentations/`** (revierte lo que la spec 040
  añadió, se retira en el Incremento F): `api/v1/catalog/schemas.py` (`VariantCreate` / `VariantUpdate`
  / `VariantResponse` con `presentation_id`) y `api/v1/catalog/router.py` (`_resolve_presentation_id`,
  import de `Presentation`); en `pos-heladeria`, `modules/products/services/product.service.ts` y
  `modules/tables/services/diner.service.ts`.

**Testing**: `unittest` vía `python -m unittest` (sin pytest, sin `conftest.py`). Characterization
tests sobre SQLite en memoria (SQLite **no** valida el ancho de `VARCHAR` ni ejecuta la migración
`@for_each_tenant_schema` — la migración se prueba contra PostgreSQL real, `quickstart.md` Paso 1).
Script de CI `app/scripts/test_promotions_rules.py` (único de promociones en CI): **reescritura
completa** — hoy ejercita `priority`, `_matching_target` (targets), `_line_discount` de
`qty_price` y `fixed`, todo eliminado. Los tests `"CONGELA comportamiento actual:"` y
`"CONGELA comportamiento corregido:"` afectados se reescriben con casos equivalentes en el modelo
nuevo, citando esta spec (FR-028, [contracts/migracion.md](./contracts/migracion.md) §"Inventario
de tests"). Los tests de la spec 040 (`test_promotions_presentation_pricing.py`,
`test_promotions_presentation_rules.py`, `test_presentations_service.py`, `presentation_fixtures.py`)
se **eliminan** (no llevan prefijo CONGELA).

**Target Platform**: Linux server (`../pos-backend` en producción) + SPA Angular
(`../pos-heladeria` en producción).

**Project Type**: Web application (API FastAPI + frontend Angular), dos repos independientes.

**Performance Goals**: Sin objetivo nuevo. `evaluate_variant_sets` corre una vez por cálculo de
cobro/preview sobre las líneas de un pedido (decenas, no miles) y es **más barato** que el motor
actual: una consulta por `promotion_variants` + `active_variant_set_promotions` (índice
`ix_promotions_status_ends_at` ya existente), sin la reconciliación de punto fijo de spec 040
(research.md D6, hoy `combined_discount_detailed` puede iterar |pool| veces).

**Constraints**:
- **FR-020 (no congelar el descuento)**: cada cobro parte del estado actual del pedido y de las
  promociones vigentes; ninguna tabla nueva guarda montos calculados **antes** de emitir. La
  persistencia de FR-021 es un **registro del resultado ya cobrado**.
- **FR-008 / FR-008a / SC-005 (reparto determinista, cuadra al peso)**: consumo codicioso
  descendente de precio; desempate `product_variant_id` asc y luego `line_id` asc; el residuo del
  redondeo y las unidades sueltas se asignan a la variante de identificador **más alto**
  (desempate: `line_id` más alto). Nunca depende del orden de las líneas del pedido.
- **FR-009 (nunca encarece, nunca negativo)**: el descuento de un grupo se topa en el precio
  normal de sus unidades; un grupo no se aplica si deja la línea con total mayor que sin promo.
- **FR-014 / FR-014a (bloqueo de solape real)**: variante compartida **y** intersección
  simultánea en fecha, días y horas; dimensión no definida = cubre todo su dominio.
- **FR-003 (alcance solo por lista de variantes)**: el conjunto se resuelve SIEMPRE por las filas
  de `promotion_variants`; los filtros por producto/categoría/texto del formulario **solo pueblan
  el selector**, nunca se guardan (FR-004). Ninguna variante creada después entra sola.
- **Principio VII (datos históricos)**: ninguna `Sale`/`Invoice`/`Payment` ya emitida se
  recalcula; las columnas `combo_id` históricas y las líneas marcadas con combo no se alteran
  (FR-024); `063a` solo agrega estructura y cambia `status` de promociones no emitidas, `063b`
  solo borra estructura ya sin uso — ningún importe emitido cambia en ninguna de las dos.
- **Compatibilidad hacia atrás**: `applied_promotions` nace `'[]'` para todo lo existente;
  `customer_orders.discount` nace `0`; sin recálculo (no retroactivo, FR-021).

**Scale/Scope**: 1 tabla nueva + 4 tablas borradas + 2 columnas borradas + 3 columnas JSONB +
1 columna marca + `CHECK`s ajustados, en **2 migraciones `@for_each_tenant_schema`**: `063a` aditiva
(con paso de datos) en el Incremento A, `063b` destructiva en el Incremento F.
5 tipos de promoción → 2. 1 función de motor nueva que reemplaza ~6 funciones y ~180 líneas de
`promotions/service.py`. Rewire de los **~9 call-sites** de cobro/preview
(`checkout.auto_discount`, `sales/service.py`, `table_sessions/service.py` ×3,
`tables_advanced.py`, `cart/service.py` ×2, `menu/router.py`). Borrado del paquete
`api/v1/presentations/`, de la integración de `Presentation` en `api/v1/catalog/`, y del módulo
Angular `modules/presentations/` (+ `presentation_id` en `product.service.ts` / `diner.service.ts`).
Reescritura del formulario de promociones (`promotions-page.component.ts`, ~3.200 líneas) y de
`promotion-pricing.util.ts`. Cero cambios retroactivos a ventas/facturas emitidas.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` aprobado: 6 historias priorizadas, 28 FR, 8 SC, checklist `requirements.md` al 100%, sesión de clarificación 2026-08-31 (24 preguntas resueltas). Este plan es posterior. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | Refactorización de comportamiento en producción con **8 cambios observables**, cada uno listado en spec.md §"Cambios de comportamiento respecto de producción" y registrado en `registro-de-anomalias.md` como **A-58** (elimina `priority`), **A-59** (solape advertencia→bloqueo), **A-60** (elimina alcance por categoría y su "captura futura"), **A-61** (elimina `combo` y su selección explícita; `combo` vigentes→`Finalizada`), **A-62** (elimina `qty_price`/`qty_price_presentation`/`fixed`; vigentes→`Finalizada`; solo `percent` migra), **A-63** (elimina la entidad `Presentation`; revierte la parte de modelo de datos de spec 040), **A-64** (persiste `applied_promotions` en `sales`/`invoices`/`customer_orders`; resuelve **A-29**), **A-65** (deroga "precio uniforme por presentación"). Decisión de negocio: propietario del repositorio (leonardogomez306@gmail.com), 2026-08-31, en las Clarifications de `spec.md` (research.md D18). No retroactivo (Principio VII). | PASS (con A-58…A-65) |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | El cambio de modelo obliga a reescribir los tests `CONGELA` que congelaban `priority`, alcance por categoría, `combo`, `qty_price` y `fixed`; **cada reescritura cita esta spec y la decisión (A-58…A-65)** en el mismo commit (Principio III, FR-028). El **comportamiento congelado que sigue vigente** (una sola promoción por línea, descuento tope al precio normal, remanente a precio normal, vigencia en hora local, cruce de medianoche + A-57) se **re-congela** con casos equivalentes. Inventario test por test en [contracts/migracion.md](./contracts/migracion.md) §"Inventario de tests" (amplía la tabla de spec.md con los tests de `test_orders_tables_advanced.py`, `test_table_sessions_service.py`, `test_promotions_router.py`, `test_menu_router.py` y `app/scripts/test_promotions_rules.py` que la spec pedía completar aquí). Los tests de la spec 040 **se eliminan** (sin prefijo CONGELA). | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El modelo por conjunto de variantes, el consumo codicioso, el bloqueo de solape y la persistencia son comportamiento nuevo definido por `spec.md` (FR-001–FR-021). El criterio de éxito es conformidad con esos FR/SC + ausencia de regresión no autorizada en los ~9 caminos de cobro, no equivalencia con el pasado. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | Todo lo que se toca deriva de un FR. **No** se migra el resto de deuda de spec 012 (`evaluate` legacy en otros módulos) más allá de lo que exige reemplazar el motor. **No** se reescriben los caminos de cobro más allá del punto de enganche (`auto_discount`). El borrado de `combo`/`Presentation`/`priority` **no es** oportunista: es el núcleo del refactor (spec.md §"Cambios de comportamiento"). | PASS |
| **VI. Evolución Incremental** | Entrega en **6 incrementos** verificables por separado (research.md D17): (A) migración **aditiva `063a`** + modelo con `PromotionVariant` — cierra con la suite existente en verde y el motor viejo intacto; (B) CRUD + validaciones (FR-014/FR-016/FR-018); (C) motor `evaluate_variant_sets` + rewire de call-sites + persistencia (FR-021); (D) menú QR + preview de terminal (FR-022/FR-023); (E) frontend de administración; (F) **retiro de estructura legada** (revisión destructiva **`063b`** + borrado de `Presentation`/`priority`/targets/combos del ORM y del catálogo) + reescritura de tests CONGELA + script de CI + borrado de los de spec 040. Ningún incremento mezcla la **migración de datos** (solo A, `063a`) con el cambio del cálculo de cobro (solo C). El **borrado de estructura** (`063b`) se difiere a F —no por ser un cambio de comportamiento, sino porque hasta la Phase 8 hay módulos que importan `Presentation`— para que A quede 100% aditivo y verificable. | PASS |
| **VII. Compatibilidad con Datos Históricos** | Ninguna `Sale`/`SaleInvoice`/`Payment` se recalcula. `applied_promotions` nace `'[]'`, sin backfill. Las columnas `combo_id` y `promotion_id` históricas y las líneas de venta marcadas con combo **no se tocan** (FR-024). La migración cambia `status` solo de promociones no terminales (que no son un hecho contable). El `downgrade` recrea estructura vacía y no altera ningún importe emitido ([data-model.md](./data-model.md) §Rollback). | PASS |
| **VIII. Evolución del Modelo de Datos** | [data-model.md](./data-model.md) especifica tabla nueva, columnas nuevas/borradas, FKs, `ondelete`, defaults, `CHECK`s (patrón "ampliar en `063a`, estrechar en `063b`"), unicidad, compatibilidad, el **paso de datos** de `063a` (percent→conjunto; resto→finished) y el `downgrade` simétrico de cada revisión, con `@for_each_tenant_schema` + guarda `_has_table`. El borrado de `presentations` / `presentation_id` está acotado a `063b` y su `downgrade` recrea la estructura de spec 040 vacía. | PASS |
| **IX. Dependencias Nuevas Permitidas con Justificación** | Ninguna dependencia nueva (Technical Context). | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de `spec.md` tiene "Independent Test"; [quickstart.md](./quickstart.md) los traduce a `unittest` ejecutables + verificación manual del menú QR, la terminal y el panel de administración. Se ejercita el consumo codicioso (FR-008), el reparto al peso (SC-005) con el caso de división no exacta, el determinismo por orden, el bloqueo de solape (SC-003), la migración (SC-006) y el anuncio del menú (SC-007). | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | Las decisiones de negocio están en `spec.md` (dos tipos, consumo codicioso favorable al cliente, bloqueo de solape real, `combo`/`fixed`/`qty_price` no se migran, `percent` sí, persistencia agregada + lista). Las decisiones técnicas —tabla puente vs. reusar `PromotionTarget`, `package_price` como valor nuevo vs. reusar `qty_price`, motor único vs. mantener la orquestación, JSONB vs. tabla puente para `applied_promotions`, marca `closed_by_refactor_at` vs. tabla de avisos— están en [research.md](./research.md), cada una con su alternativa descartada. | PASS |
| **XII. Trazabilidad** | Cadena: `spec.md` (Necesidad + contrato) → este `plan.md` / `research.md` / `data-model.md` / `contracts/` (decisión técnica) → `registro-de-anomalias.md` A-58…A-65 (decisión de negocio) → `tasks.md` (Fase 2, `/speckit-tasks`) → implementación → tests reescritos citando la decisión + tests nuevos por historia → `quickstart.md` (verificación). | PASS |
| **XIII. Todo en Español de Colombia** | Este plan y sus artefactos, los nombres de tests nuevos, los mensajes de error de negocio (conjunto vacío, porcentaje > 100, precio de paquete sin descuento, solape bloqueado, edición de promoción activa) y los mensajes de commit se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones que justificar en Complexity Tracking. Punto de atención documentado, no
violación: la spec revierte la parte de modelo de datos de la spec 040 (`Presentation`) menos de
tres meses después de mergearla — research.md D13 detalla qué de la spec 040 **se conserva**
(vigencia por día/hora, cruce de medianoche + A-57, anuncio en el menú QR) y qué se revierte
(la entidad y sus reglas), para que el refactor no arrastre trabajo útil.

## Project Structure

### Documentation (this feature)

```text
specs/063-promociones-por-variante/
├── plan.md                    # Este fichero (/speckit-plan)
├── research.md                # Fase 0 — 18 decisiones técnicas y alternativas descartadas
├── data-model.md              # Fase 1 — tabla puente, columnas, borrados, migración y rollback
├── quickstart.md              # Fase 1 — validación ejecutable por historia de usuario
├── contracts/                 # Fase 1
│   ├── motor-y-persistencia.md      # evaluate_variant_sets, reparto FR-008a, applied_promotions
│   ├── administracion-promociones.md # tipos, conjunto de variantes, FR-014, FR-016, FR-018, estados
│   ├── superficies-consumo.md       # menú QR (FR-022), terminal de staff (FR-023), carrito
│   └── migracion.md                 # paso de datos, borrado de Presentation/combo, inventario de tests
├── checklists/
│   └── requirements.md        # Ya existente, 100%
└── tasks.md                   # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorios sibling de `pos-specs`)

Rutas relativas a la raíz de cada repo. La spec vive en `pos-specs`; el código en `../pos-backend`
y `../pos-heladeria` (Constitución §Alcance).

```text
# ../pos-backend
app/
├── models/
│   ├── promotion.py                 # MODIFICADO — Incremento A (T009): clase nueva `PromotionVariant`
│   │                                   (promotion_id FK CASCADE, product_variant_id FK CASCADE,
│   │                                   UNIQUE(promotion_id, product_variant_id)); relación
│   │                                   `Promotion.variants` (cascade all, delete-orphan); columna
│   │                                   nueva `closed_by_refactor_at` (nullable); `CHECK min_qty >= 1`;
│   │                                   `PROMOTION_TYPES` y `ck_promotion_type` AMPLIADOS con
│   │                                   `package_price`. Incremento F (T061c): `ck_promotion_type` con
│   │                                   escape `OR status='finished'`; `PROMOTION_TYPES` se queda en 6
│   │                                   (lee las `finished` históricas); se BORRA `priority`, las
│   │                                   clases `PromotionTarget` / `PromotionComboItem` /
│   │                                   `PromotionPresentationRule`, las relaciones `targets` /
│   │                                   `combo_items` / `presentation_rules`, `ck_promotion_qty_price_pack`
│   │                                   y el import de `Presentation`.
│   ├── presentation.py              # BORRADO (FR-027) — Incremento F (T061c).
│   ├── product_variant.py           # MODIFICADO — Incremento F (T061c): se BORRA `presentation_id`,
│   │                                   su FK y su índice, la relación `presentation` y el import de
│   │                                   Presentation.
│   ├── sale.py                      # MODIFICADO — columna nueva `applied_promotions` (JSONB, NOT
│   │                                   NULL, server_default '[]'). `promotion_id` y
│   │                                   `sale_items.combo_id` SE CONSERVAN (histórico, Principio
│   │                                   VII); dejan de escribirse.
│   ├── invoice.py                   # MODIFICADO — columna nueva `applied_promotions` (JSONB).
│   ├── customer_order.py            # MODIFICADO — columnas nuevas `discount` (Numeric(12,2), NOT
│   │                                   NULL, server_default 0) y `applied_promotions` (JSONB).
│   ├── order_item.py / cart_item.py # SIN CAMBIO de esquema — `combo_id` / `promotion_id` quedan
│   │                                   nullable, dejan de escribirse (FR-024).
│   └── __init__.py                  # MODIFICADO — Incremento A (T015): agrega PromotionVariant.
│                                       Incremento F (T061c): quita Presentation / PromotionTarget /
│                                       PromotionComboItem / PromotionPresentationRule.
│
├── api/v1/promotions/
│   ├── schemas.py                   # MODIFICADO — `PromotionType` de ENTRADA = {PERCENT,
│   │                                   PACKAGE_PRICE} (`PromotionResponse.type` se queda `str` para
│   │                                   serializar promociones finished de tipo viejo); se
│   │                                   BORRA COMBO/QTY_PRICE/QTY_PRICE_PRESENTATION de la entrada,
│   │                                   TargetIn, ComboItemIn, PresentationRuleIn, `priority`,
│   │                                   `targets`, `combo_items`, `presentation_rules`, los flags
│   │                                   confirm_*, OverlapResponse, PromotionWithOverlaps y sus
│   │                                   validadores. PromotionCreate/PromotionShapeUpdate ganan
│   │                                   `variant_ids: list[UUID]` (>= 1). PromotionResponse expone
│   │                                   `variants: list[{product_variant_id, description,
│   │                                   unit_price}]` (FR-005). `_PromotionRules._percent_range`
│   │                                   se conserva; validador nuevo para package_price
│   │                                   (`value > 0`, `min_qty >= 1`). Cuerpo del 409 de FR-014
│   │                                   (`OverlapConflict`: promoción + variantes compartidas).
│   ├── service.py                   # REESCRITURA MAYOR —
│   │                                     · BORRA: AUTO_TYPES, _matching_target, _pack_terms,
│   │                                       _line_discount, _best_line_match, best_line_discount,
│   │                                       evaluate, evaluate_detailed, LineDiscount,
│   │                                       PromotionResult, get_active_combo, expand_combo,
│   │                                       combo_discount_for_lines, ComboComponent,
│   │                                       PresentationDiscountResult, CombinedDiscountResult,
│   │                                       active_presentation_promotions,
│   │                                       _presentation_reference_unit_price, _unit_sort_key,
│   │                                       _rule_discount_by_line,
│   │                                       presentation_package_discount_for_lines,
│   │                                       _line_by_line_discounts, combined_discount_detailed,
│   │                                       _ranges_overlap/_csv_overlap/_times_overlap/
│   │                                       _scope_overlap/find_overlaps, _apply_targets,
│   │                                       _apply_combo_items, _apply_presentation_rules,
│   │                                       presentation_overlap_conflicts,
│   │                                       _guard_presentation_overlap,
│   │                                       _active_variants_for_presentation,
│   │                                       _check_presentation_rule_prices,
│   │                                       _validate_shape_presentation_rules,
│   │                                       _validate_presentation_ids, QTY_PRICE_PRESENTATION.
│   │                                     · CONSERVA sin cambio de cuerpo: `_tz`, `local_now`,
│   │                                       `_in_time_window`, `_valid_now` (con A-57 intacto).
│   │                                     · NUEVO:
│   │                                       `active_variant_set_promotions(db, now)` (hermana de
│   │                                         active_discount_promotions: status active + ends_at
│   │                                         + _valid_now, selectinload de `variants`);
│   │                                       `_greedy_units(eligible_lines, min_qty)` (consumo
│   │                                         codicioso descendente, FR-008);
│   │                                       `_distribute_group_discount(group_units, discount)`
│   │                                         (reparto por importe cobrado, residuo a la variante
│   │                                         más alta, FR-008a);
│   │                                       `evaluate_variant_sets(db, promo_lines, now) ->
│   │                                         SetDiscountResult(total, by_line,
│   │                                         applied: list[AppliedPromotion])`;
│   │                                       `variant_set_condition_text(promo)` (lenguaje llano,
│   │                                         FR-022/FR-023);
│   │                                       `_guard_variant_overlap(db, promo, variant_ids)`
│   │                                         (FR-014/FR-014a: variante compartida + intersección
│   │                                         fecha ∧ días ∧ horas, dimensión abierta = todo);
│   │                                       `_guard_package_is_discount(db, promo)` (FR-016:
│   │                                         value >= min_qty × precio de la variante más barata
│   │                                         del conjunto → 409);
│   │                                       `_apply_variant_set(db, promo, variant_ids)`.
│   │                                     · CRUD: create/update/update_shape/change_status/
│   │                                       duplicate pierden targets/combo/presentation/priority;
│   │                                       update_shape valida conjunto no vacío + FR-016 +
│   │                                       FR-014; change_status→active revalida FR-014 y FR-016
│   │                                       (una promo creada en draft sin conflicto puede
│   │                                       chocar al activar); update (escalares en active/
│   │                                       paused) permite name/description/ends_at/days_of_week/
│   │                                       start_time/end_time y BLOQUEA value/min_qty (FR-018)
│   │                                       — `variant_ids` y `type` solo por `shape` en draft.
│   ├── router.py                    # MODIFICADO — respuestas sin `overlaps` (se BORRA
│   │                                   `_with_overlaps`, `PromotionWithOverlaps`); `_serialize`
│   │                                   resuelve `variants` con descripción + precio normal
│   │                                   vigente (FR-005). 409 legible para FR-014 y FR-016. Nuevos
│   │                                   query params de `list_promotions`: `closed_by_refactor`
│   │                                   (para el aviso de FR-025). `list_query` ordena por
│   │                                   `name` (ya no por `priority`).
│
├── api/v1/presentations/            # BORRADO — paquete completo (router, schemas, service) (FR-027);
│                                       en el Incremento F (T061d), tras liberar sus últimos
│                                       consumidores.
│
├── api/v1/catalog/
│   ├── schemas.py                   # MODIFICADO — `VariantCreate` / `VariantUpdate` /
│   │                                   `VariantResponse` pierden `presentation_id` (revierte spec 040)
│   │                                   (FR-027, Incremento F / T061d).
│   └── router.py                    # MODIFICADO — se BORRA el import de `Presentation`, la función
│                                       `_resolve_presentation_id` y sus llamadas en `create_variant`
│                                       / `update_variant` (FR-027, Incremento F / T061d).
│
├── api/v1/orders/
│   ├── checkout.py                  # MODIFICADO — `promo_lines_for` deja de traer
│   │                                   `product_id`/`category_id`/`presentation_id` (targets y
│   │                                   presentación eliminados), conserva
│   │                                   `product_variant_id`/`unit_price`/`line_id`/`quantity`/
│   │                                   `_variant_active` y agrega la descripción de la línea para
│   │                                   `applied_promotions`. `auto_discount` llama
│   │                                   `evaluate_variant_sets` y devuelve
│   │                                   `(total, promotion_id, applied)`; `pay_order` pasa
│   │                                   `applied` a `build_sale` y fija `order.discount` +
│   │                                   `order.applied_promotions` (FR-021). Se BORRA todo lo de
│   │                                   `combo_discount` y la reconciliación por pool.
│   ├── consolidation.py             # MODIFICADO — se retira `add_item_to_table` la rama de
│   │                                   `combo_id` (expansión de combo, FR-024).
│   └── tables_advanced.py           # MODIFICADO — `group_bill` usa el retorno nuevo de
│                                       `auto_discount`.
│
├── api/v1/table_sessions/service.py # MODIFICADO — `compute_bill` (186), `release_paid_session`
│                                       (656) y `_close_split` (753/762): retorno nuevo de
│                                       `auto_discount`; `_close_unified`/`_close_split` fijan
│                                       `applied_promotions` en `Sale` y en la `CustomerOrder`.
│
├── api/v1/sales/service.py          # MODIFICADO — venta de mostrador (272): retorno nuevo de
│                                       `combined_discount_detailed` → `evaluate_variant_sets`;
│                                       `applied_promotions` a la venta.
│
├── api/v1/sales/builder.py          # MODIFICADO — `build_sale` acepta
│                                       `applied_promotions: list[dict]` y lo persiste en
│                                       `Sale.applied_promotions`.
│
├── api/v1/invoices/service.py       # MODIFICADO — `issue_for_sale` copia
│                                       `sale.applied_promotions` a `invoice.applied_promotions`
│                                       (snapshot, FR-021).
│
├── api/v1/cart/
│   ├── service.py                   # MODIFICADO — `_line_discount` (por ítem) y `serialize_cart`
│   │                                   usan el desglose `by_line` de `evaluate_variant_sets`
│   │                                   (una sola llamada) en vez de `best_line_discount` +
│   │                                   `combined_discount_detailed`; `submit_cart` toma el
│   │                                   snapshot de descuento del mismo desglose (spec 038 sigue
│   │                                   cuadrando). Se BORRA `_add_combo` y la rama `combo_id` de
│   │                                   `add_item` (FR-024). `_cart_promo_lines` pierde
│   │                                   product_id/category_id/presentation_id.
│   └── schemas.py                   # MODIFICADO — `CartItemIn` pierde `combo_id`.
│
├── api/v1/menu/
│   ├── router.py                    # MODIFICADO — `_build_menu` usa una función mínima
│   │                                   `menu_unit_discount(promos, variant_id, price)` (solo
│   │                                   `percent` con `min_qty == 1` baja el precio unitario;
│   │                                   `package_price` no, igual que hoy `qty_price`).
│   │                                   `_build_menu_promotions` se adapta: el texto sale del
│   │                                   **conjunto de variantes** (`variant_set_condition_text`),
│   │                                   ya no de reglas de presentación (FR-022). `_build_menu`
│   │                                   sigue devolviendo `list[MenuCategoryResponse]` (CONGELA
│   │                                   de `test_menu_router.py`).
│   └── schemas.py                   # MODIFICADO — `MenuPromotionRule` describe el conjunto (p.
│                                       ej. "8 variantes"), ya no una presentación.
│
├── alembic/versions/
│   ├── XXXX_063a_promociones_por_conjunto_aditivo.py   # NUEVO (Incremento A) — down_revision
│   │                                   "e1c455751dbc". @for_each_tenant_schema + guarda _has_table:
│   │                                   crea promotion_variants; agrega closed_by_refactor_at,
│   │                                   sales/invoices.applied_promotions,
│   │                                   customer_orders.discount / .applied_promotions +
│   │                                   ck_customer_order_discount_non_negative; ck_promotion_min_qty;
│   │                                   ck_promotion_type AMPLIADO con 'package_price'; PASO DE DATOS
│   │                                   (percent → filas; combo/fixed/qty_price/
│   │                                   qty_price_presentation → finished). downgrade: drop de lo
│   │                                   aditivo (las tablas viejas siguen intactas).
│   └── YYYY_063b_promociones_retiro_estructura_legada.py  # NUEVO (Incremento F) — down_revision
│                                       "XXXX_063a". Borra promotion_targets / promotion_combo_items /
│                                       promotion_presentation_rules / presentations; borra
│                                       product_variants.presentation_id (+FK+índice) y
│                                       promotions.priority; drop ck_promotion_qty_price_pack;
│                                       ck_promotion_type ESTRECHADO con escape a
│                                       type IN ('percent','package_price') OR status='finished'.
│                                       downgrade simétrico (recrea estructura de spec 013/040 vacía).
│
├── scripts/
│   └── test_promotions_rules.py     # REESCRITURA COMPLETA (CI) — se van las secciones de
│                                       `priority`, `_matching_target`, `_line_discount` de
│                                       qty_price/fixed; entran: consumo codicioso descendente
│                                       (FR-008), grupos completos + remanente (FR-007), reparto
│                                       al peso con residuo (FR-008a/SC-005), tope al precio
│                                       normal (FR-009), vigencia local + cruce de medianoche +
│                                       A-57 (se conservan), `type` admite {percent,
│                                       package_price}.
│
└── characterization_tests/
    ├── test_promotions_service.py            # NUEVO — motor por conjunto (US2): los 10
    │                                            Acceptance Scenarios + edge cases + SC-005.
    ├── test_promotions_rules_admin.py        # NUEVO — CRUD (US1/US5): conjunto vacío, % > 100,
    │                                            FR-016, FR-014 (crear y activar), FR-018,
    │                                            duplicar, máquina de estados, permisos.
    ├── test_promotions_migration.py          # NUEVO — US6: percent→conjunto (foto fija),
    │                                            combo/fixed/qty_price/qty_price_presentation→
    │                                            Finalizada + aviso, Sale/Invoice previas intactas.
    ├── test_promotions_router.py             # MODIFICADO — sin `overlaps`, sin `priority`, enum
    │                                            reducido; `X-Server-Time` intacto; cita esta spec.
    ├── test_menu_router.py                   # MODIFICADO — A-08 re-congelado con conjunto de
    │                                            variantes; clase US5 reescrita (anuncio por
    │                                            conjunto). `_build_menu` no cambia de firma.
    ├── test_orders_checkout.py               # MODIFICADO — `test_promo_lines_for_*`,
    │                                            `test_pay_order_construye_sale_real_*`,
    │                                            `test_pay_order_dos_combos_*` reescritos al
    │                                            modelo nuevo (FR-021, sin combos); los casos
    │                                            nuevos de spec 040 (pool, FR-023) se sustituyen
    │                                            por los del motor por conjunto.
    ├── test_cart_service.py                  # MODIFICADO — `test_add_item_combo` y
    │                                            `test_serialize_cart_combo_*` se van (FR-024);
    │                                            `test_serialize_cart_discounted_total_con_promocion_activa`
    │                                            y `_sin_promocion` re-congelados con conjunto;
    │                                            `test_submit_cart_snapshot_*` (spec 038) cuadra
    │                                            con el motor nuevo.
    ├── test_orders_consolidation.py          # MODIFICADO — `test_add_item_to_table_combo_*` se
    │                                            va (FR-024).
    ├── test_table_sessions_service.py        # MODIFICADO — `test_close_session_unified_a29_*`
    │                                            reescrito: A-29 se resuelve por
    │                                            `applied_promotions` (FR-021), sin combos.
    ├── test_orders_tables_advanced.py        # MODIFICADO — `test_group_bill_aplica_promocion_
    │                                            percent_*` re-congelado con conjunto;
    │                                            `test_group_bill_aplica_combo_vigente_*` se va.
    ├── test_presentations_service.py         # BORRADO (spec 040, sin prefijo CONGELA).
    ├── test_promotions_presentation_pricing.py  # BORRADO (spec 040).
    ├── test_promotions_presentation_rules.py    # BORRADO (spec 040).
    ├── presentation_fixtures.py              # BORRADO (spec 040).
    ├── cart_fixtures.py / orders_fixtures.py /
    │   table_sessions_fixtures.py            # MODIFICADO — entra `add_variant_to_promotion` y
    │                                            `make_promotion` gana `type` (T017, Incremento A);
    │                                            `make_promotion_target` / `make_combo_item` y la
    │                                            limpieza de `priority` se van en T062 (Incremento F),
    │                                            ya sin uso.
    └── ...

# ../pos-heladeria
src/app/
├── modules/promotions/
│   ├── interfaces/promotion.interface.ts    # MODIFICADO — PromotionType = 'percent' |
│   │                                            'package_price'; se van PromotionTarget,
│   │                                            ComboItem, PresentationRule*, PromotionOverlap,
│   │                                            PromotionWithOverlaps, AUTO_TYPES, `priority`,
│   │                                            PROMOTION_TRANSITIONS intacto. Entra
│   │                                            `variant_ids: string[]` y
│   │                                            `PromotionVariant { product_variant_id,
│   │                                            description, unit_price }`. Cuerpo del 409 de
│   │                                            FR-014 (`OverlapConflictError`).
│   ├── services/promotion.service.ts        # MODIFICADO — payloads sin targets/combo/
│   │                                            presentation/priority/confirm_*; con
│   │                                            `variant_ids`. Método para el aviso de FR-025
│   │                                            (`?closed_by_refactor=true`).
│   ├── services/promotion-pricing.util.ts   # MODIFICADO — `getPromoDisplay` para 2 tipos;
│   │                                            `percent` con `min_qty === 1` previsualizable;
│   │                                            `package_price` fuera de la previsualización de
│   │                                            precio unitario (como `qty_price` hoy). Se van
│   │                                            las ramas de combo/qty_price/presentation.
│   ├── pages/promotions-page.component.ts    # REESCRITURA MAYOR — un solo formulario (Descuento
│   │                                            % / Precio de paquete); selector de conjunto de
│   │                                            variantes con filtros por producto, categoría y
│   │                                            texto **solo para poblar** (FR-004); resumen
│   │                                            legible antes de guardar (tipo, condición, lista
│   │                                            de variantes con precio normal, FR-005); diálogo
│   │                                            del 409 de FR-014 (nombra promoción + variantes);
│   │                                            mensaje del 409 de FR-016; edición de una
│   │                                            promoción activa limitada a nombre/descr./fin de
│   │                                            vigencia/días/horas (FR-018); banner (una vez,
│   │                                            descartable) con las promociones finalizadas por
│   │                                            la migración (FR-025). Se van: selector de tipo
│   │                                            combo/qty_price/qty_price_presentation, el
│   │                                            formulario de combo, el de precio por target, el
│   │                                            de reglas por presentación, el input de
│   │                                            `priority`, el panel de `overlaps`.
│   └── components/scope-picker.component.ts  # BORRADO — lo reemplaza el selector de conjunto de
│                                                variantes (targets eliminados).
│
├── modules/presentations/                   # BORRADO — módulo completo (lista + form + service
│                                                + interface) (FR-027).
│
├── modules/products/pages/product-form.component.ts  # MODIFICADO — se BORRA el selector de
│                                                        presentación por variante (FR-027).
├── modules/products/interfaces/product.interface.ts  # MODIFICADO — `ProductVariant` / `VariantForm`
│                                                        / `VariantCreatePayload` / `VariantUpdatePayload`
│                                                        / el draft pierden `presentation_id` /
│                                                        `presentationId` (FR-027).
├── modules/products/services/product.service.ts     # MODIFICADO — quita `presentation_id` de
│                                                        `VariantResponse` (raw), `VariantCreatePayload`,
│                                                        `VariantUpdatePayload`, `VariantForm` y de
│                                                        `createVariant` / `updateVariant` / los mapeos
│                                                        de `variantDrafts` (FR-027).
│
├── modules/tables/
│   ├── components/combo-select.component.ts  # BORRADO (FR-024).
│   ├── components/product-select.component.ts / pos-catalog-drawer.component.ts  # MODIFICADO —
│   │                                            se retira el flujo de "agregar combo".
│   ├── services/diner.service.ts             # MODIFICADO — `MenuPromotionAnnouncement.rules` pasa de
│   │                                            `{presentation_name, pack_price, min_qty, text}` a
│   │                                            `{text, variant_count, min_qty, value}`; el parser
│   │                                            de `promotions[].rules[]` deja de leer
│   │                                            `presentation_name` / `pack_price` (FR-022).
│   ├── pages/public-menu.component.ts        # MODIFICADO — el banner de anuncio usa el texto por
│   │                                            conjunto de variantes (FR-022).
│   └── services/pos-terminal.store.ts        # MODIFICADO — para cada variante de un conjunto de
│                                                promoción vigente, muestra SIEMPRE la condición
│                                                en lenguaje llano; muestra el descuento efectivo
│                                                cuando el pedido en curso alcanza `min_qty`
│                                                unidades elegibles (FR-023), calculado con el
│                                                preview del cobro (no localmente).
│
├── core/config/navigation.config.ts         # MODIFICADO — quita la entrada "Presentaciones".
├── modules/dashboard/routes.ts              # MODIFICADO — quita la ruta /dashboard/presentations.
└── core/services/menu.service.ts            # MODIFICADO — tipado del campo `promotions`.
```

**Structure Decision**: el refactor se apoya en **una** tabla puente (`promotion_variants`, sin
atributos de precio — clarification 2026-08-31) y **un** motor (`evaluate_variant_sets`), y en el
**borrado** de todo lo que el modelo nuevo deja sin sentido: `PromotionTarget`,
`PromotionComboItem`, `PromotionPresentationRule`, `Presentation`, `priority`, `find_overlaps` y la
orquestación de tres mecanismos. Ese borrado (código + revisión `063b`) se ejecuta en el
**Incremento F**, después de que todos sus consumidores dejen de referenciarlo (hasta la Phase 8
`menu/router.py` importa `Presentation`); el **Incremento A** es 100% aditivo (revisión `063a`). El motor único es viable porque el bloqueo de solape real de
FR-014 garantiza que **una variante nunca está en dos promociones vigentes al mismo instante**,
así que no hay conflicto que resolver (por eso `priority` desaparece, no se reemplaza). El
frontend colapsa dos módulos y un formulario multi-tipo en un formulario de dos tipos con un
selector de conjunto de variantes. La persistencia de FR-021 se resuelve con una columna JSONB de
snapshot (`applied_promotions`) en las tres entidades, el patrón que `sale_items.options` ya usa,
en vez de tres tablas puente nuevas (research.md D14).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
