# Contrato: migración `063c`/`063d` y reescritura de tests (partición Promoción/Regla)

> **Reemplaza** la versión de este contrato escrita para la migración del modelo plano (`063a`/
> `063b`) — esa migración **ya corrió** en la rama de feature de `pos-backend` y su inventario de
> tests **ya se implementó** (`test_promotions_service.py`, `test_promotions_rules_admin.py`,
> `test_promotions_migration.py`, `test_promotions_router.py` existen y pasan, verificado
> 2026-09-01). Este documento cubre **solo** la migración nueva (`063c` aditiva, `063d`
> destructiva, [data-model.md](../data-model.md) §"Migración y rollback") y qué de esos tests ya
> escritos hay que volver a tocar.

---

## 1. Migración de datos (`063c`)

| Situación previa (rama de feature, modelo plano) | Después de `063c` | Detalle |
|---|---|---|
| Promoción con `type`/`value`/`min_qty` propios y N filas en `promotion_variants` apuntando a `promotion_id` | Una fila nueva en `promotion_rules` con ese mismo `type`/`value`/`min_qty`; las N filas de `promotion_variants` repuntadas a `promotion_rule_id` = esa regla nueva | data-model.md §"upgrade() — paso 5"; 1:1 porque toda promoción existente tiene exactamente una combinación |
| Promoción `Finalizada` por la migración `063a` (`closed_by_refactor_at IS NOT NULL`, `type` histórico como `combo`/`fixed`/etc.) | Igual que cualquier otra: una regla con ese `type` histórico y su `value`/`min_qty` — el paso de datos no filtra por `status` ni por `closed_by_refactor_at` | El `type` histórico sobrevive dentro de `promotion_rules.type` hasta que algo lo lea; como esas promociones están `Finalizada`, nunca vuelven a evaluarse por el motor, así que un `type` fuera del enum vigente ahí no causa ningún error en tiempo de ejecución (mismo criterio que ya toleraba `Promotion.type` histórico) |
| `Sale` / `Invoice` / `CustomerOrder` con `applied_promotions` ya escrito (sin `rule_id`, porque `PromotionRule` no existía al cobrar) | Sin cambio — el paso de datos de `063c` no toca `sales`/`invoices`/`customer_orders` | Principio VII; FR-021 "el registro NO es retroactivo" |

**`063d`** no tiene paso de datos propio — solo borra columnas/constraints que ya nadie lee (ver
data-model.md).

---

## 2. Inventario de tests a reescribir (segunda vez)

Los tests listados abajo **ya existen y ya pasan** contra el modelo plano (implementados en la
rama de feature). Se reescriben **de nuevo**, citando esta revisión del plan, solo en lo que la
partición Promoción/Regla les cambia — no se reinventan desde cero.

### 2.1 Backend

| Test | Archivo | Qué cambia | Qué NO cambia |
|---|---|---|---|
| Fixture `make_promotion` | `cart_fixtures.py` / `orders_fixtures.py` / `table_sessions_fixtures.py` | Deja de aceptar `type`/`value`/`min_qty`/`variant_ids` directos; nuevo helper `add_rule_to_promotion(db, promotion, *, type, value, min_qty, variant_ids)` que inserta la regla + sus `promotion_variants`. La mayoría de los tests que hoy hacen `make_promotion(type="percent", ...)` pasan a `make_promotion(...); add_rule_to_promotion(promo, type="percent", ...)`. | El resto de campos de `make_promotion` (nombre, vigencia, estado) sin cambio. |
| `test_promotions_service.py` (los 10 Acceptance Scenarios de US2 + edge cases + SC-005) | mismo archivo | Montaje vía `add_rule_to_promotion`; el resto de la aserción (totales, reparto, determinismo) **no cambia de valor esperado** — son los mismos números, la regla produce exactamente el mismo descuento que producía la promoción plana. **Caso nuevo**: dos reglas de la misma promoción con variante compartida → 409 al guardar (FR-001a) — no existía porque no era posible en el modelo plano. |
| `test_promotions_rules_admin.py` (CRUD, FR-014, FR-016, FR-018) | mismo archivo | Payloads con `rules: [...]` en vez de campos sueltos; **caso nuevo** FR-001a (dos reglas del mismo payload comparten variante → 409, antes de siquiera comparar contra otras promociones); duplicar verifica que se copian **todas** las reglas, no una. | El criterio de FR-014/FR-016/FR-018 en sí (cuándo bloquea, qué mensaje) no cambia — solo el nivel (regla en vez de promoción). |
| `test_promotions_migration.py` | mismo archivo | Se agrega un caso: correr `063c` sobre una promoción ya migrada por `063a` (`percent` con conjunto) produce exactamente una regla con ese `type`/`value`/`min_qty`, y `promotion_variants` sigue apuntando al mismo conjunto (ahora vía `promotion_rule_id`). | Los casos ya existentes de `063a` (percent → conjunto foto fija; combo/fixed/qty_price → finished) no se reabren — son de una migración distinta ya aplicada. |
| `test_promotions_router.py` | mismo archivo | `PromotionResponse` anida `rules`; los tests de forma de respuesta (`X-Server-Time`, etc.) se ajustan al nuevo shape. | El comportamiento de headers/paginación no cambia. |
| `test_menu_router.py` (clase de anuncio) | mismo archivo | Con una promoción de N reglas, el anuncio trae N elementos en `rules[]` (antes siempre 1) — caso nuevo que ejercita la cardinalidad. | El criterio de vigencia (`_valid_now`) no cambia. |
| `test_orders_checkout.py` (`test_pay_order_construye_sale_real_con_promocion_activa`, etc.) | mismo archivo | `applied_promotions` esperado gana `rule_id` por entrada. | El monto total cobrado no cambia. |
| `test_cart_service.py` (`test_serialize_cart_discounted_total_con_promocion_activa`) | mismo archivo | Montaje vía `add_rule_to_promotion`. | El invariante `discounted_total = None` sin promo no cambia. |
| `app/scripts/test_promotions_rules.py` (script de CI) | mismo archivo | Las funciones puras que ejercita (`_greedy_units`, `_distribute_group_discount`) no cambian de comportamiento — solo el montaje de los casos de prueba pasa por una regla en vez de una promoción directa. | Todo el contenido normativo (consumo codicioso, reparto FR-008a, tope FR-009, vigencia A-57) sin cambio de valor esperado. |

### 2.2 Frontend

| Spec | Qué cambia |
|---|---|
| `promotions-page.component.spec.ts` | Casos nuevos para agregar/quitar filas de regla, y para el 409 de FR-001a. Los casos ya existentes (FR-014, FR-016, FR-018, migración) se ajustan al payload con `rules`. |
| `promotion-pricing.util.spec.ts` | `getPromoDisplay`/`discountInfo`/`effectivePrice` reciben una regla en vez de una promoción — mismos casos, mismo resultado esperado por caso. |
| `promotion.service.spec.ts` | Payloads de `create`/`updateShape`/`duplicate` con `rules`. |

---

## 3. Tests nuevos (no existían en el modelo plano)

| Cobertura | Archivo |
|---|---|
| FR-001a: crear una promoción con dos reglas cuyos conjuntos comparten una variante → 409, sin llegar a comparar contra otras promociones | `test_promotions_rules_admin.py` |
| FR-001a: `update_shape` en `Borrador` que introduce una variante repetida entre dos reglas → 409 | `test_promotions_rules_admin.py` |
| Creación por lote: una promoción con 6 reglas (caso Springfield) creada en una sola llamada, cada una con su propio conjunto disjunto, vigencia compartida | `test_promotions_rules_admin.py` |
| Mantenimiento por lote: pausar una promoción con 6 reglas deja las 6 sin efecto con una sola llamada a `change_status` | `test_promotions_service.py` |
| Migración `063c`: 1 promoción existente → exactamente 1 regla, con el mismo `type`/`value`/`min_qty`, `promotion_variants` repuntada | `test_promotions_migration.py` |
| Motor: dos reglas de la **misma** promoción, ambas con descuento en el mismo cobro (conjuntos disjuntos, ambas vigentes) → dos entradas en `applied_promotions` con el mismo `promotion_id` y distinto `rule_id` | `test_promotions_service.py`, `test_orders_checkout.py` |

---

## 4. Verificación de no-regresión (Principio X)

Con toda promoción migrada a exactamente una regla (estado inmediatamente después de `063c`, antes
de que el Incremento H exponga la creación multi-regla), todos los totales de cobro deben ser
**idénticos** a los que producía el modelo plano — es la garantía central de que la migración 1:1
no cambia ningún número, solo la estructura que lo porta. `quickstart.md` §"Verificación de
regresión 1:1" lo ejercita corriendo la suite de `test_promotions_service.py` **antes** de que
cualquier promoción tenga más de una regla.
