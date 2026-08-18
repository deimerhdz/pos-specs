# Phase 1 Data Model: Corrección de la cuenta de mesas fusionadas (`group_bill`)

Esta spec no introduce ni modifica ninguna tabla, modelo ORM ni esquema de base de datos — corrige
puramente el cálculo en tiempo de consulta de una función existente (FR-004, Assumptions de
`spec.md`: "no se requiere migración de datos"). Las "entidades" relevantes aquí son de código y de
comportamiento, no de datos nuevos: los Key Entities que ya define `spec.md`, detallados a nivel de
función para guiar la corrección.

## Entidad: `group_bill` (`app/api/v1/orders/tables_advanced.py:92-114`)

| Aspecto | Antes (defectuoso, A-01 camino C) | Después (esta spec) |
|---|---|---|
| Firma | `group_bill(db: Session, group_id: UUID) -> dict` | Sin cambios |
| Consulta de órdenes | Todas las del `merged_group_id`, sin filtro de `status` | Todas las del `merged_group_id` para el chequeo de existencia (404); las billables (`status not in ("pagada", "cancelada")`) para el cálculo — FR-001, research.md Decisión 2 |
| Cálculo por orden billable | `sub = sum(unit_price * quantity` para ítems no `anulado`)` — sin descuento | `lines = checkout.order_sale_lines(db, order.id)` (ya excluye `anulado` en su propio `WHERE`); `raw = sum(line.line_total for line in lines)`; `promo, _ = promotions.evaluate(db, checkout.promo_lines_for(db, lines), now)`; `combo = promotions.combo_discount_for_lines(db, lines, now)`; `sub = raw - promo - combo` — FR-002/FR-003, research.md Decisión 1 |
| Órdenes terminales | Incluidas en el total | Excluidas del total; siguen listadas en `orders[]` con `subtotal=0` (research.md Decisión 3) |
| `total` | `sum` de todos los `sub`, terminales incluidos | `sum` de `sub` solo de las billables — invariante `total == sum(orders[].subtotal)` se preserva |
| 404 | Grupo sin ninguna orden (de cualquier status) | Sin cambios — el chequeo de existencia no se mueve al conjunto billable (research.md Decisión 3) |

**Dependencias nuevas del módulo** (todas ya existentes en `pos-backend`, ninguna nueva —
Constitución Principio IV no aplica):

- `app.api.v1.orders.checkout.TERMINAL` — tupla `("pagada", "cancelada")`, ya pública, ya usada por
  `_billable_orders` (vía su propia copia) y por `tables_advanced.py` (su propia copia idéntica en
  la línea 17); `group_bill` reutiliza la de `checkout` en vez de la local para el filtro de status,
  evitando una tercera copia.
- `app.api.v1.orders.checkout.order_sale_lines` — ya usada por `table_sessions/service.py:163`.
- `app.api.v1.orders.checkout.promo_lines_for` — ya usada por `table_sessions/service.py:165`.
- `app.api.v1.promotions.service.evaluate` — ya usada por `table_sessions/service.py:165`.
- `app.api.v1.promotions.service.combo_discount_for_lines` — ya usada por
  `table_sessions/service.py:166`.

## Modelos ORM consultados (sin modificar — referencia)

- **`CustomerOrder`** (`app/models/customer_order.py`): `status` (determina inclusión, FR-001),
  `merged_group_id` (agrupa el grupo), `dining_table_id`, `id` — sin cambios de schema.
- **`OrderItem`** (`app/models/order_item.py`): `estado_cocina` (determina inclusión por ítem,
  FR-003 — ya lo filtra `order_sale_lines` en su propio `WHERE OrderItem.estado_cocina !=
  "anulado"`), `unit_price`, `quantity`, `combo_id`, `product_variant_id` — sin cambios de schema.
- **`Promotion`**/**`PromotionTarget`**/**`PromotionComboItem`** — consultadas sin cambios por
  `promotions.evaluate`/`combo_discount_for_lines`, ya existentes.

## Entidad: Grupo de mesas fusionadas (`merged_group_id`)

Sin cambios respecto a `spec.md` §Key Entities: no es una tabla propia, es un campo compartido
(`CustomerOrder.merged_group_id`) entre las órdenes del grupo. Esta spec no le añade ni le quita
ningún campo ni comportamiento de agrupación — solo corrige cómo se calcula la cuenta sobre las
órdenes ya agrupadas.

## Entidad: Contrato de comportamiento (Constitución, Principio II)

El conjunto que arbitra si la corrección preserva/corrige el comportamiento esperado:

- **Test modificado** (research.md Decisión 4): el `CONGELA` existente
  (`test_group_bill_a01_camino_c_incluye_todos_los_status_sin_descuentos`,
  `test_orders_tables_advanced.py:124-160`) — se actualiza para verificar el comportamiento
  corregido, citando A-01 en el commit.
- **Tests nuevos** (FR-006): al menos uno que verifique, de forma independiente entre sí (como ya
  hacen las Independent Test de cada historia en `spec.md`):
  1. Exclusión de una orden `pagada` del total (Historia 1, escenario 1).
  2. Exclusión de una orden `cancelada` del total (Historia 1, escenario 2).
  3. Aplicación de una promoción vigente sobre una orden `abierta` sin órdenes terminales en el
     grupo (Historia 2, escenario 1).
  4. El caso combinado citado en A-01/`spec.md` SC-005: orden `pagada` $20.000 + orden `abierta`
     $15.000 brutos con 10% vigente → total $13.500 (Historia 2, escenario 2).
  5. Consistencia con `table_sessions.compute_bill` para una mesa fusionada de una sola mesa
     (Historia 3) — compara el total de `group_bill` contra el de `compute_bill` para el mismo
     conjunto de órdenes.
  6. Grupo íntegramente `pagada`/`cancelada` → `total=0`, sin error (Edge Case, research.md
     Decisión 3).
- **Entrada citada del registro de anomalías**: A-01 (autoriza el cambio de comportamiento) — como
  referencia normativa en el nombre/comentario del test, no como artefacto ejecutable.
- **Sin golden master nuevo**: el alcance (una función, ~15 líneas de cambio real) no tiene
  interacción encadenada entre varias funciones que lo justifique — mismo razonamiento que la spec
  018 aplicó a un caso comparable (research.md de esa spec, Decisión 4).

## Transiciones de estado

No aplica en sentido de máquina de estados de negocio nueva — `CustomerOrder.status` ya tiene su
ciclo documentado en el modelo (`recibida → abierta → bloqueada → pagada`, `↘ cancelada`); esta spec
no le agrega ni le quita transiciones, solo cambia qué estados cuentan para el cálculo de
`group_bill` (FR-001).
