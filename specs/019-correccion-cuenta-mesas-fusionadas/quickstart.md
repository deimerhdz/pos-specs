# Quickstart: validar la corrección de `group_bill`

Guía de ejecución para comprobar que la corrección cumple su contrato. No repite firmas ni tablas
ya detalladas en [data-model.md](./data-model.md) y
[contracts/group-bill-endpoint.md](./contracts/group-bill-endpoint.md) — solo enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` activado (Python 3.14), ejecutado desde la raíz
de `../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL real: `app/characterization_tests/orders_fixtures.py` crea SQLite en
memoria (`fx.new_session()`, `fx.make_dining_table`, `fx.make_customer_order`, `fx.make_order_item`,
`fx.make_promotion`, `fx.make_promotion_target`).

## Paso 1 — Confirmar el defecto actual (antes de tocar el código)

```bash
python3 -m unittest app.characterization_tests.test_orders_tables_advanced -v
```

**Resultado esperado antes del fix**: el test T046
(`test_group_bill_a01_camino_c_incluye_todos_los_status_sin_descuentos`) pasa en verde
**congelando el defecto** — confirma con datos reales el escenario de A-01 (total $35.000 con una
orden pagada de $20.000 + una abierta de $15.000 con 50% de descuento vigente ignorado).

## Paso 2 — Aplicar la corrección

Editar `app/api/v1/orders/tables_advanced.py:92-114` según
[data-model.md](./data-model.md) §Entidad `group_bill` y
[research.md](./research.md) Decisiones 1-3:

1. Import nuevo: `from app.api.v1.orders import checkout` y
   `from app.api.v1.promotions import service as promotions` (mismas dependencias que ya declara
   `table_sessions/service.py:27,29`).
2. Filtrar `CustomerOrder.status.notin_(checkout.TERMINAL)` para el conjunto billable (FR-001), sin
   mover el chequeo de 404 (sigue sobre todas las órdenes del grupo, research.md Decisión 3).
3. Por cada orden billable: `lines = checkout.order_sale_lines(db, o.id)`, calcular `raw`, `promo`,
   `combo` y `sub = raw - promo - combo` (FR-002/FR-003, research.md Decisión 1).
4. Las órdenes terminales quedan en `orders[]` con `subtotal=Decimal("0")`, sin sumar a `total`.

## Paso 3 — Modificar el test `CONGELA` existente (Principio II)

Actualizar `test_group_bill_a01_camino_c_incluye_todos_los_status_sin_descuentos` en
`test_orders_tables_advanced.py:124-160` para verificar el comportamiento corregido, **citando A-01
en el mismo commit** (research.md Decisión 4). El nombre pasa a reflejar el comportamiento
corregido (p. ej. `test_group_bill_a01_camino_c_excluye_terminales_y_aplica_descuentos`).

## Paso 4 — Tests nuevos de FR-006

Añadir, en el mismo fichero, un test por escenario de
[data-model.md](./data-model.md) §Tests nuevos — como mínimo:

```bash
python3 -m unittest app.characterization_tests.test_orders_tables_advanced -v
```

**Resultado esperado tras el fix**:

- Historia 1, escenario 1: grupo con orden A `pagada` ($20.000) + orden B `abierta` ($15.000, sin
  promoción) → `total == Decimal("15000")`, orden A excluida del cálculo pero presente en
  `orders[]` con `subtotal=0`.
- Historia 1, escenario 2: orden `cancelada` excluida igual que `pagada`.
- Historia 2, escenario 1: orden `abierta` sola con 10% vigente sobre su categoría, $15.000 brutos
  → `total == Decimal("13500")`.
- Historia 2, escenario 2 (ejemplo A-01/SC-005): orden `pagada` $20.000 + orden `abierta` $15.000
  brutos con 10% vigente → `total == Decimal("13500")`, no `Decimal("35000")`.
- Edge case: grupo íntegramente `pagada`/`cancelada` → `total == Decimal("0")`, sin `HTTPException`.
- 404: grupo sin ninguna orden (`merged_group_id` inexistente) sigue respondiendo 404 — sin cambios.

## Paso 5 — Consistencia con `table_sessions.compute_bill` (Historia 3)

Verificación cruzada, no un test aislado nuevo obligatorio (ya cubierta si se sigue el patrón del
Independent Test de la Historia 3 en `spec.md`):

1. Sembrar una mesa con una `table_session`, varias órdenes con una combinación de status y una
   promoción vigente (usando `fx.make_table_session`, `fx.make_participant`,
   `fx.make_customer_order`, `fx.make_order_item`).
2. Calcular `table_sessions.service.compute_bill(db, ts.id).total`.
3. Fusionar esa misma mesa sola en un grupo (`tables_advanced.merge_orders(db, [o.id for o in
   orders])`) y calcular `tables_advanced.group_bill(db, group_id)["total"]`.
4. **Resultado esperado**: ambos totales son idénticos (`assertEqual`), centavo a centavo — SC-003.

## Verificación final — SC-001 a SC-005

```bash
python3 -m unittest app.characterization_tests.test_orders_tables_advanced -v
```

Todos los tests de este módulo en verde, incluidos los 4 preexistentes (T042-T045, sin cambios) más
el T046 modificado y los nuevos de FR-006, es la señal de que la corrección está completa y no
introdujo ninguna regresión en `set_table_status`/`move_order`/`merge_orders` (Principio II).
