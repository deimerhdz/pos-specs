---

description: "Task list for feature implementation"
---

# Tasks: Extracción del motor de stock de inventario a `app/inventory_engine/`

**Input**: Design documents from `/specs/018-extraccion-motor-inventario/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/module-api.md](./contracts/module-api.md), [quickstart.md](./quickstart.md)

**Path convention**: todas las rutas de código son relativas a la raíz del repositorio sibling
`../pos-backend` (ejecutado desde `pos-specs/`), tal como establece `plan.md` §Project Structure.
Las rutas bajo `specs/` o `.specify/` son relativas a este repositorio (`pos-specs`).

**Tests**: los 16 characterization tests de `test_inventory_stock.py` ya existen y actúan como
árbitro (Constitución, Principio II); no se generan tests nuevos con TDD. La única "escritura de
test nuevo" es el gate temporal de la Historia 2, pedido explícitamente por FR-010.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias pendientes)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3)

---

## Phase 1: Setup

**Purpose**: preparar el entorno de `pos-backend` para la extracción

- [X] T001 Activar el entorno virtual de `pos-backend` (`cd ../pos-backend && source env/bin/activate`) y confirmar que `app/inventory_engine/` todavía no existe (`find app -iname "*inventory_engine*"`), sin colisión de nombre (research.md Decisión 1)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: confirmar la línea base verde antes de mover ni una línea de código

**⚠️ CRITICAL**: sin esta línea base verificada en verde, ninguna de las tres historias tiene con
qué comparar su resultado

- [X] T002 Ejecutar en `../pos-backend` los 16 characterization tests contra el código legado (`python3 -m unittest app.characterization_tests.test_inventory_stock -v`) y confirmar que los 16 pasan hoy (`RecordMovementTests`: 7, `ApplyAdjustmentTests`: 6, `LockItemsTests`: 3), antes de tocar nada

**Checkpoint**: línea base verde confirmada — puede empezar la Historia 1

---

## Phase 3: User Story 1 - Extraer el motor de stock a `app/inventory_engine/` sin alterar su salida (Priority: P1) 🎯 MVP

**Goal**: mover las tres funciones (`lock_items`, `record_movement`, `apply_adjustment`) de
`app/api/v1/inventory/stock.py` a `app/inventory_engine/`, sin dividir núcleo/adaptador (las tres
son impuras), conservando el patrón `.with_for_update()` exactamente en los mismos puntos.

**Independent Test**: ejecutar los 16 characterization tests apuntando sus imports a
`app/inventory_engine/`, sin tocar ningún fichero consumidor.

### Implementation for User Story 1

- [X] T003 [US1] Crear el paquete `app/inventory_engine/` con `__init__.py` vacío y `stock.py` conteniendo las tres funciones (`lock_items`, `record_movement`, `apply_adjustment`) copiadas verbatim desde `app/api/v1/inventory/stock.py:1-121` (docstring del módulo, los tres imports de `sqlalchemy`/`app.core.exceptions`/`app.models.*`, y el cuerpo íntegro de las tres funciones sin dividir núcleo/adaptador — research.md Decisión 2), al mismo nivel que `app/models/`, `app/core/` y `app/catalog_engine/` (FR-001, FR-002, FR-003, contracts/module-api.md Contrato A)

- [X] T004 [US1] Completar `app/inventory_engine/__init__.py` reexportando los tres símbolos públicos (`lock_items`, `record_movement`, `apply_adjustment`) definidos en T003, importables como `from app.inventory_engine import <símbolo>` (contracts/module-api.md Contrato A, depende de T003)

- [X] T005 [US1] Apuntar únicamente la línea de import (sin tocar ninguna aserción) de `app/characterization_tests/test_inventory_stock.py` a `app.inventory_engine` en vez de `app.api.v1.inventory.stock`, según contracts/module-api.md Contrato C (FR-007, depende de T004)

- [X] T006 [P] [US1] Ejecutar `python3 -m unittest app.characterization_tests.test_inventory_stock -v` desde `../pos-backend` y confirmar que los 16 tests pasan en verde con cero aserciones modificadas, incluyendo `test_rn_inv_05_allow_negative_true_permite_dejar_stock_negativo`, `test_rn_inv_11_motivo_no_es_obligatorio` y `test_rn_inv_10_delta_cero_lanza_valueerror_no_http_exception` (SC-001, Acceptance Scenarios 1/4/5, depende de T005)

- [X] T007 [P] [US1] Verificar por inspección estática el patrón de bloqueo (Acceptance Scenario 2, SC-006): `grep -n "with_for_update" app/inventory_engine/stock.py` debe listar exactamente tres apariciones (una por función); `diff <(grep -n "with_for_update" app/api/v1/inventory/stock.py | sed 's/^[0-9]*://') <(grep -n "with_for_update" app/inventory_engine/stock.py | sed 's/^[0-9]*://')` debe devolver vacío (mismo texto de línea, mismo orden relativo) (depende de T003)

- [X] T008 [P] [US1] Verificar la firma exacta de las tres funciones (Acceptance Scenario 3) ejecutando el script de `inspect.signature` de [quickstart.md](./quickstart.md) Historia 1 paso 5, comparando `app.api.v1.inventory.stock` (legado) contra `app.inventory_engine` (nuevo) para `lock_items`, `record_movement`, `apply_adjustment` — debe imprimir "firmas idénticas" sin ningún `AssertionError` (depende de T004)

**Checkpoint**: `app/inventory_engine/` existe, pasa los 16 tests, conserva el patrón de bloqueo y
las firmas exactas — la Historia 1 es verificable de forma independiente

---

## Phase 4: User Story 2 - Verificación de equivalencia comparativa masiva (Priority: P2)

**Goal**: batería determinista (semilla fija, 100-200 casos) que compara campo a campo la salida
del código legado contra `app/inventory_engine/` sobre el mismo estado de datos, más la revisión
manual documentada de FR-009 (ya resuelta en research.md Decisión 4, sin golden master nuevo).

**Independent Test**: corre como su propio script/test una vez que `app/inventory_engine/` existe
(Historia 1 completa), sin depender de la Historia 3.

### Implementation for User Story 2

- [X] T009 [US2] Citar research.md Decisión 4 (tabla de revisión manual de los tres sub-hallazgos de A-35 en alcance) como evidencia de FR-009 opción (b) y SC-002 — sin acción de código adicional, este artefacto ya satisface la Historia 2 en cuanto a la decisión golden master vs. revisión manual (Acceptance Scenario 1 de la Historia 2)

- [X] T010 [US2] Implementar el generador determinista + comparador en `app/characterization_tests/inventory_engine_equivalence_gate.py`: `random.Random(seed)` genera entre 100 y 200 casos combinando tipo de movimiento (`in`/`out`/rechazado), cantidad, `allow_negative`, `signed_delta` (positivo/negativo/cero) y `current_stock` inicial en y cerca de cero, usando las factorías existentes de `fixtures.py` (`f.new_session`, `f.make_inventory_item`) sin fixture nuevo, ejecuta ambas implementaciones (`app.api.v1.inventory.stock` legado y `app.inventory_engine` nuevo) sobre el mismo estado de datos por caso, y captura `caso_id`, `entrada`, `legado`, `nuevo`, `campo_divergente` (FR-010, research.md Decisión 5, data-model.md §Batería comparativa, depende de T004)

- [X] T011 [US2] Añadir la verificación de reproducibilidad del generador: correr el generador dos veces con la misma semilla y confirmar que la lista de casos generados es idéntica byte a byte, antes de comparar implementaciones (Acceptance Scenario 2 de la Historia 2, depende de T010)

- [X] T012 [US2] Añadir la aserción de igualdad campo a campo entre `legado` y `nuevo` por caso — incluyendo el `current_stock` resultante, los campos del `InventoryMovement` creado, y tipo/mensaje de cualquier excepción levantada (`InsufficientStockError`, `ValueError`) — fallando con `caso_id` + campo divergente + valor legado vs. nuevo en cuanto aparezca una diferencia (Acceptance Scenarios 3-4 de la Historia 2, FR-011, depende de T010)

- [X] T013 [US2] Ejecutar `python3 -m unittest app.characterization_tests.inventory_engine_equivalence_gate -v` desde `../pos-backend` y confirmar cero divergencias en el 100% de los casos generados, incluyendo los que ejercitan los tres sub-hallazgos de A-35 en alcance (SC-003, depende de T011, T012)

- [X] T014 [US2] Si T013 reporta alguna divergencia: triarla según el Edge Case de spec.md — corregir el bug de la extracción antes de continuar, o documentar una anomalía nueva en `../pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` antes de decidir cómo tratarla; nunca ajustar el gate para que deje de detectarla (FR-011, depende de T013)

**Checkpoint**: el gate de equivalencia pasa en verde (o toda divergencia quedó corregida o
documentada) y FR-009 queda satisfecho — hay base para conmutar con confianza en la Historia 3

---

## Phase 5: User Story 3 - Conmutación final a fachada (Priority: P3)

**Goal**: convertir `app/api/v1/inventory/stock.py` en fachada pura que reexporta desde
`app/inventory_engine/`, sin que los cuatro ficheros consumidores cambien una línea salvo el
ajuste mínimo de import permitido en `inventory/service.py`.

**Independent Test**: correr la suite completa de tests del backend después de la conmutación,
sin haber tocado ningún fichero consumidor salvo el import mínimo de `inventory/service.py`.

**Prerequisito**: Historias 1 y 2 en verde (spec.md: sin ellas verificadas no hay base para
conmutar con confianza).

### Implementation for User Story 3

- [X] T015 [US3] Convertir `app/api/v1/inventory/stock.py` en fachada pura: reemplazar el cuerpo de las tres funciones por imports de reexport nombrado desde `app.inventory_engine` (`from app.inventory_engine import lock_items as lock_items`, y equivalente para `record_movement`, `apply_adjustment`), sin funciones wrapper y sin lógica propia de cálculo/validación/consulta (FR-005, contracts/module-api.md Contrato B, depende de T004)

- [X] T016 [US3] Verificar si `inventory/service.py:17` necesita ajustar su import de `record_movement` (de `app.api.v1.inventory.stock` directo a `app.inventory_engine`, u dejarlo apuntando a la fachada de T015) — único cambio permitido en este fichero, ningún otro (FR-008, depende de T015)

- [X] T017 [US3] Verificar diff vacío en los otros tres consumidores: `git diff --stat -- app/api/v1/inventory/router.py app/api/v1/sales/consumption.py app/api/v1/orders/consumption.py` desde `../pos-backend` no debe producir salida (SC-004, FR-006, depende de T015)

- [X] T018 [US3] Verificar que `inventory/service.py` solo cambió, a lo sumo, la línea de import: `git diff -- app/api/v1/inventory/service.py` desde `../pos-backend` debe mostrar diff vacío o una única línea de import modificada (FR-008, depende de T016)

- [X] T019 [US3] Re-ejecutar `python3 -m unittest app.characterization_tests.test_inventory_stock -v` desde `../pos-backend`, ahora contra la fachada que delega en `app.inventory_engine`, y confirmar que los 16 siguen en verde tras la conmutación (depende de T017, T018)

- [X] T020 [US3] Ejecutar la suite completa del backend desde `../pos-backend` (`python3 -m unittest discover -s app -p "test_*.py" -v`) y confirmar el mismo conjunto de tests en verde/rojo que antes de empezar la extracción (SC-005, depende de T019)

- [X] T021 [US3] Retirar o archivar `app/characterization_tests/inventory_engine_equivalence_gate.py`, documentando su resultado final (cero divergencias, N casos ejecutados) — a partir de aquí "legado" y "nuevo" son el mismo código, comparar deja de tener sentido (Edge Cases de spec.md, depende de T020)

**Checkpoint**: las tres historias están completas — la extracción quedó conmutada y verificada

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de extremo a extremo

- [X] T022 [P] Confirmar que el diff de `test_inventory_stock.py` respecto a su versión previa a T005 se limita a la línea de import (cero aserciones tocadas), según FR-007
- [X] T023 Recorrer [quickstart.md](./quickstart.md) de punta a punta (Historias 1 → 2 → 3) como pase de sanidad final sobre el estado ya conmutado del repositorio

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA las tres historias
- **User Story 1 (Phase 3)**: depende de Foundational; es la base técnica de las otras dos
- **User Story 2 (Phase 4)**: depende de que Historia 1 esté completa (`app/inventory_engine/` debe existir para poder compararlo contra el legado) — igual que la spec 014, esta spec define las historias como secuenciales (plan.md §Summary), no independientes en paralelo
- **User Story 3 (Phase 5)**: depende de que Historias 1 y 2 estén en verde (prerequisito explícito de spec.md)
- **Polish (Phase 6)**: depende de que las tres historias estén completas

### User Story Dependencies

- **User Story 1 (P1)**: sin dependencia de otra historia — es el punto de partida
- **User Story 2 (P2)**: depende de User Story 1 (necesita `app/inventory_engine/` para comparar)
- **User Story 3 (P3)**: depende de User Story 1 y User Story 2 en verde

### Parallel Opportunities

- T006, T007, T008 son de solo lectura/verificación sobre lo producido en T003-T005 y pueden correr en paralelo entre sí
- T022 es independiente del resto de Polish y puede correr en paralelo con T023
- Dentro de Historia 1, T003 y T004 son secuenciales (T004 reexporta lo creado en T003) — a diferencia de la spec 014, aquí no hay ficheros de módulo distintos (`core.py`/`pricing.py`/`consumption.py`) que mover en paralelo, porque research.md Decisión 2 mantiene las tres funciones juntas en un único `stock.py`

---

## Parallel Example: User Story 1

```bash
# Tras T005 (import de los tests apuntado a app.inventory_engine), verificar en paralelo:
Task: "Ejecutar los 16 characterization tests contra app.inventory_engine"
Task: "Verificar por inspección estática el patrón with_for_update"
Task: "Verificar la firma exacta de las tres funciones con inspect.signature"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (línea base verde confirmada)
3. Completar Phase 3: User Story 1
4. **PARAR y VALIDAR**: los 16 tests en verde contra `app/inventory_engine/`, `with_for_update()` en los mismos tres puntos, firmas idénticas
5. En este punto `app/inventory_engine/` existe en paralelo al código legado, sin que nada dependa de él todavía — es seguro parar aquí si hace falta

### Incremental Delivery

1. Setup + Foundational → línea base verde confirmada
2. Historia 1 → `app/inventory_engine/` existe y es equivalente (MVP técnico, aún sin conmutar)
3. Historia 2 → gate de equivalencia masiva en verde + FR-009 resuelto (confianza adicional antes de conmutar)
4. Historia 3 → conmutación a fachada (la extracción se vuelve real para el resto del sistema)
5. Cada historia añade una señal de verificación sin romper la anterior — igual que la spec 014, el orden P1→P2→P3 es también el orden de ejecución obligatorio, no solo de prioridad

---

## Notes

- [P] = ficheros distintos o solo-lectura, sin dependencias pendientes entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- Esta spec no pide tests nuevos por TDD: los 16 characterization tests ya existen y actúan como el árbitro (Constitución, Principio II); la única "escritura de test nuevo" es el gate temporal de la Historia 2 (T010-T012)
- Ningún fichero de los cuatro consumidores se toca hasta que T017/T018 lo verifiquen con diff vacío (salvo el import de `service.py`) — si cualquier tarea anterior requiriera tocar uno de ellos, eso es señal de que algo se desvió del contrato (FR-006)
- `app/characterization_tests/test_core_inventory_reasons.py` (catálogo de motivos, RN-INV-13) no forma parte de esta spec: caracteriza `app/core/inventory_reasons.py`, no `inventory/stock.py` — verificado, sin tarea asociada
- Commitear tras cada tarea o grupo lógico
- Parar en cada checkpoint para validar la historia de forma independiente antes de seguir
