# Quickstart: validar la red de characterization tests de `orders`

Guía de ejecución y validación end-to-end de esta spec. No repite el detalle de diseño (ver
`data-model.md` y `contracts/test-harness-api.md`) ni incluye código de implementación (eso vive
en `tasks.md` y en los propios ficheros de test).

## Prerrequisitos

- Repositorio `pos-backend` (sibling de `pos-specs`) con las dependencias ya instaladas:
  `pip install -r requirements.txt` desde la raíz de `pos-backend`.
- Python 3.12+ (CI usa 3.12; el entorno local con `env/` corre 3.14, ambos válidos).
- No hace falta Postgres, Redis ni ningún servicio externo levantado: toda la suite corre sobre
  SQLite en memoria.

## Escenario 1 — Cada historia corre aislada (Independent Test de cada User Story)

Desde la raíz de `pos-backend`:

```bash
python -m unittest app.characterization_tests.test_orders_consolidation -v   # Historia 1 (A-04)
python -m unittest app.characterization_tests.test_orders_checkout -v        # Historia 2
python -m unittest app.characterization_tests.test_orders_kitchen -v         # Historia 3 (A-16, A-25)
python -m unittest app.characterization_tests.test_orders_tables_advanced -v # Historia 4 (A-26, A-01c)
python -m unittest app.characterization_tests.test_orders_service -v        # Historia 5
```

**Resultado esperado**: cada comando termina en `OK`, sin depender de que los demás ficheros
existan o se hayan ejecutado antes (SC-001 por fichero). Si `test_orders_consolidation.py` es el
único fichero escrito hasta el momento, su comando corre igual de verde sin los otros cuatro.

## Escenario 2 — La suite completa de `orders` en conjunto

```bash
python -m unittest discover -s app/characterization_tests -p 'test_orders_*.py' -v
```

**Resultado esperado**: los ~35-45 métodos de test de los cinco ficheros pasan en verde, en
cualquier orden (FR-013 — ninguno depende del orden de ejecución de otro).

## Escenario 3 — Cobertura de las 23 funciones públicas (SC-001)

```bash
grep -rhoE 'def test_\w+' app/characterization_tests/test_orders_*.py | sort -u | wc -l
```

Verificación manual cruzada contra la tabla de `data-model.md` §Entidades conceptuales: cada una
de las 23 funciones públicas (`create_order`; `block_order`, `compute_bill`, `order_sale_lines`,
`promo_lines_for`, `pay_order`, `confirm_order`, `cancel_order`, `close_participants`,
`close_table_sessions`, `release_table`; `active_table_session_id`,
`get_or_create_table_session_id`, `get_or_create_open_order`, `consolidate_table`,
`add_item_to_table`; `transition_kitchen`, `mark_order_ready`, `void_item`; `set_table_status`,
`move_order`, `merge_orders`, `group_bill`) aparece citada en al menos un nombre o docstring de
test.

## Escenario 4 — Las siete anomalías están representadas (SC-002)

```bash
grep -rn "A-01\|A-04\|A-16\|A-25\|A-26\|A-29\|A-38" app/characterization_tests/test_orders_*.py
```

**Resultado esperado**: al menos una coincidencia por cada una de las siete citas (A-01 debe
aparecer con sus dos caminos B y C; A-38 con sus dos hallazgos RN-ORD-31 y RN-ORD-32) en el
docstring o el nombre del método de test correspondiente.

## Escenario 5 — Determinismo (SC-003), incluyendo el caso no determinista de `merge_orders`

```bash
for i in 1 2 3; do
  python -m unittest discover -s app/characterization_tests -p 'test_orders_*.py' 2>&1 | tail -5
done
```

**Resultado esperado**: las tres corridas terminan en `OK` con el mismo número de tests
ejecutados y cero fallos — incluyendo el test de `merge_orders` (RN-ORD-63), que pasa las tres
veces aunque el *grupo ganador* que observa internamente pueda variar entre corridas (su
aserción es de conjunto, no de valor fijo — ver `research.md` §3).

## Escenario 6 — La suite queda cubierta por el paso de CI existente (SC-004, Historia 6)

```bash
grep -n "characterization_tests" .github/workflows/deploy.yml
```

**Resultado esperado**: el paso `python -m unittest discover -s app/characterization_tests
-p 'test_*.py' -v` ya presente en `.github/workflows/deploy.yml` (instalado desde
`specs/015-caracterizacion-cart/`) descubre los cinco ficheros nuevos por el patrón de nombre, sin
necesitar ninguna línea nueva en el workflow. Confirmar además, sobre un PR trivial de esta rama,
que el job `test` de GitHub Actions los recoge y ejecuta (Acceptance Scenario 1 de la Historia 6).

## Escenario 7 — Cero líneas modificadas en producción (SC-005, SC-006)

```bash
git diff --stat main -- \
  app/api/v1/orders/service.py \
  app/api/v1/orders/checkout.py \
  app/api/v1/orders/consolidation.py \
  app/api/v1/orders/kitchen.py \
  app/api/v1/orders/tables_advanced.py \
  requirements.txt
```

**Resultado esperado**: salida vacía — ningún diff en los cinco ficheros de producción en alcance
ni en `requirements.txt` (cero dependencias nuevas).

## Escenario 8 — Los tres scripts legado quedan migrados y no corren aparte (SC-007)

```bash
grep -rln "test_cancel_inventory\|test_receta_obligatoria\|test_session_ttl" app/scripts/ \
  --include="*.yml" .github/workflows/
grep -c "def test_" app/characterization_tests/test_orders_checkout.py \
  app/characterization_tests/test_orders_consolidation.py \
  app/characterization_tests/test_orders_service.py
```

**Resultado esperado**: el primer `grep` no encuentra ningún workflow que invoque los tres
scripts legado como paso aparte; el segundo confirma que los ficheros que migran sus casos
(`test_orders_checkout.py` para `test_cancel_inventory.py` y la porción de `test_session_ttl.py`
que toca `orders`; `test_orders_checkout.py`/`test_orders_consolidation.py`/
`test_orders_service.py` para `test_receta_obligatoria.py`, uno por cada camino que invoca la
guarda) tienen al menos un método de test correspondiente.

## Si algo falla

Por Principio II de la Constitución: un test `"CONGELA comportamiento actual:"` en rojo, recién
escrito contra el código actual sin modificar, significa que el defecto está en el test — se
corrige el test hasta que refleje la realidad observada (FR-015). Ningún resultado de esta guía
autoriza tocar `service.py`, `checkout.py`, `consolidation.py`, `kitchen.py` ni
`tables_advanced.py`.
