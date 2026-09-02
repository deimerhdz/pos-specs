# Phase 0 — Research

**Spec**: [spec.md](./spec.md) · **Fecha**: 2026-09-02

No hay ningún `NEEDS CLARIFICATION` pendiente en el Technical Context (todo se resolvió leyendo
el código real de `pos-backend`/`pos-heladeria` antes de escribir este plan — no quedó ninguna
suposición sin verificar). Este documento registra las decisiones técnicas tomadas durante ese
reconocimiento, no elección de tecnología (no hay ninguna nueva).

---

## D1 — Dónde vive el defecto del resumen de regla

**Decisión**: el texto roto ("Precio de paquete - Paga $ 12.000 llevando 2 unidades.") lo genera
`ruleSummaryText(rule)` en
[promotions-page.component.ts:1043-1050](../../../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts#L1043-L1050),
usado únicamente por la tarjeta **colapsada** de una regla en el listado (línea 491). Es
independiente de `ruleConditionPreview(ruleIndex)` (líneas 1082-1093), que **ya** nombra las
variantes seleccionadas — lo agregó la spec 066 (FR-018) para la pantalla de revisión antes de
guardar, usando `conditionText()` de `promotion-condition.util.ts`.

**Rationale**: confirmar esto antes de tocar código evita "arreglar" la función equivocada.
`ruleConditionPreview` ya cumple FR-001 en la pantalla de revisión; el defecto reportado es
exclusivamente de la tarjeta colapsada, un método corto y sin dependencias externas.

**Alternativas consideradas**: reemplazar `ruleSummaryText` por una llamada directa a
`ruleConditionPreview`/`conditionText()` — descartada porque esa función arma una oración
distinta ("Llevando 2 Gaseosa - Única pagas $12.000") a la que pide esta spec (FR-002/FR-003:
"Precio de paquete - Paga $ 12.000 llevando 2 unidades Gaseosa - Única."), que además coincide
con la forma que ya tenía `ruleSummaryText` (tipo aparte, "Paga $X llevando N unidades") solo que
sin nombres. Se reutiliza `setDescriptor()` (el algoritmo de orden alfabético + tope de tres +
"y N más" de FR-004, exportado en `promotion-condition.util.ts:58`) para no reimplementar esa
lógica una tercera vez, pero la oración final es propia de esta spec.

---

## D2 — Cómo separar "seleccionados" de "resultados de búsqueda" sin nueva llamada al backend

**Decisión**: el catálogo completo del tenant ya vive en memoria como el signal computado
`catalogVariants()` ([promotions-page.component.ts:679-697](../../../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts#L679-L697)),
poblado una sola vez por `menu.loadMenu()` en `ngOnInit`. El método existente
`selectedVariantsForRule(ruleIndex)` (línea 885) ya filtra ese catálogo por `rule.variantIds` —
es exactamente el listado de FR-006, y ya lo usa la pantalla de revisión. `visibleVariantsForRule`
(línea 867), que hoy filtra el catálogo completo por texto/categoría **sin** exigir ninguno de
los dos, se renombra a `searchResultsForRule` y gana la guarda de FR-008: si no hay categoría
específica **y** el texto está vacío, devuelve `[]`.

**Rationale**: cero llamadas nuevas a la API — todo el catálogo ya está en el cliente. Separar en
dos métodos (uno para "qué hay en el conjunto", otro para "qué encontró la búsqueda") es un
cambio de plantilla, no de arquitectura: la lista de la izquierda usa
`selectedVariantsForRule` (checkbox siempre marcado, desmarcar = quitar), la de la derecha (solo
visible con filtro activo) usa `searchResultsForRule` (checkbox refleja pertenencia actual,
marcar = agregar).

**Alternativas consideradas**: paginar o pedir la búsqueda al backend — descartada porque el
catálogo de un tenant de heladería (decenas a un par de cientos de variantes) ya carga completo
para el menú y no justifica un endpoint nuevo; SC-002 (<10s para encontrar un producto) se cumple
con un `Array.prototype.filter` en memoria sin percibirse ninguna latencia.

---

## D3 — Cómo bloquear tipo/valor/cantidad mínima de una regla existente sin bloquear el conjunto

**Decisión**: `canEditShape()` ([línea 810](../../../pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts#L810))
hoy es un único booleano (`!isReadOnly() && isDraft()`) que gobierna a la vez: el botón
"+ Agregar regla", "Quitar regla", los botones de tipo, los inputs de valor/cantidad mínima, y
las casillas del conjunto. Esta spec necesita **dos** permisos distintos una vez que `Pausada`
entra en juego:

- `canEditRuleSet()` = `!isReadOnly() && (isDraft() || isPaused())` — gobierna agregar/quitar
  reglas y el conjunto de variantes de cualquiera de ellas (FR-014).
- `canEditRuleTypeValue(rule)` = `!isReadOnly() && (isDraft() || (isPaused() && !rule.isExisting))`
  — gobierna tipo, valor y cantidad mínima; en `Pausada` solo se habilita para una regla que
  **no** existía cuando se abrió el formulario (FR-015).

`PromotionRuleForm` gana un campo cliente-only `isExisting: boolean`: `false` en `emptyRule()`
(toda regla agregada con "+ Agregar regla" es nueva, sin importar el estado), `true` al mapear
las reglas de `openEdit(p)`. No viaja al backend: `toRules()` en `promotion.service.ts:278-285`
ya arma el payload explícitamente campo por campo, así que un campo de más en el objeto de
TypeScript no se filtra nunca al `PromotionRuleInPayload`.

**Rationale**: el backend no necesita saber cuál regla es "nueva" ni cuál "existente" —
`update_shape` **siempre** borra todas las `PromotionRule` de la promoción y las vuelve a crear a
partir de la lista completa que manda el cliente
([service.py:757-777](../../../pos-backend/app/api/v1/promotions/service.py#L757-L777)); es
reemplazo total, no upsert por fila. La distinción "ya existía vs. es nueva" es puramente una
regla de **qué inputs deja tocar el formulario antes de enviar**, así que vive enteramente en el
cliente.

**Alternativas consideradas**: mandar `rule.id` en el payload y hacer upsert en el backend —
descartada por ser un cambio de arquitectura no pedido por ninguna historia de esta spec
(Principio V); el reemplazo total ya resuelve correctamente "agregar una regla" y "quitar una
regla" sin necesitar identidad estable entre ediciones.

---

## D4 — Qué cambia en el backend para permitir `Pausada`

**Decisión**: un único condicional en `update_shape`
([service.py:760](../../../pos-backend/app/api/v1/promotions/service.py#L760)):
`if promo.status != "draft":` pasa a `if promo.status not in ("draft", "paused"):`, con el mensaje
de error actualizado. El docstring del método y el `summary` del router
([router.py:99-101](../../../pos-backend/app/api/v1/promotions/router.py#L99-L101)) se actualizan
para decir "borrador o pausada". Ninguna otra línea de `update_shape` cambia:
`_guard_no_shared_variants_within_payload` (FR-001a), `_guard_package_is_discount` (FR-016) y
`_guard_variant_overlap` (FR-014/FR-014a) ya corren sin condicionar por estado más que excluir la
propia promoción (`Promotion.id != promo.id`) y filtrar candidatas por
`status.in_(("draft", "active", "paused"))` — exactamente el conjunto de estados que ya
considera "vigente" para efectos de solape. El mínimo de una regla (FR-018) ya lo exige
`PromotionShapeUpdate.rules: list[PromotionRuleIn] = Field(..., min_length=1)`
([schemas.py:171](../../../pos-backend/app/api/v1/promotions/schemas.py#L171)) — sin cambio.

**Rationale**: todas las validaciones de guardado que esta spec necesita para `Pausada`
(FR-016) ya existen y ya se ejecutan dentro de `update_shape`; relajar el estado permitido es
literalmente la única línea de lógica de negocio nueva en el backend. Cambiar la condición de
estado no reintroduce el riesgo de "descuento congelado": `Pausada` no aplica descuento
(`active_variant_set_rules` filtra `Promotion.status == "active"`,
[service.py:158](../../../pos-backend/app/api/v1/promotions/service.py#L158)), así que editar el
conjunto mientras está pausada no afecta ningún cobro en curso.

**Alternativas consideradas**: ninguna — es el cambio mínimo que satisface FR-014 a FR-017 sin
tocar el motor de cálculo ni el modelo de datos.

---

## D5 — Decisión de negocio y characterization tests

**Decisión**: el cambio de FR-013 a FR-018 se registra como **A-69** en
[`registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md) (ya escrita como
parte de este plan, con quién/cuándo/qué cambia). Se verificó que
`test_ca2_cambiar_reglas_de_una_activa_bloquea`
([test_promotions_rules_admin.py:241-249](../../../pos-backend/app/characterization_tests/test_promotions_rules_admin.py#L241-L249))
prueba el caso `Activa` (que **no** cambia) y no lleva el prefijo `"CONGELA comportamiento
actual:"` — es un test de la funcionalidad de la spec 063, no un characterization test protegido
por el Principio III. Ningún test con ese prefijo referencia `update_shape` ni el estado
`paused` (verificado por `grep` sobre los cuatro ficheros de `characterization_tests/` que tocan
promociones).

**Rationale**: satisface el Principio II (decisión de negocio registrada antes de implementar) y
confirma que no hay que negociar ninguna excepción al Principio III — no existe ningún test
protegido en conflicto con este cambio.
