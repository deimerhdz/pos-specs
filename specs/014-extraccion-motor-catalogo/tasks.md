---

description: "Task list for feature implementation"
---

# Tasks: Extracción del motor de catálogo a `app/catalog_engine/`

**Input**: Design documents from `/specs/014-extraccion-motor-catalogo/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/module-api.md](./contracts/module-api.md), [quickstart.md](./quickstart.md)

**Path convention**: todas las rutas de código son relativas a la raíz del repositorio sibling
`../pos-backend` (ejecutado desde `pos-specs/`), tal como establece `plan.md` §Project Structure.
Las rutas bajo `specs/` o `.specify/` son relativas a este repositorio (`pos-specs`).

**Tests**: los tests de esta extracción ya existen (41 characterization tests + golden master);
no se generan tests nuevos con TDD — las tareas que crean tests nuevos son las de la Historia 2
(gate de equivalencia), explícitamente pedidas por FR-013.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias pendientes)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3)

---

## Phase 1: Setup

**Purpose**: preparar el entorno de `pos-backend` para la extracción

- [X] T001 Activar el entorno virtual de `pos-backend` (`cd ../pos-backend && source env/bin/activate`) y confirmar que `app/catalog_engine/` todavía no existe (`ls app/`), sin colisión de nombre

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: confirmar la línea base verde antes de mover ni una línea de código

**⚠️ CRITICAL**: sin esta línea base verificada en verde, ninguna de las tres historias tiene con
qué comparar su resultado

- [X] T002 Ejecutar en `../pos-backend` los 41 characterization tests y el golden master contra el código legado (`python3 -m unittest app.characterization_tests.test_catalog_line_pricing app.characterization_tests.test_catalog_consumption_plan app.characterization_tests.test_golden_master_pricing_consumption -v`) y confirmar que los 41 + el golden master pasan hoy, antes de tocar nada

**Checkpoint**: línea base verde confirmada — puede empezar la Historia 1

---

## Phase 3: User Story 1 - Extraer el motor a `app/catalog_engine/` sin alterar su salida (Priority: P1) 🎯 MVP

**Goal**: mover las trece funciones y `ConsumptionLine` a `app/catalog_engine/`, separando el
núcleo puro de los adaptadores, sin que ningún fichero consumidor cambie una línea.

**Independent Test**: ejecutar los 41 characterization tests y el golden master (sin regenerar)
apuntando sus imports a `app/catalog_engine/`, sin tocar ningún fichero consumidor.

### Implementation for User Story 1

- [X] T003 [US1] Crear el esqueleto del paquete en `app/catalog_engine/`: ficheros vacíos `__init__.py`, `core.py`, `pricing.py`, `consumption.py`, al mismo nivel que `app/models/` y `app/core/` (research.md Decisión 1-2)

- [X] T004 [P] [US1] Mover el núcleo puro a `app/catalog_engine/core.py`: dataclass `ConsumptionLine` (`frozen=True`, campos `inventory_item_id`, `quantity`, `source`, desde `consumption_plan.py:49-56`), `compute_line_price(variant: ProductVariant, options: list[Option]) -> Decimal` (desde `line_pricing.py:191-196`), `_exige_maximo(gid: UUID, lo: int, consumen: set[UUID]) -> bool` (desde `line_pricing.py:94-105`) — copiados verbatim; usar `from __future__ import annotations` + import de `ProductVariant`/`Option` bajo `TYPE_CHECKING` para que ningún import de `sqlalchemy` llegue a este fichero (FR-002, FR-003, SC-006, data-model.md §Núcleo puro)

- [X] T005 [P] [US1] Mover los cuatro adaptadores del grupo `pricing` a `app/catalog_engine/pricing.py`: `load_valid_options`, `grupos_que_descuentan`, `validate_option_selection`, `check_availability` — copiados verbatim desde `line_pricing.py:43-220` con la misma firma y `db: Session` como primer parámetro, reproduciendo tal cual A-05, A-06, A-32 (parte `grupos_que_descuentan`), A-47 (FR-004, FR-005, FR-006, contracts/module-api.md Contrato A)

- [X] T006 [P] [US1] Mover los siete adaptadores del grupo `consumption` a `app/catalog_engine/consumption.py`: `load_recipe`, `load_variant_groups`, `group_discounts`, `plan_line_consumption`, `required_consumption`, `variant_label`, `ensure_lines_consume_inventory` — copiados verbatim desde `consumption_plan.py:59-227`, importando `ConsumptionLine` desde `app.catalog_engine.core`, reproduciendo tal cual A-02, A-32 (parte `group_discounts`), A-33 (FR-004, FR-005, FR-006, contracts/module-api.md Contrato A)

- [X] T007 [US1] Completar `app/catalog_engine/__init__.py` reexportando exactamente los catorce símbolos públicos (los definidos en T004, T005, T006) según contracts/module-api.md Contrato A (depende de T004, T005, T006)

- [X] T008 [US1] Apuntar únicamente la línea de import (sin tocar ninguna aserción) de `app/characterization_tests/test_catalog_line_pricing.py`, `app/characterization_tests/test_catalog_consumption_plan.py` y `app/characterization_tests/golden_master_core.py` a `app.catalog_engine` en vez de `app.api.v1.catalog.*`, según contracts/module-api.md Contrato C (FR-011, FR-012) (depende de T007)

- [X] T009 [P] [US1] Ejecutar `python3 -m unittest app.characterization_tests.test_catalog_line_pricing app.characterization_tests.test_catalog_consumption_plan -v` desde `../pos-backend` y confirmar que los 41 tests pasan en verde con cero aserciones modificadas (SC-001, depende de T008)

- [X] T010 [P] [US1] Ejecutar `python3 -m unittest app.characterization_tests.test_golden_master_pricing_consumption -v` desde `../pos-backend` y confirmar que pasa contra `golden_master/pricing_consumption.master.json` sin regenerarlo (SC-002, depende de T008)

- [X] T011 [P] [US1] Verificar por inspección estática la frontera núcleo/adaptador: `grep -n sqlalchemy app/catalog_engine/core.py` y `grep -n "db: Session" app/catalog_engine/core.py` deben devolver vacío; `grep -n "db: Session" app/catalog_engine/pricing.py app/catalog_engine/consumption.py` debe listar las once funciones adaptadoras con `db` como primer parámetro (SC-006, Acceptance Scenarios 3-4, depende de T004, T005, T006)

- [X] T012 [US1] Confirmar manualmente, usando el caso de la invariante `[PROTEGIDA]` A-02 ya cubierto por los characterization tests recién ejecutados en T009, que `plan_line_consumption`/`required_consumption` en `app.catalog_engine.consumption` descuentan únicamente la cantidad del tamaño cuando tamaño y opción tienen `quantity_per_option`/`item_quantity` distintos — nunca la suma (Acceptance Scenario 5, depende de T009)

**Checkpoint**: `app/catalog_engine/` existe, pasa los 41 tests + golden master, y cumple la
frontera núcleo/adaptador — la Historia 1 es verificable de forma independiente

---

## Phase 4: User Story 2 - Verificación de equivalencia comparativa masiva (Priority: P2)

**Goal**: batería determinista (semilla fija, 100-200 casos) que compara campo a campo la salida
del código legado contra `app/catalog_engine/` sobre el mismo estado de datos.

**Independent Test**: corre como su propio script/test una vez que `app/catalog_engine/` existe
(Historia 1 completa), sin depender de la Historia 3.

### Implementation for User Story 2

- [X] T013 [US2] Implementar el generador determinista + comparador en `app/characterization_tests/catalog_engine_equivalence_gate.py`: `random.Random(seed)` genera entre 100 y 200 casos combinando variantes/grupos de opciones/recetas/niveles de stock tomados del fixture existente (`fixtures.py`, sin fixture nuevo), ejecuta ambas implementaciones (`app.api.v1.catalog.*` legado y `app.catalog_engine.*` nuevo) sobre el mismo estado de datos por caso, y captura `caso_id`, `entrada`, `legado`, `nuevo`, `campo_divergente` (FR-013, research.md Decisión 5, data-model.md §Batería comparativa)

- [X] T014 [US2] Añadir la verificación de reproducibilidad del generador: correr el generador dos veces con la misma semilla y confirmar que la lista de casos generados es idéntica byte a byte, antes de comparar implementaciones (Acceptance Scenario 1 de la Historia 2, depende de T013)

- [X] T015 [US2] Añadir la aserción de igualdad campo a campo entre `legado` y `nuevo` por caso (incluyendo tipo/`status_code`/`detail` de cualquier excepción levantada), fallando con el `caso_id` + campo divergente + valor legado vs. nuevo en cuanto aparezca una diferencia (Acceptance Scenarios 2-3 de la Historia 2, FR-014, depende de T013)

- [X] T016 [US2] Ejecutar `python3 -m unittest app.characterization_tests.catalog_engine_equivalence_gate -v` desde `../pos-backend` y confirmar cero divergencias en el 100% de los casos generados, incluyendo los que ejercitan A-02, A-05, A-06, A-32 y A-33 (SC-003, depende de T014, T015)

- [X] T017 [US2] Si T016 reporta alguna divergencia: triarla según el Edge Case de spec.md — corregir el bug de la extracción antes de continuar, o documentar una anomalía nueva en `../pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` antes de decidir cómo tratarla; nunca ajustar el gate para que deje de detectarla (depende de T016)

**Checkpoint**: el gate de equivalencia pasa en verde (o toda divergencia quedó corregida o
documentada) — hay base para conmutar con confianza en la Historia 3

---

## Phase 5: User Story 3 - Conmutación final a fachada (Priority: P3)

**Goal**: convertir `line_pricing.py` y `consumption_plan.py` en fachadas puras de
`app/catalog_engine/`, sin que los siete ficheros consumidores cambien una línea.

**Independent Test**: correr la suite completa de tests del backend después de la conmutación,
sin haber tocado ningún fichero consumidor.

**Prerequisito**: Historias 1 y 2 en verde (spec.md: sin ellas verificadas no hay base para
conmutar con confianza).

### Implementation for User Story 3

- [X] T018 [US3] Convertir `app/api/v1/catalog/line_pricing.py` en fachada pura: reemplazar todo el cuerpo por imports de reexport nombrado desde `app.catalog_engine` (`compute_line_price`, `load_valid_options`, `check_availability`, `validate_option_selection`, `grupos_que_descuentan`, `_exige_maximo`), preservando el reexport actual de las líneas 31-36 (`ConsumptionLine`, `load_variant_groups`, `plan_line_consumption`, `required_consumption`) del que depende `cart/service.py:31-36` — sin funciones wrapper (FR-008, FR-009, contracts/module-api.md Contrato B)

- [X] T019 [P] [US3] Convertir `app/api/v1/catalog/consumption_plan.py` en fachada pura: reemplazar todo el cuerpo por imports de reexport nombrado desde `app.catalog_engine` (`ConsumptionLine`, `load_recipe`, `load_variant_groups`, `plan_line_consumption`, `required_consumption`, `group_discounts`, `variant_label`, `ensure_lines_consume_inventory`) — sin funciones wrapper (FR-008, contracts/module-api.md Contrato B)

- [X] T020 [US3] Verificar diff vacío en los siete ficheros consumidores: `git diff --stat -- app/api/v1/sales/service.py app/api/v1/sales/consumption.py app/api/v1/orders/service.py app/api/v1/orders/consolidation.py app/api/v1/orders/kitchen.py app/api/v1/orders/consumption.py app/api/v1/cart/service.py` desde `../pos-backend` no debe producir salida (SC-004, depende de T018, T019)

- [X] T021 [US3] Re-ejecutar los 41 characterization tests y el golden master (mismos comandos de T009/T010), ahora contra las fachadas que delegan en `app.catalog_engine`, y confirmar que siguen en verde tras la conmutación (depende de T020)

- [X] T022 [US3] Ejecutar la suite completa del backend desde `../pos-backend` (`python3 -m unittest discover -s app -p "test_*.py" -v`) y confirmar el mismo conjunto de tests en verde/rojo que antes de empezar la extracción (SC-005, depende de T021)

- [X] T023 [US3] Retirar o archivar `app/characterization_tests/catalog_engine_equivalence_gate.py`, documentando su resultado final (cero divergencias, N casos ejecutados) según la Clarification de sesión 2026-08-17 — legado y nuevo pasan a ser el mismo código, así que comparar deja de tener sentido (depende de T022)

**Checkpoint**: las tres historias están completas — la extracción quedó conmutada y verificada

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de extremo a extremo

- [X] T024 [P] Confirmar que el diff de `test_catalog_line_pricing.py`, `test_catalog_consumption_plan.py` y `golden_master_core.py` respecto a su versión previa a T008 se limita a la línea de import (cero aserciones tocadas), según FR-011/FR-012
- [X] T025 Recorrer [quickstart.md](./quickstart.md) de punta a punta (Historias 1 → 2 → 3) como pase de sanidad final sobre el estado ya conmutado del repositorio

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA las tres historias
- **User Story 1 (Phase 3)**: depende de Foundational; es la base técnica de las otras dos
- **User Story 2 (Phase 4)**: depende de que Historia 1 esté completa (`app/catalog_engine/` debe existir para poder compararlo contra el legado) — a diferencia del patrón habitual de historias independientes, esta spec las define secuenciales (plan.md §Summary)
- **User Story 3 (Phase 5)**: depende de que Historias 1 y 2 estén en verde (prerequisito explícito de spec.md)
- **Polish (Phase 6)**: depende de que las tres historias estén completas

### User Story Dependencies

- **User Story 1 (P1)**: sin dependencia de otra historia — es el punto de partida
- **User Story 2 (P2)**: depende de User Story 1 (necesita `app/catalog_engine/` para comparar)
- **User Story 3 (P3)**: depende de User Story 1 y User Story 2 en verde

### Parallel Opportunities

- T004, T005, T006 (núcleo, pricing, consumption) tocan ficheros distintos y pueden ejecutarse en paralelo una vez existe T003
- T009, T010, T011 son de solo lectura/verificación sobre lo producido en T004-T007 y pueden correr en paralelo entre sí
- T018 y T019 tocan ficheros distintos (`line_pricing.py` vs. `consumption_plan.py`) y pueden ejecutarse en paralelo
- T024 es independiente del resto de Polish y puede correr en paralelo con T025

---

## Parallel Example: User Story 1

```bash
# Tras T003 (esqueleto creado), mover el contenido en paralelo:
Task: "Mover el núcleo puro a app/catalog_engine/core.py"
Task: "Mover los adaptadores pricing a app/catalog_engine/pricing.py"
Task: "Mover los adaptadores consumption a app/catalog_engine/consumption.py"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (línea base verde confirmada)
3. Completar Phase 3: User Story 1
4. **PARAR y VALIDAR**: los 41 tests + golden master en verde contra `app/catalog_engine/`, cero imports de `sqlalchemy` en `core.py`
5. En este punto `app/catalog_engine/` existe en paralelo al código legado, sin que nada dependa de él todavía — es seguro parar aquí si hace falta

### Incremental Delivery

1. Setup + Foundational → línea base verde confirmada
2. Historia 1 → `app/catalog_engine/` existe y es equivalente (MVP técnico, aún sin conmutar)
3. Historia 2 → gate de equivalencia masiva en verde (confianza adicional antes de conmutar)
4. Historia 3 → conmutación a fachada (la extracción se vuelve real para el resto del sistema)
5. Cada historia añade una señal de verificación sin romper la anterior — a diferencia de otras specs, aquí el orden P1→P2→P3 es también el orden de ejecución obligatorio, no solo de prioridad

---

## Notes

- [P] = ficheros distintos, sin dependencias pendientes entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- Esta spec no pide tests nuevos por TDD: los 41 characterization tests y el golden master ya existen y actúan como el árbitro (Constitución, Principio II); la única "escritura de test nuevo" es el gate temporal de la Historia 2 (T013-T015)
- Ningún fichero de los siete consumidores se toca hasta que T020 lo verifique con diff vacío — si cualquier tarea anterior requiriera tocar uno de ellos, eso es señal de que algo se desvió del contrato (FR-010)
- Commitear tras cada tarea o grupo lógico
- Parar en cada checkpoint para validar la historia de forma independiente antes de seguir
