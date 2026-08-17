# Quickstart: validar la extracción del motor de catálogo

Guía de ejecución para comprobar, en cada historia, que la extracción cumple su contrato. No
repite firmas ni tablas ya detalladas en [data-model.md](./data-model.md) y
[contracts/module-api.md](./contracts/module-api.md) — solo enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` activado (`source env/bin/activate`, Python
3.14), ejecutado desde la raíz de `../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL ni Redis reales: `app/characterization_tests/fixtures.py` crea SQLite en
memoria y rellena variables de entorno inertes (ver docstring de `fixtures.py`).

## Historia 1 — `app/catalog_engine/` existe y es equivalente

1. Crear el paquete según [contracts/module-api.md](./contracts/module-api.md) Contrato A.
2. Apuntar temporalmente los imports de los tests a `app.catalog_engine` (o, preferible, dejar
   que ya funcionen porque `line_pricing.py`/`consumption_plan.py` empiezan a delegar — pero en
   esta historia lo mínimo exigido es correr los tests contra el paquete nuevo sin haber tocado
   aún las fachadas; ver Acceptance Scenario 1-2 de la Historia 1 en `spec.md`).
3. Ejecutar los 41 characterization tests:

   ```bash
   python3 -m unittest app.characterization_tests.test_catalog_line_pricing -v
   python3 -m unittest app.characterization_tests.test_catalog_consumption_plan -v
   ```

   **Resultado esperado**: 25 + 16 tests en verde, cero aserciones modificadas respecto al
   fichero original (diff del propio test file vacío salvo la línea de import).

4. Ejecutar el golden master sin regenerarlo:

   ```bash
   python3 -m unittest app.characterization_tests.test_golden_master_pricing_consumption -v
   ```

   **Resultado esperado**: pasa contra `golden_master/pricing_consumption.master.json` existente,
   sin tocar ese fichero.

5. Verificar la frontera núcleo/adaptador (Acceptance Scenarios 3-4, SC-006):

   ```bash
   grep -n sqlalchemy app/catalog_engine/core.py   # debe devolver vacío
   grep -n "db: Session" app/catalog_engine/core.py  # debe devolver vacío
   grep -n "db: Session" app/catalog_engine/pricing.py app/catalog_engine/consumption.py
   # debe listar las once funciones adaptadoras, cada una con db como primer parámetro
   ```

**Independent Test** (según `spec.md`): estos cinco pasos son autocontenidos — no requieren que
la Historia 2 o 3 estén hechas, y no tocan ningún fichero consumidor.

## Historia 2 — Batería comparativa (gate temporal, previo a la Historia 3)

1. Implementar `app/characterization_tests/catalog_engine_equivalence_gate.py` según
   [research.md](./research.md) Decisión 5 (generador `random.Random(seed)`, 100-200 casos,
   reutilizando `fixtures.py`) y el formato de reporte de
   [data-model.md](./data-model.md) §Batería comparativa.
2. Ejecutar:

   ```bash
   python3 -m unittest app.characterization_tests.catalog_engine_equivalence_gate -v
   ```

3. **Verificar reproducibilidad del generador primero** (Acceptance Scenario 1 de la Historia 2):
   correr dos veces con la misma semilla y confirmar que la lista de casos generados es idéntica
   byte a byte, antes de confiar en la comparación de implementaciones.
4. **Resultado esperado**: cero diferencias campo a campo entre `app/api/v1/catalog/*` (legado) y
   `app/catalog_engine/*` (nuevo) en el 100% de los casos, incluyendo los que ejercitan A-02,
   A-05, A-06, A-32 y A-33 (SC-003).
5. Si aparece una diferencia: el reporte debe identificar caso exacto + campo + valor legado vs.
   nuevo (Acceptance Scenario 3 de la Historia 2). Tratarla según el Edge Case de `spec.md`: bug
   de la extracción (corregir antes de continuar) o caso real no caracterizado (documentar en
   `registro-de-anomalias.md` antes de decidir) — nunca ajustar la batería para dejar de
   detectarla.

**Independent Test**: puede correr aislado como su propio script/test, pero solo tiene sentido una
vez `app/catalog_engine/` existe (Historia 1 completa).

## Historia 3 — Conmutación a fachada

1. Convertir `line_pricing.py` y `consumption_plan.py` en fachadas según
   [contracts/module-api.md](./contracts/module-api.md) Contrato B. **Prerequisito**: Historias 1
   y 2 en verde (spec.md: "sin las historias 1 y 2 ya verificadas [...] no hay base para hacerla
   con confianza").
2. Verificar diff vacío en los siete consumidores (SC-004):

   ```bash
   git diff --stat -- \
     app/api/v1/sales/service.py app/api/v1/sales/consumption.py \
     app/api/v1/orders/service.py app/api/v1/orders/consolidation.py \
     app/api/v1/orders/kitchen.py app/api/v1/orders/consumption.py \
     app/api/v1/cart/service.py
   ```

   **Resultado esperado**: sin salida (cero ficheros modificados).

3. Ejecutar la suite completa del backend, no solo catálogo (SC-005):

   ```bash
   python3 -m unittest discover -s app -p "test_*.py" -v
   ```

   **Resultado esperado**: mismo conjunto de tests en verde/rojo que antes de empezar la
   extracción (ninguna regresión nueva, ninguna corrección de otro módulo colada aquí).

4. Retirar o archivar el gate de la Historia 2, documentando su resultado final (Clarifications,
   sesión 2026-08-17) — a partir de aquí "legado" y "nuevo" son el mismo código, comparar deja de
   tener sentido. Los 41 characterization tests y el golden master quedan como la red de
   regresión permanente (re-ejecutarlos apuntando ya a las fachadas, que a su vez apuntan al
   paquete nuevo, confirma que la conmutación no rompió nada — mismo comando del paso Historia 1,
   punto 3-4).

**Independent Test**: correr la suite completa del backend después de la conmutación, sin haber
tocado ningún fichero consumidor, es suficiente para validar esta historia de forma aislada.
