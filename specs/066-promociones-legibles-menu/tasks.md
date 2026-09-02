---

description: "Task list for spec 066 — promociones legibles y precios reales en el menú QR"
---

# Tasks: Promociones legibles y precios reales en el menú QR

**Input**: Design documents from `/specs/066-promociones-legibles-menu/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: esta spec exige characterization tests por Principio III y verificación ejecutable por
Principio X (Constitución) — **no son opcionales** en este proyecto. Cinco tests heredados cambian
de aserto (ninguno bajo el veto `"CONGELA comportamiento actual:"`, verificado el 2026-09-01) y
cada historia incluye sus propios tests de aceptación.

**Organization**: tareas agrupadas por historia de usuario (`spec.md`), en orden de prioridad
P1 (US1) → P1 (US2) → P2 (US3) → P3 (US4), que coincide con las Fases de entrega de
[plan.md](./plan.md) §"Fases de entrega".

**Sin migraciones, sin dependencias nuevas**: `alembic/versions/` no gana ningún fichero,
`requirements.txt` y `package.json` no cambian (FR-019, `data-model.md` §"Resumen: cero cambios de
esquema"). Cualquier tarea que parezca necesitar una migración está fuera de alcance — parar y
revisar.

## Path Conventions

Dos repos sibling de `pos-specs`: `../pos-backend` (FastAPI/Python 3.14) y `../pos-heladeria`
(Angular 20). Las rutas de cada tarea son relativas a la raíz del repo que se indica. Ambos repos
están hoy en `develop` con la spec 063 ya mergeada; el trabajo va en la rama
`feature/066-promociones-legibles-menu` de cada uno (T002).

Los números de línea citados son los verificados el 2026-09-01 (`research.md` §"Estado del código
verificado"); si el código avanzó, confirmar contra el fichero real antes de editar.

---

## Phase 1: Setup

**Purpose**: autorizar el cambio de comportamiento y fijar la línea base antes de tocar código.

- [X] T001 **🚦 GATE BLOQUEANTE (Principio II)** — verificar que las tres decisiones de negocio están
      registradas a mano en `specs/000-reconocimiento/registro-de-anomalias.md`, corriendo
      `grep -nE "^### A-6[678] " specs/000-reconocimiento/registro-de-anomalias.md` desde la raíz de
      `pos-specs`. Deben salir **tres** líneas (A-66 texto por nombres, A-67 insignia genérica,
      A-68 `package_price` `min_qty 1` como precio vigente), en el formato de A-62 a A-65 (qué
      cambia, por qué, quién decide, cuándo, funcionalidades afectadas). **Hoy el registro llega a
      A-65 y el `grep` no devuelve nada.** Esta tarea **no redacta** las entradas —es un paso previo
      externo a la feature (`spec.md` §"Cómo se registran")—: solo comprueba que existen. Si no
      salen las tres, **detener la implementación aquí**; ninguna tarea posterior está autorizada.
- [X] T002 Confirmar el punto de partida de los dos repos de código: `git branch --show-current` da
      `develop` y `git status --porcelain` sale vacío en `../pos-backend` y en `../pos-heladeria`;
      crear en cada uno la rama `feature/066-promociones-legibles-menu` a partir de `develop`.
- [X] T003 Fijar la línea base de tests, sin editar nada: en `../pos-backend`,
      `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` y
      `python -m app.scripts.test_promotions_rules`; en `../pos-heladeria`, `npm test` y
      `ng build`. Anotar el número de tests y los warnings preexistentes para comparar en Polish
      (quickstart.md §2).

**Checkpoint**: A-66/A-67/A-68 registrados, ramas creadas, línea base en verde y anotada.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: el descriptor del conjunto y la resolución de nombres. Son las dos piezas que
consumen **todas** las historias: US1 arma el texto con ellas, US2 las necesita para
`condition_text` dentro del bloque de promoción, US4 replica el descriptor en TypeScript.

**⚠️ CRITICAL**: ninguna fase de historia puede empezar hasta que esta esté completa y verificada
por T006.

- [X] T004 Añadir el descriptor del conjunto (FR-002, FR-003) en
      `../pos-backend/app/api/v1/promotions/service.py`: `_sort_key(name)` con
      `unicodedata.normalize("NFD", …)` + descarte de `unicodedata.combining` + `casefold()`, y
      `_set_descriptor(names) -> tuple[str, bool] | None` que recorta espacios, descarta vacíos,
      **deduplica por el nombre mostrado**, ordena por `_sort_key` con desempate por el nombre
      original y compone según cuántos nombres distintos `D` queden: 1 → `Pequeño 8oz`; 2 →
      `A y B`; 3 → `A, B y C`; > 3 → `A, B, C y {D-3} más`; 0 → `None`. Devuelve también
      `multiple = D > 1`. Tabla normativa en
      [contracts/texto-condicion.md §2](./contracts/texto-condicion.md). Único import nuevo:
      `unicodedata` (biblioteca estándar, sin dependencia — Principio IX).
- [X] T005 Añadir `variant_display_names(db, variant_ids) -> dict[UUID, str]` en
      `../pos-backend/app/api/v1/promotions/service.py`: **una sola** consulta
      `SELECT ProductVariant.id, ProductVariant.name, Product.name … WHERE ProductVariant.id IN (…)`
      por llamada. El nombre utilizable es `variant.name.strip()` y, si queda vacío,
      `product.name.strip()`; una variante sin ninguno de los dos **no aparece en el mapa** (FR-006,
      `research.md` D-3). Nunca llamarla dentro de un bucle de variantes ni de reglas
      (`research.md` D-12: coste constante, jamás un N+1).
- [X] T006 Verificar T004 y T005 en `../pos-backend/app/scripts/test_promotions_rules.py`: añadir una
      sección nueva `_descriptor_casos()` que ejercite `_set_descriptor` con 0, 1, 2, 3 y 5 nombres
      distintos, con duplicados (ocho `Pequeño 8oz` → un solo nombre, FR-003), con tildes y
      mayúsculas mezcladas (`Ácai` antes de `Almendra`, `research.md` D-2) y con nombres en blanco.
      Correr `python -m app.scripts.test_promotions_rules` en verde.

**Checkpoint**: el descriptor produce texto determinista y los nombres se resuelven en una consulta.
Las historias pueden arrancar.

---

## Phase 3: User Story 1 — El comensal entiende qué le ofrece el cartel de promociones (Priority: P1) 🎯 MVP

**Goal**: el texto de condición de una regla nombra las variantes de su conjunto en vez de
contarlas, en el cartel del menú QR, en el listado de administración y en la terminal (que ya lee
`condition_text`). Es la Fase 1 de [plan.md](./plan.md).

**Independent Test**: con la promoción "Semana feliz en granizados" activa y sus tres reglas de
precio de paquete (conjuntos `Pequeño 8oz`, `Mediano 12oz`, `Grande 16oz`), abrir el menú QR dentro
de su horario y comprobar que las tres líneas nombran el tamaño y ninguna dice "de estas N
variantes" (quickstart.md §4).

### Implementación

- [X] T007 [US1] Cambiar la firma a
      `variant_set_condition_text(rule: PromotionRule, names: Mapping[UUID, str]) -> str | None` en
      `../pos-backend/app/api/v1/promotions/service.py:313`, con `names` **posicional y
      obligatorio** (`research.md` D-1: un call site olvidado debe romper en carga, no degradar en
      silencio al texto por conteo y romper SC-005). Orden de las guardas, que importa: **primero**
      `rule.type not in LIVE_TYPES` → `None` (una regla histórica no se anuncia), **después** el
      descriptor de T004 y, si devuelve `None`, el respaldo por conteo. Emitir los cuatro textos de
      FR-004 con `e = "entre "` cuando `multiple` (y **nunca** en `percent` con `min_qty == 1`):
      `Llevando {n} {e}{d} pagas {valor}` · `Cada {e}{d} a {valor}` · `{pct}% en {d}` ·
      `{pct}% llevando {n} {e}{d}`. Conservar sin tocar `_money()` y el despojo de ceros del
      porcentaje (`12.5%` con punto — cambiar el separador sería un cambio visible fuera de A-66,
      [contracts/texto-condicion.md §3](./contracts/texto-condicion.md)).
- [X] T008 [US1] Actualizar el call site `_serialize_rule` en
      `../pos-backend/app/api/v1/promotions/service.py:685` para pasarle el mapa de nombres
      construido a partir del `by_id` que `serialize_promotion` **ya tiene** (`:696-701`).
      **Cero consultas nuevas**: si esta tarea añade un `SELECT`, está mal resuelta
      ([contracts/texto-condicion.md §1](./contracts/texto-condicion.md)).
- [X] T009 [US1] Actualizar el call site `_build_menu_promotions` en
      `../pos-backend/app/api/v1/menu/router.py:206` para resolver los nombres con
      `variant_display_names(db, …)` sobre la **unión** de los conjuntos de las reglas vigentes,
      **una vez por llamada y fuera de cualquier bucle**. `MenuPromotionRule.variant_count` se
      conserva aunque el texto deje de mencionarlo (campo publicado de la spec 063, Principio V).

### Tests de la historia (Principio III — tests heredados afectados)

- [X] T010 [P] [US1] Actualizar
      `../pos-backend/app/characterization_tests/test_menu_router.py::test_vigente_se_anuncia_con_texto_legible`:
      el aserto pasa de `"Llevando 2 de estas 8 variantes pagas $12.000"` a
      `"Llevando 2 entre sabor-0, sabor-1, sabor-2 y 5 más pagas $12.000"` (los nombres que ya
      generan sus fixtures). Citar la spec 066 en el comentario del test. **No tocar** el
      `"CONGELA comportamiento corregido:"` del docstring de módulo: es otro prefijo, referido a
      A-08 (zona horaria), y sus dos tests usan `percent` `min_qty 1`, que esta spec no altera
      (`research.md` D-8).
- [X] T011 [P] [US1] Actualizar
      `../pos-backend/app/characterization_tests/test_promotions_rules_admin.py::test_ca1_ca6_paquete_nace_borrador_con_condicion`:
      `condition_text` pasa a `"Llevando 2 entre licor-0, licor-1, licor-2 y 5 más pagas $12.000"`.
      Citar la spec 066.
- [X] T012 [P] [US1] Actualizar
      `../pos-backend/app/characterization_tests/test_promotions_router.py:75::test_el_header_no_cambia_la_forma_de_la_respuesta`
      — **el 5.º test afectado, que la spec no lista** (`research.md` D-8). Su variante se crea con
      `fx.make_variant(db, product=prod, price=8000)` **sin `name`**, así que el fixture le pone
      `variante-{uid}`, no determinista: pasarle un `name` explícito (p. ej. `name="Pequeño 8oz"`) y
      afirmar `"10% en Pequeño 8oz"`. Citar la spec 066.
- [X] T013 [US1] Actualizar `_regla_texto` en
      `../pos-backend/app/scripts/test_promotions_rules.py` (§5) para suministrar el mapa de nombres
      y cubrir la **tabla de casos completa** de
      [contracts/texto-condicion.md §5](./contracts/texto-condicion.md), los 10 casos: conjunto de
      un solo nombre, de 1 variante, de 3 nombres **en orden alfabético** (`Grande 16oz` primero),
      de 5 nombres con `y 2 más`, `percent` `min_qty 1` y `min_qty 3`, `package_price` `min_qty 1`
      (`Cada Pequeño 8oz a $6.000`), respaldo por conteo sin ningún nombre utilizable, regla `combo`
      histórica → `None`, y `12.5%` con punto. (Mismo fichero que T006, secciones distintas: hacerlo
      después.)
- [X] T014 [US1] Crear
      `../pos-backend/app/characterization_tests/test_promociones_legibles.py` con los tests de
      aceptación de US1 (CA1 a CA6) sobre `GET /menu/promotions`: conjunto de 8 variantes con el
      mismo nombre, conjunto de **1** variante (el defecto reportado — nunca "de estas 1
      variantes"), conjunto de 3 nombres, conjunto de 5 nombres, `percent` 10% `min_qty 1`, y
      promoción activa **fuera** de su franja horaria → el cartel no anuncia ninguna de sus reglas.
      Docstring del módulo citando la spec 066 y A-66.

**Checkpoint**: US1 completa y desplegable sola — el cartel del menú, el listado de administración
y la terminal (que ya consume `condition_text`) mejoran de golpe. Correr la suite del backend en
verde antes de seguir.

---

## Phase 4: User Story 2 — El comensal ve el costo real de la presentación que va a elegir (Priority: P1)

**Goal**: cada presentación cubierta por una regla vigente muestra su condición corta y su
equivalente por unidad, y una regla de `package_price` con `min_qty 1` pasa a mostrarse como precio
vigente — el defecto de importe que A-68 autoriza a corregir. Es la Fase 2 de [plan.md](./plan.md).

**Independent Test**: con "2 x $12.000" vigente sobre `Pequeño 8oz` ($8.000), abrir el producto y
ver `$8.000` con `2 x $12.000 · $6.000 c/u` debajo; luego, con una regla de $6.000 `min_qty 1`,
ver `$8.000` tachado y `$6.000` vigente, y comprobar que el carrito cobra $6.000 (quickstart.md §5).

### Implementación — backend

- [X] T015 [US2] Añadir el DTO `MenuVariantPromotion` en
      `../pos-backend/app/api/v1/menu/schemas.py` con los nueve campos de
      [contracts/menu-info-promocion.md §1](./contracts/menu-info-promocion.md) (`condition_text`,
      `short_condition`, `unit_equivalent`, `unit_equivalent_approx`, `unit_equivalent_text`,
      `display_text`, `type`, `min_qty`, `value`) y el campo **aditivo**
      `promotion: MenuVariantPromotion | None = None` en `MenuVariantResponse`. `MenuProductResponse`
      **no** gana ningún campo: la insignia se deriva en el frontend (`research.md` D-11).
- [X] T016 [US2] Añadir
      `menu_variant_promotion(rules, variant_id, unit_price, names) -> dict | None` en
      `../pos-backend/app/api/v1/promotions/service.py`: primera regla de `rules` cuyo conjunto
      contenga `variant_id`, **sin criterio de desempate** (FR-012, la spec 063 FR-014 impide el
      solape); exacto `Decimal(value)/Decimal(min_qty)` para `package_price` —el mismo para todas
      las variantes del conjunto— y `Decimal(unit_price) * (100 - Decimal(value))/100` para
      `percent`, sobre el precio de **esa** variante (FR-008); `aprox = exacto % 1 != 0` y
      `valor = exacto.quantize(Decimal("1"), rounding=ROUND_HALF_UP)` (FR-009, `research.md` D-5);
      textos `"{min_qty} x {_money(value)}"` / `"{min_qty} x -{pct}%"`,
      `"≈ {_money(v)} c/u"` / `"{_money(v)} c/u"` y
      `display_text = "{short_condition} · {unit_equivalent_text}"`, con el separador
      `" · "` (espacio, U+00B7, espacio). Mantener la guarda de `variant_set_condition_text` → `None`
      → sin bloque, por si el filtro de `LIVE_TYPES` cambia.
- [X] T017 [US2] Extender `menu_unit_discount` en
      `../pos-backend/app/api/v1/promotions/service.py:300-310` a `package_price` con
      `min_qty == 1`, devolviendo **`rule.value` tal cual** (FR-010, A-68), y poblar `discount_kind`
      con el `type` **real** de la regla (`"percent"` o `"package_price"`, `research.md` D-13). El
      invariante que hay que leer despacio: para `package_price` `min_qty 1` el valor de la regla es
      `discounted_price` **siempre**, incluso si resulta mayor o igual que `price` — no se recorta,
      no se descarta, no se sustituye por `price`, porque es el importe que el cobro aplica
      ([contracts/menu-info-promocion.md §4.2](./contracts/menu-info-promocion.md)). `percent` con
      `min_qty > 1` y `package_price` con `min_qty > 1` siguen devolviendo `None`.
      **No tocar `evaluate_variant_sets`** ni ninguna función del motor de cobro (FR-019, SC-006).
- [X] T018 [US2] En `../pos-backend/app/api/v1/menu/router.py`, `_build_menu` (`:158-171`): resolver
      `names = variant_display_names(db, {…})` sobre los conjuntos de `active_variant_set_rules`
      **una vez por llamada, fuera del bucle de variantes** (dentro sería un N+1), y poblar
      `promotion = menu_variant_promotion(rules, v.id, v.price, names) if rules else None` en cada
      `MenuVariantResponse` ([contracts/menu-info-promocion.md §3](./contracts/menu-info-promocion.md)).

### Tests de la historia — backend

- [X] T019 [US2] Ampliar
      `../pos-backend/app/characterization_tests/test_promociones_legibles.py` con los tests de
      aceptación de US2 (CA1 a CA8) sobre `GET /menu`: bloque `promotion` con
      `2 x $12.000 · $6.000 c/u`; `3 x -15% · $9.350 c/u` **sin `≈`** (el exacto es entero);
      `3 x $13.000 · ≈ $4.333 c/u` y `2 x -12.5% · ≈ $7.613 c/u` **con `≈`** (FR-009, quickstart §5.10);
      `package_price` `min_qty 1` → `discounted_price == rule.value` y `discount_kind ==
      "package_price"` (FR-010); `percent` `min_qty 1` → `discounted_price` y `discount_kind`
      **sin cambio** respecto de producción (no-regresión) **y** bloque `promotion` poblado con
      `display_text == "1 x -10% · $7.200 c/u"` (FR-008 con `n` = 1); solo la variante
      cubierta lleva bloque en un producto de tres presentaciones; y la **tabla de nulidad completa**
      de [contracts/menu-info-promocion.md §5](./contracts/menu-info-promocion.md), incluida
      promoción activa fuera de su franja → `promotion is None` y `discounted_price is None`.
- [X] T020 [US2] Añadir a `test_promociones_legibles.py` el caso de FR-015 que parece inalcanzable y
      no lo es (`research.md` D-6): crear y **activar** una regla `package_price` `min_qty 1` de
      $6.000 sobre una variante de $8.000 (pasa `_guard_package_is_discount`), luego **bajar el
      precio del catálogo** a $5.000 —la guarda no corre en ese camino— y afirmar que
      `discounted_price` sigue siendo $6.000, mayor que `price`. La rama "sin tachado" del frontend
      depende de este caso; sin test, queda sin cobertura.

### Implementación — frontend

- [X] T021 [P] [US2] Añadir la interfaz `MenuVariantPromotion` y el campo
      `promotion?: MenuVariantPromotion | null` a `MenuVariant` en
      `../pos-heladeria/src/app/modules/products/interfaces/product.interface.ts`
      ([contracts/superficies.md §1](./contracts/superficies.md)).
- [X] T022 [US2] Mapear el campo `promotion` en
      `../pos-heladeria/src/app/modules/tables/services/diner.service.ts:352-362` (transporte del
      comensal; `discounted_price` / `discount_kind` ya se mapean y siguen igual).
- [X] T023 [US2] En
      `../pos-heladeria/src/app/modules/tables/components/product-select.component.ts`
      (**compartido** por el comensal y las dos superficies del cajero, `research.md` D-9 — un solo
      cambio, **sin ninguna rama por superficie**): (a) añadir bajo el precio de cada fila
      (`:63-93`), cuando `v.promotion`, una línea secundaria con `v.promotion.display_text` en
      tipografía menor y tono discreto, sin competir con el precio; (b) acotar la insignia de
      porcentaje de `:80-86` a `discount_kind === 'percent'`, para que `package_price` `min_qty 1`
      no fabrique un `-25%` que la regla nunca enuncia (`research.md` D-13); (c) mostrar
      `promotion.condition_text` completo bajo la lista cuando la presentación seleccionada lo trae
      (cubre FR-016 de paso, [contracts/superficies.md §2.5](./contracts/superficies.md)).
      **No tocar** `discountInfo` ni `effectivePrice` (`promotion-pricing.util.ts:82-100`): el
      tachado y el total del botón salen correctos solos, comparando importes que ya llegaron
      calculados (`research.md` D-7, §2.3 y §2.4 del contrato).
- [X] T024 [US2] Crear
      `../pos-heladeria/src/app/modules/tables/components/product-select.component.spec.ts` (hoy el
      componente no tiene spec) cubriendo: fila con `promotion` pinta `display_text`; fila sin
      `promotion` no pinta nada; `discounted_price < price` → tachado + precio vigente;
      `discounted_price >= price` → **solo** el precio vigente, sin tachado ni señal de descuento
      (FR-015, el caso de T020); `discount_kind === 'package_price'` → **sin** insignia de
      porcentaje; `available === false` → `Agotado` gana a todo.

**Checkpoint**: US2 completa y desplegable sola. Es la fase que corrige el defecto de importe: lo
que el menú muestra coincide con lo que el cobro aplica (SC-003).

---

## Phase 5: User Story 3 — Las tarjetas del menú señalan qué productos tienen promoción (Priority: P2)

**Goal**: la tarjeta del menú QR lleva una insignia genérica `🎉 Promo` cuando alguna de sus
presentaciones trae información de promoción. Es la Fase 3 de [plan.md](./plan.md); **depende de
US2**, que es quien puebla `variants[].promotion`.

**Independent Test**: con una promoción de paquete vigente sobre las variantes `Pequeño 8oz` de dos
productos distintos, recorrer la carta y comprobar que esos dos productos —y solo esos— llevan la
insignia (quickstart.md §6).

- [X] T025 [US3] En
      `../pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts`: añadir
      `hasPromotion(product) { return product.variants.some(v => v.promotion != null); }` —una
      lectura, **no** una evaluación de reglas ni de vigencia: eso ya lo hizo el backend al poblar
      `promotion` (FR-013, `research.md` D-11)— y **reemplazar** el bloque
      `@if (productDiscount(product); as disc)` de `:382-389`, que hoy pinta `🏷️ -10%` / `🏷️ -$2.000`,
      por `@if (hasPromotion(product)) { … 🎉 Promo … }`. Una sola insignia, igual para porcentaje y
      para paquete (A-67). **Dejar intactos** el bloque de tachado + precio de `:402-406`,
      `productDiscount()` (`:775`) y `priceLabel()` (`:752-756`): FR-015 conserva el tachado y
      `priceLabel` ya usa `effectivePrice`, así que el "Desde $X" refleja el valor de la regla solo
      ([contracts/superficies.md §3](./contracts/superficies.md)).
- [X] T026 [US3] Añadir a
      `../pos-heladeria/src/app/modules/tables/pages/public-menu.component.spec.ts` los tests de
      aceptación de US3 (CA1 a CA5): producto con una variante en una regla `package_price` vigente
      → insignia (**el caso que hoy no produce ninguna señal**); producto con una variante en una
      regla `percent` → **la misma** insignia, no una distinta por tipo; producto sin variantes
      cubiertas → sin insignia; producto cuya promoción está fuera de ventana (el backend no pobló
      `promotion`) → sin insignia; y `percent` `min_qty 1` → sigue mostrando precio tachado y precio
      con descuento **y además** la insignia genérica (FR-015).

**Checkpoint**: US3 completa. El comensal identifica desde la carta qué productos tienen promoción
sin abrir ninguno (SC-007).

---

## Phase 6: User Story 4 — El cajero y el administrador leen la misma condición que el comensal (Priority: P3)

**Goal**: la terminal y el formulario de administración usan el mismo texto de condición que el
menú. Es la Fase 4 de [plan.md](./plan.md). La terminal ya gana la condición por T023 (componente
compartido); aquí falta que su transporte mapee el campo, y que la vista previa del formulario
replique el algoritmo sobre las variantes todavía no guardadas.

**Independent Test**: con la misma regla vigente, copiar el texto del cartel del menú, el de la
terminal y el de la columna «condición» del listado de administración, y comprobar que son
idénticos carácter por carácter (quickstart.md §7).

- [X] T027 [P] [US4] Mapear **solo** el campo `promotion` en
      `../pos-heladeria/src/app/core/services/menu.service.ts:88-110` (transporte de la terminal).
      **NO mapear `discounted_price` ni `discount_kind`**: hoy los descarta a propósito y debe
      seguir descartándolos. Mapearlos "ya que estamos" haría que `effectivePrice` empezara a
      mostrar precios con descuento en la terminal — expansión de alcance no pedida, choque con
      FR-017 y con la spec 063 FR-023 (el importe de la terminal lo resuelve el preview del cobro),
      y cambio de comportamiento sin decisión de negocio registrada (`research.md` D-10). Es el
      riesgo principal del diseño.
- [X] T028 [P] [US4] Crear
      `../pos-heladeria/src/app/modules/promotions/services/promotion-condition.util.ts` con
      `setDescriptor(names)` y `conditionText(rule, names, variantCount)`, réplica exacta del
      algoritmo de T004/T007 en TypeScript: `sortKey` con
      `name.normalize('NFD').replace(/\p{M}/gu, '').toLowerCase()`, misma deduplicación, mismo corte
      a tres nombres, mismos cuatro textos y mismo respaldo por conteo. El porcentaje se formatea
      con `String(value)` —**no** `toLocaleString('es-CO')`— para igualar al backend
      ([contracts/texto-condicion.md §6](./contracts/texto-condicion.md)). Es la única
      reimplementación del algoritmo, y existe porque el formulario describe variantes que todavía
      no están guardadas (FR-018).
- [X] T029 [US4] Crear
      `../pos-heladeria/src/app/modules/promotions/services/promotion-condition.util.spec.ts` con
      **la misma tabla de casos** que T013 ([contracts/texto-condicion.md §5](./contracts/texto-condicion.md),
      los 10 casos). Es la garantía de SC-005 entre los dos lenguajes: si un caso da distinto en TS
      que en Python, las superficies se separaron.
- [X] T030 [US4] Reescribir `ruleConditionPreview` en
      `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts:1070` sobre
      `conditionText(…)`, cambiando la firma de `ruleConditionPreview(rule)` a
      `ruleConditionPreview(ruleIndex: number)` para poder pedir
      `selectedVariantsForRule(ruleIndex)` (`:884`) y derivar los nombres con el mismo criterio que
      `variant_display_names` (`variantName?.trim() || productName?.trim()`, descartando vacíos).
      Actualizar la llamada de la plantilla en `:554` a `ruleConditionPreview($index)` — `$index` ya
      está en ese alcance (`:556` lo usa). **No tocar** la lista completa de variantes con su precio
      del resumen (`:556-559`, spec 063 FR-005) ni el listado de `:228`, que recibe el texto nuevo
      del backend sin cambios de código.
- [X] T031 [US4] Actualizar
      `../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.spec.ts:56,61`
      a la firma nueva `ruleConditionPreview($index)` y a los textos por nombres (hoy afirman
      `'Llevando 2 de estas 3 variantes'` y `'10% en estas 3 variantes'`), citando la spec 066.
      `promotion-pricing.util.spec.ts:34` menciona `condition_text` **solo como dato de fixture** y
      ningún aserto lo lee: **no** es un test afectado, no tocarlo.
- [X] T032 [US4] Crear `../pos-heladeria/src/app/core/services/menu.service.spec.ts` (hoy no existe)
      con el **test de no-regresión de importes de la terminal** (quickstart.md §7.6, `research.md`
      D-10): dada una respuesta de `GET /menu` con `promotion` poblado **y** `discounted_price` de
      $6.000 sobre una variante de $8.000, la categoría mapeada expone `promotion` y deja
      `discounted_price` / `discount_kind` en `undefined`/`null`, de modo que el modal de la terminal
      sigue mostrando $8.000 y el total del botón sigue siendo `variant.price`.
- [X] T033 [US4] Añadir a
      `../pos-backend/app/characterization_tests/test_promociones_legibles.py` el test de SC-005:
      para la **misma** regla, `GET /menu/promotions` (`rules[].text`), `GET /menu`
      (`variants[].promotion.condition_text`) y el listado de administración (`condition_text`)
      devuelven la misma cadena, comparada **carácter por carácter** con `assertEqual`.

**Checkpoint**: las cuatro historias completas. Las tres superficies hablan del mismo conjunto con
las mismas palabras y la terminal no ganó ni un importe.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: cerrar la verificación obligatoria (Principio X) y comprobar los límites que la spec
impone al cambio.

- [X] T034 Correr la suite completa del backend en verde:
      `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` y
      `python -m app.scripts.test_promotions_rules` en `../pos-backend`. **Verificación de SC-006**:
      `test_promotions_service.py` debe pasar **sin que se haya modificado ningún aserto de
      importe**; comprobarlo con `git diff develop -- app/characterization_tests/test_promotions_service.py`
      (debe salir vacío). Si un aserto de importe hubo que tocarlo, el diseño se salió del alcance —
      parar y revisar.
- [X] T035 [P] Correr `npm test` y `ng build` en verde en `../pos-heladeria`, comparando contra la
      línea base de T003 (mismos warnings preexistentes, ninguno nuevo).
- [X] T036 [P] Verificar los límites del cambio: `alembic/versions/` sin ficheros nuevos
      (`git status`/`git diff --stat develop -- alembic/`), `requirements.txt` y `package.json` sin
      cambios (FR-019, Principios VIII y IX), y ningún fichero del motor de cobro tocado
      (`git diff --stat develop` no debe listar cambios en `evaluate_variant_sets`,
      `_greedy_units`, `_distribute_group_discount`, `_valid_now`, `_guard_variant_overlap`,
      `_guard_package_is_discount`, `PROMOTION_TRANSITIONS`).
- [X] T037 Ejecutar los escenarios manuales de US1 en local: quickstart.md §4, casos 4.1 a 4.6,
      incluida la comprobación cruzada de SC-005 (copiar y pegar los tres textos y compararlos).
      **Ojo con 4.3 y 4.4**: el orden es alfabético (FR-002), no el de selección.
- [X] T038 Ejecutar los escenarios manuales de US2 en local: quickstart.md §5, casos 5.1 a 5.8, más
      §5.9 (la rama sin tachado: activar la regla y **después** bajar el precio del catálogo) y §5.10
      (marca de aproximado en los dos tipos). En 5.5, cobrar el pedido y confirmar que el importe
      cobrado coincide con el que mostró el modal (SC-003).
- [X] T039 Ejecutar los escenarios manuales de US3 y US4 en local: quickstart.md §6 (casos 6.1 a
      6.5, con el conteo de SC-004: los productos con insignia son exactamente los que tienen al
      menos una variante cubierta) y §7 (casos 7.1 a 7.5, incluido 7.5 — la insignia por producto de
      la terminal **conserva** `-10%` / `Paquete $12.000`).
- [X] T040 Cerrar la lista de verificación de quickstart.md §8: los siete criterios SC-001 a SC-007
      comprobados, los cinco tests heredados actualizados **citando la spec 066 en el mensaje de
      commit** (Principio III y XII), y commits separados por fase para poder desplegar cada
      historia por su cuenta (Principio VI).

---

## Dependencies & Execution Order

### Dependencias de fase

- **Setup (Phase 1)**: sin dependencias. **T001 bloquea absolutamente todo lo demás** — sin A-66,
  A-67 y A-68 registrados, el cambio de comportamiento no está autorizado (Principio II).
- **Foundational (Phase 2)**: depende de Setup. **Bloquea las cuatro historias.**
- **US1 (Phase 3)**: depende de Foundational.
- **US2 (Phase 4)**: depende de Foundational y de **T007** (necesita
  `variant_set_condition_text(rule, names)` para el `condition_text` del bloque de promoción). El
  resto de US1 (call sites y tests) no la bloquea.
- **US3 (Phase 5)**: depende de **US2** (T015, T018, T021, T022): sin `variants[].promotion` no hay
  nada de donde derivar la insignia.
- **US4 (Phase 6)**: depende de Foundational y de T007 (T028/T030 replican su algoritmo) y de T015
  (T027 mapea el campo). T023 ya le entregó el pintado de la condición en la terminal.
- **Polish (Phase 7)**: depende de todas las historias que se quieran entregar.

### Dentro de cada historia

- Modelos/DTO antes que servicios; servicios antes que routers; backend antes que el frontend que lo
  consume; implementación antes que sus tests de aceptación, salvo los tests heredados (T010-T013),
  que se actualizan junto con el cambio que los rompe, en el mismo commit.

### Oportunidades de paralelismo

- **Phase 3**: T010, T011 y T012 son ficheros distintos → en paralelo. T013 y T014 después (T013
  comparte fichero con T006).
- **Phase 4**: T021 (interfaz del frontend) es independiente del backend → en paralelo con
  T015-T020. T022 y T023 después de T021.
- **Phase 6**: T027, T028 y T029 tocan ficheros distintos; T027 es independiente de la cadena
  T028 → T029 → T030 → T031.
- **Phase 7**: T035 y T036 en paralelo con T034.
- **Entre repos**: todo el trabajo de `pos-heladeria` de una historia puede ir en paralelo con el de
  `pos-backend` una vez fijado el contrato del DTO (T015).

---

## Parallel Example: User Story 1

```bash
# Los tres tests heredados del backend, en ficheros distintos:
Task: "Actualizar test_menu_router.py::test_vigente_se_anuncia_con_texto_legible"          # T010 [P]
Task: "Actualizar test_promotions_rules_admin.py::test_ca1_ca6_..."                        # T011 [P]
Task: "Actualizar test_promotions_router.py::test_el_header_no_cambia_la_forma..."         # T012 [P]
```

## Parallel Example: User Story 2

```bash
# Backend y frontend a la vez, en cuanto el DTO (T015) esté fijado:
Task: "menu_variant_promotion() en promotions/service.py"                                  # T016 [US2]
Task: "MenuVariantPromotion + promotion en product.interface.ts"                           # T021 [P] [US2]
```

---

## Implementation Strategy

### MVP primero (US1 + US2)

1. Completar Phase 1 — **con T001 en verde, o parar**.
2. Completar Phase 2 (Foundational) — bloquea todo.
3. Completar Phase 3 (US1) → **PARAR y VALIDAR** con quickstart.md §4: el cartel nombra las
   variantes. Desplegable.
4. Completar Phase 4 (US2) → **PARAR y VALIDAR** con quickstart.md §5: el modal muestra el
   equivalente por unidad y el importe mostrado coincide con el cobrado. Desplegable.

US1 + US2 es el MVP: resuelve los dos defectos con impacto directo en el comensal, incluido el
único defecto de **importe** de la spec.

### Entrega incremental

1. Foundational → descriptor y nombres, sin superficie nueva expuesta.
2. + US1 → cartel, listado de administración y terminal ganan el texto por nombres (A-66).
3. + US2 → modal con condición y equivalente; `package_price` `min_qty 1` mostrado = cobrado (A-68).
4. + US3 → insignia genérica en la carta (A-67).
5. + US4 → vista previa del formulario y transporte de la terminal.
6. + Polish → verificación completa y cierre de SC-001 a SC-007.

### Estrategia con varios desarrolladores

Tras Foundational:

- Developer A: US1 (backend puro) → luego T033 (SC-005, reutiliza sus fixtures).
- Developer B: US2 backend (T015-T020) en cuanto T007 esté mergeado.
- Developer C: US2 frontend (T021-T024) en cuanto el DTO de T015 esté fijado → luego US3 (T025-T026,
  depende de su propio trabajo) → luego US4 frontend.

Nadie entra en Polish hasta que las historias que se van a entregar estén completas.

---

## Notes

- `[P]` = ficheros distintos, sin dependencia entre sí dentro de la misma fase.
- `[US#]` mapea cada tarea a su historia de `spec.md` para trazabilidad (Principio XII).
- **T001 no es una formalidad**: es el gate del Principio II. Las tareas de esta feature verifican
  que A-66, A-67 y A-68 existen; **no las redactan** (`spec.md` §"Cómo se registran").
- Ninguna tarea toca el modelo de datos, el motor de cálculo, el criterio de vigencia, el bloqueo de
  solapamiento, la máquina de estados ni la persistencia del descuento en la venta (FR-019, FR-020).
  Todo lo que esta feature calcula es información derivada que se descarta tras responder.
- Todo texto visible de la UI y todo nombre de test nuevo, en español de Colombia (Principio XIII).
- Los números de línea citados son los del 2026-09-01 sobre `develop` (`pos-backend` `199b3f5`,
  `pos-heladeria` `302d7ee`); si el código avanzó, confirmar contra el fichero real antes de editar.
- Commitear después de cada tarea o grupo lógico; parar en cualquier checkpoint para validar una
  historia de forma independiente antes de seguir.
