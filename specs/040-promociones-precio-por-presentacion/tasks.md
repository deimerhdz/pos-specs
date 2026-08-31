---

description: "Task list para Promociones de Precio por Cantidad Configuradas por Presentación"
---

# Tasks: Promociones de Precio por Cantidad Configuradas por Presentación

**Input**: Design documents en `/specs/040-promociones-precio-por-presentacion/`

**Prerequisites**: [plan.md](./plan.md) (requerido), [spec.md](./spec.md) (requerido para historias de
usuario), [research.md](./research.md), [data-model.md](./data-model.md),
[contracts/presentaciones-api.md](./contracts/presentaciones-api.md),
[contracts/promociones-precio-por-presentacion.md](./contracts/promociones-precio-por-presentacion.md),
[contracts/cobro-y-preview.md](./contracts/cobro-y-preview.md),
[contracts/menu-qr-anuncio.md](./contracts/menu-qr-anuncio.md), [quickstart.md](./quickstart.md)

**Tests**: **incluidos y NO opcionales para esta spec.** `plan.md` §Constitution Check (Principio X)
y `quickstart.md` fijan de antemano qué characterization test crea o extiende cada historia
(research.md Decisión 14). Los tests nuevos por historia aparecen abajo como tareas de
implementación. Ningún test `"CONGELA comportamiento actual:"` se modifica (research.md D14, FR-016).

**Organization**: tareas agrupadas por historia de usuario (US1–US5, prioridades de `spec.md`) para
que cada una sea implementable y verificable de forma independiente. Esta spec toca **dos
repositorios sibling** de `pos-specs` (Constitución §Alcance, plan.md §Project Structure).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (ficheros distintos, sin dependencia de una tarea sin terminar)
- **[Story]**: historia de usuario a la que pertenece (US1–US5); Setup / Foundational / Polish sin etiqueta
- Cada tarea incluye la ruta de fichero exacta, con el prefijo del repo sibling que corresponda

## Path Conventions

- **Backend**: `pos-backend/app/...` (repo sibling `../pos-backend`)
- **Frontend**: `pos-heladeria/src/app/...` (repo sibling `../pos-heladeria`)
- **Documentación de gobierno**: `pos-specs/specs/...`

---

## Phase 1: Setup

**Purpose**: preparar ramas y confirmar la línea base antes de tocar código (quickstart.md Paso 0).

- [X] T001 [P] Crear la rama `040-promociones-precio-por-presentacion` en `pos-backend` y en
  `pos-heladeria`, partiendo de la rama base de cada repo (hoy ambos en `refactor/promotions`;
  confirmar con el propietario cuál es la base correcta antes de ramificar).
  → Decisión del propietario (2026-08-27): trabajar directamente sobre `refactor/promotions` en
  ambos repos, sin rama nueva.

- [X] T002 Confirmar línea base **en verde** en `pos-backend` (quickstart.md Paso 0), sin tocar
  código todavía:
  `python -m unittest app.characterization_tests.test_promotions_router -v`,
  `python -m unittest app.characterization_tests.test_menu_router -v`,
  `python -m unittest app.characterization_tests.test_orders_checkout -v`,
  `python -m unittest app.characterization_tests.test_cart_service -v`,
  `python app/scripts/test_promotions_rules.py` (único script de promociones en CI). Guardar los
  totales de cobro de referencia para el chequeo de aditividad-segura de T045 (research.md D6).

- [X] T003 Verificar el head real de Alembic en `pos-backend` con `alembic heads` (plan.md asume
  `187e491e597a`). Si el head difiere, usar el valor real como `down_revision` de la migración de
  T004. Confirmar además con `alembic heads` que hay **una sola** cabeza (sin bifurcaciones).

**Checkpoint**: entorno listo, línea base fijada, head de Alembic conocido.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Incremento A (research.md D17) — el catálogo de presentaciones, la tabla hija de reglas,
la columna FK en la variante, el tipo de promoción nuevo en el modelo, la corrección de `_valid_now`
(FR-004 / A-55) y las fixtures de test compartidas. **Ninguna historia de usuario puede empezar
hasta que esta fase esté completa**: US1 necesita presentaciones seleccionables, US2 necesita la
tabla de reglas y la columna `presentation_id`, US3/US4/US5 dependen de ambas, y US1/US5 dependen de
la vigencia correcta al cruzar medianoche (T009a).

**⚠️ CRITICAL**: el trabajo de las historias de usuario no arranca hasta terminar esta fase.

- [X] T004 Crear la migración Alembic
  `pos-backend/alembic/versions/{rev}_presentations_y_reglas_por_presentacion.py`
  (`down_revision` = head verificado en T003). `upgrade` y `downgrade` decorados con
  `@for_each_tenant_schema` + guarda `_has_table(schema, "promotions")` (patrón de
  `d3e4f5a6b7c8_promotions.py` / `e3f4a5b6c7d8_products_tracks_inventory.py`), nombres de constraint
  con `op.f(...)`. `upgrade(schema)`: (1) `create_table("presentations", ...)` — `id`, `name`
  (`String(100)`, `UniqueConstraint` por schema), `active` (`server_default="true"`),
  `created_at`/`updated_at`; (2) `create_table("promotion_presentation_rules", ...)` — FK
  `promotion_id → promotions.id` `ondelete="CASCADE"` + índice, FK `presentation_id → presentations.id`
  `ondelete="CASCADE"`, `min_qty` `CHECK >= 1`, `pack_price` `Numeric(12,2)` `CHECK >= 0`,
  `UniqueConstraint(promotion_id, presentation_id)`; (3) `add_column("product_variants",
  "presentation_id" UUID nullable)` + `create_foreign_key(... ondelete="SET NULL")` +
  `create_index`; (4) `drop_constraint("ck_promotions_type")` + `create_check_constraint` con
  `type IN ('percent','fixed','combo','qty_price','qty_price_presentation')`. `downgrade(schema)`
  exactamente inverso (restaurar el `CHECK` vigente hoy `('percent','fixed','combo','qty_price')`,
  `drop_column` `presentation_id` + FK + índice, `drop_table` de las dos tablas nuevas)
  ([data-model.md](./data-model.md) §Migración y rollback, research.md D15).
  → **Corrección (2026-08-27)**: la migración también hace
  `op.alter_column("promotions", "type", varchar(20) → varchar(50))` (y el `downgrade` inverso
  `50 → 20`). `qty_price_presentation` tiene 22 caracteres y `promotions.type` se creó como
  `varchar(20)` en `d3e4f5a6b7c8`; sin ensanchar, el `INSERT` revienta en PostgreSQL con
  `StringDataRightTruncation` → 500. El modelo (`app/models/promotion.py`) pasa de `String(20)` a
  `String(50)`.

- [X] T005 Aplicar la migración contra una base **real** (PostgreSQL): `alembic upgrade head` →
  `alembic downgrade -1` → `alembic upgrade head`. Verificar: tras `upgrade`, toda variante existente
  tiene `presentation_id IS NULL` (FR-008); tras `downgrade`, el esquema queda idéntico a antes de la
  spec sin pérdida de dato histórico ([data-model.md](./data-model.md) §Rollback; quickstart.md Paso
  1). Depende de T004.
  → **Gap detectado (2026-08-27)**: T005 solo probó aplicar/revertir la migración, **nunca insertó
  una promoción `qty_price_presentation`**, así que no detectó el truncamiento de `promotions.type`
  (`varchar(20)`). Los characterization tests tampoco lo cogen porque **SQLite no valida el ancho de
  `VARCHAR`**. Corregido en T004; `quickstart.md` Paso 1 ahora exige crear una promoción del tipo
  nuevo contra Postgres real.

- [X] T006 [P] Crear el modelo `Presentation` en `pos-backend/app/models/presentation.py`:
  `UUIDPrimaryKeyMixin` + `TimestampMixin`, `__table_args__` con `{"schema": "tenant"}` y
  `UniqueConstraint("name", name=op.f-equivalente)`, columnas `name: Mapped[str]` (String(100), no
  vacío) y `active: Mapped[bool]` (`server_default=text("true")`). Registrar el import en
  `pos-backend/app/models/__init__.py` ([data-model.md](./data-model.md) §Presentation, research.md
  D1).

- [X] T007 [P] En `pos-backend/app/models/promotion.py`: (a) `PROMOTION_TYPES` (línea 30) pasa a
  `("percent", "fixed", "combo", "qty_price", "qty_price_presentation")`; (b) `CheckConstraint`
  `ck_promotion_type` (líneas 92-95) amplía su lista `type IN (...)` con `qty_price_presentation`;
  (c) clase nueva `PromotionPresentationRule(UUIDPrimaryKeyMixin, Base)` — `promotion_id`
  (FK `promotions.id` `ondelete="CASCADE"`, indexado), `presentation_id` (FK `presentations.id`
  `ondelete="CASCADE"`), `min_qty: Mapped[int]` (`CheckConstraint("min_qty >= 1")`),
  `pack_price: Mapped[Decimal]` (`Numeric(12, 2)`, `CheckConstraint("pack_price >= 0")`),
  `UniqueConstraint("promotion_id", "presentation_id")`, `__table_args__` schema `tenant`; (d)
  relación `Promotion.presentation_rules: Mapped[list[PromotionPresentationRule]]` con
  `cascade="all, delete-orphan"` (igual que `targets` / `combo_items`, líneas 84-89).
  `AUTO_TYPES` de `service.py` **NO se toca** ([data-model.md](./data-model.md) §PromotionPresentationRule,
  research.md D3/D4).

- [X] T008 [P] En `pos-backend/app/models/product_variant.py`: columna nueva
  `presentation_id: Mapped[uuid.UUID | None]` (FK `presentations.id`, `ondelete="SET NULL"`,
  `index=True`, nullable) + relación `presentation: Mapped["Presentation" | None]`. Las columnas
  existentes y el `UniqueConstraint("product_id", "name")` no cambian ([data-model.md](./data-model.md)
  §ProductVariant, research.md D1/D2).

- [X] T009 [P] Enum del tipo nuevo en ambos repos, **sin** agregarlo a `AUTO_TYPES` en ninguno
  (paridad, research.md D4): `PromotionType.QTY_PRICE_PRESENTATION = "qty_price_presentation"` en
  `pos-backend/app/api/v1/promotions/schemas.py` y `PromotionType += 'qty_price_presentation'` en
  `pos-heladeria/src/app/modules/promotions/interfaces/promotion.interface.ts` (+ tipo
  `PresentationRule` en la interfaz).

- [X] T009a [P] Corregir `_valid_now` en `pos-backend/app/api/v1/promotions/service.py:94-109`
  (FR-004, CL-8, decisión **A-55** / research.md D18): cuando `start_time > end_time` (ventana que
  cruza medianoche) **y** `now.time() <= end_time` (tramo posterior a la medianoche), la fecha de
  referencia para evaluar vigencia es `(now - timedelta(days=1))` — se compara contra `days_of_week`
  con `.weekday()` y contra `ends_at` con `.date()`; en cualquier otro caso, sin cambio. `starts_at`
  y `start_time` no cambian. Afecta a **todos** los tipos de promoción (`_valid_now` es
  compartido) — por eso el cambio va con entrada en `registro-de-anomalias.md` (A-55), no es
  retroactivo (Principio VII). Agregar un `check()` a
  `pos-backend/app/scripts/test_promotions_rules.py` (sección 1, tras la línea 80): ventana
  `22:00`–`02:00` los "lunes" → vigente lunes 23:00 y martes 01:00 (sigue siendo el día de inicio),
  no vigente martes 03:00 ni miércoles 01:00. **No editar** ningún `check()` existente ni ningún
  test `CONGELA`. Depende de T002 (línea base en verde).

- [X] T010 Crear el paquete backend `pos-backend/app/api/v1/presentations/` (CRUD, Incremento A):
  `router.py` (`GET /presentations` lista paginada con `?active` y `?search`; `POST`; `PATCH /{id}`;
  `DELETE /{id}`; autorización `require_tenant_admin`), `schemas.py`
  (`PresentationCreate` {name}, `PresentationUpdate` {name?, active?}, `PresentationResponse` con
  `applicable_variant_count`), `service.py` (CRUD + `applicable_variants(db, presentation_id)` →
  variantes **activas** que la referencian + **guarda FR-020**: `DELETE` y `PATCH active=false`
  devuelven **409** `{ "error": ..., "promotions": [{id, name}, ...] }` si alguna regla de una
  promoción `status == "active"` referencia la presentación). Montar el router en el agregador de
  `api/v1`. Renombrar **no** está bloqueado por uso
  ([contracts/presentaciones-api.md](./contracts/presentaciones-api.md) §1-4, research.md D10).

- [X] T011 [P] Campo `presentation_id` opcional en el payload de variante,
  `pos-backend/app/api/v1/catalog/schemas.py`: `VariantCreate` y `VariantUpdate` ganan
  `presentation_id: UUID | None = None` (enviar `null` desasigna); `VariantResponse` expone
  `presentation_id`. En el servicio de variantes (`catalog/service.py`), validar que
  `presentation_id` referencia una presentación **existente y activa** o es `null` → 422
  "Presentación no encontrada o inactiva". Cambiar la presentación **no** dispara la verificación de
  uniformidad de FR-017 ([contracts/presentaciones-api.md](./contracts/presentaciones-api.md) §5).

- [X] T012 [P] Fixtures compartidos de characterization tests
  `pos-backend/app/characterization_tests/presentation_fixtures.py`: `make_presentation(db, **kw)`,
  `make_presentation_rule(db, promotion, presentation, min_qty, pack_price, **kw)`, y un helper para
  asignar `presentation_id` a variantes creadas por las fixtures existentes
  (`catalog_fixtures` / `orders_fixtures`). Mismo patrón `_uid()` + `db.add` + `db.flush()` que el
  resto de fixtures del repo; exportar en `__all__`.

- [X] T013 [P] Frontend: módulo nuevo `pos-heladeria/src/app/modules/presentations/` — 
  `interfaces/presentation.interface.ts`, `services/presentation.service.ts` (list/create/update/
  delete, mismo patrón que servicios CRUD existentes),
  `pages/presentations-page.component.ts` (lista + formulario: crear / renombrar / activar /
  desactivar / eliminar). Ruta en el módulo de administración
  ([contracts/presentaciones-api.md](./contracts/presentaciones-api.md) §6).

- [X] T014 [P] Frontend: selector opcional de presentación por variante/tamaño en
  `pos-heladeria/src/app/modules/products/pages/product-form.component.ts`, poblado con las
  presentaciones activas desde `PresentationService`; texto de ayuda "una variante sin presentación
  no participa de promociones por presentación" (FR-008)
  ([contracts/presentaciones-api.md](./contracts/presentaciones-api.md) §6). Depende de T013.

**Checkpoint**: la migración aplica y revierte limpio; el catálogo de presentaciones funciona
(crear, asignar a variantes, la baja de una referenciada por promoción activa se bloquea con 409);
el modelo conoce el tipo `qty_price_presentation` pero ningún cobro lo usa todavía; `_valid_now`
atribuye correctamente el día al cruzar medianoche (FR-004) y el script de CI sigue en verde.

---

## Phase 3: User Story 1 - El administrador configura una promoción con una o más reglas por presentación (Priority: P1) 🎯 MVP

**Goal**: un administrador crea una promoción de tipo `qty_price_presentation`, agrega una o más
reglas (presentación + cantidad mínima + precio de paquete, una por presentación), ve el resumen con
cuántos productos alcanza cada regla, y el sistema le impide reglas duplicadas o que se solapen con
otra promoción activa del mismo tipo.

**Independent Test**: crear una promoción con nombre, fechas, días y horas, agregar dos reglas de
presentaciones distintas, ver el resumen antes de guardar, e intentar agregar una tercera regla
repitiendo una presentación ya usada; con una promoción activa sobre "8oz", intentar activar otra
con regla sobre "8oz" y ver el 409 que nombra el conflicto.

**Depende de**: Phase 2 (Foundational).

### Implementation for User Story 1

- [X] T015 [P] [US1] Schemas de promoción en
  `pos-backend/app/api/v1/promotions/schemas.py`: `PresentationRuleIn` (`presentation_id: UUID`,
  `min_qty: int >= 1`, `pack_price: Decimal >= 0`, 12/2); `PromotionCreate` y `PromotionShapeUpdate`
  aceptan `presentation_rules: list[PresentationRuleIn] | None` + flags opcionales
  `confirm_precio_no_uniforme: bool = False` y `confirm_sin_descuento: bool = False`;
  `PromotionResponse` expone `presentation_rules` con `presentation_name` y `applicable_variant_count`
  por regla. Validadores Pydantic: lista requerida y no vacía cuando
  `type == qty_price_presentation`; sin `presentation_id` repetido (FR-006 1ª parte); `422` si se
  envían `presentation_rules` con otro `type`; `422` si se envían `targets` y `presentation_rules` a
  la vez ([contracts/promociones-precio-por-presentacion.md](./contracts/promociones-precio-por-presentacion.md)
  §2-3, research.md D3/D4).

- [X] T016 [US1] Servicio de promoción en `pos-backend/app/api/v1/promotions/service.py`: en
  `create` y `update_shape`, (a) validar la forma de las reglas (tipo ↔ reglas, cada
  `presentation_id` existe y está activa), refuerzo en servicio de "no dos reglas para la misma
  presentación" (FR-006 1ª parte, además del validador Pydantic y el `UniqueConstraint`); (b)
  **validación de solape entre promociones** (FR-006 2ª parte): si alguna regla apunta a una
  `presentation_id` ya cubierta por una regla de **otra** promoción `type == "qty_price_presentation"`
  con `status == "active"` → **409** `{ "error": ..., "conflicts": [{promotion_id, promotion_name,
  presentation_id}] }`. Invocar la **misma** validación de solape en `change_status` cuando el
  destino es `active` (research.md D8). Ajuste acotado de `find_overlaps` /
  `active_discount_promotions` para reconocer el tipo nuevo donde aplique, sin cambiar su semántica
  para el resto ([contracts/promociones-precio-por-presentacion.md](./contracts/promociones-precio-por-presentacion.md)
  §3, research.md D8). Depende de T015.

- [X] T017 [US1] `POST /promotions/{id}/duplicate` en
  `pos-backend/app/api/v1/promotions/service.py` copia también `presentation_rules` (una fila nueva
  por regla, sin compartir `id`), igual que ya copia `targets` y `combo_items`; la copia nace
  `draft` y el solape de FR-006 se revalida al activarla, no al duplicar
  ([contracts/promociones-precio-por-presentacion.md](./contracts/promociones-precio-por-presentacion.md)
  §5). Depende de T015.

- [X] T018 [US1] Router de promociones en `pos-backend/app/api/v1/promotions/router.py`: mapear el
  conflicto de solape a un **409** legible (sin endpoints nuevos, sin cambio de forma en el PATCH
  escalar / `delete` / `list`)
  ([contracts/promociones-precio-por-presentacion.md](./contracts/promociones-precio-por-presentacion.md)
  §3). Depende de T016.

- [X] T019 [P] [US1] Char test nuevo
  `pos-backend/app/characterization_tests/test_promotions_presentation_rules.py`: (1) crear promoción
  con dos reglas ("8oz" 2×$12.000, "16oz" 2×$16.500) → 201 con `presentation_rules` y
  `applicable_variant_count` por regla (FR-005, CA-1); (2) tercera regla repitiendo "8oz" → 422
  (FR-006 1ª parte, CA-2); (3) con promoción activa sobre "8oz", crear/activar otra
  `qty_price_presentation` con regla sobre "8oz" → 409 nombrando el conflicto, y repetir el intento
  vía `PATCH /promotions/{id}/status {"status":"active"}` sobre una creada en `draft` → mismo 409
  (CA-3, CL-4, research.md D8); (4) ventana `22:00`–`02:00` aceptada como cruce de medianoche al
  guardar (FR-003, CA-4); (5) una promoción `qty_price_presentation` activa con ventana
  `22:00`–`02:00` los "lunes" está vigente el lunes 23:00 **y** el martes 01:00 (atribución de día
  al día de inicio, FR-004/CL-8, A-55), y NO vigente el martes 03:00 ni el miércoles 01:00. Usa
  `presentation_fixtures` (quickstart.md §US1). Depende de T009a, T012, T016, T017.

- [X] T020 [P] [US1] Frontend `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`:
  tipo `qty_price_presentation` en `editableTypes` y en el selector de tipo; editor de reglas
  (agregar/quitar; cada regla con selector de presentación + cantidad mínima + precio de paquete,
  una por presentación); panel lateral "Productos Aplicables" (usa `applicable_variant_count` + la
  lista) y "Resumen de la Regla" (FR-005, mockup de spec §Assumptions); mensaje del 409 de solape
  (FR-006) nombrando la(s) promoción(es) en conflicto
  ([contracts/promociones-precio-por-presentacion.md](./contracts/promociones-precio-por-presentacion.md)
  §6). Depende de T013, T015.
  → Primera versión: editor compacto de filas. **Reimplementado (2026-08-27, ajuste posterior del
  propietario, ver T047)** como el bento del mockup de `spec.md` §Assumptions: tarjetas
  "Información General" + "Configuración de Reglas" a la izquierda, paneles "Productos Aplicables" +
  "Resumen de la Regla" a la derecha (`#presentationForm`). Aplica al crear
  (`@case ('presentation')`) y al editar borrador (`@case ('edit')` con `isDraft()`). Estilo
  Tailwind actual de la app, sin Material 3. Contrato §6.1 actualizado.

- [X] T021 [P] [US1] Frontend
  `pos-heladeria/src/app/modules/promotions/services/promotion-pricing.util.ts`: `getPromoDisplay`
  reconoce `qty_price_presentation` para elegir la insignia (a partir de los datos de la regla);
  el tipo **queda fuera** de `bestProductDiscount` / `discountInfo`, igual que `qty_price` ya lo está
  (research.md D13). Depende de T009.

**Checkpoint**: un administrador ya puede crear, editar, duplicar y activar promociones
`qty_price_presentation` con reglas por presentación, con el resumen de alcance y los bloqueos de
duplicado/solape — **sin** que el cálculo del cobro las use todavía.

---

## Phase 4: User Story 2 - El cajero cobra pedidos que combinan distintos productos de una misma presentación (Priority: P1)

**Goal**: al cobrar, el sistema agrupa las unidades elegibles de una presentación sin importar de
qué producto vienen, forma tantos paquetes completos como permita la regla, cobra cada unidad de
paquete a `pack_price ÷ min_qty` y cada unidad sobrante a un precio unitario normal único (el menor
vigente); el reparto por línea es determinista y no depende del orden de las líneas.

**Independent Test**: con una regla "2 × 8oz por $12.000" activa y variantes de 8oz a $7.000 c/u,
cobrar combinaciones de líneas con distintos productos en 8oz (2, 3, 5 unidades; con y sin otra
presentación en el mismo pedido) y verificar el total y el reparto del descuento por línea, en
cualquier orden de las líneas.

**Depende de**: Phase 3 (US1) — necesita reglas creadas y activas para evaluar.

### Implementation for User Story 2

- [X] T022 [P] [US2] `_presentation_reference_unit_price(...)` en
  `pos-backend/app/api/v1/promotions/service.py`: dado el conjunto de líneas que aportan unidades a
  una presentación en el cobro, devuelve el **menor** `unit_price` vigente entre sus variantes
  elegibles — único para todas esas unidades, nunca variante por variante (FR-011, FR-017; mismo
  criterio `min(...)` que `combo_discount_for_lines`, `service.py:428-431`).

- [X] T023 [US2] `presentation_package_discount_for_lines(db, lines, now) -> PresentationDiscountResult`
  en `pos-backend/app/api/v1/promotions/service.py`, hermana de `combo_discount_for_lines`
  (`service.py:399-448`); devuelve **total** + **desglose `{line_index: Decimal}`**. Algoritmo
  (research.md D5): traer promociones `qty_price_presentation` con `status == "active"` y
  `_valid_now(p, now)`; por regla, reunir unidades cuya variante tenga
  `presentation_id == regla.presentation_id` **y** `ProductVariant.active` (FR-015) **y** la línea no
  venga de un combo (`combo_id is None`); `paquetes = total_unidades // min_qty` (solo completos,
  FR-010); `precio_ref` de T022; precio por unidad de paquete
  `(pack_price / min_qty).quantize(Decimal("1"), ROUND_HALF_UP)` a peso, con el **residuo** asignado
  a la unidad de la línea de **identificador más alto** — `max` sobre `product_variants.id` (UUID);
  desempate: `max` sobre el `id` de la fila de línea (ver data-model.md §"Definición operativa de
  identificador más alto"); unidades sobrantes a `precio_ref`, tomadas de la(s) línea(s) con el
  mismo criterio; descuento por línea derivado (`unidades_línea × precio_ref − cobrado_a_la_línea`),
  la suma cuadra al peso con el total (SC-005). Redondeo del total una sola vez, `ROUND_HALF_UP`
  (patrón `service.py:336,448`). Nunca mezcla
  presentaciones distintas en un paquete (FR-009); presentación sin regla o sin mínimo no descuenta
  (FR-012). Depende de T022.

- [X] T024 [US2] `combined_discount_detailed(db, promo_lines, now) -> CombinedDiscountResult` en
  `pos-backend/app/api/v1/promotions/service.py` (orquestador, research.md D6): ejecuta
  `evaluate_detailed` (línea-por-línea: `percent`/`fixed`/`qty_price` de producto/categoría, una
  sola vez sobre todas las líneas), `combo_discount_for_lines` (líneas con `combo_id`, no compiten)
  y `presentation_package_discount_for_lines` (T023). **Reconciliación con recálculo del pool hasta
  punto fijo** (research.md D6): `pool` = líneas elegibles para alguna regla de presentación
  vigente; se calcula el paquete sobre `pool`; una línea `L` **sale de `pool`** si
  `evaluate_detailed` la deja con total **estrictamente menor** (empate exacto → se queda) o si el
  descuento por presentación la deja peor que sin promoción (FR-023); si alguna salió, **se
  recalcula el paquete sobre el `pool` más chico** y se repite (converge en ≤ |pool| iteraciones,
  `pool` solo se encoge). Ninguna línea acumula dos descuentos (FR-013). Devuelve `total` +
  `promotion_id` único cuando una sola promoción explica todas las líneas descontadas (misma regla
  que `single_promotion_id`, `service.py:191-195`). **No** se migra el resto de `evaluate` →
  `evaluate_detailed` (deuda de spec 012, fuera de alcance, Principio V)
  ([contracts/cobro-y-preview.md](./contracts/cobro-y-preview.md) §1, §3). Depende de T023.

- [X] T025 [US2] `promo_lines_for` en `pos-backend/app/api/v1/orders/checkout.py:240-256`: agregar
  a cada dict de línea los campos que `presentation_package_discount_for_lines` (T023) y
  `_presentation_reference_unit_price` (T022) necesitan y que hoy el dict NO lleva —
  `presentation_id`, `product_variant_id`, `unit_price` y el `id` de la fila de línea (desempate de
  FR-011)— resueltos desde la variante / la línea junto con `product_id`/`category_id`. `evaluate` y
  `combo_discount_for_lines` ignoran los campos nuevos (sin regresión). Los demás call-sites que
  construyen sus propias listas de línea (`table_sessions/service.py`, `sales/service.py`,
  `cart/service.py`) agregan los mismos campos
  ([contracts/cobro-y-preview.md](./contracts/cobro-y-preview.md) §1).

- [X] T026 [US2] Reemplazar el par `promotions.evaluate(...)` + `promotions.combo_discount_for_lines(...)`
  (y el cálculo de `final_promotion_id`) por una sola llamada a
  `promotions.combined_discount_detailed(db, promo_lines_for(db, lines), now)` en los 5 call-sites de
  `pos-backend/app/api/v1/orders/checkout.py`: `pay_order` (275), `checkout_and_send` (466),
  `approve_payment_attempt` (849), `confirm_cash` (969), `compute_bill` (136)
  ([contracts/cobro-y-preview.md](./contracts/cobro-y-preview.md) §2). Depende de T024, T025.

- [X] T027 [P] [US2] Idem en `pos-backend/app/api/v1/table_sessions/service.py`: `compute_bill`
  (186, por comensal), `release_paid_session` (656), `_close_split` (753). La unidad de agrupación es
  el subconjunto de líneas evaluadas juntas en ese cobro — en split, por comensal (research.md D16)
  ([contracts/cobro-y-preview.md](./contracts/cobro-y-preview.md) §2). Depende de T024, T025.

- [X] T028 [P] [US2] Idem en `pos-backend/app/api/v1/sales/service.py` (venta de mostrador, 279) y
  `pos-backend/app/api/v1/orders/tables_advanced.py` (`group_bill`, 155)
  ([contracts/cobro-y-preview.md](./contracts/cobro-y-preview.md) §2). Depende de T024, T025.

- [X] T029 [P] [US2] Carrito del comensal por QR en `pos-backend/app/api/v1/cart/service.py`:
  `serialize_cart` (269) y el snapshot de `submit_cart` (~635) reflejan el descuento por presentación
  en `discounted_total` llamando `combined_discount_detailed` sobre las líneas del carrito cuando las
  cantidades alcanzan algún `min_qty` vigente. **No se persiste** (FR-014); se recalcula en cada
  `GET /cart` y al cobrar. El precio unitario por variante del menú
  (`menu/router.py:157`, `best_line_discount`) **no** cambia
  ([contracts/cobro-y-preview.md](./contracts/cobro-y-preview.md) §4). Depende de T024.

- [X] T030 [P] [US2] Char test nuevo
  `pos-backend/app/characterization_tests/test_promotions_presentation_pricing.py`: los 10 escenarios
  de cálculo de quickstart.md §US2 (tabla #1–#10: paquete con productos distintos → $12.000; 3
  unidades → $19.000 con la suelta decidida por identificador de variante más alto; mismo pedido en
  otro orden → total y reparto idénticos; 8oz+16oz → $28.500 sin mezclar presentaciones; 5 × 8oz →
  $31.000; ninguna alcanza el mínimo → $16.500 sin etiqueta; día no incluido → sin descuento;
  ventana `08:00`–`22:00` a las 07:59 vs 08:01; **regla con división no exacta "3 × 8oz por
  $10.000" → total $10.000 exacto, residuo de $1 a la línea de identificador de variante más alto,
  Σ descuentos por línea cuadra al peso — CL-9**; **1× 8oz activa + 1× 8oz con variante
  `active=false` → $14.000, 0 paquetes porque la variante desactivada no es unidad elegible —
  CL-1c/FR-015**) + **SC-005** (Σ descuento por línea cuadra al peso, cualquier orden) + CA-5
  (determinismo). Usa `presentation_fixtures` (quickstart.md §US2). Depende de T023, T026.

- [X] T031 [US2] Casos nuevos en `pos-backend/app/characterization_tests/test_orders_checkout.py`
  (**sin tocar los tests `CONGELA`**): (1) una línea elegible a la vez para un `qty_price` a nivel
  de producto y para una regla por presentación → recibe el descuento de **una sola** (la de menor
  total para esa línea), nunca la suma, nunca deja la línea peor que sin promoción (FR-013/FR-023);
  (2) **recálculo del pool** (research.md D6): 3× X (8oz $7.000) + 1× Y (8oz $7.000), regla
  presentación "2×8oz $12.000" **activa** + `qty_price` de producto "3×X por $15.000" → X se
  adjudica al descuento de producto ($15.000); Y, que sola no completa un paquete, se cobra a precio
  normal ($7.000); **total $22.000, NO $21.000** (Y no conserva el precio de paquete tras salir X
  del pool). Depende de T024, T026.

**Checkpoint**: US1 + US2 funcionan juntas — el cobro combina productos de una misma presentación,
forma paquetes completos, reparte el descuento por línea de forma determinista y coexiste con las
promociones `qty_price` de producto sin acumular ni empeorar.

---

## Phase 5: User Story 3 - El sistema advierte si los productos de una presentación no tienen precio uniforme (Priority: P2)

**Goal**: al guardar una regla, el sistema verifica que todas las variantes que alcanza tengan el
mismo precio en esa presentación; si difieren, exige confirmación explícita antes de guardar. Igual
para el caso en que la regla no representa un descuento real.

**Independent Test**: con dos productos en la misma presentación a precios distintos, crear una
regla sobre esa presentación y verificar que el sistema muestra la diferencia y bloquea el guardado
hasta confirmación explícita; y que una regla `2 × 8oz por $14.000` con precio de referencia $7.000
también exige confirmación.

**Depende de**: Phase 3 (US1) — extiende `create` / `update_shape` de promociones y el fichero de
test de US1. `reference_unit_price` reutiliza `_presentation_reference_unit_price` (T022, Phase 4);
si US3 se implementa antes que US2, crear ese helper aquí.

### Implementation for User Story 3

- [X] T032 [US3] Verificación de **uniformidad de precio** (FR-017) en
  `pos-backend/app/api/v1/promotions/service.py`, **solo** en `create` y `update_shape` (nunca
  retroactivo, FR-018): para cada regla, reunir las variantes **activas** que referencian su
  `presentation_id` y sus precios vigentes; si no todos son iguales y
  `confirm_precio_no_uniforme == False` → **422** `{ "error": ..., "presentation_id": ...,
  "reference_unit_price": <el menor>, "variants": [{variant_id, description, price}, ...] }`. Con el
  flag en `True` → se guarda (nunca en silencio)
  ([contracts/promociones-precio-por-presentacion.md](./contracts/promociones-precio-por-presentacion.md)
  §4). Depende de T016.

- [X] T033 [US3] Verificación de **"no es descuento real"** (FR-022) en el mismo punto, mismo
  patrón: si `pack_price / min_qty >= reference_unit_price` y `confirm_sin_descuento == False` →
  **422** `{ "error": ..., "presentation_id": ..., "pack_unit_price": <pack_price/min_qty>,
  "reference_unit_price": ... }`. Con el flag en `True` → se guarda; el motor igual nunca deja una
  línea peor que sin promoción en el cobro (FR-023, T024)
  ([contracts/promociones-precio-por-presentacion.md](./contracts/promociones-precio-por-presentacion.md)
  §4). Depende de T032.

- [X] T034 [P] [US3] Char tests en
  `pos-backend/app/characterization_tests/test_promotions_presentation_rules.py` (fichero de T019):
  (1) dos productos en "8oz" a $7.000 y $8.000, regla sin flag → 422 con detalle
  (`reference_unit_price == 7000`), reenviar con `confirm_precio_no_uniforme=true` → 201 (CA-10);
  (2) promoción ya activa, un producto cambia de precio después → **no** se revalida (FR-018, CL-1);
  (3) variante nueva en "8oz" con precio distinto con la promoción activa → entra sin pasar por la
  verificación (CL-1b); (4) regla `2 × 8oz por $14.000` con `reference_unit_price` $7.000 sin flag →
  422 (FR-022), con `confirm_sin_descuento=true` → se guarda (quickstart.md §US3). Depende de T032,
  T033.

- [X] T035 [P] [US3] Frontend
  `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`: diálogos de
  confirmación para FR-017 y FR-022 — leen el detalle estructurado del 422 (variantes/precios,
  `reference_unit_price`, `pack_unit_price`) y, al confirmar, reenvían el mismo payload con el flag
  correspondiente en `true`
  ([contracts/promociones-precio-por-presentacion.md](./contracts/promociones-precio-por-presentacion.md)
  §4, §6). Depende de T020.

**Checkpoint**: US1 + US2 + US3 funcionan de forma independiente — ninguna regla con precios no
uniformes o sin descuento real se guarda en silencio, y el chequeo nunca corre retroactivamente.

---

## Phase 6: User Story 4 - Un producto nuevo con la presentación de una regla activa entra automáticamente a la promoción (Priority: P2)

**Goal**: un producto o variante creado con la presentación de una regla de una promoción activa
queda alcanzado por la regla en el siguiente cobro, sin editar la promoción; y una presentación
referenciada por una regla activa no se puede eliminar ni desactivar.

**Independent Test**: con una regla activa sobre "8oz", crear un producto nuevo con una variante en
"8oz" y verificar que un pedido con ese producto combina para el paquete sin haber tocado la
promoción; e intentar eliminar/desactivar la presentación "8oz" y ver el 409 con la lista de
promociones.

**Depende de**: Phase 2 (guarda FR-020 ya implementada en T010) y Phase 4 (US2) para la verificación
de entrada automática en el cálculo del cobro. Esta historia **no agrega código de producción
nuevo** — el alcance por referencia sale del diseño (research.md D11) y FR-020 ya está en T010; la
fase la verifica y pule la UX del 409.

### Implementation for User Story 4

- [X] T036 [P] [US4] Char test nuevo
  `pos-backend/app/characterization_tests/test_presentations_service.py`: (1) crear "8oz", crearla de
  nuevo → 409 unicidad; (2) asignar `presentation_id` de "8oz" a variantes de dos productos
  distintos → `applicable_variant_count == 2` (FR-007); (3) crear un producto/variante **nuevo** con
  "8oz" → `applicable_variant_count == 3` sin tocar ninguna promoción (FR-019, CA-9); (4) con una
  promoción `qty_price_presentation` **activa** con regla sobre "8oz": `DELETE /presentations/{id}` y
  `PATCH {active:false}` → **409** con la lista de promociones (FR-020, CL-2); pausar la promoción →
  la baja procede y las variantes quedan con `presentation_id NULL` (quickstart.md §US4 / "US4
  parte"). Usa `presentation_fixtures`. Depende de T010, T012.

- [X] T037 [P] [US4] Caso en
  `pos-backend/app/characterization_tests/test_promotions_presentation_pricing.py` (fichero de T030):
  con una regla activa sobre "8oz", crear una variante nueva en "8oz" **después** de activar la
  promoción y verificar que un pedido con esa variante combina para el paquete en el siguiente cobro
  sin editar la promoción (FR-019, CA-9; el alcance se resuelve por `presentation_id` en cada cobro,
  research.md D11). Depende de T023, T030.

- [X] T038 [P] [US4] Frontend `pos-heladeria/src/app/modules/presentations/`: el mensaje del 409 de
  FR-020 muestra la lista de promociones que referencian la presentación y enlaza a cada una; texto
  que pide editar o pausar esas promociones antes de continuar
  ([contracts/presentaciones-api.md](./contracts/presentaciones-api.md) §6, FR-020). Depende de T013.

**Checkpoint**: US1–US4 funcionan de forma independiente — los productos nuevos entran solos por
referencia a la presentación y el catálogo protege las presentaciones en uso.

---

## Phase 7: User Story 5 - El cliente ve la promoción anunciada en el menú QR (Priority: P3)

**Goal**: el menú QR público anuncia cada promoción de precio por presentación **vigente en ese
momento** (ventana de día y hora, zona horaria del tenant), con su condición en lenguaje llano, sin
que el cliente agregue nada al carrito.

**Independent Test**: con una promoción activa "2 × 8oz por $12.000", abrir el menú QR sin agregar
productos dentro de su horario y verificar que la condición es visible; volver a abrirlo fuera de
horario y verificar que el anuncio ya no aparece.

**Depende de**: Phase 2 (tipo y reglas en el modelo). Independiente de US2/US3/US4.

### Implementation for User Story 5

- [X] T039 [US5] `pos-backend/app/api/v1/menu/schemas.py`: clase nueva `MenuPromotionAnnouncement`
  (`promotion_id`, `promotion_name`, `rules: [{ presentation_name, min_qty, pack_price, text }]`,
  con `text` legible construido en backend: "Llevando {min_qty} de cualquier sabor en presentación
  {presentation_name} por {pack_price}"). `MenuCategoryResponse` y el `response_model` de
  `public_menu` (`list[MenuCategoryResponse]`) **NO cambian**; `_build_menu` **NO se toca** — es
  entrada del `CONGELA` de `test_menu_router.py`, de `cart_fixtures.py:379` y del endpoint QR
  (research.md D12, [contracts/menu-qr-anuncio.md](./contracts/menu-qr-anuncio.md) §1).

- [X] T040 [US5] Función nueva `_build_menu_promotions(db, now)` en
  `pos-backend/app/api/v1/menu/router.py` (hermana de `_build_menu`, **sin modificar `_build_menu`**):
  devuelve **solo** las promociones `type == "qty_price_presentation"` con `status == "active"` **y**
  `_valid_now(p, now)` verdadero — vigencia en ese instante, hora local del tenant (FR-021,
  aclaración 2026-08-26). Exponerla en: (a) endpoint hermano nuevo `GET /menu/promotions` →
  `list[MenuPromotionAnnouncement]`; (b) clave `"promotions"` en el `dict` del flujo QR con token
  (`menu/router.py:210-214`). Construir `now` como `datetime.now(timezone.utc)` (aware) — no
  arrastrar el bug A-08 ([contracts/menu-qr-anuncio.md](./contracts/menu-qr-anuncio.md) §2,
  research.md D12). Depende de T039.

- [X] T041 [P] [US5] Casos nuevos en `pos-backend/app/characterization_tests/test_menu_router.py`
  (**sin tocar los tests `CONGELA`**; añadir una `TestCase` nueva): (1) promoción vigente en ese
  instante → `_build_menu_promotions` / `GET /menu/promotions` la incluye con el texto legible,
  visible con carrito vacío (CA-11); (2) promoción `active` pero fuera de su ventana de día u hora →
  la lista sale vacía para ella (SC-006); (3) `now` en zona horaria del tenant: ventana que en UTC
  caería fuera pero en hora local está dentro → se anuncia (no arrastra A-08); (4) **regresión**:
  los dos tests `CONGELA` existentes (`test_a08_*`, que hacen `_build_menu(db)[0]...`) siguen en
  verde sin editar (quickstart.md §US5). Depende de T040.

- [X] T042 [P] [US5] Frontend
  `pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts`: banner/sección de anuncio
  visible **sin agregar nada al carrito**, con el `text` de cada regla; si la lista de anuncios
  viene vacía, no se renderiza nada; `pos-heladeria/src/app/core/services/menu.service.ts` agrega
  un método/tipo nuevo para `GET /menu/promotions` (o la clave `promotions` del contexto QR con
  token) — el tipo de retorno de `getMenu()` **no cambia**
  ([contracts/menu-qr-anuncio.md](./contracts/menu-qr-anuncio.md) §3). Depende de T039.

**Checkpoint**: las 5 historias de usuario son funcionales de forma independiente.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de no-regresión que atraviesa todas las historias (quickstart.md
§Verificación final; plan.md §Constitution Check Principios III y X).

- [X] T043 [P] No-regresión backend en `pos-backend`:
  `python -m unittest discover -s app/characterization_tests -v` +
  `python app/scripts/test_promotions_rules.py`. Confirmar que `test_promotions_router.py` (tiene
  `CONGELA`) y el script de CI pasan (percent/fixed/qty_price/combo validan igual con el enum
  ampliado; `_in_time_window`/`_line_discount`/`best_line_discount` sin cambio; **`_valid_now` con
  el `check()` nuevo de FR-004 en verde y los `check()` previos sin editar**, T009a/A-55) y que
  ningún test `"CONGELA comportamiento actual:"` quedó en rojo, en particular los `test_a08_*` de
  `test_menu_router.py` que usan `_build_menu` (C1, research.md D12/D14). Depende de T007, T009a,
  T026, T027, T028, T040.

- [X] T044 [P] No-regresión / compilación frontend en `pos-heladeria`: `ng build --configuration
  development` (o el runner habitual del repo) sin errores de tipos, y la suite `*.spec.ts` en verde.
  Depende de T014, T020, T021, T035, T038, T042.

- [X] T045 Ejecutar `quickstart.md` de punta a punta: Paso 0 (línea base), Paso 1 (migración
  `upgrade`/`downgrade`/`upgrade` contra base real), y las secciones US1–US5. Verificar en particular
  **SC-005** (reparto al peso, incluido el caso #9 de división no exacta), **SC-006** (ventana del
  anuncio) y la **aditividad-segura**: sin ninguna promoción `qty_price_presentation` activa y
  vigente, todos los totales de cobro son idénticos a los de la línea base de T002 (research.md D6)
  — **salvo** el cambio esperado de `_valid_now` (A-55): una promo de cualquier tipo con ventana que
  cruza medianoche + días restringidos ahora está vigente en el tramo posterior a la medianoche del
  día de inicio. Verificación manual del frontend: menú QR dentro/fuera de ventana, panel de
  promociones (editor de reglas, paneles, diálogos, 409 de solape), CRUD de presentaciones y 409 de
  FR-020. Depende de T043, T044.
  → Ejecutado (2026-08-27): Paso 0 (línea base en verde), Paso 1 (`upgrade`→`downgrade`→`upgrade`
  contra PostgreSQL real, `presentation_id IS NULL` para las 14 variantes existentes), y US1–US5
  vía la suite de characterization (487 tests en verde) + `test_promotions_rules.py`. SC-005
  cubierto por `test_9_division_no_exacta_residuo_al_peso`; SC-006 por
  `TestMenuPromotionsAnnouncementUS5`; aditividad-segura por la suite completa sin regresión.
  Frontend verificado vía `ng build --configuration development` (sin errores de tipos) y la suite
  `*.spec.ts` (mismos 12 fallos preexistentes que en la línea base, ninguno nuevo). El
  click-through manual en navegador queda pendiente para el propietario.

- [X] T046 Confirmar que la **única** entrada en
  `pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` de esta spec es **A-55** (FR-004,
  corrección de `_valid_now`, T009a): la modalidad de descuento en sí se suma y no requiere registro
  (research.md D14/D18; `spec.md` §"Naturaleza de esta spec" y §"Out of Scope", 5º ítem; FR-016).
  Dejar constancia en el cuerpo del PR / mensaje de commit de que A-55 quedó registrada y de que no
  hace falta ninguna otra entrada.

- [X] T047 [P] [Frontend] **Ajuste posterior del propietario (2026-08-27)**, sobre `pos-heladeria`:
  (a) **rediseño del formulario** de `qty_price_presentation` según el mockup de `spec.md`
  §Assumptions — `promotions-page.component.ts`, nuevo template `#presentationForm` con tarjetas
  "Información General" + "Configuración de Reglas" y paneles "Productos Aplicables" + "Resumen de
  la Regla" (cumple FR-005 de forma explícita); aplica al **crear** (`@case ('presentation')`) y al
  **editar borrador** (`@case ('edit')` con `isDraft()`); las promociones ya activas siguen en la
  vista de solo lectura. Estilo Tailwind actual de la app (índigo/gris, SVG inline) — **sin**
  Material 3 / Google Fonts / Material Symbols. (b) **Retirada de `qty_price` de la creación**: la
  tarjeta "Paquete" sale del selector "¿Qué quieres crear?" (`chooseKind` → `'discount' |
  'presentation'`); las promociones `qty_price` existentes se siguen listando y editando (tipo en
  `editableTypes`, `@case ('pack')` intacto). **Backend sin cambios**: `qty_price` sigue soportado
  en modelo, `AUTO_TYPES`, motor y API.
  Ficheros: `promotions-page.component.ts` (+ `.spec.ts`, 3 tests nuevos).
  Verificación: `ng build --configuration development` sin errores; `ng test` = 12 fallos
  preexistentes, 0 nuevos, +3 tests nuevos en verde (470 total). Contrato §6.1/§6.2, `plan.md` y
  `quickstart.md` §Frontend actualizados. Click-through manual pendiente para el propietario.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — arranca de inmediato.
- **Foundational (Phase 2)**: depende de Setup. **BLOQUEA todas las historias de usuario** — nadie
  empieza US1–US5 hasta que T004–T014 estén completas.
- **User Stories (Phase 3–7)**: todas dependen de Foundational. Además:
  - **US1 (Phase 3)** es la base de US3 y US4 (extiende `create`/`update_shape` y el catálogo).
  - **US2 (Phase 4)** depende de US1 (necesita reglas activas). Introduce
    `_presentation_reference_unit_price` (T022) que US3 reutiliza — si se hace US3 antes que US2,
    T022 se crea en la Phase 5.
  - **US3 (Phase 5)** depende de US1; comparte helper con US2.
  - **US4 (Phase 6)** depende de Foundational (T010 ya trae la guarda FR-020) y de US2 para verificar
    la entrada automática en el cobro (T037). No agrega código de producción nuevo.
  - **US5 (Phase 7)** depende solo de Foundational — independiente de US2/US3/US4.
- **Polish (Phase 8)**: depende de que todas las historias a entregar estén completas.

### User Story Dependencies

- **US1 (P1)**: sin dependencia de otras historias (tras Foundational).
- **US2 (P1)**: depende de US1.
- **US3 (P2)**: depende de US1.
- **US4 (P2)**: depende de Foundational + US2 (para T037).
- **US5 (P3)**: sin dependencia de otras historias (tras Foundational).

### Within Each User Story

- Schemas antes que servicio antes que router (mismo endpoint).
- La función de cálculo (T023, T024) antes de cualquier call-site que la invoque (T026–T029).
- `_presentation_reference_unit_price` (T022) antes de `presentation_package_discount_for_lines`
  (T023) y antes de las verificaciones de US3 (T032, T033).
- El fichero de fixtures (T012) antes de cualquier char test nuevo.
- Backend antes que el frontend que lo consume, dentro de la misma historia.

### Ficheros compartidos (secuenciales, no paralelos entre sí)

- `pos-backend/app/models/promotion.py`: T007 (Foundational).
- `pos-backend/app/api/v1/promotions/service.py`: T009a (Foundational, solo `_valid_now`) → T016,
  T017 (US1) → T022, T023, T024 (US2) → T032, T033 (US3). Todas secuenciales.
- `pos-backend/app/api/v1/promotions/schemas.py`: T009 (Foundational) → T015 (US1).
- `pos-backend/app/api/v1/orders/checkout.py`: T025, T026 (US2) — secuenciales.
- `pos-backend/app/api/v1/table_sessions/service.py`: T027 (US2).
- `pos-backend/app/characterization_tests/test_promotions_presentation_rules.py`: T019 (US1) → T034
  (US3).
- `pos-backend/app/characterization_tests/test_promotions_presentation_pricing.py`: T030 (US2) → T037
  (US4).
- `pos-heladeria/.../promotions/pages/promotions-page.component.ts`: T020 (US1) → T035 (US3).
- `pos-heladeria/.../modules/presentations/`: T013 (Foundational) → T038 (US4).

### Parallel Opportunities

- **Setup**: T001 mientras se prepara el resto.
- **Foundational**: tras T004/T005, los modelos T006/T007/T008 en paralelo (ficheros distintos);
  T009, T009a, T011, T012 en paralelo (T009a solo toca `_valid_now`, aislado); T013/T014 (frontend)
  en paralelo con el backend. T010 depende de T006/T007/T008 (usa los tres modelos).
- **US1**: T015 (schemas) y T020 (frontend) y T021 (util) pueden avanzar en paralelo tras
  Foundational; T019 (test) en paralelo una vez T016/T017 listas.
- **US2**: T022 independiente; T027 y T028 en paralelo entre sí (ficheros distintos) tras T024/T025;
  T029 en paralelo; T030 en paralelo tras T023/T026.
- **US3**: T034 (test) y T035 (frontend) en paralelo tras T032/T033.
- **US4**: T036, T037 y T038 en paralelo (ficheros distintos).
- **US5**: T041 (test) y T042 (frontend) en paralelo tras T040.
- **Polish**: T043 (backend) y T044 (frontend) en paralelo.
- Historias enteras en paralelo: **US5 puede desarrollarse en paralelo a US2/US3/US4** (solo depende
  de Foundational, ficheros disjuntos). US2/US3 comparten `promotions/service.py` — no en paralelo.

---

## Parallel Example: Foundational (Phase 2)

```bash
# Tras T004 (migración) y T005 (aplicarla), los modelos en paralelo:
Task: "Modelo Presentation en pos-backend/app/models/presentation.py"
Task: "PromotionPresentationRule + enum de tipo en pos-backend/app/models/promotion.py"
Task: "Columna presentation_id en pos-backend/app/models/product_variant.py"

# Y en paralelo con lo anterior:
Task: "Campo presentation_id en el payload de variante, pos-backend/app/api/v1/catalog/schemas.py"
Task: "Fixtures presentation_fixtures.py en pos-backend/app/characterization_tests/"
Task: "Módulo frontend modules/presentations/ en pos-heladeria"
```

## Parallel Example: User Story 2

```bash
# Tras T024 (combined_discount_detailed) y T025 (promo_lines_for):
Task: "Integrar combined_discount_detailed en table_sessions/service.py (3 call-sites)"
Task: "Integrar combined_discount_detailed en sales/service.py y orders/tables_advanced.py"
Task: "Reflejar el descuento por presentación en el carrito del comensal, cart/service.py"

# Tests, tras los call-sites de checkout.py (T026):
Task: "Char test test_promotions_presentation_pricing.py con los 10 escenarios + SC-005 (incl. residuo y variante desactivada)"
```

---

## Implementation Strategy

### MVP First (Incremento A + US1 + US2)

1. Completar Phase 1 (Setup) y Phase 2 (Foundational) — el catálogo de presentaciones ya es
   utilizable de forma aislada (crear, asignar a variantes, baja bloqueada).
2. Completar Phase 3 (US1) — el administrador ya configura promociones con reglas por presentación,
   con resumen de alcance y bloqueos de duplicado/solape. **DETENER y VALIDAR**.
3. Completar Phase 4 (US2) — el cobro ya aplica el descuento de paquete por presentación combinando
   productos distintos. **DETENER y VALIDAR** con los 10 escenarios de quickstart.md §US2.
4. Este es el valor central de la spec (spec.md "Why this priority" de US2). Desplegar/demostrar.

### Incremental Delivery

1. Setup + Foundational → catálogo de presentaciones (Incremento A) → demo.
2. + US1 → configuración de promociones con reglas → demo.
3. + US2 → cobro con paquetes por presentación (valor central) → demo.
4. + US3 → avisos de precio no uniforme / sin descuento real → demo.
5. + US4 → entrada automática verificada + UX del 409 de baja bloqueada → demo.
6. + US5 → anuncio en el menú QR → demo.
7. Polish → no-regresión completa (cuerpo de los `CONGELA` intacto; script de CI con el `check()`
   nuevo de FR-004 y los previos sin editar; aditividad-segura salvo el cambio esperado de A-55).

### Parallel Team Strategy

Con más de una persona, tras Foundational:

1. Persona A: US1 (Phase 3), luego US2 (Phase 4) — camino crítico, `promotions/service.py`.
2. Persona B: US5 (Phase 7) en paralelo desde el inicio (solo depende de Foundational).
3. Tras US1: Persona B o C toma US3 (Phase 5) cuando A libere `promotions/service.py`.
4. Tras US2: US4 (Phase 6) — mayormente tests y UX, repartible.

---

## Notes

- **[P]** = ficheros distintos, sin dependencia de una tarea sin terminar.
- **[Story]** mapea cada tarea a su historia para trazabilidad (Principio XII).
- **FR-016 (promociones `qty_price` de producto intactas)** y **`AUTO_TYPES` sin cambio** son
  restricciones **negativas**: no generan tarea propia, se verifican por omisión en T043 (ningún
  test `CONGELA` ni el script de CI se editan) y en T045 (aditividad-segura).
- **FR-004 (atribución de día al cruzar medianoche)** es la **única** excepción: cambia el cuerpo de
  `_valid_now` para todos los tipos de promoción (T009a). Va con entrada **A-55** en
  `registro-de-anomalias.md` (Principio II); no es retroactivo (Principio VII). El anuncio del menú
  (T040) **no** cambia `_build_menu` — usa una función hermana + endpoint `GET /menu/promotions`
  para no tocar el `CONGELA` de `test_menu_router.py` (research.md D12).
- **FR-014 (descuento nunca persistido)**: ninguna tarea agrega una columna que guarde montos
  calculados; el desglose se recalcula en cada cobro/preview.
- **FR-007/FR-019 (alcance por referencia)**: el conjunto de variantes de una regla se resuelve
  SIEMPRE por `product_variants.presentation_id`, nunca por `ProductVariant.name` (T023, T036, T037).
- **Principio VII (datos históricos)**: ninguna `Sale`/`SaleInvoice`/`Payment` se recalcula; la
  migración solo agrega estructura vacía, `presentation_id` nace `NULL`, sin backfill (T004, T005).
- Verificar que T002 (línea base) pasa en verde **antes** de tocar cualquier código de producción.
- Detenerse en cada checkpoint para validar la historia de forma independiente.
- Todo en español de Colombia: nombres de tests, mensajes de error de negocio (solape, uniformidad,
  "no es descuento real", baja bloqueada) y mensajes de commit (Principio XIII).
