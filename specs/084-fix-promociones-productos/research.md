# Research: Corrección de Bugs en Promociones y Productos

**Input**: [spec.md](./spec.md), verificación directa del código en `../pos-backend` y
`../pos-heladeria` (ambos en `develop`, 2026-09-17).

## D0 — Verificación de código real (punto de partida)

No hay ningún `NEEDS CLARIFICATION` de tecnología: ambos repos ya existen, con stack fijo
(FastAPI/SQLAlchemy/Alembic + Angular 21). El trabajo de esta fase fue **verificar el código real**
antes de diseñar, no elegir tecnología. Hallazgos clave que cambian el diseño respecto a una
lectura ingenua de `spec.md`:

- **Bug 1 no requiere tocar el backend.** `menu_variant_promotion`
  (`app/api/v1/promotions/service.py:357-411`) ya calcula `display_text`/`unit_equivalent` para
  **cualquier** `min_qty` (no solo 1) y ya viaja en `MenuVariantResponse.promotion`
  (`app/api/v1/menu/schemas.py:41`, servido desde `app/api/v1/menu/router.py:184`). La función que
  el frontend usa hoy para decidir qué mostrar en la tarjeta (`menu_unit_discount`,
  `service.py:318-354`) es la que **solo** cubre `min_qty == 1` — es la causa raíz exacta del bug,
  y ya hay un dato mejor (`menu_variant_promotion`) sin consumir. Ver D6.
- **Bug 3 ya tiene una implementación parcial, con semántica distinta a la pedida.**
  `addRuleRow()` (`promotions-page.component.ts:1656-1706`) ya combina los productos candidatos del
  Paso 1 que comparten una presentación — pero genera **una sola regla compartida** (un
  `PromotionRuleForm` cuyo `variantIds` es la unión de variantes de N productos), no N reglas
  independientes editables por separado. `spec.md` FR-017/FR-019 exige lo segundo. Esto es un
  **reemplazo de comportamiento ya en producción** (spec 083, mergeada), no una función nueva desde
  cero — de ahí la anomalía A-77 (D7). Ver D4.
- **El botón "Configurar" no tiene ninguna guarda hoy.** `promotions-page.component.ts:317-323` no
  tiene `[disabled]`; `openEdit(p)` (línea 1528) tampoco valida `p.status`. El patrón "estado real,
  no badge" que `spec.md` FR-008 exige **ya existe** en `canDelete(p)` (línea 1458-1460:
  `return p.status !== 'active'`) — se replica el mismo criterio, no se inventa uno nuevo.
- **El backend ya expone `status` real** en `PromotionResponse` (`schemas.py:226`) — bug 4 no
  necesita ningún cambio de contrato de API para el `[disabled]` del listado.
- **`PATCH /promotions/{id}/shape` ya bloquea `active`/`finished`** a nivel de servicio
  (`update_shape`, `service.py:781`: `if promo.status not in ("draft", "paused")`) — esa guarda
  cubre la **forma** (reglas), no necesariamente un endpoint separado de nombre/vigencia si existe
  uno. Marcado como verificación pendiente en tasks (no bloquea el diseño, ver Constraints de
  `plan.md`).
- **No existe hoy ninguna guarda de exclusividad por producto completo.** Solo
  `_guard_variant_overlap` (`service.py:579-632`), que compara **la misma variante exacta** entre
  promociones cuya vigencia se cruza (`draft/active/paused`). La exclusividad de producto de FR-021
  es código enteramente nuevo, con un criterio de estado más angosto (solo `active`).
- **El menú de acciones ⋮ no usa CDK Overlay ni `mat-menu`.** Es un `<div class="absolute ...">`
  dentro de una celda con `overflow-x: auto` en el contenedor de la tabla — la causa técnica exacta
  del recorte: fijar `overflow-x: auto` sin fijar `overflow-y` fuerza a `overflow-y: auto` también
  (regla del spec CSS de `overflow`), recortando cualquier descendiente absoluto que se extienda
  verticalmente fuera del contenedor. `@angular/cdk` (`^21.2.14`) ya está instalado, sin ningún uso
  de `Overlay`/`Menu` en el repo — no hay patrón previo que replicar ni romper.
- **El patrón de menú ⋮ con recorte solo existe en `promotions-page.component.ts`.**
  `products-page.component.ts:192-230` usa botones de icono inline, sin dropdown — confirma que el
  alcance de bug 5 (FR-026–028) no necesita tocar el listado de productos hoy (el Assumption del
  spec ya lo dejaba condicionado a "si existieran otros listados con el mismo patrón").

## D1 — Modelo de la asociación variante↔presentación: columna vs. tabla puente

**Decisión**: columna nueva `product_variants.presentation_id` (FK nullable a `presentations.id`),
relación 1:1 por variante.

**Rationale**: una variante representa como máximo una presentación (spec.md FR-006: dos variantes
del mismo producto no pueden compartir presentación) — es exactamente la cardinalidad de una FK
simple, igual que `product_variants.product_id`. No hay atributos propios de la asociación (fecha,
quién la creó, etc.) que justifiquen una tabla intermedia.

**Alternatives considered**: tabla puente `variant_presentations` (espejo de
`category_presentations`) — descartada por sobre-modelar una relación 1:1; habría exigido un JOIN
adicional en cada lectura de variante (recibos, menú, promociones) para un beneficio que una FK
simple ya cubre.

## D2 — Sincronización de nombre: columna copiada + cascada vs. nombre derivado (JOIN)

**Decisión**: `product_variants.name` sigue siendo una columna propia, almacenada. Al asociar o
reasociar una presentación (`FR-002`/`FR-003`), el backend copia `Presentation.name` dentro de la
misma transacción. Al renombrar una `Presentation` (`PATCH /presentations/{id}`, `FR-004`), el
backend ejecuta `UPDATE product_variants SET name = :nuevo_nombre WHERE presentation_id = :id` en
la misma transacción que el `UPDATE` de `Presentation`.

**Rationale**: `product_variants.name` ya es `NOT NULL` con `UniqueConstraint(product_id, name)` y
lo leen directamente recibos, el motor de promociones, el menú QR y exportes — derivarlo vía JOIN
obligaría a tocar todos esos puntos de lectura para un beneficio marginal (evitar una cascada de
`UPDATE`, que en la práctica son unas pocas filas por tenant). La cascada es la operación
equivalente a la que ya hace, por ejemplo, cualquier catálogo con nombre desnormalizado en este
código base (mismo patrón que otros catálogos del panel).

**Alternatives considered**: nombre derivado en tiempo de lectura (sin columna `name` propia,
`COALESCE(presentation.name, variant.own_name)` vía JOIN) — descartado por el alto radio de
impacto en código que hoy asume `variant.name` como columna simple, para un caso de uso (cambiar el
nombre de una presentación ya usada) que en la práctica es infrecuente.

## D3 — Exclusividad de producto entre promociones: guarda nueva vs. extender `_guard_variant_overlap`

**Decisión**: función nueva y separada (`_guard_product_overlap`, nombre de trabajo) junto a
`_guard_variant_overlap` en `app/api/v1/promotions/service.py`, invocada desde los mismos puntos
(`create()`, `update_shape()`).

**Rationale**: el criterio de estado es distinto — `_guard_variant_overlap` compara contra
`draft/active/paused` con cruce de vigencia (fechas/días/horas); la exclusividad de producto
(`spec.md` FR-021, confirmada en Clarifications) compara **solo** contra `status=active`, sin
evaluar vigencia horaria. Son dos guardas con semántica distinta que deben poder evolucionar por
separado; fusionar ambas en una sola función mezclaría dos criterios de estado diferentes y haría
más difícil verificar cada uno de forma aislada (Principio VI).

**Alternatives considered**: extender `_guard_variant_overlap` para que, además de la variante
exacta, también compare por `product_id` — descartada porque cambiaría el comportamiento de una
guarda ya protegida (spec 063 FR-014, con su propio criterio de vigencia) sin que ningún FR de esta
spec pida modificarla.

## D4 — `addRuleRow()`: refactor a N reglas independientes vs. agregar checkboxes sobre la regla combinada

**Decisión**: refactorizar `addRuleRow()`/`resolvedVariantIdsForLabel` para que, cuando más de un
producto candidato comparte la presentación elegida, genere **una `PromotionRuleForm` por
producto** (cada una con la única variante de ese producto), en vez de una regla combinada con
`variantIds` de varios productos. La lista de checkboxes (FR-016) decide **para cuáles** productos
se genera esa fila antes de confirmar.

**Rationale**: `spec.md` FR-017 y FR-019 exigen que cada fila resultante sea **editable o
eliminable de forma individual sin afectar las demás** — imposible si varias variantes de distintos
productos viven dentro de la misma `PromotionRuleForm`/`PromotionRule`. Agregar solo checkboxes
sobre el comportamiento actual no resolvería esto: seguiría existiendo una única regla combinada
para todos los productos marcados.

**Alternatives considered**: mantener la regla combinada y agregar checkboxes solo para decidir qué
variantes entran en su `variantIds` — descartada porque no cumple FR-019 (edición/eliminación
individual por producto) ni refleja lo que el propietario del producto confirmó en Clarifications
("cada fila de regla independiente con su propia lista explícita de variantes").

## D5 — Menú de acciones (bug 5): Angular CDK Overlay vs. fix CSS manual

**Decisión**: reemplazar el `<div>` absoluto por Angular CDK Overlay (`OverlayModule`/`CdkMenu` de
`@angular/cdk`), ya instalado en `pos-heladeria` sin uso previo.

**Rationale**: CDK Overlay renderiza el menú en un contenedor global (`cdk-overlay-container`),
fuera del árbol DOM de la tabla — no queda sujeto al `overflow` del contenedor con scroll (causa
raíz confirmada en D0) y ya resuelve el reposicionamiento al hacer scroll/resize (`FR-027`/`FR-028`)
sin código manual de cálculo de posición. Es el patrón estándar de Angular para este problema
exacto, y no hay ningún uso previo de CDK en el repo que este cambio pueda romper.

**Alternatives considered**: fix puramente CSS (`overflow-y: visible` explícito en el contenedor de
la tabla, manteniendo `overflow-x: auto`, más un `position: fixed` calculado a mano con
`getBoundingClientRect()` en scroll/resize) — descartado porque replicaría a mano exactamente lo
que CDK Overlay ya resuelve, con más superficie de bugs futuros (recalcular posición en cada evento
de scroll/resize manualmente) para un ahorro de dependencia que no aplica (`@angular/cdk` ya está
instalado).

## D6 — Precio mínimo de la tarjeta del menú QR: cálculo en frontend vs. campo agregado en backend

**Decisión**: calcular el precio mínimo (`FR-013`) en frontend, iterando `product.variants[].promotion`
(ya presente en `MenuProductResponse`) y tomando el `unit_equivalent` más bajo; sin agregar ningún
campo nuevo a `MenuProductResponse` ni tocar `app/api/v1/menu/`.

**Rationale**: el dato por variante (`display_text`, `unit_equivalent`) ya viaja completo en cada
respuesta del menú (confirmado en D0); agregar un campo agregado en el backend sería redundante —
el frontend ya recibe todo lo necesario para reducirlo a un mínimo. Mantiene el alcance de bug 1
100% frontend, coherente con `spec.md` ("no reabre el motor de cálculo").

**Alternatives considered**: campo `MenuProductResponse.min_promo_price` calculado en el backend —
descartado por redundante y porque hubiera exigido tocar `app/api/v1/menu/router.py`/`schemas.py`
sin ninguna necesidad de cálculo que el frontend no pueda hacer con el dato que ya recibe.

## D7 — Anomalías pendientes de registrar antes de implementar (Principio II)

Próximo número disponible en `registro-de-anomalias.md`: **A-76** (el último registrado es A-75,
spec 083).

- **A-76** — Bloqueo total de "Configurar" en promociones `active`: reemplaza, para ese estado, la
  edición parcial (nombre/fin de vigencia/días/horas editables) que spec 083 FR-018 dejó vigente.
  Cita `spec.md` §Clarifications (pregunta 2, bug 4) como decisión de negocio.
- **A-77** — Reemplazo de la regla combinada de `addRuleRow()` (comportamiento en producción desde
  el merge de spec 083, PR #83) por N reglas independientes, una por producto. Cita `spec.md`
  §Clarifications (pregunta 1 y pregunta de `/speckit-clarify` sobre checkboxes, bug 3).
- **A-78** — Exclusividad de producto entre promociones `active`: restricción nueva que puede
  rechazar, hacia adelante, selecciones de producto que hoy son válidas (un producto en dos
  promociones activas simultáneas, mientras cubran variantes distintas). Cita `spec.md`
  §Clarifications (preguntas sobre exclusividad de producto, bug 3). No retroactiva (FR-025).

Estas tres entradas deben existir en `registro-de-anomalias.md` **antes** de mergear los commits
que implementan cada una — mismo tratamiento que A-74/A-75 de spec 083.

## D8 — Alcance de bug 5 confirmado por código

Confirmado en D0: el patrón de menú ⋮ con recorte solo existe hoy en
`promotions-page.component.ts`. `products-page.component.ts` usa botones de icono inline sin
dropdown — no hay nada que corregir ahí. El Assumption de `spec.md` ("si existieran otros
listados con el mismo patrón, se corrigen al encontrarlos") queda sin efecto práctico por ahora:
el alcance real de FR-026–028 es un solo archivo.

## D9 — Endpoint de nombre/vigencia de una promoción `active` (confirmado tras `/speckit-analyze`)

**Confirmado**: sí existe un endpoint separado — `PATCH /promotions/{promotion_id}`
(`app/api/v1/promotions/router.py:81`, `update_promotion`), que llama a `service.update()`
(`app/api/v1/promotions/service.py:762-778`). Esa función edita `name`, `description`, `ends_at`,
`days_of_week`, `start_time`, `end_time` — **sin ninguna guarda de `status` hoy** (a diferencia de
`update_shape`, que sí bloquea `active`/`finished`). Esto confirma que el bug 4 reportado es real y
exacto: hoy se puede editar el nombre/vigencia de una promoción `active` sin restricción.

**Tests de characterization que protegen este comportamiento hoy** (`app/characterization_tests/
test_promotions_rules_admin.py`, clase `TestUS5DuplicarEditarEstados` y
`TestUS5MantenimientoPorLote`):

- `test_ca1_editar_escalares_de_una_activa` (línea 280): crea una promoción, la activa
  (`change_status(..., "active")`), y llama `service.update()` con `name`/`ends_at`/
  `days_of_week`/`start_time`/`end_time` nuevos — **asertando que se aplican con éxito**.
- `test_editar_vigencia_de_promocion_multi_regla_afecta_a_todas_con_una_accion` (línea 363): mismo
  patrón, con una promoción `active` de 6 reglas — asertando que `service.update(..., ends_at=...)`
  tiene éxito y afecta a las 6 reglas de una sola vez.

**Implicación (Principio III)**: implementar FR-008/FR-011 (bloquear `service.update()` cuando
`promo.status == "active"`) rompe **directamente** estos dos tests tal como están escritos hoy. No
es opcional actualizarlos — son characterization tests protegidos, y el Principio III exige que su
actualización sea explícita, cite la decisión de negocio (**A-76**) y evidencie que no afecta otros
comportamientos protegidos del mismo archivo (en particular, `test_ca2_cambiar_reglas_de_una_activa_bloquea`,
`test_ca4_duplicar_copia_borrador_con_las_mismas_reglas` y los de `TestUS5MantenimientoPorLote`
sobre pausar/reactivar deben seguir en verde sin tocarse — no dependen de `service.update()` en
estado `active`).

**Guarda correcta** (no la de `update_shape`): FR-009 exige que "Configurar" — y por extensión
`service.update()` — siga editable en `Borrador`, `Pausada` **y `Finalizada`**, no solo en
`draft`/`paused`. La condición de `update_shape` (`status not in ("draft", "paused")`) es más
amplia de lo que pide esta spec — bloquear con ella también dejaría `Finalizada` sin poder
editarse, violando FR-009. La guarda nueva en `service.update()` debe ser específica:
`if promo.status == "active": raise HTTPException(409, ...)`.
