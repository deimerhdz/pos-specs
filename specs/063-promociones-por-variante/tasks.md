---

description: "Task list para la refactorización del módulo de promociones — modelo por conjunto explícito de variantes"
---

# Tasks: Refactorización del módulo de promociones — modelo por conjunto explícito de variantes

**Input**: Design documents en `/specs/063-promociones-por-variante/`

**Prerequisites**: [plan.md](./plan.md) (requerido), [spec.md](./spec.md) (requerido para historias de
usuario), [research.md](./research.md), [data-model.md](./data-model.md),
[contracts/motor-y-persistencia.md](./contracts/motor-y-persistencia.md),
[contracts/administracion-promociones.md](./contracts/administracion-promociones.md),
[contracts/superficies-consumo.md](./contracts/superficies-consumo.md),
[contracts/migracion.md](./contracts/migracion.md), [quickstart.md](./quickstart.md)

**Tests**: **incluidos y NO opcionales para esta spec.** `plan.md` §Constitution Check (Principios III
y X), `quickstart.md` y [contracts/migracion.md](./contracts/migracion.md) §2-3 fijan de antemano qué
test crea, reescribe o elimina cada historia. Los tests nuevos por historia y las reescrituras de los
`CONGELA` afectados aparecen abajo como tareas de implementación; **cada reescritura cita esta spec y
la decisión A-58…A-65 en el mismo commit** (Principio III, FR-028).

**Organization**: tareas agrupadas por historia de usuario (US1–US6, prioridades de `spec.md`) para
que cada una sea implementable y verificable de forma independiente. Esta spec toca **dos
repositorios sibling** de `pos-specs` (Constitución §Alcance, plan.md §Project Structure). El orden
de fases sigue los **6 incrementos** de [research.md](./research.md) D17 (A→F).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (ficheros distintos, sin dependencia de una tarea sin terminar)
- **[Story]**: historia de usuario a la que pertenece (US1–US6); Setup / Foundational / Polish sin etiqueta
- Cada tarea incluye la ruta de fichero exacta, con el prefijo del repo sibling que corresponda

## Path Conventions

- **Backend**: `pos-backend/app/...` (repo sibling `../pos-backend`)
- **Frontend**: `pos-heladeria/src/app/...` (repo sibling `../pos-heladeria`)
- **Documentación de gobierno**: `pos-specs/specs/...`

---

## Phase 1: Setup

**Purpose**: preparar ramas, confirmar la línea base y los prerrequisitos de Principio II antes de
tocar código (quickstart.md Paso 0 y §"Prerrequisito de Principio II").

- [X] T001 [P] Crear la rama `refactor/063-promociones-por-variante` en `pos-backend` y en
  `pos-heladeria` partiendo de la rama base de cada repo (confirmar la base con el propietario antes
  de ramificar; `pos-specs` ya está en esa rama). — Ambas ramas creadas desde `develop` (confirmado
  por el propietario 2026-08-31).

- [X] T002 Confirmar la línea base **en verde** en `../pos-backend` (quickstart.md Paso 0), sin tocar
  código todavía: — **514 tests OK** (`unittest discover`), CI script `test_promotions_rules` en
  verde. Línea base de aditividad-segura: con cero promociones `active` todos los totales de cobro
  son los de la suite existente (referencia para T064).
  `python -m unittest app.characterization_tests.test_promotions_router -v`,
  `python -m unittest app.characterization_tests.test_menu_router -v`,
  `python -m unittest app.characterization_tests.test_orders_checkout -v`,
  `python -m unittest app.characterization_tests.test_cart_service -v`,
  `python -m unittest app.characterization_tests.test_table_sessions_service -v`,
  `python -m unittest app.characterization_tests.test_orders_tables_advanced -v`,
  `python -m unittest app.characterization_tests.test_orders_consolidation -v`,
  `python -m unittest app.characterization_tests.test_promotions_presentation_pricing -v`,
  `python app/scripts/test_promotions_rules.py`. **Guardar los totales de cobro de referencia** para
  el chequeo de aditividad-segura de T064 (research.md D6, contracts/migracion.md §4).

- [X] T003 Verificar el head real de Alembic en `../pos-backend` con `alembic heads` (plan.md asume
  **`e1c455751dbc`**, `e1c455751dbc_merge_domicilio_presentations.py`). Confirmar que hay **una sola**
  cabeza (sin bifurcaciones). Si el head difiere, usar el valor real como `down_revision` de la
  migración de T005. — `alembic heads` → `e1c455751dbc (head)`, una sola cabeza. Coincide con plan.md.

- [X] T004 Confirmar que las 8 entradas **A-58 … A-65** están registradas en
  `pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` (detalle en research.md D18) y que
  **A-29** quedó marcada como resuelta por la spec 063 (vía A-64). Es **prerrequisito del Incremento
  A**: sin esto ningún commit puede citar la decisión que lo autoriza (Principio II/III, FR-028). —
  A-58…A-65 presentes (líneas 1631-1818); A-29 marcada RESUELTA en el índice, en su entrada y en la
  tabla de `memoria-historica.md`.

**Checkpoint**: ramas listas, línea base fijada, head de Alembic conocido, decisiones de negocio
registradas.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: **Incremento A** (research.md D17), **100% aditivo** — la tabla puente
`promotion_variants`, las columnas `applied_promotions` + `customer_orders.discount` + marca
`closed_by_refactor_at`, el `CHECK` `ck_promotion_min_qty`, la **ampliación** de `ck_promotion_type`
para admitir `package_price` (sin quitar los valores viejos todavía), el **paso de datos** de la
migración (revisión `063a`) y las fixtures de test compartidas. El modelo gana `PromotionVariant`
y los dos tipos nuevos **sin** perder nada del modelo viejo.

**⚠️ El retiro de la estructura legada se difiere al Incremento F** (Principio VI): el borrado de
`PromotionTarget` / `PromotionComboItem` / `PromotionPresentationRule` / `Presentation` / `priority` /
`presentation_id`, el estrechamiento con escape de `ck_promotion_type`
(`type IN ('percent','package_price') OR status='finished'`) y la revisión **destructiva `063b`**
van en la Phase 9 (T061a–T061d), porque hasta la Phase 8 (`menu/router.py`, T055–T056) hay módulos
que importan `Presentation` / `active_presentation_promotions`. Así el Incremento A cierra con **la
suite existente en verde y el motor viejo intacto**.

**⚠️ CRITICAL**: ninguna historia de usuario arranca hasta terminar esta fase. US1–US5 necesitan
`promotion_variants` y los schemas nuevos; US2 necesita las columnas de persistencia; US6 verifica el
paso de datos que se escribe aquí (T006).

- [X] T005 Crear la revisión Alembic **aditiva** `063a`
  `pos-backend/alembic/versions/387ef3e638cd_063a_promociones_por_conjunto_aditivo.py`
  (`down_revision = e1c455751dbc`). `promotion_variants`, `closed_by_refactor_at`,
  `sales/invoices.applied_promotions`, `customer_orders.discount + .applied_promotions +
  ck_customer_order_discount_non_negative`, `ck_promotion_min_qty`, `ck_promotion_type` ampliado con
  `package_price`. `upgrade`/`downgrade -1`/`upgrade` limpio contra Postgres real. (`down_revision` =
  head verificado en T003). `upgrade` y `downgrade` decorados con `@for_each_tenant_schema` + guarda
  `_has_table(schema, "promotions")` (patrón de `d3e4f5a6b7c8_promotions.py` / `f03274730367`),
  nombres de constraint con `op.f(...)`. **Solo estructura nueva y compatible hacia atrás**
  (data-model.md §Migración `063a`): `create_table("promotion_variants", ...)` — `id`, `promotion_id`
  (FK `promotions.id` `ondelete="CASCADE"`, indexado), `product_variant_id`
  (FK `product_variants.id` `ondelete="CASCADE"`, indexado),
  `UniqueConstraint("promotion_id", "product_variant_id")`; `add_column("promotions",
  "closed_by_refactor_at" TIMESTAMP NULL)`; `add_column("sales", "applied_promotions" JSONB NOT NULL
  server_default "'[]'")`; ídem `invoices`; `add_column("customer_orders", "discount" NUMERIC(12,2)
  NOT NULL server_default "0")` + `applied_promotions` JSONB + `create_check_constraint(
  "ck_customer_order_discount_non_negative", "discount >= 0")`;
  `create_check_constraint("ck_promotion_min_qty", "min_qty >= 1")`; **reemplazar** `ck_promotion_type`
  por `type IN ('percent','fixed','combo','qty_price','qty_price_presentation','package_price')`
  (se **amplía** con `package_price`, NO se quitan los viejos — eso es `063b`, T061a). La revisión
  **destructiva `063b`** (borrado de tablas/columnas; `ck_promotion_type` estrechado **con escape**
  `OR status='finished'`; drop de `ck_promotion_qty_price_pack`) se crea y aplica en la **Phase 9**
  (T061a–T061b).

- [X] T006 `migrate_promotions_data(bind, schema=None)` en el módulo de `063a` — función pura con
  Core tipado (`UUID(as_uuid=True)`), corre en Postgres y en SQLite (T050). (a) `percent` → filas en
  `promotion_variants` (targets de producto/categoría, o todas las activas si global); (b) `combo`/
  `fixed`/`qty_price`/`qty_price_presentation` no terminales → `finished` + `closed_by_refactor_at`.
  Invocada al final de `upgrade`.
  En la misma revisión `063a` de T005, el **PASO DE DATOS** (data-model.md §Migración `063a`,
  research.md D12) — se ejecuta **con `promotion_targets` / `promotion_combo_items` /
  `promotion_presentation_rules` todavía presentes** (esas tablas se borran en `063b`, T061a):
  extraer la lógica a una función pura testeable
  `migrate_promotions_data(bind, schema)` en el mismo módulo de migración —
  (a) por cada `promotions.type = 'percent'`: `INSERT INTO promotion_variants` las
  `product_variants.active = true` alcanzadas por sus `promotion_targets` de producto/categoría, o
  **todas** las activas del tenant si no tiene targets (percent global), `ON CONFLICT
  (promotion_id, product_variant_id) DO NOTHING` (foto fija, FR-026);
  (b) `UPDATE promotions SET status='finished', closed_by_refactor_at=now() WHERE type IN
  ('combo','fixed','qty_price','qty_price_presentation') AND status <> 'finished'` (FR-025). Las ya
  `finished` no se tocan. Depende de T005.

- [X] T007 ⏸ (ejecutada en el Incremento F vía T061a/T061b) **Diferida al Incremento F — su contenido está en T061a / T061b (Phase 9).** El borrado
  de `promotion_presentation_rules` / `promotion_combo_items` / `promotion_targets` / `presentations`,
  de `promotions.priority` y de `product_variants.presentation_id`, el estrechamiento con escape de
  `ck_promotion_type` (`… OR status='finished'`) y el drop de `ck_promotion_qty_price_pack`
  **no pueden ejecutarse en el Incremento A**: hasta la Phase 8 (`menu/router.py`, T055–T056) hay
  código que importa `Presentation` / `active_presentation_promotions`. Todo eso va en la revisión
  destructiva `063b` con `downgrade` simétrico (T061a), aplicada y verificada en T061b.

- [X] T008 Aplicar la revisión **`063a`** contra **PostgreSQL real** (quickstart.md Paso 1): sembrar un
  tenant con promociones de los cinco tipos (`percent` de producto, de categoría y global, `fixed`,
  `combo` de 2 componentes, `qty_price`, `qty_price_presentation`) + al menos una `Sale` + `Invoice`
  con descuento emitidas; `alembic upgrade head` → `alembic downgrade -1` → `alembic upgrade head`.
  Verificar: `percent` con filas en `promotion_variants` (foto fija); `combo`/`fixed`/`qty_price`/
  `qty_price_presentation` no terminales → `finished` + `closed_by_refactor_at`; `Sale`/`Invoice`
  previas **sin cambio de importe**; `downgrade` de `063a` limpio (las tablas viejas siguen intactas).
  **El borrado de tablas/columnas y su ciclo se verifican en T061b** (revisión `063b`, Phase 9).
  Depende de T005, T006.

- [X] T009 [P] Modelo `pos-backend/app/models/promotion.py` — **solo lo aditivo del Incremento A**:
  clase nueva `PromotionVariant` (`promotion_id` FK CASCADE indexado, `product_variant_id` FK CASCADE
  indexado, `UniqueConstraint("promotion_id","product_variant_id")`, `__table_args__` schema
  `tenant`); relación `Promotion.variants: Mapped[list[PromotionVariant]]` con
  `cascade="all, delete-orphan"`; `CheckConstraint("min_qty >= 1", name="ck_promotion_min_qty")`;
  columna `closed_by_refactor_at: Mapped[Optional[datetime]]` (`DateTime`, nullable);
  `PROMOTION_TYPES = ("percent", "fixed", "combo", "qty_price", "qty_price_presentation",
  "package_price")` y `ck_promotion_type` **ampliado** a esos seis valores (coincide con `063a`).
  **NO** se tocan todavía `priority`, las clases `PromotionTarget` / `PromotionComboItem` /
  `PromotionPresentationRule`, las relaciones `targets` / `combo_items` / `presentation_rules`,
  `ck_promotion_qty_price_pack` ni el import de `Presentation` — ese borrado va en **T061c** (Phase 9),
  tras reescribir `promotions/service.py` (T028). Depende de T003.

- [X] T010 ⏸ (ejecutada en el Incremento F vía T061c) **Diferida al Incremento F — su contenido está en T061c (Phase 9).** Borrar
  `pos-backend/app/models/presentation.py` no puede hacerse aquí: `promotions/service.py:39`,
  `catalog/router.py:12` y `menu/router.py` (vía `active_presentation_promotions`) lo importan hasta
  las Phases 4/8 (FR-027, A-63).

- [X] T011 ⏸ (ejecutada en el Incremento F vía T061c) **Diferida al Incremento F — su contenido está en T061c (Phase 9).** Quitar
  `product_variants.presentation_id` (columna, FK, `index`, relación, import) del ORM
  `pos-backend/app/models/product_variant.py` no puede hacerse aquí: `catalog/router.py` (T061d),
  `checkout.py:264` (T029), `cart/service.py:275` (T037) y `promotions/service.py`
  (`_active_variants_for_presentation`, T028/T042) lo referencian (data-model.md §ProductVariant).

- [X] T012 [P] `pos-backend/app/models/sale.py`: columna nueva `applied_promotions` (`JSONB`, NOT
  NULL, `server_default "'[]'"`). **Conservar** `promotion_id` (FK `SET NULL`) y `sale_items.combo_id`
  (histórico, FR-024, Principio VII); dejan de escribirse (data-model.md §sales).

- [X] T013 [P] `pos-backend/app/models/invoice.py`: columna nueva `applied_promotions` (`JSONB`, NOT
  NULL, `server_default "'[]'"`) (data-model.md §invoices).

- [X] T014 [P] `pos-backend/app/models/customer_order.py`: columnas nuevas `discount`
  (`Numeric(12, 2)`, NOT NULL, `server_default="0"`) y `applied_promotions` (`JSONB`, NOT NULL,
  `server_default "'[]'"`) + `CheckConstraint("discount >= 0",
  name="ck_customer_order_discount_non_negative")` (data-model.md §customer_orders).

- [X] T015 [P] `pos-backend/app/models/__init__.py`: **agregar** el import/export de
  `PromotionVariant`. Los exports de `Presentation` / `PromotionTarget` / `PromotionComboItem` /
  `PromotionPresentationRule` **se conservan** hasta T061c (Phase 9), porque hay módulos que aún los
  importan.

- [X] T016 ⏸ (ejecutada en el Incremento F vía T061d) **Diferida al Incremento F — su contenido está en T061d (Phase 9).** Borrar el paquete
  `pos-backend/app/api/v1/presentations/` y desmontar su router del agregador de `api/v1` no puede
  hacerse aquí: `promotions/router.py:17` (`from app.api.v1.presentations import service`) lo usa
  hasta la Phase 3 (T020) y el resto de consumidores hasta la Phase 8 (FR-027, A-63).

- [X] T017 [P] Fixtures de characterization tests en `pos-backend/app/characterization_tests/` —
  **solo lo aditivo**: `make_promotion` de `cart_fixtures.py` / `orders_fixtures.py` /
  `table_sessions_fixtures.py` gana `type` (default `"percent"`) y sigue aceptando `priority`
  (ignorado / pass-through, la columna existe hasta T061c); **agregar**
  `add_variant_to_promotion(db, promo, variant)` (inserta en `promotion_variants`, mismo patrón
  `_uid()` + `db.add` + `db.flush()`). **`make_promotion_target`, `make_combo_item` y
  `presentation_fixtures.py` se conservan** hasta que la última reescritura de tests (T039/T040/T041/
  T057) las deje sin uso; su borrado y la limpieza de `priority` en `make_promotion` van en T062
  (Incremento F) (contracts/migracion.md §2.6).

**Checkpoint**: `063a` aplica y revierte limpio contra Postgres real; `promotion_variants` poblado por
el paso de datos; el modelo conoce `PromotionVariant` y admite `package_price`; **el motor y los
endpoints actuales siguen funcionando sin cambios y la suite existente pasa en verde** (`Presentation`
/ `priority` / targets / combos siguen en el modelo, sin uso nuevo); ningún cobro usa el modelo nuevo
todavía.

---

## Phase 3: User Story 1 - El administrador arma una promoción sobre un conjunto explícito de variantes (Priority: P1) 🎯 MVP

**Goal**: el administrador crea una promoción eligiendo tipo (porcentaje | precio de paquete), valor,
cantidad mínima y el **conjunto explícito de variantes**; ve un resumen legible (tipo, condición en
lenguaje llano, lista de variantes con precio normal) antes de guardar; el sistema rechaza el
conjunto vacío, el porcentaje > 100 y el precio de paquete que no representa descuento (FR-016).

**Independent Test**: crear "2X Pequeños con licor $12.000" agregando las 8 variantes Pequeño con
licor, ver el resumen, guardar, y comprobar que queda en `Borrador` con esas 8 variantes
(quickstart.md §US1).

**Depende de**: Phase 2 (Foundational). **Corresponde al Incremento B** (parcial) + **E** (parcial).

### Implementation for User Story 1

- [X] T018 [P] [US1] `pos-backend/app/api/v1/promotions/schemas.py`: `PromotionType = {PERCENT,
  PACKAGE_PRICE}` **solo para entrada** (`PromotionCreate` / `PromotionShapeUpdate`); **borrar** de la
  entrada `COMBO` / `QTY_PRICE` / `QTY_PRICE_PRESENTATION`, `TargetIn`, `ComboItemIn`,
  `PresentationRuleIn`, `priority`, `targets`, `combo_items`, `presentation_rules`,
  `confirm_precio_no_uniforme`, `confirm_sin_descuento`, `OverlapResponse`, `PromotionWithOverlaps` y
  sus validadores; `PromotionCreate` y `PromotionShapeUpdate` ganan `variant_ids: list[UUID]` (**≥ 1**,
  sin repetidos); conservar `_PromotionRules._percent_range` (`0 < value <= 100`); validador nuevo
  para `package_price` (`value > 0`, `min_qty >= 1`). **`PromotionResponse.type` se queda como `str`
  (no el enum de entrada)** para poder serializar las promociones `finished` que la migración dejó
  con `type` = `combo` / `qty_price` / `qty_price_presentation` / `fixed` (FR-025) — el aviso de
  FR-025 (`?closed_by_refactor=true`) las lista. `PromotionResponse` expone además `variants:
  list[{product_variant_id, description, unit_price}]`, `condition_text` y `closed_by_refactor_at`, y
  **pierde** `priority` / `targets` / `combo_items` / `presentation_rules` / `overlaps`
  (contracts/administracion-promociones.md §1, §3).

- [X] T019 [US1] `pos-backend/app/api/v1/promotions/service.py`: funciones nuevas
  `_apply_variant_set(db, promo, variant_ids)` (valida no vacía, sin repetidos, cada uuid existe y es
  del tenant; puebla `promotion_variants`), `variant_set_condition_text(promo)` (tabla de textos de
  contracts/motor-y-persistencia.md §4, español de Colombia), `_guard_package_is_discount(db, promo)`
  (FR-016: `type == "package_price"` y `value >= min_qty × menor price del conjunto` → **409**
  `{error, value, min_qty, cheapest_unit_price, variant_id}`). Invocar `_apply_variant_set` +
  `_guard_package_is_discount` en `create`. **Borrar** de `create`/`update` el andamiaje de targets /
  combo / presentation (`_apply_targets`, `_apply_combo_items`, `_apply_presentation_rules`,
  `_validate_presentation_ids`, `_validate_shape_presentation_rules`, `_check_presentation_rule_prices`,
  `QTY_PRICE_PRESENTATION`, `AUTO_TYPES`). Depende de T009, T018.

- [X] T020 [US1] `pos-backend/app/api/v1/promotions/router.py`: `_serialize` resuelve `variants` con
  `"{product.name} - {variant.name}"` + precio normal vigente (FR-005) y `condition_text`; **borrar**
  `_with_overlaps` / `PromotionWithOverlaps`; `list_query` ordena por `Promotion.name` (ya no
  `priority`); query param nuevo `closed_by_refactor: bool` → filtra `closed_by_refactor_at IS NOT
  NULL` (FR-025); 409 legible para FR-016; `X-Server-Time` intacto (A-09)
  (contracts/administracion-promociones.md §3). Depende de T019.

- [X] T021 [P] [US1] Test nuevo `pos-backend/app/characterization_tests/test_promotions_rules_admin.py`
  — casos US1 (quickstart.md §US1): `POST` con `variant_ids=[]` → **422** "Una promoción necesita al
  menos una variante" (CA3); `POST` `type=percent value=150` → **422** (CA4); `POST`
  `type=package_price value=12000 min_qty=2` con 8 variantes → **201** `status='draft'`,
  `condition_text` correcto, `variants[].unit_price` presente (CA1, CA5); guardar una lista concreta y
  comprobar que una variante creada **después** no aparece en `variants[]` (CA2, FR-003/FR-004);
  `POST` `package_price value=12000 min_qty=2` con conjunto `{$8.000, $6.000}` → **409** FR-016
  (SC-002). Usa `add_variant_to_promotion`. **Cita esta spec (A-58…A-65).** Depende de T017, T019,
  T020.

- [X] T022 [P] [US1] Frontend
  `pos-heladeria/src/app/modules/promotions/interfaces/promotion.interface.ts`: `PromotionType =
  'percent' | 'package_price'`; **borrar** `PromotionTarget`, `ComboItem`, `PresentationRule*`,
  `PromotionOverlap`, `PromotionWithOverlaps`, `AUTO_TYPES`, `priority`; conservar
  `PROMOTION_TRANSITIONS`; agregar `variant_ids: string[]`, `PromotionVariant { product_variant_id,
  description, unit_price }` y `OverlapConflictError` (cuerpo del 409 de FR-014).

- [X] T023 [P] [US1] Frontend
  `pos-heladeria/src/app/modules/promotions/services/promotion.service.ts`: payloads sin
  `targets` / `combo` / `presentation` / `priority` / `confirm_*`, con `variant_ids`; método nuevo
  para el aviso de FR-025 (`GET /promotions?closed_by_refactor=true`). Depende de T022.

- [X] T024 [US1] Frontend
  `pos-heladeria/src/app/modules/promotions/services/promotion-pricing.util.ts`: `getPromoDisplay`
  para 2 tipos (`percent` con `min_qty === 1` previsualizable a nivel de precio unitario;
  `package_price` fuera de la previsualización, como `qty_price` hoy); **borrar** las ramas de
  `combo` / `qty_price` / `qty_price_presentation` (contracts/superficies-consumo.md §2). Depende de
  T022.

- [X] T025 [US1] Frontend
  `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts` (reescritura mayor):
  un solo formulario con selector de **dos tipos** ("Descuento %" / "Precio de paquete"); selector de
  conjunto de variantes con filtros por producto, categoría y texto **solo para poblar** (FR-004);
  resumen legible antes de guardar — tipo, condición en lenguaje llano, lista de variantes con precio
  normal (FR-005); **borrar** el selector de tipo combo/qty_price/presentación, el formulario de
  combo, el de precio por target, el de reglas por presentación, el input de `priority` y el panel de
  `overlaps`. **Borrar** `pos-heladeria/src/app/modules/promotions/components/scope-picker.component.ts`
  (contracts/superficies-consumo.md §3). Depende de T022, T023, T024.

**Checkpoint**: un administrador ya crea promociones de dos tipos sobre un conjunto explícito de
variantes, con el resumen de FR-005 y el bloqueo de FR-016 — **sin** que el cobro las use todavía.

---

## Phase 4: User Story 2 - El cajero cobra un pedido y el paquete combina variantes distintas del conjunto (Priority: P1)

**Goal**: al cobrar, el sistema reúne todas las unidades del pedido cuyas variantes pertenecen al
conjunto de una promoción vigente, arma `total // min_qty` **grupos completos** por **consumo
codicioso descendente de precio**, descuenta solo esos grupos y reparte el descuento entre las líneas
contribuyentes repartiendo el importe cobrado (residuo a la variante de id más alto). El resultado no
depende del orden de las líneas. Al emitir se persiste `applied_promotions` (FR-021).

**Independent Test**: con "2X Pequeños con licor $12.000" activa ($8.000 c/u), cobrar 1 Ojo de Diablo
Pequeño + 1 Perla Negra Pequeño → total $12.000; luego 3 unidades → $20.000; en cualquier orden de
captura (quickstart.md §US2).

**Depende de**: Phase 2 (Foundational) + Phase 3 (US1 — necesita promociones activas con conjunto).
**Corresponde al Incremento C.**

### Implementation for User Story 2

- [X] T026 [US2] `pos-backend/app/api/v1/promotions/service.py`: dataclasses `AppliedPromotion`
  (`promotion_id`, `name` snapshot, `amount`) y `SetDiscountResult` (`total`, `by_line`, `applied`);
  función `active_variant_set_promotions(db, now)` (hermana de `active_discount_promotions`:
  `status == "active"` + `ends_at` no vencido (índice `ix_promotions_status_ends_at`) +
  `_valid_now(p, now)` + `selectinload(Promotion.variants)`). **Conservar sin cambio de cuerpo**
  `_tz`, `local_now`, `_in_time_window`, `_valid_now` (A-57 intacto)
  (contracts/motor-y-persistencia.md §2). Depende de T019.

- [X] T027 [US2] `pos-backend/app/api/v1/promotions/service.py`: `_greedy_units(eligible_units,
  min_qty)` (ordena por `(-unit_price, product_variant_id, line_id)`; las primeras
  `grupos * min_qty` unidades en bloques consecutivos de `min_qty`; el resto es remanente — FR-008,
  FR-007) y `_distribute_group_discount(group_units, discount)` (reparto por **importe cobrado**:
  `cobrado_L = floor(aporte_L - discount * aporte_L / aporte_total)`; el residuo hasta
  `aporte_total - discount` se suma al `cobrado_L` de la línea cuya variante tiene el
  `product_variant_id` más alto, desempate `line_id` más alto — FR-008a, contract §3). Depende de
  T026.

- [X] T028 [US2] `pos-backend/app/api/v1/promotions/service.py`: `evaluate_variant_sets(db,
  promo_lines, now) -> SetDiscountResult` (algoritmo normativo de contracts/motor-y-persistencia.md
  §2: unidades elegibles con `product_variant_id ∈ S` **y** `_variant_active` **y** `combo_id is
  None`; `grupos = n // min_qty`; consumo codicioso; `package_price` → `max(0, normal_g - value)`;
  `percent` → `round(value% × normal_g)` a peso; reparto vía `_distribute_group_discount`; `total`
  con `.quantize(Decimal("0.01"), ROUND_HALF_UP)` **una sola vez**; `applied` ordenado por
  `promotion_id`). **Borrar** `evaluate`, `evaluate_detailed`, `combined_discount_detailed`,
  `combo_discount_for_lines`, `presentation_package_discount_for_lines`, `_best_line_match`,
  `best_line_discount`, `_line_discount`, `_matching_target`, `_pack_terms`, `get_active_combo`,
  `expand_combo`, `ComboComponent`, `LineDiscount`, `PromotionResult`, `_line_by_line_discounts`,
  `_rule_discount_by_line`, `_presentation_reference_unit_price`, `_unit_sort_key`,
  `active_presentation_promotions` y toda la reconciliación por pool. Depende de T027.

- [X] T029 [US2] `pos-backend/app/api/v1/orders/checkout.py`: `promo_lines_for` deja de traer
  `product_id` / `category_id` / `presentation_id` (línea ~264, `variant.presentation_id`) y agrega
  `description`; conserva `product_variant_id` / `unit_price` / `line_id` / `quantity` /
  `_variant_active` / `combo_id` (contracts/motor-y-persistencia.md §1).
  `auto_discount(db, lines, now)` llama `evaluate_variant_sets` y devuelve `(total, promotion_id,
  applied)` (`promotion_id` = `applied[0].promotion_id` si `len(applied) == 1`, si no `None`).
  **Borrar** todo lo de `combo_discount` y la reconciliación por pool. Depende de T028.

- [X] T030 [P] [US2] `pos-backend/app/api/v1/sales/builder.py`: `build_sale` acepta
  `applied_promotions: list[dict]` y lo persiste en `Sale.applied_promotions`
  (contracts/motor-y-persistencia.md §5). Depende de T012.

- [X] T031 [P] [US2] `pos-backend/app/api/v1/invoices/service.py`: `issue_for_sale` copia
  `sale.applied_promotions` a `invoice.applied_promotions` dentro de la transacción del cobro
  (snapshot inmutable, como `discount=sale.discount`). Depende de T013, T030.

- [X] T032 [US2] `pos-backend/app/api/v1/orders/checkout.py`: `pay_order` (orden de mesa legada) pasa
  `applied` a `build_sale` y fija `order.discount` + `order.applied_promotions` en la
  `CustomerOrder` (FR-021). Depende de T029, T030.

- [X] T033 [P] [US2] `pos-backend/app/api/v1/sales/service.py` (venta de mostrador,
  `checkout_and_send`): reemplaza la llamada a `combined_discount_detailed` por `auto_discount` /
  `evaluate_variant_sets`; `applied_promotions` a la venta (y a la `CustomerOrder` si viene de orden).
  Depende de T029, T030.

- [X] T034 [P] [US2] `pos-backend/app/api/v1/table_sessions/service.py`: `compute_bill` (preview por
  comensal / split), `release_paid_session` / `_close_unified` y `_close_split` usan el retorno nuevo
  de `auto_discount`; `_close_unified` / `_close_split` fijan `applied_promotions` en la `Sale`, la
  `Invoice` y cada `CustomerOrder` de la sesión (contracts/motor-y-persistencia.md §5, research.md
  D7). Depende de T029, T030.

- [X] T035 [P] [US2] `pos-backend/app/api/v1/orders/tables_advanced.py`: `group_bill` (mesas
  fusionadas) usa el retorno de tres valores de `auto_discount`. Depende de T029.

- [X] T036 [P] [US2] `pos-backend/app/api/v1/orders/consolidation.py`: retirar de `add_item_to_table`
  la rama de `combo_id` (expansión de combo, FR-024); `AddItemToTableIn` pierde `combo_id`.

- [X] T037 [US2] `pos-backend/app/api/v1/cart/service.py`: `_cart_promo_lines` pierde `product_id` /
  `category_id` / `presentation_id` (`ProductVariant.presentation_id` en el `select` ~L275 y en el
  dict ~L294), agrega `description`; `_line_discount` (por ítem) y `serialize_cart` usan el
  `by_line` de **una sola** llamada a `evaluate_variant_sets`; `submit_cart` toma el snapshot de
  descuento del **mismo** `by_line` (spec 038 sigue cuadrando); **borrar** `_add_combo` y la rama
  `combo_id` de `add_item` (FR-024). `pos-backend/app/api/v1/cart/schemas.py`: `CartItemIn` pierde
  `combo_id` (contracts/motor-y-persistencia.md §6). Depende de T028.

- [X] T038 [P] [US2] Test nuevo `pos-backend/app/characterization_tests/test_promotions_service.py` —
  los **10 Acceptance Scenarios** de US2 + edge cases (quickstart.md §US2 tabla #1–#11): 2 variantes
  distintas → $12.000 (CA1, SC-008); 3 unidades → $20.000 con la suelta determinista (CA2); otro
  orden → total y reparto idénticos (CA3, SC-005); `percent` `min_qty` 1 mixto → $20.700 (CA4); "15%
  llevando 3 medianos" → $33.500 con reparto −$3.300 / −$1.200 (CA5, SC-005); no alcanza `min_qty` →
  sin descuento (CA6); día no incluido (CA7); ventana horaria 14:59 vs 15:01 (CA8); variante
  desactivada no cuenta (CA9, FR-011); "3 Pequeños sin licor por $16.000" → residuo $1 a la variante
  de id más alto, Σ por línea cuadra al peso en cualquier orden (SC-005, división no exacta);
  conjunto mixto → $18.000. **Cita esta spec (A-58…A-65).** Depende de T028.

- [X] T039 [P] [US2] Reescribir en
  `pos-backend/app/characterization_tests/test_orders_checkout.py` los `CONGELA` afectados
  (contracts/migracion.md §2.1): `test_promo_lines_for_camino_feliz_y_sin_promocion_aplicable`
  (forma de `promo_lines` en el modelo nuevo), `test_pay_order_construye_sale_real_con_promocion_activa`
  (`Sale.discount` + `applied_promotions`, FR-021), `test_pay_order_dos_combos_distintos_a29_promotion_id_none`
  (dos promociones distintas → `promotion_id` NULL **pero** `applied_promotions` con las dos, A-64).
  Sustituir los casos de spec 040 (`test_una_linea_recibe_una_sola_promocion_la_de_menor_total`,
  `test_recalculo_del_pool_*`, `test_fr023_nunca_deja_la_linea_peor_que_sin_promocion`) por los del
  motor por conjunto (FR-007/FR-008/FR-009). **Cita esta spec (A-58…A-65).** Depende de T029, T032.

- [X] T040 [P] [US2] En `pos-backend/app/characterization_tests/test_cart_service.py`: **eliminar**
  `test_add_item_combo` y `test_serialize_cart_combo_no_recibe_descuento_adicional` (FR-024);
  **re-congelar** `test_serialize_cart_discounted_total_con_promocion_activa` (conjunto de variantes
  en vez de categoría) y `test_serialize_cart_discounted_total_sin_promocion` (invariante
  `discounted_total = None`); `test_submit_cart_snapshot_de_descuento_coincide_con_el_carrito` (spec
  038) cuadra con `evaluate_variant_sets` (contracts/migracion.md §2.1). **Cita esta spec.** Depende
  de T037.

- [X] T041 [P] [US2] Reescrituras restantes de `CONGELA` (contracts/migracion.md §2.1-2.2):
  `pos-backend/app/characterization_tests/test_table_sessions_service.py`
  (`test_close_session_unified_a29_promotion_id_no_registra_combos_multiples` → `applied_promotions`
  en `Sale` y `CustomerOrder`, sin combos);
  `pos-backend/app/characterization_tests/test_orders_consolidation.py` (**eliminar**
  `test_add_item_to_table_combo_expande_componentes_a_precio_normal`);
  `pos-backend/app/characterization_tests/test_orders_tables_advanced.py`
  (`test_group_bill_aplica_promocion_percent_*` y `test_group_bill_a01_camino_c_*` re-congelados con
  conjunto de variantes; **eliminar** `test_group_bill_aplica_combo_vigente_sin_terminales_fr_002`).
  **Cita esta spec (A-58…A-65).** Depende de T034, T035, T036.

**Checkpoint**: US1 + US2 funcionan juntas — el cobro combina variantes distintas del conjunto, arma
grupos completos por consumo codicioso, reparte al peso y persiste `applied_promotions` en
`Sale` / `Invoice` / `CustomerOrder`. SC-005 verificado con el caso de división no exacta.

---

## Phase 5: User Story 3 - Vigencia por días y franja horaria; sin solapamiento real entre promociones (Priority: P2)

**Goal**: el administrador define días y franja horaria (con cruce de medianoche); el sistema impide
crear o activar una promoción cuyo conjunto comparta ≥1 variante con otra no terminal **si además**
sus rangos de fecha **y** conjuntos de días **y** ventanas horarias se intersectan (dimensión no
definida = cubre todo su dominio).

**Independent Test**: activar "10% en granizados" todos los días; intentar activar "20% en granizados"
también todos los días sobre variantes que se solapan → **bloqueado**; cambiar la segunda a
`daysOfWeek = {martes}` y ventana 00:00–14:59 mientras la primera va de 15:00 a cierre → **permitido**
(quickstart.md §US3).

**Depende de**: Phase 3 (US1 — extiende `create` / `update_shape` de `promotions/service.py`).
**Corresponde al Incremento B** (parcial).

### Implementation for User Story 3

- [X] T042 [US3] `pos-backend/app/api/v1/promotions/service.py`: `_guard_variant_overlap(db, promo,
  variant_ids)` (FR-014 / FR-014a — detalle normativo en contracts/administracion-promociones.md §2:
  variante compartida con otra promoción en `draft`/`active`/`paused` **y** intersección simultánea
  de **fecha ∧ días ∧ horas**; dimensión abierta = todo su dominio; reutiliza `_in_time_window` y las
  primitivas `_ranges_overlap` / `_csv_overlap` / `_times_overlap` acotadas a intersección
  simultánea). **Borrar** `find_overlaps`, `_scope_overlap`, `presentation_overlap_conflicts`,
  `_guard_presentation_overlap`, `_active_variants_for_presentation`. Invocar `_guard_variant_overlap`
  en `create` y `update_shape` (research.md D4). Depende de T019.

- [X] T043 [US3] `pos-backend/app/api/v1/promotions/schemas.py` + `router.py`: cuerpo del 409 de
  FR-014 (`OverlapConflict`: `{error, conflicts: [{promotion_id, promotion_name, variant_ids: [...]}]}`);
  el router lo mapea a un 409 legible; confirmar que ninguna respuesta arrastra `overlaps`
  (contracts/administracion-promociones.md §2-3). Depende de T020, T042.

- [X] T044 [P] [US3] Casos US3 en
  `pos-backend/app/characterization_tests/test_promotions_rules_admin.py` (quickstart.md §US3):
  activar 2ª promo con variante compartida y ventanas que se cruzan → **409** nombrando la promoción
  y las variantes compartidas (CA2); ventanas que no se cruzan (08:00–15:00 / 15:00–22:00) →
  **permitido** (CA3); "Happy hour 22:00–01:00 vie/sáb, 25% en litros" cobrado **sábado 00:30** →
  descuenta (madrugada del sábado pertenece al viernes, A-57) (CA1); promo `Activa` **sin franja**
  vs otra con franja 15:00–17:00 y días/fechas que se cruzan → **409** (FR-014a) (CA5); ventana
  22:00–02:00 aceptada al guardar (CA4). **Cita esta spec (A-58…A-65).** Depende de T042.

- [X] T045 [P] [US3] Frontend
  `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`: diálogo del 409 de
  FR-014 (nombra `conflicts[].promotion_name` y las variantes compartidas); confirmar que el panel de
  `overlaps` (advertencia) ya no existe (contracts/superficies-consumo.md §3). Depende de T025.

**Checkpoint**: US1 + US2 + US3 funcionan de forma independiente — el bloqueo de solape real acota la
coexistencia de promociones sobre variantes compartidas a ventanas disjuntas.

---

## Phase 6: User Story 5 - Duplicar, editar una promoción activa, cambiar de estado (Priority: P2)

**Goal**: duplicar una promoción (la copia nace en `Borrador` con el mismo conjunto y condición),
editar solo campos no estructurales de una `Activa` (nombre, descripción, fin de vigencia, días,
horas) y moverla por la máquina de estados `Borrador → Activa → Pausada → Finalizada` (`Finalizada`
terminal).

**Independent Test**: activar una promoción, editar su nombre y su fin de vigencia (permitido),
intentar cambiar su valor y su conjunto (bloqueado), duplicarla, y en la copia cambiar el valor
(quickstart.md §US5).

**Depende de**: Phase 5 (US3 — comparte `promotions/service.py`; `change_status → active` revalida
`_guard_variant_overlap`). **Corresponde al Incremento B** (parcial) + **E** (parcial).

### Implementation for User Story 5

- [X] T046 [US5] `pos-backend/app/api/v1/promotions/service.py`: `create` / `update` / `update_shape`
  / `change_status` / `duplicate` pierden `targets` / `combo` / `presentation` / `priority`; `update`
  (escalares en `active` / `paused`) permite `name` / `description` / `ends_at` / `days_of_week` /
  `start_time` / `end_time` y **bloquea** `value` / `min_qty` (422 "Duplica la promoción para
  cambiar el valor o la cantidad") evaluado contra `promo.status` real; `update_shape` exige `draft`
  para `type` / `variant_ids`; `change_status → active` revalida `_guard_variant_overlap` +
  `_guard_package_is_discount` + conjunto no vacío; `duplicate` copia tipo / valor / `min_qty` /
  conjunto de variantes / vigencia, nombre distinto, nace `draft` (FR-017, FR-018;
  contracts/administracion-promociones.md §2, research.md D11). Depende de T042.

- [X] T047 [US5] `pos-backend/app/api/v1/promotions/schemas.py`: `PromotionUpdate` = `{name?,
  description?, ends_at?, days_of_week?, start_time?, end_time?}`; `value?` / `min_qty?` se aceptan en
  el schema pero el servicio los rechaza si `status != "draft"` (FR-018); `starts_at` no editable
  tras crear (contracts/administracion-promociones.md §1). Depende de T018.

- [X] T048 [P] [US5] Casos US5 en
  `pos-backend/app/characterization_tests/test_promotions_rules_admin.py` (quickstart.md §US5):
  editar `name` / `description` / `ends_at` / `days_of_week` / `start_time` / `end_time` de una
  `Activa` → 200 (CA1); `PATCH` con `value` distinto o `PATCH /shape` con `variant_ids` distinto →
  422 / 409 que sugiere duplicar (CA2); `PATCH /status {"status":"active"}` sobre `Finalizada` →
  **409** (terminal) (CA3); `POST /promotions/{id}/duplicate` → copia `Borrador` con mismo conjunto,
  nombre distinto (CA4); usuario cajero `POST /promotions` → **403** (CA5). **Cita esta spec.**
  Depende de T046.

- [X] T049 [P] [US5] Frontend
  `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`: edición de una
  promoción `Activa` limitada a nombre / descripción / fin de vigencia / días / horas (FR-018);
  diálogo del 422 de FR-018 ("El valor y la cantidad de una promoción activa no se pueden cambiar.
  Duplícala…"); acción de duplicar (contracts/superficies-consumo.md §3). Depende de T025.

**Checkpoint**: US1 + US2 + US3 + US5 funcionan de forma independiente — el ciclo de vida diario del
administrador (duplicar, editar activa, transiciones de estado, permisos) está cubierto.

---

## Phase 7: User Story 6 - Las promociones existentes en producción se migran o se cierran de forma predecible (Priority: P2)

**Goal**: al desplegar, las `percent` se migran automáticamente a conjuntos de variantes (foto fija)
y las `fixed` / `combo` / `qty_price` / `qty_price_presentation` pasan a `Finalizada` con un aviso;
ninguna `Sale` / `Invoice` ya emitida cambia de importe.

**Independent Test**: con una `percent` de categoría, una `fixed`, un `combo` de dos componentes y una
`qty_price_presentation` activas antes del despliegue, correr la migración y verificar el estado y la
forma de cada una después (quickstart.md §US6).

**Depende de**: Phase 2 (T006 — el paso de datos `migrate_promotions_data` de la revisión `063a`).
**Corresponde al Incremento A** (verificación del paso de datos) + **F** (tests + el borrado de tablas
va en `063b`/T061b) + **E** (banner).

### Implementation for User Story 6

- [X] T050 [P] [US6] Test nuevo
  `pos-backend/app/characterization_tests/test_promotions_migration.py` — ejercita
  `migrate_promotions_data` (T006) sobre SQLite con el esquema pre-refactor sembrado
  (contracts/migracion.md §3): `percent` de categoría → tipo porcentaje, valor/`min_qty`/vigencia/
  estado conservados, conjunto = variantes **activas** de la categoría al migrar (CA1); `combo` "1
  Litro + 1 Cono $30.000" → `status='finished'`, `closed_by_refactor_at` no nulo, aparece en
  `GET /promotions?closed_by_refactor=true`, líneas de venta históricas con `combo_id` intactas
  (CA2); `qty_price_presentation` activa → `Finalizada` + aviso (CA3); `fixed` "$2.000 de descuento
  por línea" → `Finalizada`, **no** se convierte en porcentaje ni precio de paquete (CA4); `Sale` ya
  emitida con descuento → `discount` / `total` / factura sin cambio (CA5). **Cita esta spec
  (A-61/A-62).** Depende de T006, T017.

- [X] T051 [US6] Verificación end-to-end del **paso de datos de `063a`** contra **PostgreSQL real**
  (quickstart.md Paso 1, ampliando T008): con el tenant sembrado con los cinco tipos + `Sale`/`Invoice`
  emitidas, confirmar que cada `percent` quedó con su conjunto en `promotion_variants` (foto fija) y
  estado/vigencia conservados; que `combo` / `fixed` / `qty_price` / `qty_price_presentation` no
  terminales pasaron a `finished` + `closed_by_refactor_at`; que `GET /promotions?closed_by_refactor=true`
  las lista exactamente; que ninguna `Sale` / `Invoice` emitida cambió de importe. (El borrado de
  `promotion_targets` / `presentations` / `priority` / `presentation_id` y su ciclo se verifican en
  T061b, revisión `063b`.) Depende de T008, T050.

- [X] T052 [P] [US6] Frontend
  `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`: banner **descartable**
  (una vez por administrador, descarte en `localStorage`) que, si `GET /promotions?closed_by_refactor=true`
  devuelve resultados, lista las promociones finalizadas por la migración e invita a recrearlas
  (FR-025, contracts/superficies-consumo.md §3). Depende de T023, T025.

- [X] T053 [P] [US6] Frontend — borrado de la entidad `Presentation` (FR-027, A-63): eliminar
  `pos-heladeria/src/app/modules/presentations/` completo; quitar la entrada "Presentaciones" de
  `pos-heladeria/src/app/core/config/navigation.config.ts` y la ruta `/dashboard/presentations` de
  `pos-heladeria/src/app/modules/dashboard/routes.ts`; quitar el selector de presentación por
  variante de `pos-heladeria/src/app/modules/products/pages/product-form.component.ts`;
  `pos-heladeria/src/app/modules/products/interfaces/product.interface.ts` — `ProductVariant` /
  `VariantForm` / `VariantCreatePayload` / `VariantUpdatePayload` / el draft pierden `presentation_id`
  y `presentationId`; `pos-heladeria/src/app/modules/products/services/product.service.ts` — quitar
  `presentation_id` de `VariantResponse` (raw), `VariantCreatePayload`, `VariantUpdatePayload`,
  `VariantForm` y de `createVariant` / `updateVariant` / los mapeos de `variantDrafts` (~L78/308/400/618);
  `pos-heladeria/src/app/core/services/menu.service.ts` — tipado del campo `promotions`.

- [X] T054 [P] [US6] Frontend — retiro del mecanismo de combo (FR-024): eliminar
  `pos-heladeria/src/app/modules/tables/components/combo-select.component.ts`; retirar el flujo
  "agregar combo" de `pos-heladeria/src/app/modules/tables/components/product-select.component.ts` y
  `pos-heladeria/src/app/modules/tables/components/pos-catalog-drawer.component.ts`.

**Checkpoint**: US1–US3 + US5 + US6 funcionan de forma independiente — la migración deja cada
promoción existente en un estado explícito, sin cambiar ninguna venta emitida, y el frontend ya no
arrastra `Presentation` ni el flujo de combo.

---

## Phase 8: User Story 4 - El cliente del menú QR ve la promoción y su precio efectivo (Priority: P3)

**Goal**: un cliente que consulta el menú por QR ve, mientras la promoción está vigente en ese
momento, un anuncio con su condición en lenguaje llano, aunque no haya agregado nada al carrito;
cuando el carrito alcanza `min_qty` unidades elegibles, el precio efectivo se refleja en el carrito.
La terminal de staff muestra siempre la condición y el descuento efectivo al alcanzar `min_qty`
(FR-023), calculado por el preview del cobro.

**Independent Test**: con "2X Pequeños con licor $12.000" activa, abrir el menú QR dentro de su
horario sin agregar nada y ver el anuncio; agregar 2 Pequeños con licor y ver el total $12.000;
volver a abrir fuera de horario y comprobar que el anuncio ya no aparece (quickstart.md §US4).

**Depende de**: Phase 4 (US2 — `evaluate_variant_sets` / `active_variant_set_promotions`).
**Corresponde al Incremento D.**

### Implementation for User Story 4

- [X] T055 [US4] `pos-backend/app/api/v1/menu/router.py`: función mínima `menu_unit_discount(promos,
  variant_id, unit_price) -> Decimal | None` (solo `percent` con `min_qty == 1` baja el precio
  unitario → `round(unit_price * value / 100, ROUND_HALF_UP)`; `package_price` y `percent` con
  `min_qty > 1` → `None`, igual que hoy `qty_price`). `_build_menu` usa `promos =
  active_variant_set_promotions(db, now)` y **sigue devolviendo `list[MenuCategoryResponse]`** (no
  cambia de firma — CONGELA de `test_menu_router.py`, A-08) (contracts/superficies-consumo.md §1).
  Depende de T028.

- [X] T056 [US4] `pos-backend/app/api/v1/menu/router.py` + `schemas.py`: `_build_menu_promotions(db,
  now)` se adapta — el texto de cada anuncio sale de `variant_set_condition_text(promo)` (conjunto de
  variantes), incluido solo si `_valid_now(promo, now)` (vigencia en ese instante, FR-022, SC-007);
  `GET /menu/promotions` y la clave `"promotions"` del flujo QR con token se conservan;
  `MenuPromotionRule` describe el conjunto ("estas N variantes"), ya no una presentación
  (contracts/superficies-consumo.md §1). Depende de T055.

- [X] T057 [P] [US4] Reescribir `pos-backend/app/characterization_tests/test_menu_router.py`
  (contracts/migracion.md §2.2-2.3): `test_a08_fuera_de_ventana_*` y `test_a08_dentro_de_ventana_*`
  re-congelados con conjunto de variantes (la corrección de zona horaria **se conserva**);
  `TestMenuPromotionsAnnouncementUS5` reescrita — el anuncio describe el conjunto de variantes
  (vigente en el instante → aparece; fuera de ventana → no aparece; zona horaria del tenant);
  `_build_menu` no cambia de firma. **Cita esta spec (A-58…A-65).** Depende de T056.

- [X] T058 [P] [US4] Caso US4-CA3 en
  `pos-backend/app/characterization_tests/test_cart_service.py`: carrito con 1 unidad elegible de una
  promo `min_qty` 3 → condición visible + `discounted_total` = precio normal; al llegar a 3 →
  `discounted_total` refleja el precio de paquete (contracts/migracion.md §3). Depende de T037.

- [X] T059 [P] [US4] Frontend
  `pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts`: el banner de anuncio usa el
  texto por conjunto de variantes (FR-022); si la lista viene vacía, no se renderiza nada.
  `pos-heladeria/src/app/modules/tables/services/diner.service.ts`: `MenuPromotionAnnouncement.rules`
  pasa de `{ presentation_name, pack_price, min_qty, text }` a `{ text, variant_count, min_qty, value }`
  (FR-022); el parser de `promotions[].rules[]` (~L333) deja de leer `presentation_name` / `pack_price`.
  `pos-heladeria/src/app/core/services/menu.service.ts`: tipo/método para
  `GET /menu/promotions` — el retorno de `getMenu()` no cambia
  (contracts/superficies-consumo.md §1). Depende de T023.

- [X] T060 [US4] Frontend `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`: para
  cada variante de un conjunto de promoción vigente, mostrar **siempre** la condición en lenguaje
  llano (`variant_set_condition_text`, servida por el backend); mostrar el **descuento efectivo**
  cuando el pedido en curso alcanza `min_qty` unidades elegibles (FR-023), tomado del **preview del
  cobro** (`compute_bill` / preview de `checkout_and_send`), no de un cálculo local
  (contracts/superficies-consumo.md §2, research.md D10). Depende de T024, T034.

**Checkpoint**: las 6 historias de usuario son funcionales de forma independiente.

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: **Incremento F** (cierre) — el **retiro de la estructura legada** (revisión destructiva
`063b` + borrado de `Presentation` / `priority` / targets / combos / `presentation_id` del ORM y del
catálogo), diferido del Incremento A porque hasta la Phase 8 hubo módulos que importaban `Presentation`
(Principio VI: el Incremento A quedó 100% aditivo y verificable con la suite en verde); más la
reescritura del script de CI, el borrado de los tests de la spec 040 y la verificación de no-regresión
que atraviesa todas las historias (quickstart.md §"Verificación final"; plan.md §Constitution Check
Principios III, VI y X; contracts/migracion.md §4).

**Orden dentro de la fase**: T061a → T061c → T061d → T061b (autoría de `063b` → borrado del ORM y del
catálogo → borrado del paquete `presentations` → aplicar y verificar `063b` contra Postgres real);
luego T061–T063 (tests/CI) en paralelo; luego T064–T067.

- [X] T061a [Incremento F] Crear la revisión Alembic **destructiva** `063b`
  `pos-backend/alembic/versions/{rev}_063b_promociones_retiro_estructura_legada.py`
  (`down_revision` = revisión `063a` de T005), `@for_each_tenant_schema` + guarda `_has_table`
  (data-model.md §Migración `063b`): `drop_table` `promotion_presentation_rules`,
  `promotion_combo_items`, `promotion_targets` (en ese orden); `drop_index` + `drop_constraint` FK +
  `drop_column` `product_variants.presentation_id`; `drop_table` `presentations`; `drop_column`
  `promotions.priority`; `drop_constraint` `ck_promotion_qty_price_pack`; **estrechar con escape**
  `ck_promotion_type` a `type IN ('percent','package_price') OR status = 'finished'` (ninguna
  promoción viva con tipo viejo; las `finished` que dejó `063a` conservan su `type` histórico).
  `downgrade(schema)` **simétrico** (data-model.md §downgrade `063b`): recrea `presentations` /
  `promotion_targets` / `promotion_combo_items` / `promotion_presentation_rules` **vacías** con la
  estructura de spec 013/040, restaura `priority` (INTEGER NOT NULL default 0),
  `product_variants.presentation_id` (+ FK `SET NULL` + índice), `ck_promotion_qty_price_pack` y
  `ck_promotion_type` a los cinco valores de spec 040; **sin reversa del paso de datos**
  (documentado). Depende de T005; ejecutar tras la Phase 8.

- [X] T061b [Incremento F] Aplicar y verificar `063b` contra **PostgreSQL real** (quickstart.md
  Paso 1, cierre): sobre el tenant sembrado de T008 (ya con `063a` aplicada), `alembic upgrade head`
  → `alembic downgrade -1` → `alembic upgrade head`. Verificar: `promotion_targets` /
  `promotion_combo_items` / `promotion_presentation_rules` / `presentations` borradas;
  `promotions.priority` y `product_variants.presentation_id` borradas; una promoción **viva**
  (`draft`/`active`/`paused`) de tipo `combo`/`qty_price`/`fixed` es rechazada por `ck_promotion_type`,
  pero las `finished` migradas siguen legibles; **ninguna `Sale` / `Invoice` emitida cambió de
  importe ni de representación** (Principio VII); `downgrade` de `063b` recrea la estructura vacía sin
  error.
  Depende de T061a, T008, T028 (y todas las Phases 3–8 de backend).

- [X] T061c [Incremento F] Retiro del modelo legado del ORM: `pos-backend/app/models/promotion.py` —
  **borrar** `priority` (+ `server_default`), las clases `PromotionTarget` / `PromotionComboItem` /
  `PromotionPresentationRule`, las relaciones `targets` / `combo_items` / `presentation_rules`,
  `ck_promotion_qty_price_pack`, el import de `Presentation` y el bloque `TYPE_CHECKING`; ajustar
  `ck_promotion_type` en `__table_args__` a `type IN ('percent','package_price') OR status='finished'`
  (coincide con `063b`). **`PROMOTION_TYPES` se queda en los 6 valores** — sirve para leer las
  promociones `finished` de tipo viejo; la restricción "solo dos" vive en el `PromotionType` de
  entrada de `schemas.py` (T018).
  `pos-backend/app/models/product_variant.py` — borrar la columna `presentation_id`, su `ForeignKey`,
  su `index=True`, la relación `presentation` y el import de `Presentation` (data-model.md
  §ProductVariant). Borrar `pos-backend/app/models/presentation.py` (FR-027, A-63).
  `pos-backend/app/models/__init__.py` — quitar los exports de `Presentation` / `PromotionTarget` /
  `PromotionComboItem` / `PromotionPresentationRule`. Depende de T028, T042 (el código de
  `promotions/service.py` ya no referencia esas clases) y de todas las Phases 3–8 de backend.

- [X] T061d [Incremento F] Retiro del paquete y del catálogo: borrar
  `pos-backend/app/api/v1/presentations/` completo (`router.py`, `schemas.py`, `service.py`) y
  desmontar su router del agregador de `api/v1` (FR-027, A-63). Revertir la integración de
  `Presentation` en el catálogo que la spec 040 introdujo y el plan 063 no listaba:
  `pos-backend/app/api/v1/catalog/schemas.py` — `VariantCreate` / `VariantUpdate` / `VariantResponse`
  pierden `presentation_id`; `pos-backend/app/api/v1/catalog/router.py` — borrar el import
  `from app.models.presentation import Presentation`, la función `_resolve_presentation_id` y sus
  llamadas en `create_variant` (`presentation_id=...`) y `update_variant` (rama
  `if "presentation_id" in body.model_fields_set`). Depende de T020 (`promotions/router.py` ya no
  importa el service de `presentations`) y T055–T056 (`menu/router.py` ya no usa
  `active_presentation_promotions`).

- [X] T061 [P] Reescritura **completa** del script de CI
  `pos-backend/app/scripts/test_promotions_rules.py` (contracts/migracion.md §2.5): se van las
  secciones de `priority`, `_matching_target` (targets) y `_line_discount` de `qty_price` / `fixed`;
  entran (funciones puras, sin sesión): consumo codicioso descendente (FR-008), grupos completos +
  remanente a precio normal (FR-007), `package_price` → descuento = `Σ normal − value` topado en 0
  (FR-006/FR-009), `percent` → `round(value% × Σ normal)` a peso (FR-006),
  `_distribute_group_discount` con **división no exacta** → residuo a la variante de id más alto y
  `Σ descuentos por línea == descuento del grupo` al peso en cualquier orden (FR-008a, SC-005), un
  grupo nunca encarece (FR-009), vigencia en hora local + ventana que cruza medianoche + **A-57
  conservado**, `type` admite exactamente `{percent, package_price}`. **Cita esta spec.**

- [X] T062 [P] Borrar los tests de la spec 040 (sin prefijo CONGELA, contracts/migracion.md §2.4):
  `pos-backend/app/characterization_tests/test_promotions_presentation_pricing.py`,
  `pos-backend/app/characterization_tests/test_promotions_presentation_rules.py`,
  `pos-backend/app/characterization_tests/test_presentations_service.py`. **Limpieza de fixtures**
  (diferida de T017, ya sin uso tras T039/T040/T041/T057): borrar `make_promotion_target`,
  `make_combo_item` y `presentation_fixtures.py`; `make_promotion` deja de aceptar `priority`
  (contracts/migracion.md §2.6). Depende de T039, T040, T041, T057.

- [X] T063 [P] Reescribir las specs de frontend afectadas (contracts/migracion.md §2.7):
  `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.spec.ts`,
  `pos-heladeria/src/app/modules/promotions/services/promotion-pricing.util.spec.ts`,
  `pos-heladeria/src/app/modules/promotions/services/promotion.service.spec.ts` (dos tipos, selector
  de conjunto, diálogos de FR-014 / FR-016 / FR-018); ajustar `product.service.spec.ts` /
  `product-form.component.spec.ts` (sin `presentation_id`) y cualquier spec de `diner.service` que
  arme el anuncio (forma nueva `{ text, variant_count, min_qty, value }`); **borrar**
  `pos-heladeria/src/app/modules/promotions/components/scope-picker.component.spec.ts` y
  `pos-heladeria/src/app/modules/presentations/pages/presentations-page.component.spec.ts`.

- [X] T064 No-regresión backend en `../pos-backend`:
  `python -m unittest discover -s app/characterization_tests -v` +
  `python app/scripts/test_promotions_rules.py`. Confirmar que **sin ninguna promoción `active`**
  todos los totales de cobro coinciden con la línea base de T002 (aditividad-segura, research.md D6);
  que los `CONGELA` no relacionados con promociones pasan **sin editar**; que los ficheros de la spec
  040 ya no existen; que la app importa sin `Presentation` / `PromotionTarget` (T061c/T061d).
  Depende de todas las fases de backend (T005–T060) y del retiro de estructura legada (T061a–T061d).

- [X] T065 No-regresión / compilación frontend en `../pos-heladeria`: `ng build` sin errores de tipos
  (los módulos `presentations` y `scope-picker` salieron sin romper el build) + suite `*.spec.ts` en
  verde. Depende de T045, T049, T052, T053, T054, T059, T060, T063.

- [X] T066 Ejecutar `quickstart.md` de punta a punta: Paso 0 (línea base), Paso 1 (migración
  `upgrade`/`downgrade`/`upgrade` contra base real), US1–US6 y §"Verificación final". Verificar los 8
  **SC** de `spec.md`, en particular **SC-005** (reparto al peso, caso de división no exacta),
  **SC-003** (bloqueo/permiso de solape), **SC-006** (migración) y **SC-007** (anuncio del menú).
  Verificación manual del frontend: menú QR dentro/fuera de ventana; formulario de dos tipos con
  selector de conjunto; diálogo del 409 de FR-014; edición limitada de una `Activa`; banner de
  FR-025. Depende de T064, T065.

- [X] T067 Confirmar en `pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` que las 8
  entradas **A-58 … A-65** y la resolución de **A-29** siguen coherentes con lo implementado; dejar
  constancia en el cuerpo del PR / mensajes de commit de que cada reescritura de `CONGELA` cita su
  decisión (Principio III / XII, FR-028). Depende de T066.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — arranca de inmediato.
- **Foundational (Phase 2 / Incremento A)**: depende de Setup. **BLOQUEA todas las historias** — nadie
  empieza US1–US6 hasta que las tareas **activas** de Phase 2 (T005, T006, T008, T009, T012–T015, T017)
  estén completas. **T007, T010, T011 y T016 están diferidas al Incremento F** (T061a–T061d) — ver la
  nota de la Phase 2.
- **User Stories (Phase 3–8)**: todas dependen de Foundational. Además:
  - **US1 (Phase 3)** es la base de US3, US5 y US4 (introduce `variant_ids`, `_apply_variant_set`,
    `variant_set_condition_text`, `_guard_package_is_discount` en `promotions/service.py` y los
    schemas nuevos).
  - **US2 (Phase 4)** depende de US1 (necesita promociones activas con conjunto). Introduce
    `evaluate_variant_sets` / `active_variant_set_promotions`, que US4 reutiliza.
  - **US3 (Phase 5)** depende de US1; comparte `promotions/service.py`.
  - **US5 (Phase 6)** depende de US3 (`change_status → active` revalida `_guard_variant_overlap`).
  - **US6 (Phase 7)** depende de Foundational (T006). Los tests y el banner son verificación + UX; no
    agregan lógica de cálculo.
  - **US4 (Phase 8)** depende de US2 (motor y `active_variant_set_promotions`). Independiente de
    US3/US5/US6.
- **Polish (Phase 9 / Incremento F)**: depende de que todas las historias a entregar estén completas.
  Incluye el retiro de la estructura legada (T061a–T061d: revisión `063b`, borrado de `Presentation` /
  `priority` / targets / combos / `presentation_id` del ORM y del catálogo), que **no podía ir en el
  Incremento A** porque `menu/router.py` (Phase 8) importaba `Presentation` hasta T055–T056.

### User Story Dependencies

- **US1 (P1)**: sin dependencia de otras historias (tras Foundational).
- **US2 (P1)**: depende de US1.
- **US3 (P2)**: depende de US1.
- **US5 (P2)**: depende de US3 (y por transitividad de US1).
- **US6 (P2)**: depende solo de Foundational.
- **US4 (P3)**: depende de US2.

### Ficheros compartidos (secuenciales, NO en paralelo entre sí)

- `pos-backend/app/api/v1/promotions/service.py`: T019 (US1) → T026 → T027 → T028 (US2) → T042 (US3) →
  T046 (US5). Todas secuenciales.
- `pos-backend/app/api/v1/promotions/schemas.py`: T018 (US1) → T043 (US3) → T047 (US5).
- `pos-backend/app/api/v1/promotions/router.py`: T020 (US1) → T043 (US3).
- `pos-backend/app/api/v1/orders/checkout.py`: T029 → T032 (US2). Secuenciales.
- `pos-backend/app/api/v1/cart/service.py`: T037 (US2).
- `pos-backend/app/models/promotion.py`: T009 (aditivo, Phase 2) → T061c (retiro legado, Incremento F).
- `pos-backend/app/api/v1/catalog/{router,schemas}.py`: T061d (Incremento F) únicamente.
- `pos-backend/app/characterization_tests/test_promotions_rules_admin.py`: T021 (US1) → T044 (US3) →
  T048 (US5).
- `pos-heladeria/.../promotions/pages/promotions-page.component.ts`: T025 (US1) → T045 (US3) → T049
  (US5) → T052 (US6). Secuenciales.
- `pos-heladeria/.../promotions/interfaces/promotion.interface.ts`: T022 (US1) → T043-equivalente si
  se toca de nuevo.
- Alembic: revisión **aditiva** `063a` — T005 → T006 (secuenciales, Phase 2). Revisión **destructiva**
  `063b` — T061a → T061b (Incremento F, `down_revision = 063a`).

### Within Each User Story

- Schemas antes que servicio antes que router (mismo endpoint).
- Las funciones de motor (T026 → T027 → T028) antes de cualquier call-site que las invoque
  (T029–T037).
- `build_sale` (T030) e `issue_for_sale` (T031) antes de los call-sites que persisten
  `applied_promotions` (T032–T034).
- Las fixtures (T017, Foundational) antes de cualquier char test nuevo.
- Backend antes que el frontend que lo consume, dentro de la misma historia.

### Parallel Opportunities

- **Setup**: T001 mientras se corre T002; T003/T004 en paralelo con T002.
- **Foundational**: tras T005 → T006 (revisión `063a`) y T008 (aplicarla), los modelos aditivos
  T009/T012/T013/T014 en paralelo (ficheros distintos); T015 tras ellos; T017 en paralelo.
  (T007/T010/T011/T016 diferidas a T061a–T061d.)
- **US1**: T018 (schemas) primero; luego T019 → T020; T022/T023/T024 (frontend) en paralelo con el
  backend; T021 (test) tras T019/T020.
- **US2**: T030/T031 en paralelo tras T012/T013; tras T029: T032 → luego T033/T034/T035/T036 en
  paralelo; T037 tras T028; T038 tras T028; T039/T040/T041 (reescrituras de tests) en paralelo tras
  sus call-sites.
- **US3**: T044 (test) y T045 (frontend) en paralelo tras T042/T043.
- **US5**: T048 (test) y T049 (frontend) en paralelo tras T046/T047.
- **US6**: T050, T052, T053, T054 en paralelo (ficheros distintos); T051 tras T050 + T008.
- **US4**: T057/T058 (tests) y T059 (frontend) en paralelo tras T056; T060 tras T034.
- **Polish**: T061a → T061c → T061d → T061b (secuenciales, retiro de estructura legada) primero;
  luego T061, T062, T063 en paralelo; T064 y T065 en paralelo tras ellos.
- **Historias enteras en paralelo**: **US6 puede desarrollarse en paralelo a US2/US3/US5** (solo
  depende de Foundational, ficheros disjuntos). US1/US2/US3/US5 comparten `promotions/service.py` — no
  en paralelo entre sí.

---

## Parallel Example: Foundational (Phase 2)

```bash
# Tras T005 → T006 (revisión aditiva 063a) y T008 (aplicarla contra Postgres real):
Task: "Modelo promotion.py — SOLO aditivo: PromotionVariant + Promotion.variants + closed_by_refactor_at + ck_promotion_min_qty + ampliar PROMOTION_TYPES/ck_promotion_type con package_price"
Task: "Columna applied_promotions en pos-backend/app/models/sale.py"
Task: "Columna applied_promotions en pos-backend/app/models/invoice.py"
Task: "Columnas discount + applied_promotions en pos-backend/app/models/customer_order.py"

# Y tras esos:
Task: "pos-backend/app/models/__init__.py — AGREGAR export de PromotionVariant (los viejos se conservan)"
Task: "Fixtures: add_variant_to_promotion, borrar make_promotion_target/make_combo_item/presentation_fixtures.py"

# NOTA: borrar presentation.py / presentation_id / el paquete api/v1/presentations/ y
# priority/targets/combo del modelo va en el Incremento F (T061c/T061d), no aquí.
```

## Parallel Example: User Story 2

```bash
# Tras T029 (promo_lines_for + auto_discount) y T030/T031 (build_sale + issue_for_sale):
Task: "pay_order fija order.discount + applied_promotions en checkout.py"   # T032 primero
# luego en paralelo:
Task: "Venta de mostrador en pos-backend/app/api/v1/sales/service.py"
Task: "compute_bill / _close_unified / _close_split en pos-backend/app/api/v1/table_sessions/service.py"
Task: "group_bill en pos-backend/app/api/v1/orders/tables_advanced.py"
Task: "Retirar la rama combo_id de pos-backend/app/api/v1/orders/consolidation.py"

# Tests, en paralelo tras los call-sites:
Task: "Char test test_promotions_service.py con los 10 escenarios + SC-005 (división no exacta)"
```

---

## Implementation Strategy

### MVP First (Incremento A + US1 + US2)

1. Completar Phase 1 (Setup) y Phase 2 (Foundational) — la revisión aditiva `063a` aplica y revierte
   limpio contra Postgres real, **la suite existente sigue en verde**; el modelo nuevo existe pero
   ningún cobro lo usa.
2. Completar Phase 3 (US1) — el administrador ya arma promociones de dos tipos sobre un conjunto
   explícito de variantes, con resumen (FR-005) y bloqueo de FR-016. **DETENER y VALIDAR**.
3. Completar Phase 4 (US2) — el cobro ya combina variantes distintas del conjunto, arma grupos
   completos por consumo codicioso, reparte al peso y persiste `applied_promotions`. **DETENER y
   VALIDAR** con los 10 Acceptance Scenarios de quickstart.md §US2 y SC-005.
4. Este es el valor central de la spec (spec.md "Why this priority" de US2). Desplegar/demostrar.

### Incremental Delivery (los 6 incrementos de research.md D17)

1. **A** — Setup + Foundational → revisión **aditiva `063a`** (tabla puente, columnas nuevas,
   `ck_promotion_min_qty`, `ck_promotion_type` ampliado, paso de datos) + modelo con `PromotionVariant`
   → verificar contra Postgres real **con la suite existente en verde** → demo.
2. **B** — US1 + US3 + US5 → CRUD, `variant_ids`, FR-014, FR-016, FR-018, duplicar, estados,
   permisos → demo (sin que el cobro lo use).
3. **C** — US2 → motor `evaluate_variant_sets` + rewire de los ~9 call-sites + persistencia
   (FR-021) → demo (valor central).
4. **D** — US4 → menú QR (`menu_unit_discount`, anuncio por conjunto) + terminal (condición +
   descuento efectivo) → demo.
5. **E** — Frontend de administración (reparte entre US1/US3/US5/US6: formulario de dos tipos,
   selector de conjunto, diálogos de FR-014/FR-016/FR-018, banner de FR-025, borrado de
   `modules/presentations/` y `scope-picker`/`combo-select`) → demo.
6. **F** — Polish → **retiro de la estructura legada** (revisión destructiva **`063b`**: drop de
   `promotion_targets`/`combo_items`/`presentation_rules`/`presentations`/`priority`/`presentation_id`,
   `ck_promotion_type` estrechado; borrado de `Presentation`/targets/combos del ORM y del catálogo,
   T061a–T061d) + reescritura del script de CI + borrado de los tests de la spec 040 + no-regresión
   completa.

Ningún incremento mezcla la **migración de datos** (solo A, revisión `063a`) con el cambio del cálculo
de cobro (solo C). El **borrado de estructura** que el modelo nuevo deja sin sentido se difiere a F
(revisión `063b`), no porque sea un cambio de comportamiento, sino porque hasta la Phase 8 hay módulos
que importan `Presentation`; así el Incremento A cierra 100% aditivo y verificable (Principio VI).

### Parallel Team Strategy

Con más de una persona, tras Foundational:

1. Persona A: US1 (Phase 3) → US2 (Phase 4) — camino crítico, `promotions/service.py` +
   `checkout.py`.
2. Persona B: US6 (Phase 7) en paralelo desde el inicio (solo depende de Foundational; tests + banner
   + borrado de frontend).
3. Tras US1: Persona B o C toma US3 (Phase 5) y luego US5 (Phase 6) cuando A libere
   `promotions/service.py`.
4. Tras US2: US4 (Phase 8) — menú + terminal, repartible.

---

## Notes

- **[P]** = ficheros distintos, sin dependencia de una tarea sin terminar.
- **[Story]** mapea cada tarea a su historia para trazabilidad (Principio XII); Setup / Foundational /
  Polish no llevan etiqueta.
- **FR-020 (no congelar el descuento)** es una restricción **negativa**: ninguna tarea agrega una
  columna que guarde montos calculados **antes** de emitir; `applied_promotions` (FR-021) es un
  registro del resultado ya cobrado. Se verifica por omisión en T064.
- **Principio VII (datos históricos)**: las columnas `combo_id` / `promotion_id` históricas **no se
  tocan** (T012), `063a` solo agrega estructura y cambia `status` de promociones no emitidas (T006),
  `063b` solo borra estructura ya sin uso (T061a), ninguna `Sale` / `Invoice` se recalcula (T008,
  T051, T061b).
- **A-57 se conserva**: `_valid_now` no cambia de cuerpo (T026); el `check()` de la ventana que cruza
  medianoche entra en el script de CI reescrito (T061) y en los char tests de US3 (T044).
- **Los tests de la spec 040** (`test_promotions_presentation_*`, `test_presentations_service.py`,
  `presentation_fixtures.py`) **no llevan prefijo CONGELA** y se **eliminan** en T062 (Incremento F),
  una vez que las reescrituras de tests las dejan sin uso.
- **FR-028 / Principio III**: toda reescritura de un test `"CONGELA comportamiento actual:"` cita esta
  spec y la decisión A-58…A-65 **en el mismo commit** (T021, T038, T039, T040, T041, T044, T048,
  T057).
- Todo en español de Colombia: nombres de tests nuevos, mensajes de error de negocio (conjunto vacío,
  porcentaje > 100, precio de paquete sin descuento, solape bloqueado, edición de promoción activa) y
  mensajes de commit (Principio XIII).
- Detenerse en cada checkpoint para validar la historia de forma independiente.
