# Phase 0 — Research: promociones legibles y precios reales en el menú QR

**Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md) · **Fecha**: 2026-09-01

La spec entró a planificación **sin ningún `NEEDS CLARIFICATION`**: la sesión de
`/speckit-clarify` del 2026-09-01 cerró las nueve preguntas abiertas (formato del descriptor,
qué precio muestra el modal, forma de la insignia, alcance, si el equivalente cobra, el caso
`min_qty == 1`, redondeo y marca de aproximado, quién registra las anomalías, y quién arma la
información por variante). Este documento no resuelve incógnitas de negocio: **resuelve las
decisiones técnicas** y registra lo que el reconocimiento del código encontró y la spec no sabía.

---

## Estado del código verificado (2026-09-01)

Ambos repositorios están en `develop`, con la spec 063 ya mergeada:

- `pos-backend` — `develop`, `199b3f5` (`#51`, spec 064 encima de la 063).
- `pos-heladeria` — `develop`, `302d7ee` (`#52`, spec 064 encima de la 063).

Puntos exactos que esta feature toca:

| Qué | Dónde | Hoy |
|---|---|---|
| Texto de condición | `promotions/service.py:313` `variant_set_condition_text(rule)` | Solo lee `type`/`value`/`min_qty`/`len(rule.variants)`. **No tiene acceso a ningún nombre.** |
| Call sites del texto | `menu/router.py:206`, `promotions/service.py:685` | Exactamente **dos**. |
| Precio con descuento | `menu/router.py:158-166` vía `menu_unit_discount` (`service.py:300`) | Solo `percent` + `min_qty == 1`; el resto devuelve `None`. |
| Insignia de la tarjeta QR | `public-menu.component.ts:382-389` + `productDiscount()` (`:775`) | Derivada de `discounted_price` de la presentación más barata. |
| Modal de presentaciones | `product-select.component.ts:63-93` | Muestra precio y, si hay `discounted_price`, tachado + descuento. |
| Insignia de la terminal | `pos-terminal.store.ts:410-424` `productDiscountBadge` | Calculada localmente desde `activePromotions()`; conserva su etiqueta (FR-016). |
| Vista previa del formulario | `promotions-page.component.ts:1070` `ruleConditionPreview` | Réplica local del texto por conteo. |

---

## D-1 — El descriptor necesita nombres: firma **obligatoria**, no opcional

**Decisión**: `variant_set_condition_text(rule, names)` donde `names: Mapping[UUID, str]` es
**parámetro posicional obligatorio**.

**Razón**: la función hoy recibe `rule` y nada más; los `PromotionVariant` de `rule.variants` solo
llevan `product_variant_id`. Hay que inyectar los nombres desde fuera. La tentación es hacerlo
opcional (`names=None` → texto por conteo) para no romper nada, pero eso convierte un call site
olvidado en una **degradación silenciosa**: la superficie que lo olvide seguiría emitiendo
`"de estas 8 variantes"` y **rompería SC-005** (texto idéntico carácter por carácter en las tres
superficies) sin que ningún test lo grite. Con el parámetro obligatorio, un call site sin
actualizar es un `TypeError` en el arranque de los tests. Son solo dos call sites: el coste de
hacerlo obligatorio es nulo.

**Alternativas consideradas**:
- *`names` opcional con respaldo al conteo* — descartada por la degradación silenciosa de arriba.
- *Cargar los nombres dentro de la función con una `Session`* — obligaría a pasar `db` a una
  función hoy pura, la haría intestable sin base de datos (`app/scripts/test_promotions_rules.py`
  la ejercita con `SimpleNamespace`) y dispararía una consulta por regla (N+1 en el cartel).
- *Persistir el descriptor en `promotion_rules`* — violaría FR-019 (sin cambios de modelo) y se
  quedaría obsoleto en cuanto alguien renombrara una variante del catálogo.

---

## D-2 — Orden determinista sin tildes ni mayúsculas: `unicodedata`, sin dependencia nueva

**Decisión**: la clave de ordenación de FR-002 es `NFD` + descarte de marcas combinantes +
`casefold()`, con la biblioteca estándar. En Python `unicodedata.normalize("NFD", s)` +
`unicodedata.combining(c)`; en TypeScript `s.normalize('NFD').replace(/\p{M}/gu, '').toLowerCase()`.

**Razón**: FR-002 exige orden alfabético *"sin distinguir mayúsculas ni tildes para que el texto
sea determinista"*, y SC-005 exige que backend y vista previa del formulario coincidan carácter
por carácter. Sin normalizar, `"Ácai"` y `"Almendra"` se ordenan distinto en Python (por punto de
código) que con un colador local. Ambos lenguajes traen la normalización Unicode de fábrica:
**ninguna dependencia nueva** (Principio IX ni se activa).

**Alternativas consideradas**: `PyICU` / `Intl.Collator` con `locale: 'es'` — descartadas por
introducir una dependencia (Python) y por depender de la configuración regional del navegador
(TypeScript), que es justo lo contrario de "determinista entre superficies".

---

## D-3 — El nombre utilizable, y qué pasa cuando no hay ninguno

**Decisión**: para cada variante del conjunto, el nombre utilizable es
`variant.name.strip()` y, si queda vacío, `product.name.strip()`. Si tampoco hay, esa variante
**no aporta nombre**. Si **ninguna** variante del conjunto aporta uno, el texto vuelve al
descriptor por conteo actual (FR-006). Una regla de tipo retirado sigue devolviendo `None` antes
de mirar nada (comportamiento intacto).

**Razón**: es literalmente FR-006. Lo relevante para la implementación es el orden de las guardas:
`type not in LIVE_TYPES` → `None` **primero** (una promoción histórica no se anuncia), y solo
después el respaldo por conteo. Invertirlas haría que una regla `combo` histórica produjera texto.

---

## D-4 — El equivalente por unidad lo calcula y lo **renderiza** el backend

**Decisión**: el DTO `MenuVariantPromotion` viaja con los importes **y con los textos ya
compuestos** (`unit_equivalent_text`, `display_text`). El frontend imprime cadenas.

**Razón**: FR-007 lo pide explícitamente (*"El menú NO DEBE recalcular importes, redondeos ni
textos por su cuenta"*) y hay una razón mecánica detrás: el redondeo de FR-009 (peso más cercano,
medio hacia arriba) sobre `Decimal` no tiene equivalente exacto en el `number` de JavaScript, que
es un binario de doble precisión. `12000/2` coincide en los dos lenguajes; `13000/3` o un 12,5%
sobre `$8.700` no tienen por qué. Componer el texto una sola vez en Python elimina la clase entera
de bug — y es lo que hace verificable SC-005 sin comparar dos implementaciones.

**Alternativas consideradas**: *enviar solo los números y componer en Angular* — más "limpio" en
apariencia, pero duplica el redondeo, el formato de moneda colombiano (`$12.000`) y la regla del
`≈` en dos lenguajes, y deja SC-005 dependiendo de que las dos implementaciones no se separen
nunca.

---

## D-5 — Marca de aproximado: la condición exacta

**Decisión**: `unit_equivalent_approx = (exacto % 1 != 0)`, donde `exacto` es el `Decimal` **antes**
de redondear:
- `package_price`: `Decimal(value) / Decimal(min_qty)`
- `percent`: `Decimal(price) * (100 - Decimal(value)) / 100`

Y `unit_equivalent = exacto.quantize(Decimal("1"), rounding=ROUND_HALF_UP)`.

**Razón**: FR-009 dice *"siempre que el importe exacto no sea entero en pesos"*, y los ejemplos de
la spec lo confirman en las dos direcciones: `$9.350` (15% de `$11.000`, entero) va **sin** `≈`
en el escenario 3 de US2; `≈ $4.333` (de `13.000/3`) va **con**. `ROUND_HALF_UP` es el mismo
criterio que ya usa el motor de cobro al redondear su descuento de grupo
(`service.py:260-262`), así que el número informativo no se separa del vinculante por criterio de
redondeo — solo, y como máximo en un peso, por el reparto entre líneas
(`_distribute_group_discount`, que reparte el **importe cobrado** con `ROUND_FLOOR` y manda el
residuo a la variante de id más alto). Esa es exactamente la razón por la que FR-009 exige la
marca también en `percent`.

**Cuidado de implementación**: un precio de catálogo llega como `Decimal("8000.00")`;
`Decimal("8000.00") % 1` es `Decimal("0.00")`, que compara igual a `0`. La condición funciona sin
normalizar antes.

---

## D-6 — `package_price` con `min_qty == 1` sí puede superar el precio normal (FR-015 no es letra muerta)

**Hallazgo del reconocimiento**, no estaba en la spec.

`_guard_package_is_discount` (`service.py:447-476`) rechaza con **409** cualquier regla
`package_price` donde `value >= min_qty × (precio más barato del conjunto)`. Con `min_qty == 1`
eso significa que **al crearla**, el valor de la regla es estrictamente menor que el precio normal
de *todas* las variantes de su conjunto. Leído solo así, el caso de FR-010/FR-015 ("valor mayor o
igual al precio normal") sería inalcanzable y el código de la rama sin tachado, muerto.

No lo es. La guarda corre en exactamente tres momentos —`create` (`:560`), `update_shape`
(`:603`) y `change_status` al activar (`:629`)— y **en ninguno más**. No corre cuando el
catálogo cambia un precio. Basta con que una variante del conjunto baje de precio después de
activar la promoción (o que entre al conjunto por una edición del catálogo) para que su precio
normal quede por debajo del valor de la regla. Es el mismo hueco temporal que la spec 063 ya
asume al no guardar un snapshot de precio en la regla.

**Consecuencia para el diseño**: la rama "sin tachado y sin señal de descuento" de FR-015 es
alcanzable y **debe** tener test. Se cubre en `quickstart.md` §5 bajando el precio del catálogo
después de activar, que es la única forma de llegar ahí sin desactivar la guarda.

---

## D-7 — La comparación que decide el tachado no es un recálculo

**Decisión**: el frontend decide el tachado con `discounted_price < price`, que es lo que
`discountInfo()` (`promotion-pricing.util.ts:82-100`) ya hace hoy — devuelve `null` cuando
`discounted >= original`.

**Razón**: FR-007 prohíbe **recalcular importes**; comparar dos importes que ya vinieron
calculados no es recalcularlos. Y el comportamiento que sale de esa comparación es exactamente el
que pide FR-015: con `value >= price`, `discountInfo` devuelve `null`, la plantilla cae a
`variantPrice(v)` → `effectivePrice(price, discounted_price)` → el valor de la regla, **sin
tachado y sin insignia de descuento**. O sea: FR-010 y FR-015 se cumplen sin tocar
`discountInfo`, solo poblando `discounted_price` para `package_price` `min_qty == 1`.

**Consecuencia**: no hace falta un campo `strikethrough` en el DTO. Se evita meter una decisión de
presentación en el contrato de la API.

---

## D-8 — Hay un 5.º test afectado que la spec no lista

**Hallazgo**. La spec enumera cuatro tests afectados. El `grep` sobre `condition_text` encontró
uno más:

`pos-backend/app/characterization_tests/test_promotions_router.py:75`
→ `test_el_header_no_cambia_la_forma_de_la_respuesta` afirma
`regla["condition_text"] == "10% en estas 1 variantes"`.

Y tiene una arruga propia: su variante se crea con
`fx.make_variant(db, product=prod, price=8000)` **sin `name`**, así que el fixture le pone
`f"variante-{uid}"` — un nombre no determinista. Con el texto por nombres, el aserto no se puede
reescribir literal: la tarea debe **pasarle un `name` explícito** a la variante y afirmar contra
él (p. ej. `name="Pequeño 8oz"` → `"10% en Pequeño 8oz"`).

**Estado de los cinco frente al Principio III**: `grep -rn "CONGELA comportamiento actual"` sobre
`test_menu_router.py`, `test_promotions_router.py`, `test_promotions_rules_admin.py` y
`app/scripts/test_promotions_rules.py` **no devuelve nada** (verificado 2026-09-01). Ninguno está
bajo veto; los cinco se actualizan citando esta spec. Cuidado con la lectura rápida:
`test_menu_router.py` **sí** dice `"CONGELA comportamiento corregido:"` en su docstring de módulo —
es otro prefijo, referido a A-08 (zona horaria), y esos dos tests usan `percent` con `min_qty 1`,
que FR-010 no toca.

**Textos nuevos esperados**, con los nombres que los fixtures ya generan:

| Test | Nombres del conjunto | Texto nuevo |
|---|---|---|
| `test_menu_router::test_vigente_se_anuncia_con_texto_legible` | `sabor-0` … `sabor-7` | `Llevando 2 entre sabor-0, sabor-1, sabor-2 y 5 más pagas $12.000` |
| `test_promotions_rules_admin::test_ca1_ca6_paquete_nace_borrador_con_condicion` | `licor-0` … `licor-7` | `Llevando 2 entre licor-0, licor-1, licor-2 y 5 más pagas $12.000` |
| `test_promotions_router::test_el_header_no_cambia_la_forma_de_la_respuesta` | ⚠️ hoy no determinista | requiere `name` explícito en el fixture |

`promotion-pricing.util.spec.ts:34` menciona `condition_text: '10% en estas 3 variantes'`, pero
**solo como dato de fixture** — ningún aserto lo lee. No es un test afectado.

---

## D-9 — `ProductSelectComponent` está compartido: el comensal y el cajero se resuelven de una vez

**Hallazgo del reconocimiento**, y la decisión de arquitectura más importante del plan.

`src/app/modules/tables/components/product-select.component.ts` lo consumen **tres** páginas:
- `public-menu.component.ts:498` — el comensal (menú QR),
- `manual-order-page.component.ts:308` — el cajero,
- `pos-catalog-drawer.component.ts:84` — el cajero, en el cajón de catálogo.

**Decisión**: pintar el bloque `promotion` dentro de `ProductSelectComponent`, sin ninguna rama
por superficie. FR-007 (modal del comensal) y FR-016 (condición en la terminal) quedan cubiertos
por el mismo cambio, y SC-005 se cumple por construcción: es literalmente el mismo nodo del DOM
imprimiendo el mismo campo.

---

## D-10 — La terminal recibe `promotion` pero **no** `discounted_price`

**Decisión**: `MenuService.toCategory` (`menu.service.ts:88-110`) mapea el campo `promotion` nuevo
y **sigue sin mapear** `discounted_price` / `discount_kind`.

**Razón**: es el riesgo real que abre D-9. Hoy la terminal ya llama a `GET /menu`, pero su
`MenuVariantResponse` local (`menu.service.ts:85-91`) **descarta** `discounted_price` — por eso su
modal muestra siempre precio normal. Si al compartir el componente se aprovechara para mapearlo
"ya que estamos", `product-select.component.ts:310-318` (`effectivePrice`) empezaría a mostrar
precios con descuento en la terminal. Eso sería:

- una expansión de alcance no pedida (la spec acota la terminal a FR-016/FR-017);
- un choque con FR-017 y con la spec 063 FR-023 — el importe de la terminal lo resuelve el
  **preview del cobro** del backend (`discounted_unit_price` de la línea,
  `pos-terminal.store.ts:399-403`), no el menú;
- y un cambio de comportamiento **sin decisión de negocio registrada** (Principio II), porque
  A-66/A-67/A-68 no lo cubren.

Mapear solo `promotion` deja la terminal ganando texto y cero importes. `addDraftFromSelection`
(`pos-terminal.store.ts:1289`) usa `sel.variant.price` crudo, así que tampoco por ahí se cuela
nada. Merece un test explícito de no-regresión.

---

## D-11 — La insignia genérica se deriva en el frontend, sin campo nuevo por producto

**Decisión**: la tarjeta evalúa `product.variants.some(v => v.promotion != null)`. No se añade
`has_promotion` a `MenuProductResponse`.

**Razón**: FR-013 lo dice tal cual (*"La tarjeta deriva esa señal de la información por variante
que entrega el backend, no de una evaluación propia de las reglas"*). Un booleano derivable de un
campo que ya viaja es superficie de contrato que puede desincronizarse. La condición es una
lectura, no una evaluación de reglas de vigencia: el filtro de vigencia ya lo hizo el backend al
poblar `promotion`.

**Consecuencia sobre `productDiscount()`** (`public-menu.component.ts:775`): se **conserva** tal
cual, porque FR-015 mantiene el precio tachado en la tarjeta. Lo que cambia es que deja de
gobernar la insignia: el `@if (productDiscount(product); as disc)` de `:382-389` se reemplaza por
la insignia genérica, y el `@if` de `:402-406` (tachado + precio) se queda.

---

## D-12 — Resolver los nombres: una consulta constante, nunca un N+1

**Decisión**: `variant_display_names(db, variant_ids) -> dict[UUID, str]`, una sola consulta
`SELECT ProductVariant.id, ProductVariant.name, Product.name … WHERE ProductVariant.id IN (…)`
por llamada, alimentada con la unión de los conjuntos de las reglas vigentes.

**Razón y coste**: `active_variant_set_rules` ya trae las reglas con sus `variants` cargadas
(`selectinload`), así que los ids salen gratis. `_serialize_rule` **ya tiene** el mapa que
necesita (`by_id`, construido en `serialize_promotion:696-701`): ahí no se añade ni una consulta,
solo se le pasa lo que ya tiene. En el menú se suma **una** consulta a `GET /menu` y **una** a
`GET /menu/promotions`; el flujo del QR con token llama a las dos funciones, así que son dos
consultas constantes por resolución de QR. No se comparte el mapa entre ambas a propósito: son
endpoints independientes y compartirlo obligaría a cambiar las dos firmas por un ahorro de una
consulta indexada por PK.

**Por qué no reutilizar los productos que `_build_menu` ya cargó**: el conjunto de una regla puede
incluir variantes de productos inactivos o no disponibles, que el menú **no** carga. Nombrarlas
desde el menú daría descriptores incompletos y no deterministas según qué esté agotado.

---

## D-13 — Qué le pasa al `discount_kind` de la tarjeta

**Decisión**: `discount_kind` conserva su tipo (`PromotionType | None`) y pasa a poder valer
`"package_price"` cuando FR-010 lo pobla. El frontend **deja de usarlo para elegir insignia** en
la tarjeta (FR-013 la unifica) y, en el modal, la insignia de porcentaje se muestra **solo** con
`discount_kind === 'percent'`.

**Razón**: hoy `product-select.component.ts:80-86` pinta `-{{ disc.percent }}%` para cualquier
`kind` que no sea `'fixed'`. Con `package_price` `min_qty 1` eso inventaría un porcentaje que la
regla nunca enuncia (`$8.000 → $6.000` se anunciaría como `-25%`), que es justo lo que FR-007
prohíbe. Con la guarda por `kind`, ese caso muestra tachado + precio vigente y su línea de
promoción, sin porcentaje fabricado. El valor `'fixed'` que la plantilla todavía contempla
pertenece a un tipo retirado por la spec 063 y no llega nunca; se deja como está para no mezclar
limpieza no relacionada (Principio V).

---

## Riesgos y mitigaciones

| Riesgo | Mitigación |
|---|---|
| La terminal empieza a mostrar precios con descuento al compartir el componente. | D-10: `MenuService` no mapea `discounted_price`. Test de no-regresión explícito. |
| FR-010 cambia un importe y arrastra el cobro. | El cobro no lee el menú: `evaluate_variant_sets` no se toca. SC-006 se verifica corriendo la batería de la spec 063 **sin modificar sus asertos de importe**. |
| El texto se separa entre backend y vista previa del formulario. | Un único algoritmo especificado en [contracts/texto-condicion.md](./contracts/texto-condicion.md), con la misma tabla de casos ejercitada por los tests de los dos lados. |
| Un call site del texto se queda sin nombres. | D-1: parámetro obligatorio → falla ruidosa. |
| La rama sin tachado de FR-015 queda sin cobertura por parecer inalcanzable. | D-6 documenta cómo se alcanza; quickstart §5 la ejercita. |
