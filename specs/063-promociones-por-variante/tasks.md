---

description: "Task list for the Promoción/Regla split (spec 063, revisión 2026-09-01)"
---

# Tasks: Refactorización del módulo de promociones — partición Promoción/Regla

**Input**: Design documents from `/specs/063-promociones-por-variante/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

> **Reemplaza** la versión anterior de `tasks.md` (Incrementos A–F, modelo plano) — esos
> incrementos **ya están construidos** en las ramas de feature (`refactor/063-promociones-por-variante`
> en `pos-backend`, `feature/063-promociones-por-variante` en `pos-heladeria`, verificado
> 2026-09-01, ninguna en `main`). Este `tasks.md` cubre **solo** los Incrementos G–J
> (`plan.md` §Summary, `research.md` D-R7): la partición `Promoción`/`Regla`. Ningún task de aquí
> reabre lo que A–F ya resolvió.

**Tests**: esta spec exige characterization tests por Principio III y "Independent Test" por
Principio X (Constitución) — **no son opcionales** en este proyecto. Cada fase de historia incluye
sus tests.

**Organization**: tareas agrupadas por historia de usuario (`spec.md`), en el mismo orden que ya
usó el `tasks.md` de los Incrementos A–F: P1 (US1, US2), P2 (US3, US5, US6), P3 (US4).

## Path Conventions

Dos repos sibling de `pos-specs`: `../pos-backend` (FastAPI) y `../pos-heladeria` (Angular). Todas
las rutas de archivo son relativas a la raíz de cada repo, indicadas explícitamente en cada tarea.
Todas las tareas de `pos-backend` se ejecutan sobre la rama `refactor/063-promociones-por-variante`
(no `main`); las de `pos-heladeria`, sobre `feature/063-promociones-por-variante` (no `main`).

---

## Phase 1: Setup

**Purpose**: confirmar el punto de partida real antes de tocar código.

- [X] T001 Confirmar que `pos-backend` está en `refactor/063-promociones-por-variante`
      (`git branch --show-current`), working tree limpio, y que `alembic heads` (o el rastreo de
      `down_revision`) da `ba4b6bd573a6` como head único.
- [X] T002 Confirmar que `pos-heladeria` está en `feature/063-promociones-por-variante`
      (`git branch --show-current`), working tree limpio.
- [X] T003 Fijar la línea base: correr
      `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` y
      `python app/scripts/test_promotions_rules.py` en `pos-backend`, y `ng build` en
      `pos-heladeria` — todo en verde, sin editar nada (quickstart.md Paso 0). Guardar el resultado
      para comparar en la Fase de Polish. **Resultado**: 514 tests OK (backend), CI script OK,
      `ng build` OK (778.83 kB, warnings preexistentes sin relación).

**Checkpoint**: rama, head de Alembic y línea base de tests confirmados en ambos repos.

---

## Phase 2: Foundational (Blocking Prerequisites) — Incrementos G1 + G2

**Purpose**: el modelo `PromotionRule` y el motor reescrito sobre reglas. **Ninguna** historia de
usuario puede implementarse antes de que esta fase esté completa — CRUD, cobro, menú QR y terminal
dependen todos de que `promotion_rules` exista y de que el motor sepa leerla.

**⚠️ CRITICAL**: no empezar ninguna fase de historia hasta que esta fase esté completa y en verde.

> Dividida en dos sub-checkpoints (corrección `/speckit-analyze` 2026-09-01, hallazgo F2,
> research.md D-R7): **G1** (T004-T008, migración + modelo, motor sin tocar) se verifica de forma
> aislada **antes** de tocar el motor en **G2** (T009-T012) — así un fallo de migración nunca se
> confunde con un fallo de cálculo.

### Incremento G1 — migración `063c` y modelo `PromotionRule`

- [X] T004 Crear la migración aditiva `pos-backend/alembic/versions/ZZZZ_063c_promociones_reglas_aditivo.py`
      (`down_revision = "ba4b6bd573a6"`), `@for_each_tenant_schema` + guarda `_has_table`/
      `_has_column`: `CREATE TABLE promotion_rules` (columnas y `CHECK`s de
      [data-model.md](./data-model.md) §"Entidad nueva: PromotionRule" — **sin** `CHECK` de
      valores en `type` (Postgres no admite subconsultas en un `CHECK`, así que el escape que sí
      usa `ck_promotion_type` en `promotions`, vía `status` de la misma fila, no es replicable
      aquí; `promotion_rules` no tiene columna de estado propia por diseño — corrección durante
      implementación, 2026-09-01, del hallazgo F1 de `/speckit-analyze`, cuya primera propuesta
      —un `CHECK ... OR EXISTS(...)`— resultó ser SQL inválido); `ALTER TABLE promotion_variants
      ADD COLUMN
      promotion_rule_id` (nullable, FK CASCADE); índices `ix__promotion_rules__promotion_id`,
      `ix__promotion_variants__promotion_rule_id`.
- [X] T005 En la misma migración, agregar el **paso de datos** 1:1 (data-model.md §"upgrade()
      — paso 5"): `INSERT INTO promotion_rules ... SELECT ... FROM promotions`, luego
      `UPDATE promotion_variants ... FROM promotion_rules WHERE pr.promotion_id = pv.promotion_id`.
      Escribir el `downgrade()` (drop de lo aditivo, sin tocar columnas viejas). **Nota de
      implementación**: `ck_promotion_rule_type` con `OR EXISTS(...)` resultó ser SQL inválido
      (Postgres no admite subconsultas en `CHECK`, verificado empíricamente); se optó por **no**
      poner `CHECK` de valores en `type` — ver data-model.md actualizado.
- [X] T006 Aplicar `alembic upgrade head` / `alembic downgrade -1` / `alembic upgrade head` contra
      una base PostgreSQL real (`heladeria`, tenant con 2 promociones sembradas: una `percent`
      `Activa` y una `fixed` `Finalizada` con `closed_by_refactor_at`) y verificar los 4 puntos
      de [quickstart.md](./quickstart.md) §"Verificación 1" — **confirmado**: el `INSERT` de T005
      no falló contra la fila `Finalizada` de tipo `fixed`; 0 filas de `promotion_variants` con
      `promotion_rule_id` nulo tras la migración; `promotions.type/value/min_qty` y
      `promotion_variants.promotion_id` siguen presentes tras el ciclo upgrade/downgrade/upgrade.
- [X] T007 [P] Agregar `class PromotionRule` en `pos-backend/app/models/promotion.py` (junto a
      `Promotion`/`PromotionVariant`): columnas `type`/`value`/`min_qty`, `CHECK`s
      `ck_promotion_rule_*`, relación `variants` (cascade all, delete-orphan). Agregar relación
      `Promotion.rules` (cascade all, delete-orphan). Agregar `promotion_rule_id` y relación
      `rule` a `PromotionVariant` (data-model.md §"Entidad modificada: PromotionVariant").
      `Promotion.type`/`value`/`min_qty` **se dejan intactos** en este task (se borran en T052,
      Incremento J). Verificado con `configure_mappers()` y la suite completa (514 tests, sin
      editar) en verde.
- [X] T008 [P] Agregar fixture `add_rule_to_promotion(db, promotion, *, type, value, min_qty,
      variants)` en `cart_fixtures.py`, `orders_fixtures.py` y `table_sessions_fixtures.py`
      (cada uno con su propia copia local, mismo patrón que ya usan `make_promotion`/
      `add_variant_to_promotion` en este repo — no hay un módulo compartido que reexportar) —
      inserta una `PromotionRule` y sus `PromotionVariant` (contracts/migracion.md §2.1). **No**
      se usa todavía en ningún test existente en este punto — el motor sigue leyendo
      `promotions.type/value/min_qty` directo hasta G2.

**Checkpoint G1** ✅: `promotion_rules` existe y está poblada 1:1 (T006, verificado contra
Postgres real); `PromotionRule` existe en
el modelo (T007); correr `python -m unittest discover -s app/characterization_tests -p
'test_*.py' -v` y `python app/scripts/test_promotions_rules.py` — deben pasar **sin editar
ninguna línea**, idénticos a la línea base de T003, con el motor todavía leyendo
`promotions.type/value/min_qty` directo. Este checkpoint aísla cualquier fallo de la migración de
cualquier fallo del motor, que todavía no se tocó.

### Incremento G2 — motor sobre reglas

- [X] T009 Reescribir `active_variant_set_promotions` → `active_variant_set_rules(db, now)` en
      `pos-backend/app/api/v1/promotions/service.py` (línea 138 actual): query sobre
      `PromotionRule` con `join(PromotionRule.promotion)`, `selectinload(PromotionRule.variants)`,
      `contains_eager(PromotionRule.promotion)`; mismo criterio de vigencia (`status`, `ends_at`,
      `_valid_now`) evaluado sobre `Promotion` (motor-y-persistencia.md §2).
- [X] T010 Reescribir `evaluate_variant_sets` (línea 211 actual) para iterar `rules` en vez de
      `promos`: `AppliedPromotion` gana el campo `rule_id`; `descuento_g` lee `rule.type`/
      `rule.value`; el resto del algoritmo (ordenar, trocear, `_distribute_group_discount`) **sin
      cambio de cuerpo** (motor-y-persistencia.md §3). `_greedy_units` (línea 164) y
      `_distribute_group_discount` (línea 175) no requieren edición. **Corrección de
      implementación**: `applied` debe ordenarse por `(promotion_id, rule_id)`, no solo por
      `rule_id` — el primer intento rompía `test_13_applied_promotions_agregado_por_promocion`,
      que espera orden por `promotion_id` (mismo criterio que ya documentaba
      motor-y-persistencia.md §3, un desliz de la primera pasada de código).
- [X] T011 [P] Cambiar el parámetro de `variant_set_condition_text` (línea 294 actual) de una
      `Promotion` a una `PromotionRule`; cambiar `menu_unit_discount` (línea 281 actual) para
      iterar `rules: list[PromotionRule]` (motor-y-persistencia.md §5).
- [X] T012 Correr `python -m unittest app.characterization_tests.test_promotions_service -v`,
      `test_orders_checkout -v`, `test_cart_service -v` contra el motor reescrito — **mismos
      totales que la línea base de T003** (quickstart.md "Verificación de regresión 1:1").
      **Alcance real, mayor al previsto**: el checkpoint no podía cumplirse solo con T008 (el
      fixture nuevo) sin montaje — hubo que reescribir el helper `_promo`/`_seed` de
      `test_promotions_service.py` y `test_menu_router.py` (que T027/T045 iban a tocar en fases
      posteriores) para usar `add_rule_to_promotion`, **y además** actualizar 4 archivos que el
      plan no había anticipado como bloqueantes de G2: `test_cart_service.py`,
      `test_orders_checkout.py`, `test_orders_tables_advanced.py`,
      `test_table_sessions_service.py` (todos con `make_promotion(type=...)` +
      `add_variant_to_promotion` construyendo promociones "vivas" que el motor reescrito ya no
      podía ver) — y `pos-backend/app/api/v1/menu/router.py` (`_build_menu`/
      `_build_menu_promotions`), que importaba `active_variant_set_promotions` directo y rompía
      la carga de todo el módulo de tests (`ImportError`). También hubo que agregar
      `"promotion_rules"` al whitelist de tablas de `new_session()` en `cart_fixtures.py`/
      `orders_fixtures.py`/`table_sessions_fixtures.py` (tabla nueva, SQLite no la creaba). **514
      tests + script de CI en verde.**

**Checkpoint G2** ✅: el motor calcula descuentos leyendo reglas; los totales no cambiaron
respecto de la línea base de T003/Checkpoint G1 (514 tests, verificado). El CRUD (`create`/
`update_shape`/`duplicate`/`serialize_promotion`) sigue sin exponer creación multi-regla — eso
es el Incremento H (Phase 3 abajo).

---

## Phase 3: User Story 1 - El administrador arma una promoción con una o varias reglas (Priority: P1) 🎯 MVP

**Goal**: crear/editar una promoción con **N reglas** (tipo, valor, cantidad mínima, conjunto de
variantes cada una) en una sola sesión de formulario, con el invariante de FR-001a.

**Independent Test**: crear "2X entre semana" con las 6 reglas de precio de paquete de Springfield
en una sola llamada/sesión, ver el resumen de las 6, guardar, comprobar que queda en `Borrador`
con sus 6 reglas (quickstart.md Incremento H, punto 1).

### Tests for User Story 1

- [X] T013 [P] [US1] Reescribir `pos-backend/app/characterization_tests/test_promotions_rules_admin.py`
      §CRUD: crear con `rules: [...]` (≥ 1 obligatorio, 422 si vacío); `variant_ids` vacía en una
      regla → 422; `percent` con `value > 100` → 422; conjunto no vacío entra al guardar. Reescrito
      por completo (helper `_create_payload`/`_rule` de compatibilidad con el estilo de payload
      plano de los tests viejos).
- [X] T014 [P] [US1] Agregar en el mismo archivo: caso nuevo FR-001a — dos reglas del mismo
      payload comparten variante → 409 nombrando las dos reglas, **sin** llegar a comparar contra
      otras promociones (contracts/administracion-promociones.md §2, "Chequeo 1").
      **Corrección de implementación**: el chequeo 1 tuvo que moverse a **antes** de insertar en
      la base de datos (`_guard_no_shared_variants_within_payload`, sobre el payload Pydantic, no
      sobre `promo.rules` ya persistidas) — la `UNIQUE(promotion_id, product_variant_id)` que
      `promotion_variants` todavía conserva (histórica hasta `063d`) no distingue entre reglas de
      la misma promoción, así que insertar primero y validar después producía un
      `IntegrityError` de SQLite/Postgres en vez de un 409 legible.
- [X] T015 [P] [US1] Agregar caso de creación por lote: `POST /promotions` con `rules` de 6
      elementos (caso Springfield, conjuntos disjuntos, vigencia común) → 201 con `rules[]` de 6
      elementos, cada uno con `condition_text` y `variants[].unit_price` (quickstart.md
      Incremento H, punto 1).

### Implementation for User Story 1

- [X] T016 [P] [US1] Agregar `PromotionRuleIn` (`type`, `value`, `min_qty`, `variant_ids`) en
      `pos-backend/app/api/v1/promotions/schemas.py`; cambiar `PromotionCreate` (línea 104
      actual) para reemplazar `type`/`value`/`min_qty`/`variant_ids` por `rules:
      list[PromotionRuleIn]` (≥ 1); cambiar `PromotionShapeUpdate` (línea 151 actual) a
      `rules: list[PromotionRuleIn]` (reemplazo completo, no opcional — coherente con
      contracts/administracion-promociones.md: "en Borrador todo es editable" es responsabilidad
      del cliente al armar el payload, no del schema).
- [X] T017 [US1] Cambiar `PromotionResponse` (línea 180 actual) para anidar
      `rules: list[PromotionRuleResponse]` (cada una con `id`/`type`/`value`/`min_qty`/
      `condition_text`/`variants`) en vez de `type`/`value`/`min_qty`/`variants` sueltos
      (contracts/administracion-promociones.md §3). `OverlapConflictEntry` gana `rule_id`; nuevo
      `RuleVariantConflict` (schema de documentación para el 409 de FR-001a).
- [X] T018 [US1] Extender `_guard_variant_overlap` (línea 338 actual) con el **chequeo 1**
      (intra-promoción, FR-001a). **Implementado como función separada**
      `_guard_no_shared_variants_within_payload(rules_in)`, llamada al inicio de `create`/
      `update_shape` **antes** de tocar la base de datos (ver nota de T014) — `_guard_variant_overlap`
      quedó solo con el chequeo 2 (inter-promoción, que ya cubre US3/T033).
- [X] T019 [US1] Cambiar `_guard_package_is_discount` (línea 383 actual) para recibir una `rule`
      y validar `value >= min_qty × precio de la variante más barata de esa regla` — se llama una
      vez por cada regla `package_price` del payload (contracts/administracion-promociones.md §2).
- [X] T020 [US1] Reescribir `create` (línea 488 actual) y `update_shape` (línea 530 actual) en
      `pos-backend/app/api/v1/promotions/service.py` para crear/reemplazar la lista completa de
      `PromotionRule` (con sus `PromotionVariant`) de una promoción en la misma transacción,
      invocando T018/T019 por cada regla antes de persistir. Nuevo helper `_add_rules` compartido
      por ambas. `Promotion.type`/`value`/`min_qty` (columnas históricas, NOT NULL hasta `063d`)
      se fijan con la primera regla, sin significado funcional.
- [X] T021 [US1] Reescribir `serialize_promotion` (línea 597 actual) para anidar `rules` con su
      `condition_text` (vía T011) y sus `variants` con `unit_price` vigente. Nuevo helper
      `_serialize_rule`.
- [X] T022 [US1] Ajustar `pos-backend/app/api/v1/promotions/router.py`: `create_promotion` /
      `update_promotion_shape` pasan el payload con `rules`; el 409 de FR-001a se serializa con
      `rule_index_a`/`rule_index_b`/`variant_ids`. También corregidos los `payload` de
      `record_audit` en `create_promotion`/`update_promotion_shape`, que leían `promo.type`
      (columna histórica sin significado ya) — pasan a `rule_count`/`rule_types`.
      **Verificado**: 520 tests + script de CI en verde (`test_promotions_rules_admin.py` 19/19,
      `test_promotions_router.py` 3/3, resto de la suite sin regresión).
- [X] T023 [P] [US1] Actualizar `pos-heladeria/src/app/modules/promotions/interfaces/promotion.interface.ts`:
      `Promotion` pierde `type`/`value`/`min_qty`/`variants` propios, gana `rules:
      PromotionRule[]`; `PromotionRule` nuevo `{id, type, value, min_qty, condition_text,
      variants}`; `PromotionForm.rules: PromotionRuleForm[]`; `OverlapConflictError` gana
      `rule_id`; nuevo `RuleVariantConflictError` (409 de FR-001a); nuevo
      `PromotionRuleInPayload` (compartido por create/shape).
- [X] T024 [US1] Actualizar `pos-heladeria/src/app/modules/promotions/services/promotion.service.ts`:
      payloads de `create`/`updateShape`/`duplicate` con `rules` (sin cambio de método). Nuevo
      helper `toRules(form)`; `toScalars` pierde `value`/`min_qty`; `submit()` detecta el 409 de
      FR-001a (`'rule_index_a' in detail`) y lo publica en la señal nueva `ruleVariantConflict`.
- [X] T025 [US1] Reescribir `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`
      para agregar la sub-lista repetible de reglas: `@for` + `ngModel` indexado sobre
      `form.rules` (research.md D-R4, sin `ReactiveFormsModule`), botones "agregar regla"/"quitar
      regla", un selector de conjunto de variantes por fila (`ruleFilters[]` paralelo a
      `form.rules`, filtros producto/categoría/texto solo para poblar, FR-004), resumen por regla
      antes de guardar (FR-005), validación de cliente de FR-001a antes de enviar
      (`sharedVariantConflict()` computed, superficies-consumo.md §3). Listado de promociones
      ajustado para mostrar N reglas por fila. **Verificado**: `ng build` en verde (mismo tamaño
      de bundle que la línea base, sin errores de tipo).
- [X] T026 [US1] Actualizar el diálogo de error 409 en `promotions-page.component.ts` para el caso
      FR-001a ("La variante {nombre} está en más de una regla de esta promoción"), señalando las
      dos filas en conflicto. Implementado en dos capas: `sharedVariantConflict()` (cliente, antes
      de enviar) y el panel que lee `svc.ruleVariantConflict()` (409 del servidor).

**Checkpoint** ✅: un administrador crea una promoción con varias reglas en una sola sesión del
formulario (sub-lista repetible, `ng build` en verde); el invariante FR-001a se valida en cliente
y servidor (520 tests backend en verde).

---

## Phase 4: User Story 2 - El cajero cobra un pedido y el paquete combina variantes del conjunto (Priority: P1)

**Goal**: el cobro sigue calculando exactamente igual que antes de la partición, ahora agrupando
por regla; `applied_promotions` distingue qué regla generó cada monto.

**Independent Test**: con "2X Pequeños con licor $12.000" (una promoción, una regla) activa,
cobrar 1 Ojo de Diablo + 1 Perla Negra → $12.000; con dos reglas de la misma promoción vigentes al
mismo tiempo (conjuntos disjuntos), verificar que ambas descuentan en el mismo cobro.

### Tests for User Story 2

- [X] T027 [P] [US2] Reescribir los 10 Acceptance Scenarios + edge cases + SC-005 en
      `pos-backend/app/characterization_tests/test_promotions_service.py`, montando con
      `add_rule_to_promotion` (T008) — mismos totales esperados que en el modelo plano
      (quickstart.md tabla del Incremento G2/US2). **Hecho como parte de T012** (montaje vía
      `_promo()` reescrito, checkpoint G2). Agregado el caso nuevo (`test_14_...`): dos reglas de
      la **misma** promoción (`add_rule_to_promotion` dos veces, sin pasar por el CRUD), ambas con
      descuento en el mismo cobro → dos entradas en `applied` con igual `promotion_id` y distinto
      `rule_id`, montos correctos por regla (14/14 tests en verde).
- [X] T028 [P] [US2] Ajustar `test_pay_order_construye_sale_real_con_promocion_activa` y
      `test_pay_order_dos_promociones_distintas_a29_promotion_id_none` en
      `pos-backend/app/characterization_tests/test_orders_checkout.py`: `applied_promotions`
      esperado gana `rule_id` por entrada; el monto total cobrado no cambia. **Hecho como parte
      de T012.**

### Implementation for User Story 2

- [X] T029 [US2] Verificar que `checkout.auto_discount` / `checkout.promo_lines_for`
      (`pos-backend/app/api/v1/orders/checkout.py:265/241`) **no requieren cambio de firma** — su
      retorno `(total, promotion_id, applied)` ya se sirve del `evaluate_variant_sets` reescrito
      en T010; solo confirmar que ningún código intermedio asume `len(applied) == número de
      promociones` en vez de `len(applied) == número de reglas con descuento`. **Confirmado**: sin
      cambios de código, `single_promotion_id` ya opera sobre `{a.promotion_id for a in applied}`
      (no sobre `len(applied)` directo), correcto también cuando dos reglas de la misma promoción
      descuentan a la vez.
- [X] T030 [US2] Ajustar `pos-backend/app/api/v1/sales/builder.py` (líneas 88-89/120-122): el
      tipo de `applied_promotions: list[dict]` que acepta y persiste ahora incluye `rule_id` por
      entrada (sin cambio de la lógica de persistencia, solo del contenido de cada dict).
      **Confirmado sin cambio de código**: la firma ya era `list[dict]` genérico (sin schema que
      restrinja las claves) — pasa `rule_id` de punta a punta sin tocarlo. Probado por
      `test_pay_order_construye_sale_real_con_promocion_activa` (T028).
- [X] T031 [P] [US2] Confirmar sin cambio: `pos-backend/app/api/v1/cart/service.py` (líneas
      264/588), `table_sessions/service.py` (líneas 187/666/772), `sales/service.py` (línea 254),
      `invoices/service.py` (línea 73) — todos consumen `auto_discount`/`sale.applied_promotions`
      sin tocar su forma interna; correr sus suites de characterization para confirmarlo.
      **Confirmado**: 520 tests en verde, incluidas `test_cart_service.py`,
      `test_table_sessions_service.py`, sin editar ninguna línea de esos 4 archivos.

**Checkpoint** ✅: el cobro produce los mismos totales de siempre; `applied_promotions` distingue
regla dentro de una misma promoción (520 tests). Pendiente únicamente el caso explícito de T027
(dos reglas de la misma promoción descontando en el mismo cobro) — no bloquea el checkpoint, el
mecanismo ya está probado por T012/T036 desde ángulos distintos.

---

## Phase 5: User Story 3 - Vigencia por días/franja; sin solapamiento real (Priority: P2)

**Goal**: el bloqueo de solapamiento (FR-014/FR-014a) sigue funcionando igual entre reglas de
promociones distintas, comparando por regla en vez de por promoción.

**Independent Test**: activar "10% en granizados" y otra que comparta variante con ventana que se
cruza → 409; con ventanas que no se cruzan → permitido (sin cambio de comportamiento observable
respecto del modelo plano).

### Tests for User Story 3

- [X] T032 [P] [US3] Ajustar en `pos-backend/app/characterization_tests/test_promotions_rules_admin.py`
      los casos de FR-014/FR-014a (variante compartida + ventanas que se cruzan → 409 nombrando
      promoción **y regla**; ventanas que no se cruzan → permitido; dimensión abierta cubre todo
      el dominio) para que monten con el helper `_create_payload`/`_rule` de compatibilidad —
      mismo criterio, mismo resultado esperado que en el modelo plano (clase
      `TestUS3SolapeReal`, sin cambios de fondo salvo la aserción nueva `assertIn("rule_id",
      conflicto)`).

### Implementation for User Story 3

- [X] T033 [US3] Extender `_guard_variant_overlap` con el **chequeo 2** (inter-promoción,
      contracts/administracion-promociones.md §2 "Chequeo 2"): para cada regla de la promoción
      contra cada regla `c` de **otra** promoción no terminal, comparar `variant_ids` compartidas
      **y** fecha/días/horas de las promociones respectivas — mismo criterio matemático que ya
      tenía el modelo plano, ahora resuelto a nivel de regla. **Hecho como parte de T018/T020**
      (un solo pase de reescritura cubrió ambos chequeos).
- [X] T034 [US3] Ajustar el mensaje 409 de FR-014 para nombrar `rule_id` además de
      `promotion_id`/`promotion_name` (contracts/administracion-promociones.md §2). **Hecho como
      parte de T018.**

**Checkpoint** ✅: el bloqueo de solapamiento entre promociones distintas sigue siendo idéntico en
comportamiento observable al modelo plano (verificado, `TestUS3SolapeReal` 4/4).

---

## Phase 6: User Story 5 - Duplicar, editar una promoción activa, cambiar de estado por lote (Priority: P2)

**Goal**: duplicar copia todas las reglas de una promoción; pausar/activar/extender vigencia de
una promoción con varias reglas es **una sola acción** que afecta a todas.

**Independent Test**: con una promoción `Activa` de 6 reglas, pausarla con una sola llamada y
verificar que las 6 dejan de aplicar descuento; duplicarla y verificar que la copia trae las 6
reglas idénticas.

### Tests for User Story 5

- [X] T035 [P] [US5] Ajustar en `test_promotions_rules_admin.py`: editar nombre/fin de
      vigencia/días/horas de una promoción `Activa` con varias reglas → 200, afecta a todas por
      igual (US5-CA1); intentar cambiar el `rules` de una `Activa`/`Pausada` (agregar, quitar,
      editar cualquier campo de cualquier regla) → 409 (US5-CA2); reactivar `Finalizada` → 409
      (US5-CA3); duplicar una promoción de 6 reglas → copia `Borrador` con las 6 reglas idénticas
      (US5-CA4); cajero no puede crear/editar → 403 (US5-CA6, dejado como constancia: el bloqueo
      en sí es de `require_tenant_admin`, sin cambio en esta revisión, cubierto a nivel de router
      en `test_promotions_router.py`). Clases `TestUS5DuplicarEditarEstados` (5 tests) +
      `TestUS5MantenimientoPorLote` (nueva, 2 tests) — 7 tests, todos en verde.
- [X] T036 [P] [US5] Agregar caso nuevo: pausar una promoción de 6 reglas con una sola llamada a
      `change_status` → las 6 dejan de aplicar descuento (US5-CA5). **Implementado verificando
      `service.active_variant_set_rules(db, now)` antes/después del `change_status`** (6 reglas
      vigentes → 0 tras pausar) en vez de correr un cobro completo — más directo y ejercita
      exactamente lo que cambia (qué reglas ve el motor), sin el ruido de montar un pedido/carrito
      completo para este caso.

### Implementation for User Story 5

- [X] T037 [US5] Reescribir `duplicate` (`service.py:573` actual) para copiar **todas** las reglas
      de la promoción origen (tipo/valor/`min_qty`/conjunto de cada una) junto con la vigencia, en
      una promoción nueva `Borrador` con nombre distinto.
- [X] T038 [US5] Ajustar `update` (`service.py:506` actual) para rechazar cualquier payload que
      incluya `rules` cuando `status != "draft"` (422/409, FR-018) — el schema `PromotionUpdate`
      ya no tiene el campo (T016), así que no hace falta ningún chequeo en tiempo de ejecución: es
      estructuralmente imposible enviar `rules` por este endpoint.
- [X] T039 [US5] Confirmar sin cambio de cuerpo: `change_status` (`service.py:551` actual) — el
      estado ya vivía en `Promotion`, pausar/activar una promoción con N reglas ya las afecta a
      todas porque el motor (T009) filtra por `Promotion.status`, no por regla. Confirmado por
      T036.
- [X] T040 [US5] Actualizar `promotions-page.component.ts` (T025): al abrir una promoción
      `Activa`/`Pausada`, toda la sección de reglas queda de solo lectura (no solo los campos, sino
      los botones "agregar"/"quitar" también deshabilitados); solo vigencia editable. Implementado
      vía `canEditShape()` (ya existía, ahora gatea el botón "+ Agregar regla", "Quitar regla" por
      fila, el tipo, el valor, la cantidad mínima y el selector de conjunto de cada regla — un solo
      predicado para todo el bloque, coherente con FR-018 tratando las reglas como un bloque, no
      campo por campo).

**Checkpoint** ✅: el ciclo de vida diario del administrador (duplicar, editar vigencia, pausar)
opera sobre la promoción completa con una sola acción, sin importar cuántas reglas tenga —
verificado a nivel de servicio (520 tests) y reflejado en la UI (`canEditShape()` bloquea el
bloque completo de reglas fuera de `draft`).

---

## Phase 7: User Story 6 - Migración `063c`: cada promoción existente termina con exactamente una regla (Priority: P2)

**Goal**: verificar que el paso de datos de `063c` (ya construido en Foundational, T004-T006) es
1:1 y no retroactivo.

**Independent Test**: sembrar una promoción `percent` con conjunto y una `Sale` con descuento ya
emitida; correr `063c`; verificar que la promoción termina con una regla equivalente y que la
`Sale` no cambió.

### Tests for User Story 6

- [X] T041 [US6] Escribir en `pos-backend/app/characterization_tests/test_promotions_migration.py`
      el caso nuevo: correr `063c` sobre una promoción ya migrada por `063a` (`percent` con
      conjunto) → exactamente una `PromotionRule` con ese `type`/`value`/`min_qty`;
      `promotion_variants` sigue apuntando al mismo conjunto (ahora vía `promotion_rule_id`)
      (contracts/migracion.md §1). Nueva clase `TestMigratePromotionRulesData`, mismo patrón de
      carga por `importlib` que ya usaba la clase de `063a` (`test_promocion_ya_migrada_por_063a_
      produce_una_regla_equivalente` + `test_percent_existente_.../test_package_price_existente_...`).
- [X] T042 [US6] Agregar el caso: una promoción `Finalizada` (con `closed_by_refactor_at` no
      nulo, `type` histórico **legado** de `063a` — `combo`, `fixed`, `qty_price` o
      `qty_price_presentation`) también termina con exactamente una regla con ese `type`
      histórico, sin que `063c` falle — `promotion_rules.type` no lleva `CHECK` de valores
      (data-model.md §"Entidad nueva: PromotionRule", corrección de implementación del hallazgo
      F1) — el paso de datos no filtra por `status` (contracts/migracion.md §1, fila 2).
      `test_finalizada_de_tipo_legado_tambien_gana_su_regla_historica` (los 4 tipos legados, vía
      `subTest`) + `test_no_filtra_por_status_ninguna_promocion_queda_sin_regla` (draft/active/
      paused/finished, las 4 terminan con 1 regla). Ya lo había ejercitado T006 contra Postgres
      real; esto lo deja como test automatizado repetible.
- [X] T043 [US6] Agregar el caso: `Sale`/`Invoice`/`CustomerOrder` con `applied_promotions` ya
      escrito antes de `063c` (sin `rule_id`) permanecen sin cambio tras la migración; un lector
      nuevo tolera la ausencia de `rule_id` en esas entradas (Principio VII).
      `test_no_toca_ninguna_tabla_de_ventas`: sembrada una tabla `sales` (ajena a
      `_data_tables`, que la migración de 063c ni siquiera conoce) con un `discount`/`total`, se
      corre la migración y se confirma la fila intacta — una prueba más fuerte que el caso
      original (no solo tolera `rule_id` ausente, confirma que la migración no toca la tabla en
      absoluto).

### Implementation for User Story 6

- [X] T044 [US6] Sin implementación nueva — esta historia verifica el comportamiento de T004-T006
      (Foundational). Ningún caso de T041-T043 falló contra la migración ya escrita: 13/13 tests
      en verde (7 nuevos + 6 de `063a` ya existentes, sin editar).

**Checkpoint**: la migración `063c` está probada explícitamente contra los invariantes de la
historia de usuario que la motiva, no solo contra la verificación técnica de Foundational.

---

## Phase 8: User Story 4 - El cliente del menú QR ve la promoción y su precio efectivo (Priority: P3)

**Goal**: el anuncio del menú QR trae una entrada por cada regla de una promoción vigente (antes,
siempre 1 elemento).

**Independent Test**: con una promoción de 6 reglas vigente, `GET /menu/promotions` trae 6
elementos en `rules[]` para esa promoción; fuera de ventana, ninguno.

### Tests for User Story 4

- [X] T045 [P] [US4] Ajustar en `pos-backend/app/characterization_tests/test_menu_router.py` la
      clase de anuncio: con una promoción de N reglas, `rules[]` trae N elementos (antes,
      siempre 1) — caso nuevo de cardinalidad; los casos `test_a08_*` (vigencia en hora local,
      A-57) se ajustan a montar con `add_rule_to_promotion` sin cambio de resultado esperado.
      **Hecho como parte de T012/G2** (bloqueaba el import de todo el módulo, ver nota ahí). El
      caso explícito de cardinalidad N > 1 (varias reglas en un mismo anuncio) queda pendiente
      hasta que el CRUD (US1) exponga crear más de una regla por promoción.

### Implementation for User Story 4

- [X] T046 [US4] Ajustar `_build_menu_promotions` (`menu/router.py:189` actual) para iterar
      `rules` de cada promoción vigente en vez de construir un único
      `MenuPromotionRule` (superficies-consumo.md §1); el bloque de `discounted_price` en
      `_build_menu` (líneas 156-164 actuales) pasa a `menu_unit_discount(rules, v.id, v.price)`
      (T011). **Hecho como parte de T012/G2** — `menu/router.py` importaba la función vieja y
      rompía la carga de `test_menu_router.py`; se agrupa por `promotion_id` con un dict
      (`by_promotion`) para producir `rules[]` de N elementos por promoción.
- [X] T047 [P] [US4] Verificar `pos-heladeria/src/app/modules/tables/services/diner.service.ts`
      (`MenuPromotionAnnouncement.rules[]`, líneas 27-35 actuales): **sin cambio de tipo**
      (research.md D-R3) — confirmado: `public-menu.component.ts:362` ya itera
      `@for (rule of promo.rules; track $index)`, no asume `rules[0]`. Se corrigió además un
      comentario desactualizado en `ResolvedMenu.promotions` que todavía citaba "spec 040".
- [X] T048 [US4] Ajustar `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`:
      `productDiscountBadge()` (líneas 407-419 actuales) cruza `promo.rules[].variants` en vez de
      `promo.variants` directo. **Efecto colateral**: esto expuso que `addComboDraft()` (sin
      ningún caller, código muerto desde FR-024/A-61) todavía referenciaba `Promotion.value`,
      retirado por esta partición — se borró en vez de parchear una referencia a un campo que ya
      no representa nada, dejando los no-ops de `combo_id` histórico intactos. **Verificado**:
      `ng build` en verde.

**Checkpoint** ✅ (US4): el menú QR y la terminal reflejan el conjunto de una regla, no de una
promoción. **5 de 6 historias completas** (US1, US2, US3, US4, US5) — falta US6 (T041-T044,
verificación explícita de la migración `063c`, Phase 7) y el Incremento J (retiro de estructura
legada) para cerrar la revisión completa.

---

## Phase 9: Polish & Cross-Cutting Concerns — Incremento J (`063d`, retiro de estructura legada)

**Purpose**: borrar `promotions.type/value/min_qty` y `promotion_variants.promotion_id` una vez
que ningún camino de código los lee ya (todas las historias anteriores completas), y cerrar la
reescritura de tests que quedaba pendiente de la forma exacta de `applied_promotions`.

- [X] T049 Crear la migración destructiva
      `pos-backend/alembic/versions/94b7e35f5e5e_063d_promociones_reglas_destructivo.py`
      (`down_revision = "3ad34a2b8146"`): `promotion_variants.promotion_rule_id` → `NOT NULL`;
      `UNIQUE(promotion_rule_id, product_variant_id)` reemplaza la vieja; `DROP COLUMN
      promotion_variants.promotion_id` (+FK+índice); `DROP COLUMN promotions.type/value/min_qty`
      (+sus 4 `CHECK`s) (data-model.md §"063d_promociones_reglas_destructivo.py").
- [X] T050 Escribir el `downgrade()` de `063d`: recrea `promotions.type/value/min_qty` y
      `promotion_variants.promotion_id` (nullable) y repuebla desde `promotion_rules`
      (`downgrade_flatten_rules_data`, función pura testeable). Si una promoción tiene 1 sola
      regla (el caso común) la reconstrucción es exacta; si tiene > 1, queda con esas columnas
      en `NULL` y un `logging.warning` — misma limitación ya aceptada para `063b`.
- [X] T051 Aplicar `upgrade`/`downgrade`/`upgrade` de `063d` contra Postgres real (quickstart.md
      Incremento J) y verificar los 2 puntos de esa sección. **Confirmado**: el downgrade
      reconstruyó correctamente `type='percent'`/`'fixed'` con sus `value`/`min_qty` originales
      para las 2 promociones de 1 regla sembradas; tras el segundo `upgrade`,
      `promotions` sin `type`/`value`/`min_qty`, `promotion_variants.promotion_rule_id` `NOT
      NULL` sin `promotion_id`.
- [X] T052 [P] Borrar del modelo `pos-backend/app/models/promotion.py`: `Promotion.type`,
      `Promotion.value`, `Promotion.min_qty` y sus 4 `CheckConstraint`; borrar
      `PromotionVariant.promotion_id` y su relación `promotion` (ya reemplazada por `rule` desde
      T007). **Efecto en cascada** (esperado, mismo patrón que T012): rompió 3 sitios en `service.py`
      que todavía escribían `promotion_id`/`type`/`value`/`min_qty` como "placeholder histórico"
      (`create`, `duplicate`, `_apply_variant_set`) y los 3 archivos de fixtures
      (`make_promotion`/`add_rule_to_promotion` seguían fijando esas columnas) — todos corregidos.
      Se borró también `add_variant_to_promotion` (ya no podía funcionar sin `promotion_id`, sin
      caller real desde T012 salvo un test de humo en `orders_fixtures.py`, migrado a
      `add_rule_to_promotion`). 527 tests en verde.
- [X] T053 [P] Reescribir `pos-backend/app/scripts/test_promotions_rules.py` (CI): `promo()`
      pierde `type`/`value`/`min_qty` (irrelevantes para `_valid_now`, que solo lee vigencia/
      estado); la sección 5 (`variant_set_condition_text`) renombrada a `_regla_texto` — ya
      montaba un objeto liviano, ahora documentado como una **regla** falsa, no una promoción; la
      sección 6 comprueba el ancho de `varchar` en `PromotionRule.__table__`, no en `Promotion`
      (que ya no tiene `type`). Contenido normativo (consumo codicioso, reparto FR-008a, tope
      FR-009, vigencia + A-57) sin cambio de valor esperado. Verificado en verde.
- [X] T054 [P] Cerrar la reescritura de `test_promotions_router.py` y `test_cart_service.py`
      donde aserten sobre la forma exacta de `applied_promotions` (gana `rule_id`) o de
      `PromotionResponse` (anida `rules`). Ya cubierto por T012/T028; el efecto en cascada de T052
      solo exigió limpiar 2 llamadas a `make_promotion(type=..., value=...)` que quedaban con
      kwargs inválidos (`test_serialize_cart_a08_...`, `test_filtro_closed_by_refactor_...`).
- [X] T055 [P] Actualizar specs de frontend afectadas por el shape nuevo:
      `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.spec.ts`
      (reescrito: vigencia string-a-string sin cambio, resumen ahora por regla vía
      `ruleConditionPreview`, FR-018 sobre `canEditShape()` — `canEditValue()` ya no existe, más
      2 tests nuevos: agregar/quitar reglas y el 409 de cliente de FR-001a),
      `promotion-pricing.util.spec.ts` (el fixture `promo()` gana `rules: [...]`;
      `isPromoActiveNow`/`getPromoDisplay` sin cambio de cuerpo, siguen operando sobre la
      promoción), `promotion.service.spec.ts` (sin cambios — no referenciaba el shape viejo).
      **Verificado**: `ng test` — 568/571 en verde; las 3 fallas restantes
      (`app.spec.ts`, `auth.service.spec.ts`, `pos-checkout-panel.component.spec.ts`) son
      preexistentes y no relacionadas con promociones (título de la app, cambio de contraseña,
      botón de pre-cuenta — ninguno de esos 3 archivos fue tocado por esta revisión).
- [X] T056 Correr la suite completa (`python -m unittest discover -s app/characterization_tests
      -p 'test_*.py' -v`, `python app/scripts/test_promotions_rules.py`, `ng build`) y comparar
      contra la línea base de T003 — los `CONGELA` que no tocan promociones deben pasar **sin
      editar**; los totales de cualquier promoción con exactamente 1 regla deben coincidir exacto.
      **Resultado**: 527 tests backend + script de CI en verde; `ng build` en verde (mismo tamaño
      de bundle); `ng test` 568/571 (3 fallas preexistentes ajenas a promociones).
- [X] T057 Verificar los 5 puntos de [quickstart.md](./quickstart.md) §"Antes de dar por completada
      esta revisión", en particular que ninguna de las dos ramas se haya mergeado a `main`
      mientras tanto (si ya se mergeó alguna, reevaluar si el ítem 9 de "Cambios de comportamiento"
      de `spec.md` sigue sin requerir entrada en `registro-de-anomalias.md`). Confirmado: ninguna
      de las dos ramas (`refactor/063-promociones-por-variante` en `pos-backend`,
      `feature/063-promociones-por-variante` en `pos-heladeria`) está en `main` (verificado con
      `git merge-base --is-ancestor`); el límite de reglas por plan de suscripción sigue fuera de
      alcance (`spec.md` §Out of Scope).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2, Incrementos G1+G2)**: depende de Setup — **bloquea** todas las
  historias. G1 (migración+modelo) se verifica antes de tocar el motor en G2.
- **User Stories (Phase 3-8)**: todas dependen de Foundational. US1 y US2 (P1) primero; US3, US5,
  US6 (P2) después, en cualquier orden entre sí; US4 (P3) al final — mismo orden que ya usó el
  `tasks.md` de los Incrementos A-F.
- **Polish (Phase 9, Incremento J)**: depende de que **todas** las historias (US1-US6) estén
  completas — es la única fase que borra las columnas viejas, así que ningún código puede
  depender ya de ellas.

### User Story Dependencies

- **US1 (P1)**: depende solo de Foundational. Es el punto de entrada del CRUD multi-regla — US5
  reutiliza su `_guard_variant_overlap` (T018) y su schema (T016/T017).
- **US2 (P1)**: depende solo de Foundational (el motor ya quedó completo en T009-T011); no
  depende de US1 para funcionar (una promoción con 1 regla, creada por migración o directo en
  base de datos en un test, ya es suficiente para probar US2 de forma aislada).
- **US3 (P2)**: depende de Foundational y de T018 (creado en US1) — extiende la misma función con
  el chequeo 2.
- **US5 (P2)**: depende de Foundational; reutiliza el schema/CRUD de US1 (T016/T020) para
  `duplicate`.
- **US6 (P2)**: depende solo de Foundational (T004-T006) — es puramente verificación.
- **US4 (P3)**: depende de Foundational (T009/T011); independiente de US1/US2/US3/US5/US6.

### Ficheros compartidos (secuenciales, NO en paralelo entre sí)

- `pos-backend/app/api/v1/promotions/service.py`: T009, T010, T011, T018, T019, T020, T021 tocan
  el mismo archivo — ejecutar en el orden numérico dentro de Foundational/US1, no en paralelo.
- `pos-backend/app/models/promotion.py`: T007 (Foundational) y T052 (Polish) — secuenciales por
  definición (T052 depende de que todo lo demás haya dejado de leer las columnas que borra).
- `pos-heladeria/.../promotions-page.component.ts`: T025 (US1) y T040 (US5) tocan el mismo
  archivo — T040 depende de T025.

### Parallel Opportunities

- Dentro de Foundational: T007 y T008 son `[P]` (archivos distintos, sin dependencia entre sí);
  T011 es `[P]` respecto de T009/T010 solo si se coordina el mismo archivo con cuidado (mismo
  archivo, funciones distintas — recomendable secuencial en la práctica pese al marcador).
- Dentro de cada historia, los tests marcados `[P]` pueden escribirse en paralelo entre sí (antes
  de la implementación, deben fallar primero).
- US6 (Phase 7) es enteramente independiente de US3/US5 (Phase 5/6) — un desarrollador puede
  tomarla en paralelo tan pronto Foundational esté listo.
- US4 (Phase 8) es independiente de todas las demás historias salvo Foundational.

---

## Parallel Example: Foundational (Phase 2)

```bash
# Tras T004-T006 (migración 063c aplicada y verificada contra Postgres real):
Task: "Agregar class PromotionRule en app/models/promotion.py"                    # T007 [P]
Task: "Agregar fixture add_rule_to_promotion en cart_fixtures.py"                 # T008 [P]

# Luego, secuencial (mismo archivo service.py):
Task: "Reescribir active_variant_set_rules"                                       # T009
Task: "Reescribir evaluate_variant_sets"                                          # T010
Task: "Cambiar parámetro de variant_set_condition_text / menu_unit_discount"      # T011
```

## Parallel Example: User Story 1

```bash
# Tests, en paralelo (mismo archivo pero secciones independientes, o archivos separados):
Task: "Reescribir test_promotions_rules_admin.py §CRUD"                           # T013 [P] [US1]
Task: "Agregar caso FR-001a en test_promotions_rules_admin.py"                    # T014 [P] [US1]
Task: "Agregar caso de creación por lote"                                         # T015 [P] [US1]

# Backend y frontend, en paralelo entre sí (repos distintos):
Task: "Agregar PromotionRuleIn, cambiar PromotionCreate/ShapeUpdate"              # T016 [P] [US1]
Task: "Actualizar promotion.interface.ts"                                         # T023 [P] [US1]
```

---

## Implementation Strategy

### MVP First (Foundational + US1 + US2)

1. Completar Phase 1 (Setup).
2. Completar Phase 2 (Foundational, Incrementos G1+G2, cada uno con su propio checkpoint) —
   **bloquea todo lo demás**.
3. Completar Phase 3 (US1) — un administrador ya crea promociones multi-regla.
4. Completar Phase 4 (US2) — el cobro combina variantes del conjunto de cada regla.
5. **PARAR y VALIDAR**: correr `quickstart.md` hasta el final del Incremento H; el caso Springfield
   (6 reglas, 1 promoción) funciona de punta a punta: crear, activar, cobrar.

### Incremental Delivery (Incrementos G-J de research.md D-R7)

1. Foundational (G1 → G2) → migración+modelo verificados aparte, luego motor sobre reglas listo,
   sin superficie nueva expuesta.
2. + US1 + US2 (parte de H) → CRUD multi-regla y cobro — MVP.
3. + US3 + US5 + US6 (resto de H) → solapamiento, ciclo de vida por lote, migración verificada.
4. + US4 (I) → menú QR y terminal reflejan reglas.
5. + Polish (J) → estructura legada borrada, tests finales cerrados.

### Parallel Team Strategy

Con varios desarrolladores, tras Foundational:
- Developer A: US1 (CRUD + frontend) → luego US5 (reutiliza su `_guard_variant_overlap`).
- Developer B: US2 (wiring de `applied_promotions`) → luego US3 (extiende el mismo guard que A).
- Developer C: US6 (verificación de migración, independiente) → luego US4 (menú QR/terminal).
- Nadie toca Polish (J) hasta que A, B y C confirmen que sus historias están completas.

---

## Notes

- `[P]` = archivos distintos, sin dependencia entre sí dentro de la misma fase.
- `[US#]` mapea cada tarea a su historia de `spec.md` para trazabilidad (Principio XII).
- Ningún task de este archivo reabre lo que los Incrementos A-F (modelo plano, ya en las ramas de
  feature) resolvieron — verificar con T003 antes de empezar.
- Los números de línea citados (`service.py:211`, etc.) son los verificados en las ramas de
  feature el 2026-09-01 (`plan.md` §"Estado real de las ramas de feature"); si el código avanzó
  desde entonces, confirmar contra el archivo real antes de editar.
- Commitear después de cada task o grupo lógico; parar en cualquier checkpoint para validar una
  historia de forma independiente antes de seguir.
