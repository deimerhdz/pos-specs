# Quickstart: validar la extracción del motor de stock de inventario

Guía de ejecución para comprobar, en cada historia, que la extracción cumple su contrato. No repite
firmas ni tablas ya detalladas en [data-model.md](./data-model.md) y
[contracts/module-api.md](./contracts/module-api.md) — solo enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` activado (`source env/bin/activate`, Python
3.14), ejecutado desde la raíz de `../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL real: `app/characterization_tests/fixtures.py` crea SQLite en memoria
(`f.new_session()`, `f.make_inventory_item`).

## Historia 1 — `app/inventory_engine/` existe y es equivalente

1. Crear el paquete según [contracts/module-api.md](./contracts/module-api.md) Contrato A
   (`__init__.py` + `stock.py`, sin frontera núcleo/adaptador — ver research.md Decisión 2).
2. Apuntar temporalmente el import de `test_inventory_stock.py` a `app.inventory_engine`.
3. Ejecutar los 16 characterization tests:

   ```bash
   python3 -m unittest app.characterization_tests.test_inventory_stock -v
   ```

   **Resultado esperado**: 16 tests en verde (`RecordMovementTests`: 7, `ApplyAdjustmentTests`: 6,
   `LockItemsTests`: 3), cero aserciones modificadas respecto al fichero original (diff del propio
   test file vacío salvo la línea de import).

4. Verificar el patrón de bloqueo por inspección estática (Acceptance Scenario 2, SC-006):

   ```bash
   grep -n "with_for_update" app/inventory_engine/stock.py
   # debe listar exactamente tres apariciones (una por función), en el mismo orden relativo
   # que hoy en app/api/v1/inventory/stock.py
   diff <(grep -n "with_for_update" app/api/v1/inventory/stock.py | sed 's/^[0-9]*://') \
        <(grep -n "with_for_update" app/inventory_engine/stock.py | sed 's/^[0-9]*://')
   # debe devolver vacío: mismo texto de línea, mismo orden
   ```

5. Verificar la firma exacta de las tres funciones (Acceptance Scenario 3):

   ```bash
   python3 -c "
   import inspect
   from app.api.v1.inventory import stock as legado
   from app import inventory_engine as nuevo
   for name in ('lock_items', 'record_movement', 'apply_adjustment'):
       sig_legado = inspect.signature(getattr(legado, name))
       sig_nuevo = inspect.signature(getattr(nuevo, name))
       assert sig_legado == sig_nuevo, f'{name}: {sig_legado} != {sig_nuevo}'
   print('firmas idénticas')
   "
   ```

**Independent Test** (según `spec.md`): estos cinco pasos son autocontenidos — no requieren que la
Historia 2 o 3 estén hechas, y no tocan ningún fichero consumidor.

## Historia 2 — Batería comparativa + revisión manual de A-35 (gate temporal, previo a la Historia 3)

1. **FR-009 ya resuelto en fase de planificación**: research.md Decisión 4 documenta la revisión
   manual de los tres sub-hallazgos de A-35 en alcance (tabla completa ahí) — no se construye
   golden master de inventario. Este paso no requiere ninguna acción adicional en la
   implementación, solo cita esa decisión como evidencia de SC-002.
2. Implementar `app/characterization_tests/inventory_engine_equivalence_gate.py` según
   [research.md](./research.md) Decisión 5 (generador `random.Random(seed)`, 100-200 casos,
   reutilizando `fixtures.py`) y el formato de reporte de [data-model.md](./data-model.md)
   §Batería comparativa.
3. Ejecutar:

   ```bash
   python3 -m unittest app.characterization_tests.inventory_engine_equivalence_gate -v
   ```

4. **Verificar reproducibilidad del generador primero** (Acceptance Scenario 2 de la Historia 2):
   correr dos veces con la misma semilla y confirmar que la lista de casos generados es idéntica
   byte a byte, antes de confiar en la comparación de implementaciones.
5. **Resultado esperado**: cero diferencias campo a campo entre `app.api.v1.inventory.stock`
   (legado) y `app.inventory_engine` (nuevo) en el 100% de los casos, incluyendo los que ejercitan
   los tres sub-hallazgos de A-35 en alcance (SC-003).
6. Si aparece una diferencia: el reporte debe identificar caso exacto + campo + valor legado vs.
   nuevo (Acceptance Scenario 4 de la Historia 2). Tratarla según el Edge Case de `spec.md`: bug de
   la extracción (corregir antes de continuar) o caso real no caracterizado (documentar en
   `registro-de-anomalias.md` antes de decidir) — nunca ajustar la batería para dejar de detectarla.

**Independent Test**: puede correr aislado como su propio script/test, pero solo tiene sentido una
vez `app/inventory_engine/` existe (Historia 1 completa).

## Historia 3 — Conmutación a fachada

1. Convertir `app/api/v1/inventory/stock.py` en fachada según
   [contracts/module-api.md](./contracts/module-api.md) Contrato B. **Prerequisito**: Historias 1
   y 2 en verde.
2. Si `inventory/service.py` necesita ajustar su import de `record_movement` (FR-008), hacerlo
   ahora — único cambio permitido en ese fichero.
3. Verificar diff vacío en los otros tres consumidores (SC-004):

   ```bash
   git diff --stat -- \
     app/api/v1/inventory/router.py \
     app/api/v1/sales/consumption.py \
     app/api/v1/orders/consumption.py
   ```

   **Resultado esperado**: sin salida (cero ficheros modificados).

4. Verificar que `inventory/service.py` solo cambió la línea de import, si acaso (FR-008):

   ```bash
   git diff -- app/api/v1/inventory/service.py
   ```

   **Resultado esperado**: diff vacío, o a lo sumo una línea de import modificada — ningún otro
   cambio.

5. Ejecutar la suite completa del backend, no solo inventario (SC-005):

   ```bash
   python3 -m unittest discover -s app -p "test_*.py" -v
   ```

   **Resultado esperado**: mismo conjunto de tests en verde/rojo que antes de empezar la
   extracción (ninguna regresión nueva).

6. Retirar o archivar el gate de la Historia 2 (`inventory_engine_equivalence_gate.py`),
   documentando su resultado final en el propio fichero (mismo tratamiento que
   `catalog_engine_equivalence_gate.py` de la spec 014) — a partir de aquí "legado" y "nuevo" son
   el mismo código, comparar deja de tener sentido. Los 16 characterization tests quedan como la
   red de regresión permanente (re-ejecutarlos apuntando ya a la fachada, que a su vez apunta al
   paquete nuevo, confirma que la conmutación no rompió nada — mismo comando del paso Historia 1,
   punto 3).

**Independent Test**: correr la suite completa del backend después de la conmutación, sin haber
tocado ningún fichero consumidor salvo el import mínimo de `inventory/service.py`, es suficiente
para validar esta historia de forma aislada.
