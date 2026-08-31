# Research: Promociones de Precio por Cantidad Configuradas por Presentación

Fase 0 de `/speckit-plan`. `spec.md` no dejó ningún `[NEEDS CLARIFICATION]` (checklist
`requirements.md` al 100%, 8 clarifications resueltas antes de redactar la spec). Este documento no
resuelve ambigüedades del contrato funcional: investiga el código real de `../pos-backend` y
`../pos-heladeria` para decidir **cómo** modelar la entidad "presentación compartida" (trabajo que
`spec.md` §"Out of Scope" delega explícitamente aquí, Principio VIII) y **dónde** enganchar el
cálculo, dejando registro de qué alternativas se descartaron y por qué.

Numeración de línea: estado del código al 2026-08-26.

---

## Decisión 1 — `Presentation` es una entidad de catálogo nueva, no un campo de `ProductVariant`

**Decisión**: se crea la tabla `presentations` (schema `tenant`) con `id`, `name` (único por
tenant, no vacío), `active` (bool) y timestamps. `ProductVariant` gana una columna nueva
`presentation_id` UUID **nullable**, FK → `presentations.id` con `ondelete="SET NULL"`.

**Por qué**:
- Hoy no existe ningún concepto compartido entre productos. `ProductVariant.name`
  (`app/models/product_variant.py:25`) es texto libre por producto, con
  `UniqueConstraint("product_id", "name")` — dos productos distintos pueden tener ambos una
  variante llamada "8oz" y son filas sin ninguna relación. `spec.md` FR-007 exige que el alcance de
  una regla se resuelva "por la referencia a la presentación — nunca comparando el nombre de la
  variante". Eso obliga a una entidad referenciable, no a un `string`.
- El código y el frontend ya usan la palabra "presentación" como sinónimo coloquial de "variante"
  (`app/api/v1/menu/schemas.py:41`, `product-form.component.ts:138-199`). Para no colisionar, la
  entidad nueva se llama `Presentation` en el modelo y "presentación (catálogo compartido del
  tenant)" en la documentación; la variante conserva su `name` libre (puede seguir diciendo "8oz")
  y **además** apunta, opcionalmente, a una fila de `presentations`.
- `nullable`: FR-008 exige que una variante sin presentación asignada (incluida la "Single" de un
  producto sin tamaños, `catalog/service.py:63-80`) quede fuera de toda regla por presentación.
  `NULL` es exactamente eso, y es el valor con el que nace todo el catálogo existente sin backfill
  (compatibilidad hacia atrás, Principio VII).
- `ondelete="SET NULL"` en la FK de la variante: si una presentación llega a borrarse (solo posible
  cuando **ninguna** regla activa la referencia, FR-020 — ver D10), las variantes que la apuntaban
  simplemente quedan sin presentación, sin romper el catálogo.

**Alternativas consideradas**:
- *Un `string` `presentation` libre en `ProductVariant`, agrupando por igualdad de texto*: rechazada
  — contradice FR-007 de raíz ("nunca comparando el nombre"), y "8 oz" / "8oz" / "8 Oz" romperían
  el agrupamiento en silencio, que es justo el bug de mantenimiento que la spec quiere eliminar.
- *Reusar `Category`*: rechazada — una categoría agrupa productos ("Granizados"), no tamaños; un
  Granizado 8oz y un Granizado 16oz están en la misma categoría y deben ser presentaciones
  distintas.
- *Tabla `presentations` global (no por tenant)*: rechazada — todo el modelo es schema-per-tenant
  (`{"schema": "tenant"}` en cada modelo); una tabla global rompería el aislamiento y el patrón
  `@for_each_tenant_schema` de las migraciones.

---

## Decisión 2 — La columna FK va en `product_variants`, no una tabla de enlace

**Decisión**: `product_variants.presentation_id` es una FK directa (una variante tiene **a lo sumo
una** presentación).

**Por qué**: una variante vendible es un tamaño concreto de un producto ("Granizado 8oz"); no tiene
sentido que pertenezca a dos presentaciones a la vez. FR-009 lo confirma: "El sistema NUNCA DEBE
mezclar unidades de presentaciones distintas dentro de un mismo paquete" — la presentación es una
propiedad singular de la variante. Una tabla de enlace M:N añadiría complejidad sin ningún caso de
uso que la pida.

**Alternativa considerada**: `variant_presentations(variant_id, presentation_id)` — rechazada por lo
anterior; además obligaría a los `SELECT` del motor a un `JOIN` extra en el camino caliente del
cobro.

---

## Decisión 3 — Las reglas viven en una tabla hija propia (`promotion_presentation_rules`), no en `PromotionTarget`

**Decisión**: se crea `promotion_presentation_rules` (schema `tenant`): `id`, `promotion_id`
(FK → `promotions.id` `ondelete="CASCADE"`), `presentation_id` (FK → `presentations.id`),
`min_qty` (int, `CHECK >= 1`), `pack_price` (`Numeric(12, 2)`, `CHECK >= 0`),
`UniqueConstraint("promotion_id", "presentation_id")`. `Promotion` gana la relación
`presentation_rules` con `cascade="all, delete-orphan"`, igual que `targets` y `combo_items`.

**Por qué**:
- `PromotionTarget` (`app/models/promotion.py:122-175`) tiene `product_id` XOR `category_id` con dos
  índices únicos parciales y `TargetIn._one_scope` (`schemas.py:76-82`) exige exactamente uno de los
  dos. Meter un tercer scope (`presentation_id`) reescribiría ese XOR, los dos índices y el
  validador, y afectaría `_matching_target` / `_scope_overlap` / `find_overlaps` —código que la
  spec 012 congela como contrato—. Una tabla nueva no toca nada de eso.
- `PromotionTarget.min_qty` tiene `CHECK "min_qty IS NULL OR min_qty >= 2"`
  (`promotion.py:162`) y la promoción `qty_price` tiene `CHECK "type <> 'qty_price' OR min_qty >= 2"`
  (`promotion.py:110-113`). Pero **CL-7 de `spec.md` exige que una regla con cantidad mínima 1 sea
  válida** ("equivale a un precio especial por unidad de esa presentación"). Reusar `PromotionTarget`
  obligaría a relajar o condicionar esos `CHECK`s, debilitando la triple capa que la spec 013
  (FR-009) protege para `qty_price`. La tabla nueva tiene su propio `CHECK min_qty >= 1` sin tocar
  los de `qty_price`.
- `promotion_combo_items` (`promotion.py:178-203`) ya es el precedente exacto: los combos NO usan
  `PromotionTarget`, tienen su tabla hija con su propia semántica y su `UniqueConstraint`. Las
  reglas por presentación siguen ese mismo molde.
- `pack_price` como columna dedicada (en vez de reusar `Promotion.value`): `Promotion.value` es un
  escalar único por promoción; una promoción de esta modalidad tiene **un precio de paquete
  distinto por regla** (`spec.md` FR-001: "cada regla es la tripleta (presentación, cantidad
  mínima, precio total del paquete)"). El precio vive en la fila de la regla, exactamente como
  `PromotionTarget.value` lo hace para `qty_price` a nivel de target.

**Alternativas consideradas**:
- *Ampliar `PromotionTarget` con `presentation_id`*: rechazada por todo lo anterior (XOR, índices,
  `CHECK`s de `min_qty`, código congelado de spec 012).
- *Reusar `PromotionComboItem`*: rechazada — su semántica es "variante + cantidad por bundle", sin
  precio por fila (el precio del combo es `Promotion.value` único); no encaja con "una presentación,
  su propio precio de paquete".

---

## Decisión 4 — Tipo de promoción nuevo `qty_price_presentation`, no una variante de `qty_price`

**Decisión**: `PROMOTION_TYPES` (`app/models/promotion.py:30`) gana el valor
`"qty_price_presentation"`; el `CheckConstraint "ck_promotion_type"` (`promotion.py:92-95`) y el de
la migración de creación se amplían. `PromotionType` en `schemas.py:9-13` y en
`promotion.interface.ts:13` ganan el enum equivalente. `AUTO_TYPES` (`service.py:47`) **NO** cambia.

**Por qué**:
- FR-006 necesita identificar "otra promoción **de este tipo** que esté activa" para el bloqueo de
  solape entre promociones. Un `type` distinto hace esa consulta trivial y exacta
  (`Promotion.type == "qty_price_presentation" AND status == "active"`); si fuera un sub-modo de
  `qty_price` habría que distinguir por "¿tiene reglas de presentación?", frágil y ambiguo.
- El motor automático línea-por-línea (`_best_line_match`, `service.py:238-276`) itera sobre
  `active_discount_promotions` filtradas a `AUTO_TYPES`. El descuento por presentación **no se
  calcula línea por línea** (agrupa varias líneas), así que debe quedar fuera de `AUTO_TYPES` —
  exactamente el mismo trato que `combo` recibe (`service.py:45-47`), que se calcula aparte en
  `combo_discount_for_lines`.
- `find_overlaps` (`service.py:495-518`) y la máquina de estados (`change_status`,
  `service.py:657-672`) ya se ramifican por `type`. Un tipo explícito hace que el comportamiento
  nuevo sea visible y auditable (Principio XII) en vez de escondido en una condición.
- El frontend ya lista `editableTypes: PromotionType[] = ['percent', 'fixed', 'qty_price', 'combo']`
  (`promotions-page.component.ts:1523`) — se agrega el nuevo ahí.

**Alternativa considerada**: `type = "qty_price"` con reglas de presentación como un modo detectado
por presencia de filas en `promotion_presentation_rules` — rechazada: hace ambigua la consulta de
FR-006, obliga a `_best_line_match` a saber que "este `qty_price` en realidad no es línea-por-línea",
y mezcla dos semánticas de cálculo bajo un mismo `type` (lo que spec 012/013 tratan como invariante).

---

## Decisión 5 — `presentation_package_discount_for_lines(db, lines, now)`: función hermana de `combo_discount_for_lines`, con desglose por línea

**Decisión**: función nueva en `promotions/service.py`, misma ubicación y forma que
`combo_discount_for_lines` (`service.py:399-448`). Firma:

```python
def presentation_package_discount_for_lines(
    db: Session, lines: list, now: datetime,
) -> PresentationDiscountResult   # total + desglose {line_index: amount}
```

Algoritmo (traducción directa de FR-009 a FR-012):

1. Traer las promociones `qty_price_presentation` con `status == "active"` y `_valid_now(p, now)`
   verdadero (hora local del tenant, mismo `_valid_now` que todo el motor — `service.py:94-109`).
2. Para cada regla (presentación, `min_qty`, `pack_price`) de esas promociones:
   a. Reunir de `lines` todas las unidades cuya variante tenga `presentation_id ==
      regla.presentation_id` **y** la variante esté `active` (FR-015: una variante desactivada no
      cuenta) **y** la línea no venga de un combo (`combo_id is None`, mismo criterio que
      `promo_lines_for`, `checkout.py:246`).
   b. `total_unidades = Σ quantity` de esas líneas; `paquetes = total_unidades // min_qty`
      (FR-010: solo paquetes completos).
   c. Si `paquetes == 0`: la regla no aplica (FR-012 para presentaciones que no alcanzan el mínimo).
   d. `precio_ref` = `_presentation_reference_unit_price(...)` (D6) — el **menor** precio unitario
      vigente entre las variantes elegibles que aportan unidades (FR-011).
   e. `unidades_en_paquete = paquetes * min_qty`; `unidades_sueltas = total_unidades -
      unidades_en_paquete`.
   f. Precio por unidad de paquete = `(pack_price / min_qty).quantize(Decimal("1"), ROUND_HALF_UP)`
      (peso colombiano, mismo modo que el redondeo final del motor, `service.py:336,448`); el
      **residuo** (`pack_price * paquetes - precio_unidad_paquete * unidades_en_paquete`) se asigna
      a la unidad que provenga de la línea con **identificador de variante más alto** — esto es
      `max` sobre `product_variants.id` (UUID); desempate: `max` sobre el `id` de la fila de línea
      (`order_items.id` / `cart_items.id` / `sale_items.id`). Ver data-model.md §"Definición
      operativa de identificador más alto". FR-011, CL-9.
   g. Las `unidades_sueltas` se cobran a `precio_ref` y se toman de la(s) línea(s) con el mismo
      criterio de "identificador más alto" del paso (f) — FR-011.
   h. Descuento de cada línea = `unidades_de_la_línea_en_esa_presentación * precio_ref` −
      `lo_efectivamente_cobrado_a_esa_línea`. La suma de descuentos por línea cuadra con el
      descuento total de la regla, al peso (SC-005).
3. El redondeo final del total sigue el patrón del motor: una sola vez, `ROUND_HALF_UP`
   (`service.py:336, 448`).

**Por qué esta forma**: `combo_discount_for_lines` ya resuelve el problema estructuralmente idéntico
—agrupar líneas por una clave (`combo_id`), contar bundles completos, usar el precio **mínimo**
cuando la misma variante llega con precios distintos (`service.py:428-431`), descontar solo bundles
completos— y ya está integrado como "descuento aparte que se suma" en los ~10 call-sites. Copiar ese
molde minimiza el riesgo y la superficie de cambio. La diferencia (combos agrupan por `combo_id`
explícito en la línea; presentación agrupa por `presentation_id` de la variante) es un cambio de
clave de agrupación, no de arquitectura.

**Diferencia clave que sí se agrega**: `combo_discount_for_lines` devuelve un `Decimal` escalar;
esta función devuelve **también el desglose por línea**, porque FR-011 y SC-005 exigen que el
reparto por línea sea explícito y cuadre al peso, y FR-013 necesita saber el resultado por línea
para reconciliar contra la promoción de producto (D6). Es el mismo salto conceptual que
`evaluate_detailed` hizo sobre `evaluate` (spec 012, `service.py:11-16`).

**Alternativa considerada**: extender `evaluate_detailed` para que también haga la agrupación por
presentación — rechazada: `evaluate_detailed` está construido sobre `_best_line_match`, que es
estrictamente línea-por-línea; meterle un paso de agrupación previa mezclaría dos modelos de
cálculo en una función que spec 012 congela. La separación (una función por mecanismo) es la que el
propio motor ya eligió para los combos.

---

## Decisión 6 — Precio de referencia único y coexistencia con la promoción de producto (FR-011, FR-013, FR-023)

**Decisión — precio de referencia (FR-011)**: `_presentation_reference_unit_price` toma, de las
líneas que aportan unidades a esa presentación en este cobro, el **menor** `unit_price` vigente
(el de la línea, que ya incluye recargos de opciones si los hubiera — mismo `unit_price` que
`SaleLine` y `promo_lines_for` manejan). Ese único valor se aplica a todas las unidades de esa
presentación (en paquetes y sueltas). El descuento nunca se calcula variante por variante. Es el
mismo criterio "el cliente no paga el alto" que `combo_discount_for_lines` ya aplica con `min(...)`
(`service.py:428-431`), citado literalmente en la clarification del 2026-08-26.

**Decisión — coexistencia (FR-013, FR-023)**: el descuento por presentación entra al mecanismo de
"mejor promoción por línea" como **candidata adicional**, con recálculo del pool hasta punto fijo
(el precio de paquete de una línea depende de qué otras líneas siguen en el pool):

1. `pool` = todas las líneas elegibles para alguna regla de presentación vigente (D5 paso 2a).
2. Se calcula `evaluate_detailed` sobre **todas** las líneas (motor línea-por-línea:
   `percent`/`fixed`/`qty_price` de producto/categoría) — un `LineDiscount` por línea. Una sola vez;
   no depende del `pool`.
3. Se calcula `presentation_package_discount_for_lines` sobre `pool` (D5) — desglose por línea.
4. Para cada línea `L` de `pool` que `evaluate_detailed` también descontaría, se compara el total
   que deja `L` bajo el desglose por presentación vs. bajo `evaluate_detailed`. `L` **sale de
   `pool`** si (a) `evaluate_detailed` deja `L` con total **estrictamente menor** (FR-013: gana la
   de menor total; empate exacto en el total de esa línea → `L` se queda en `pool`, elección
   determinista, el cliente paga igual por esa línea), o (b) aplicar el descuento por presentación
   deja `L` con total **mayor** que sin ninguna promoción (FR-023).
5. Si alguna línea salió de `pool` en el paso 4, volver al paso 3 (recalcular el paquete sobre el
   `pool` más chico — menos unidades ⇒ quizá menos paquetes completos, y `precio_ref` puede subir si
   la línea que salió tenía la variante más barata). Si no salió ninguna, terminar.
6. Resultado final: las líneas que quedan en `pool` con descuento por presentación positivo →
   descuento por presentación; las que salieron de `pool` (o nunca estuvieron) y `evaluate_detailed`
   las descuenta → descuento línea-por-línea; el resto → sin descuento. Ninguna línea acumula dos.

**Terminación y determinismo**: `pool` solo se encoge, así que el bucle converge en ≤ |pool|
iteraciones. El criterio de salida del paso 4 es un orden total con empate roto de forma fija ("se
queda en `pool`"), así que el resultado no depende del orden de las líneas del pedido (SC-005,
CA-5).

**Dónde vive esta reconciliación**: en una función nueva, `combined_discount_detailed(db,
promo_lines, now)`, que orquesta los tres mecanismos (línea-por-línea, combos, presentación) y
devuelve un total + `promotion_id` único cuando corresponda (misma regla que `single_promotion_id`,
`service.py:191-195`). Los call-sites la llaman en vez de encadenar `evaluate` +
`combo_discount_for_lines` a mano (D7). Los combos siguen sin competir con nada (sus líneas se
filtran por `combo_id`, sin cambio).

**Por qué no reescribir `_best_line_match`**: el bucle (pasos 3-5) envuelve la comparación por línea
sin tocarla — `_best_line_match` / `evaluate_detailed` siguen viendo solo montos por línea, sin
saber que uno de ellos vino de un cálculo agrupado que puede recalcularse. Cero cambios a la función
que spec 012 congela.

**Alternativa considerada**: una sola pasada sin recálculo del `pool` (comparar por línea el
desglose de presentación calculado sobre el `pool` máximo contra `evaluate_detailed`) — rechazada:
una línea cuya "pareja de paquete" se va al descuento de producto seguiría cobrándose a
`precio_paquete ÷ min_qty` aunque sus unidades ya no completen un paquete, rompiendo FR-013 y el
cuadre de SC-005. Ejemplo: 3× X (8oz) + 1× Y (8oz), regla "2×8oz $12.000" + producto "qty_price
3×X $15.000": X se va al producto, Y sola no forma paquete → total correcto $22.000, no $21.000.

**Alternativa considerada**: aplicar siempre el descuento por presentación primero y dejar que el
motor línea-por-línea solo compita por lo que sobre — rechazada: no garantiza "menor total por
línea" (FR-013) cuando la promoción de producto sería mejor para una línea concreta.

---

## Decisión 7 — Punto de enganche: `combined_discount_detailed` en los ~10 call-sites, sin migrar el resto de `evaluate`

**Decisión**: se identifican los call-sites que hoy combinan `promotions.evaluate(...)` +
`promotions.combo_discount_for_lines(...)` y se los cambia para llamar
`promotions.combined_discount_detailed(db, promo_lines_for(db, lines), now)` (que internamente hace
las tres cosas, D6). `promo_lines_for` (`checkout.py:240-256`) agrega `presentation_id` a cada dict
de línea. **No** se migra el resto de `evaluate` → `evaluate_detailed` (deuda de spec 012, fuera de
alcance, Principio V).

**Call-sites** (verificados con `grep -rn "promotions.evaluate\|combo_discount_for_lines"`):

| # | Fichero:línea | Camino | Cobro real / preview |
|---|---|---|---|
| 1 | `orders/checkout.py:275` | `pay_order` (orden de mesa legada) | real |
| 2 | `orders/checkout.py:466` | `checkout_and_send` (mostrador/mesero cobra antes de cocina) | real |
| 3 | `orders/checkout.py:849` | `approve_payment_attempt` (transferencia aprobada) | real |
| 4 | `orders/checkout.py:969` | `confirm_cash` (efectivo confirmado) | real |
| 5 | `orders/checkout.py:136` | `compute_bill` (preview de cuenta por mesa) | preview |
| 6 | `table_sessions/service.py:186` | `compute_bill` (preview por comensal / split) | preview |
| 7 | `table_sessions/service.py:656` | `release_paid_session` | real |
| 8 | `table_sessions/service.py:753` | `_close_split` (cierre dividido) | real |
| 9 | `sales/service.py:279` | venta de mostrador directa | real |
| 10 | `orders/tables_advanced.py:155` | `group_bill` (cuenta de mesas fusionadas) | preview |
| 11 | `cart/service.py:255` (`best_line_discount`) + `:635` (`submit_cart`) | carrito/preview del comensal QR | preview |
| 12 | `menu/router.py:157` (`best_line_discount`) | menú público (precio con descuento por variante) | preview |

Los call-sites 1-10 pasan por `combined_discount_detailed`. Los 11-12 usan `best_line_discount`
directamente para pintar el precio por variante; ahí el descuento por presentación **no se
previsualiza a nivel de precio unitario** (depende de cuántas unidades combinadas haya en el
pedido, que en el menú aún no existe — mismo motivo por el que `menu/router.py:152-153` ya asume
"cantidad 1" y `qty_price` no baja precio en el menú). El carrito del comensal (11) **sí** puede
mostrar el descuento por presentación en su `discounted_total` porque ahí ya conoce las cantidades
reales (`serialize_cart`, `cart/service.py:269`): llama `combined_discount_detailed` sobre las
líneas del carrito, sin persistir nada (FR-014).

**Por qué un solo helper y no editar 10 bloques a mano cada vez**: hoy los 10 bloques repiten
literalmente 4 líneas (`evaluate` + `combo_discount_for_lines` + `combo_ids_used` +
`final_promotion_id`). `combined_discount_detailed` encapsula esas 4 + la reconciliación nueva (D6)
en una sola llamada, reduciendo la superficie donde el descuento por presentación podría quedarse
sin integrar. No es una refactorización oportunista (Principio V): es el cambio mínimo para que la
funcionalidad nueva llegue a todos los caminos sin copiar la lógica de reconciliación 10 veces.

**Alternativa considerada**: sumar `presentation_package_discount_for_lines` como una tercera línea
suelta en cada uno de los 10 bloques, sin helper — rechazada: la reconciliación de FR-013 (elegir
por línea entre presentación y producto) no es una suma, es una comparación con estado; repetirla
10 veces es exactamente el tipo de duplicación que causa que un camino quede mal (spec 012 A-08
documenta un caso así de "la convención no llegó a todos los puntos").

---

## Decisión 8 — Validación de solape de reglas (FR-006): dentro de la promoción y contra otras activas del mismo tipo

**Decisión**: dos chequeos, ambos en `promotions/service.py`, invocados desde `create` /
`update_shape` y **además** desde `change_status` cuando el destino es `active`:

1. **Dentro de la promoción** (FR-006, 1ª parte): no se permiten dos filas de
   `promotion_presentation_rules` con la misma `presentation_id`. Se refuerza en tres capas, igual
   que el resto de reglas de forma (spec 013 FR-008/FR-009): validador Pydantic sobre
   `presentation_rules` (lista sin `presentation_id` repetido), chequeo en el servicio, y
   `UniqueConstraint("promotion_id", "presentation_id")` en la tabla como última red.
2. **Contra otras promociones** (FR-006, 2ª parte): al guardar o activar, si alguna regla apunta a
   una `presentation_id` ya cubierta por una regla de **otra** promoción
   `type == "qty_price_presentation"` con `status == "active"`, se rechaza con **409** y el detalle
   nombra la(s) promoción(es) en conflicto. A diferencia de `find_overlaps` (que es solo
   advertencia, spec 012 FR-034), **este solape sí bloquea** — es una decisión de negocio explícita
   de la clarification del 2026-08-26 (CL-4).

**Por qué en `change_status` también**: una promoción puede crearse en `draft` sin conflicto y
activarse más tarde, cuando otra ya ocupó esa presentación. El chequeo solo al crear dejaría pasar
ese caso. Mismo patrón que `change_status` ya revalida `>= 2` componentes de combo al activar
(`service.py:665-669`, spec 013 FR-003).

**Alternativa considerada**: reusar `find_overlaps` — rechazada: `find_overlaps` compara ventanas de
fecha/hora/día y devuelve advertencia, no bloqueo; FR-006 pide bloqueo duro sobre coincidencia de
`presentation_id` entre promociones activas del mismo tipo, que es una condición distinta y más
simple.

---

## Decisión 9 — Uniformidad de precio (FR-017) y "no es descuento real" (FR-022): confirmación explícita vía flag en el payload

**Decisión**: `PromotionCreate` y `PromotionShapeUpdate` ganan dos campos booleanos opcionales
(`confirm_precio_no_uniforme` y `confirm_sin_descuento`, default `False`). Al guardar una regla:

- **FR-017**: el servicio calcula, para cada regla, el conjunto de variantes activas que la
  referencian y sus precios vigentes. Si no todos son iguales y `confirm_precio_no_uniforme` es
  `False` → **422** con un detalle estructurado (lista de variantes y precios divergentes, y el
  `precio_ref` = el menor, que es el que se cobrará). Si el flag es `True` → se guarda igual.
- **FR-022**: si `pack_price / min_qty >= precio_ref` (la regla no representa descuento) y
  `confirm_sin_descuento` es `False` → **422** con el detalle. Con el flag `True` → se guarda.

El frontend muestra el detalle en un diálogo, y al confirmar reenvía el mismo payload con el flag en
`True` (`promotions-page.component.ts`). "La regla PUEDE guardarse igual, pero NUNCA en silencio"
(FR-022) se cumple: sin el flag no pasa.

**Por qué un flag y no un `?force=true` en la query**: los otros overrides del proyecto que "avisan
y dejan continuar" no existen aún en promociones; el patrón más cercano es el detalle estructurado
de error que ya usa el checkout (`checkout.py:963-967` monto insuficiente). Un flag en el body
mantiene la operación idempotente y auditable (el `record_audit` de `create`/`update_shape`,
`router.py:67-69,104-105`, registra que se confirmó).

**FR-018 (no revalida retroactivamente)**: el chequeo corre **solo** en `create` / `update_shape`.
Un producto que cambia de precio después, o una variante nueva a la que se asigna la presentación de
una regla activa (CL-1b), no dispara nada — no hay job ni trigger que revise uniformidad. `precio_ref`
se recalcula en cada cobro (FR-011, CL-3), y esa variación es comportamiento esperado, no alerta
pendiente.

**Alternativa considerada**: bloquear duro (422 sin override) si el precio no es uniforme —
rechazada: `spec.md` (Historia 3, FR-017) es explícita en que "la regla puede guardarse igual".

---

## Decisión 10 — Baja de presentación bloqueada mientras una regla activa la referencia (FR-020)

**Decisión**: `DELETE /presentations/{id}` y `PATCH /presentations/{id}` con `active=false` chequean
primero si existe alguna fila en `promotion_presentation_rules` cuya `promotion.status == "active"`
apunte a esa presentación. Si la hay → **409**, con la lista de promociones que la referencian en el
detalle, y el mensaje pide editar o pausar esas promociones antes de continuar (FR-020, CL-2).

Si **ninguna** promoción activa la referencia, la baja procede. Al borrar, la FK
`product_variants.presentation_id` (`ondelete="SET NULL"`, D1) deja las variantes sin presentación.
Reglas de promociones en `draft`/`paused`/`finished` no bloquean (solo `active`, como en FR-006) —
pero sus filas quedarían apuntando a una presentación inexistente: se resuelve con
`ondelete="CASCADE"` en `promotion_presentation_rules.presentation_id` para reglas huérfanas, o
`ondelete="RESTRICT"` para forzar limpieza manual. **Se elige `CASCADE`**: una regla de una
promoción no activa que pierde su presentación no tiene sentido y arrastrarla sería basura; una
promoción `draft` con una regla menos se puede volver a completar antes de activar (que revalida
todo, D8).

**Por qué `active` y no cualquier estado**: FR-020 dice "una regla de una promoción **activa**".
Una promoción en `draft` todavía se está armando; bloquear la gestión del catálogo por un borrador
sería demasiado restrictivo y la spec no lo pide.

---

## Decisión 11 — Entrada automática de variantes nuevas (FR-007/FR-019): sale del diseño, sin código adicional

**Decisión**: no se necesita ningún mecanismo especial. `presentation_package_discount_for_lines`
(D5) resuelve el conjunto de variantes elegibles **en cada cobro** con un `SELECT` sobre
`product_variants WHERE presentation_id == regla.presentation_id AND active` — nunca contra una
lista materializada al crear la promoción. Una variante creada después con esa `presentation_id`
entra en el siguiente cobro automáticamente (FR-019, CA-9), sin editar la promoción. Una variante
sin `presentation_id` nunca aparece en ese `SELECT` (FR-008).

**Verificación**: se confirma leyendo `combo_discount_for_lines` — hace lo análogo con la receta del
combo (`SELECT PromotionComboItem WHERE promotion_id == combo_id`, `service.py:415-417`) en cada
llamada, sin cache. El mismo patrón "resolver el alcance al momento de calcular" es lo que hace que
FR-019 sea gratis.

---

## Decisión 12 — Anuncio en el menú QR (FR-021): endpoint hermano, sin tocar `_build_menu`, filtrado por vigencia instantánea

**Decisión**: `_build_menu` (`menu/router.py:80`) **NO cambia de firma** — sigue devolviendo
`list[MenuCategoryResponse]`. Es entrada del test `"CONGELA comportamiento corregido:"` de
`test_menu_router.py` (`menu = _build_menu(db); menu[0].products[0]...`), de la fixture compartida
`cart_fixtures.py:379` y del endpoint QR con token (`menu/router.py:210-214`); cambiar su retorno
rompería un test protegido (Principio III) y obligaría a una decisión de negocio que la spec no
contempla.

Los anuncios se exponen aparte, con una función nueva `_build_menu_promotions(db, now) ->
list[MenuPromotionAnnouncement]` (hermana de `_build_menu`, mismo módulo) y dos superficies: un
endpoint hermano nuevo `GET /menu/promotions` (público) y la clave `"promotions"` del `dict` del
flujo QR con token. Cada anuncio lleva un texto legible por regla (p. ej. "Llevando 2 de cualquier
sabor en presentación 8oz por $12.000") construido en el backend a partir de `(presentación.name,
min_qty, pack_price)`. Se incluye solo si la promoción `qty_price_presentation` pasa
`_valid_now(p, now)` —**vigencia en ese instante**, no solo `status == "active"`— usando la hora
local del tenant (FR-021, aclaración 2026-08-26, SC-006). Fuera de la ventana de día/hora no se
anuncia.

`menu/router.py:82` ya construye `now = datetime.now(timezone.utc)` (aware), y `_valid_now` →
`local_now` (`service.py:62-73`) lo convierte a hora local correctamente — este camino **no**
arrastra el bug A-08 (que afecta a `cart/service.py`/`menu` cuando construyen `now` naive); se
verifica que `menu/router.py` usa el `datetime` aware.

El frontend (`public-menu.component.ts`) renderiza un banner con esos textos, visible sin agregar
nada al carrito (CA-11).

**Por qué un campo nuevo y no reusar `discounted_price` por variante**: FR-021 pide un anuncio de la
**condición** de la promoción ("llevando 2..."), independiente de cualquier variante y visible con
el carrito vacío. `discounted_price` es un precio por variante que además no baja para `qty_price`
(cantidad 1 asumida). Son dos cosas distintas.

**Alternativa considerada**: anuncio por categoría en vez de a nivel de menú — rechazada: una
presentación ("8oz") puede abarcar variantes de productos en categorías distintas; el anuncio es del
menú, no de una categoría.

**Alternativa considerada**: cambiar la raíz de `GET /menu` (y el retorno de `_build_menu`) a un
objeto `{categories, promotions}` — rechazada: rompe el `CONGELA` de `test_menu_router.py`, la
fixture `cart_fixtures.py:379` y el endpoint QR, todos los cuales tratan el retorno como lista
(Principio III).

---

## Decisión 13 — El frontend no previsualiza el precio de esta modalidad en el cliente; el backend es la autoridad

**Decisión**: `promotion-pricing.util.ts` se toca lo mínimo — `getPromoDisplay`
(`promotion-pricing.util.ts:245`) reconoce `qty_price_presentation` para elegir la insignia, pero el
tipo **queda fuera** de `bestProductDiscount` / `discountInfo`, igual que `qty_price` ya está fuera
("`qty_price` no puede cuantificarse y queda fuera de la previsualización",
`promotion-pricing.util.ts:127`). El precio y el total con descuento los calcula siempre el backend:
el POS de staff los toma de `compute_bill` / del preview del cobro (call-sites 5-8 en D7), no de un
cálculo local.

**Por qué**: el descuento por presentación depende de cuántas unidades combinadas de esa
presentación hay en todo el pedido (varias líneas, varios productos) — el `util` opera línea por
línea y no tiene ese contexto. Replicarlo en TypeScript reintroduciría el riesgo de divergencia que
`registro-de-anomalias.md` A-09/A-10 ya documenta para el port. El proyecto ya aceptó este trade-off
para `qty_price`; esta modalidad sigue la misma línea. La insignia ("2 x 8oz por $12.000") sí se
muestra a partir de los datos de la regla, sin recalcular montos.

---

## Decisión 14 — Tests: qué se agrega, qué `CONGELA` se verifica intacto, sin entrada en `registro-de-anomalias.md`

**Decisión — tests nuevos** (`unittest`, SQLite en memoria, mismo patrón que el resto de
`characterization_tests`), uno por historia:

| Historia | Fichero | Qué cubre |
|---|---|---|
| US1 | `test_promotions_presentation_rules.py` (nuevo) | resumen de reglas + conteo de aplicables (CA-1); rechazo de 2ª regla misma presentación (CA-2); 409 de solape entre promociones activas (CA-3); ventana 22:00–02:00 aceptada (RF-10) |
| US2 | `test_promotions_presentation_pricing.py` (nuevo) | los 10 Acceptance Scenarios de cálculo (2×$12.000, 3 unidades → $19.000 con reparto determinista, otro orden → idéntico, 8oz+16oz → $28.500, 5 unidades, sin mínimo, día no incluido, ventana horaria, **división no exacta "3×8oz $10.000" → residuo/CL-9**, **1 activa + 1 `active=false` → $14.000/FR-015**); SC-005 (suma de descuentos por línea cuadra al peso, incl. el caso con residuo) |
| US3 | `test_promotions_presentation_rules.py` | uniformidad: 422 sin flag, guarda con flag (CA-10); no revalida retroactivamente (CL-1); variante nueva sin re-chequeo (CL-1b); FR-022 "no es descuento real" |
| US4 | `test_presentations_service.py` (nuevo) + `test_promotions_presentation_pricing.py` | variante nueva entra sin editar la promoción (CA-9); baja de presentación referenciada bloqueada con 409 (CL-2) |
| US5 | `test_menu_router.py` (MODIFICADO, casos nuevos) | anuncio visible dentro de ventana; ausente fuera de ventana (SC-006). Los `CONGELA` de este fichero NO se tocan |
| FR-013/FR-023 | `test_orders_checkout.py` (MODIFICADO, casos nuevos) | coexistencia con `qty_price` de producto sobre la misma línea (gana la de menor total, nunca ambas, nunca deja peor); **recálculo del pool (D6): 3×X + 1×Y en 8oz, regla "2×8oz $12.000" + producto "3×X $15.000" → X al producto, Y a precio normal, total $22.000 no $21.000** |

**Decisión — `CONGELA` y CI, verificados intactos** (no asumidos):

- `app/scripts/test_promotions_rules.py` (único script en CI, `.github/workflows/deploy.yml`): sus
  `check()` ejercitan `_valid_now`, `_in_time_window`, `_matching_target`, `_line_discount`,
  `best_line_discount` (`test_promotions_rules.py:45,183-219`). `_in_time_window`, `_matching_target`,
  `_line_discount` y `best_line_discount` no cambian de firma ni de cuerpo — se agregan funciones
  hermanas (`presentation_package_discount_for_lines`, `combined_discount_detailed`,
  `_presentation_reference_unit_price`). **`_valid_now` sí cambia de cuerpo** (FR-004, D18, A-55): la
  atribución de día al cruzar medianoche. Los `check()` actuales del script usan ventanas normales y
  siguen en verde; se agrega un `check()` para la combinación "día restringido + ventana que cruza
  medianoche". Ningún test `"CONGELA comportamiento actual:"` se edita.
- `app/characterization_tests/test_promotions_router.py` (tiene `CONGELA`): ejercita el CRUD y la
  paginación de `GET /promotions` (`test_promotions_router.py:34,50`). El tipo nuevo se suma al
  enum; los casos existentes (percent/fixed/qty_price/combo) siguen validando igual. Se verifica en
  verde; no se edita.
- `test_menu_router.py`, `test_cart_service.py`, `test_catalog_line_pricing.py`,
  `test_orders_checkout.py`: se agregan casos nuevos en ficheros que ya existen; los `CONGELA`
  presentes se dejan intactos y se corren en verde como parte de `quickstart.md`.

**Decisión — una sola entrada en `registro-de-anomalias.md` (A-55), por FR-004**: la modalidad de
descuento en sí no es la corrección de un comportamiento existente, es una modalidad nueva que se
suma (`spec.md` §"Naturaleza de esta spec", §"Out of Scope" 5º ítem; FR-016) — para eso el Principio
II no exige registro. **La excepción** es FR-004 (D18): corrige `_valid_now` para que las horas tras
la medianoche de una ventana que cruza medianoche pertenezcan al día de inicio, y eso cambia el
comportamiento observable de promociones existentes de todos los tipos → se registra como **A-55**.
No es retroactivo (Principio VII).

---

## Decisión 15 — Migración: una sola revisión `@for_each_tenant_schema`, con rollback simétrico

**Decisión**: una revisión Alembic nueva, `down_revision = "187e491e597a"` (head actual verificado
con `alembic heads`). `upgrade` y `downgrade` decorados con `@for_each_tenant_schema` y guardados
con `_has_table(schema, "product_variants")` / `_has_table(schema, "promotions")` (patrón de
`d3e4f5a6b7c8_promotions.py` y `e3f4a5b6c7d8_products_tracks_inventory.py`):

`upgrade(schema)`:
1. `create_table("presentations", ...)` — `id`, `name` (con `UniqueConstraint` por schema),
   `active` (`server_default="true"`), `created_at`/`updated_at`.
2. `create_table("promotion_presentation_rules", ...)` — FKs a `promotions` (`ondelete="CASCADE"`) y
   `presentations` (`ondelete="CASCADE"`, D10), `CHECK min_qty >= 1`, `CHECK pack_price >= 0`,
   `UniqueConstraint("promotion_id", "presentation_id")`, índice en `promotion_id`.
3. `add_column("product_variants", Column("presentation_id", UUID, nullable=True))` +
   `create_foreign_key(... ondelete="SET NULL")` + índice.
4. Ampliar `ck_promotions_type`: `drop_constraint` + `create_check_constraint` con la lista
   `('percent','fixed','combo','qty_price','qty_price_presentation')`. (El `CHECK` de creación
   original en `d3e4f5a6b7c8` incluía `buy_x_get_y`; se replica el `CHECK` **vigente** del modelo,
   `promotion.py:92-95`, más el valor nuevo.)

`downgrade(schema)`: exactamente inverso — restaurar el `CHECK` anterior, `drop_column`
`presentation_id` (+ FK, + índice), `drop_table` `promotion_presentation_rules`, `drop_table`
`presentations`.

**Sin pérdida de dato histórico en el rollback**: nada histórico se escribe en estas estructuras
(el descuento nunca se persiste, FR-014; `Sale.promotion_id` es una FK `SET NULL` que no se rompe si
la promoción se borra). Revertir la migración deja el sistema exactamente como antes de la spec.

**Sin backfill**: `presentation_id` nace `NULL` para todo el catálogo (compatibilidad hacia atrás).
Asignar presentaciones a las variantes existentes es trabajo del administrador desde el formulario
de producto, no de la migración (la spec no pide migrar nada automáticamente — §"Out of Scope").

---

## Decisión 16 — Unidad de agrupación de un paquete: las líneas evaluadas juntas en ese cobro

**Decisión**: `presentation_package_discount_for_lines` agrupa las líneas que recibe en esa llamada
— igual que `combo_discount_for_lines`. En un cobro unificado (mostrador, `checkout_and_send`,
`_close_unified`) eso es el pedido completo. En un **cierre dividido por comensal** (`_close_split`,
`compute_bill` por comensal, `table_sessions/service.py:181-188`) eso es el subconjunto de líneas de
cada comensal — los paquetes se forman dentro de la porción de cada quien.

**Por qué**: FR-011 habla de "las variantes elegibles que aportan unidades a esa presentación **en
el pedido**", y FR-014 exige "recalcular desde el estado actual del pedido cada vez". El sistema ya
evalúa promociones y combos **por comensal** en el split (`compute_bill` lo documenta:
"mismo cálculo por comensal que usa `_close_split`"). La modalidad nueva hereda esa unidad de
agrupación sin inventar una distinta — dos sabores del mismo comensal combinan; dos sabores de dos
comensales que piden cuentas separadas, no (cada uno paga su porción). El caso central de la spec
("un cliente que lleva dos sabores") es un único pagador y queda cubierto.

**Nota**: esto no está en un FR explícito de `spec.md` porque sus Independent Tests usan cobro
unificado; se documenta aquí como la lectura coherente con el resto del motor. Si el negocio
quisiera que los paquetes se formaran a nivel de mesa completa incluso en cuentas separadas, sería
un cambio posterior con su propia decisión (no se asume).

---

## Decisión 17 — Entrega incremental (Principio VI)

**Decisión**: 5 incrementos, cada uno verificable por separado antes de seguir:

- **A — Catálogo de presentaciones**: tabla `presentations`, `product_variants.presentation_id`,
  CRUD backend (`api/v1/presentations/`), CRUD frontend (`modules/presentations/`), selector en el
  formulario de producto, baja bloqueada por regla activa (FR-020). Verificable sin ninguna
  promoción nueva: se crean presentaciones, se asignan a variantes, se intenta borrar una en uso.
- **B — Tipo y reglas de promoción**: `qty_price_presentation`, `promotion_presentation_rules`,
  schemas, validaciones de forma/solape (FR-006), uniformidad (FR-017) y "no es descuento real"
  (FR-022), panel "Productos Aplicables" + "Resumen" en el formulario. Verificable creando/activando
  promociones y comprobando los rechazos, **sin** que el cálculo del cobro las use todavía.
- **C — Motor de evaluación**: `presentation_package_discount_for_lines`,
  `_presentation_reference_unit_price`, `combined_discount_detailed`, integración en los 10
  call-sites de cobro/preview (D7) + carrito del comensal. Verificable con los Acceptance Scenarios
  de US2 y el de coexistencia FR-013/FR-023.
- **D — Anuncio en menú QR**: campo `promotions` en la respuesta del menú, filtro por vigencia
  instantánea, banner en `public-menu.component.ts` (FR-021, US5).
- **E — Superficies de preview de staff/comensal**: insignia del tipo nuevo en
  `promotion-pricing.util.ts` / panel de staff, ajuste de `discounted_total` del carrito. (Parte ya
  cubierta por C para el backend; E es el pulido de UI.)

Ningún incremento mezcla la migración (solo en A) con cambio de comportamiento de cobro (solo en C).
`tasks.md` (Fase 2) ordenará las tareas dentro y entre incrementos.

---

## Decisión 18 — FR-004 (atribución de día al cruzar medianoche): corregir `_valid_now` para todos los tipos

**Hallazgo (2026-08-27, durante `/speckit-analyze`)**: `_valid_now` (`service.py:94-109`) evalúa
`days_of_week` contra `now.weekday()` y la ventana horaria contra `_in_time_window` de forma
**independiente**. Con una ventana que cruza la medianoche (`start_time > end_time`) y `days_of_week`
restringido, la promoción no descuenta en el tramo posterior a la medianoche del día correcto (una
promo "lunes 22:00–02:00" evaluada el lunes 00:30 —que el sistema lee como martes— sale no vigente).
`_in_time_window` (corrección previa) solo arregló la comparación de **hora**, no la de **día**.
FR-004/CL-8 exigen la semántica correcta y ninguna tarea la implementaba; `data-model.md` afirmaba
erróneamente que "ya está soportado".

**Decisión**: corregir `_valid_now` en el sitio compartido, no envolver solo el tipo nuevo. Cuando
`start_time > end_time` (ventana que cruza medianoche) **y** `now.time() <= end_time` (estamos en el
tramo posterior a la medianoche), la fecha de referencia para evaluar vigencia es
`(now - timedelta(days=1))`: se compara contra `days_of_week` (`.weekday()`) y contra `ends_at`
(`.date()`). `starts_at` y `start_time` no cambian. En cualquier otro caso, sin cambio.

**Por qué global y no solo para `qty_price_presentation`**:
- Duplicar la lógica de vigencia solo para el tipo nuevo deja el mismo defecto vivo para
  `percent`/`fixed`/`qty_price`/`combo` y crea dos definiciones de "vigente ahora" que pueden
  divergir — justo lo que el motor unificado evita.
- El cambio es acotado y verificable: un `check()` nuevo en `test_promotions_rules.py` para la
  combinación, más los char test de US1/US2. Los `check()` existentes de `_valid_now` (ventanas
  normales, líneas 76-80) y de `_in_time_window` (función pura, líneas 92-96) no se ven afectados.
- Es comportamiento de negocio observable en promociones existentes → se registra como **A-55** en
  `registro-de-anomalias.md` (Principio II). No es retroactivo (Principio VII): no recalcula ninguna
  venta ya emitida.

**Alternativa considerada**: envoltura `_presentation_valid_now` solo para el tipo nuevo — rechazada
por la duplicación y la divergencia potencial descritas arriba.
