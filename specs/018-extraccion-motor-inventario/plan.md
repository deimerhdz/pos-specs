# Implementation Plan: Extracción del motor de stock de inventario a `app/inventory_engine/`

**Branch**: `018-extraccion-motor-inventario` | **Date**: 2026-08-17 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/018-extraccion-motor-inventario/spec.md`

## Summary

Mover las tres funciones hoy en `app/api/v1/inventory/stock.py` (121 líneas: `lock_items`,
`record_movement`, `apply_adjustment`) del repositorio `pos-backend` a un paquete nuevo,
`app/inventory_engine/`, sin dividir núcleo puro de adaptador — a diferencia del motor de catálogo
(spec 014), las tres funciones son impuras (reciben `db: Session`, usan `.with_for_update()`), así
que no hay ninguna función sin `db` que aislar. Es una extracción de código por estrangulamiento
(Constitución, Principio III), no una reescritura: cero reglas de negocio cambian, y los tres
sub-hallazgos de A-35 en alcance (`allow_negative` sin llamador, motivo no obligatorio,
`signed_delta=0` sin handler) se reproducen tal cual. El enfoque técnico sigue las tres historias
de la spec: (1) crear `app/inventory_engine/` con las tres funciones verificadas contra los 16
characterization tests existentes, sin golden master nuevo — se documenta en su lugar una revisión
manual explícita de los tres sub-hallazgos (ver Decisión 4 de research.md); (2) correr una batería
comparativa generada con semilla fija (100-200 casos, mismas factorías de `fixtures.py`) como gate
único antes de conmutar, mismo patrón que `catalog_engine_equivalence_gate.py` de la spec 014; (3)
convertir `app/api/v1/inventory/stock.py` en fachada que reexporta desde `app/inventory_engine/`,
sin tocar ningún fichero consumidor salvo el ajuste mínimo de import permitido en
`inventory/service.py` (FR-008).

## Technical Context

**Language/Version**: Python 3.14 (venv de `pos-backend`, `env/pyvenv.cfg`)

**Primary Dependencies**: SQLAlchemy (ORM — `Session`, `select`, `.with_for_update()`) y
`app.core.exceptions.InsufficientStockError` (subclase de `HTTPException`, ya usada por el código
legado); ninguna dependencia nueva (Constitución, Principio IV — no hace falta justificar nada
porque no se añade ninguna)

**Storage**: PostgreSQL 16 schema-per-tenant en producción (sin cambios: el motor no gana ni pierde
acceso a datos — `inventory_items`/`inventory_movements` bajo `schema: "tenant"`); SQLite en
memoria vía `app/characterization_tests/fixtures.py` (`f.new_session()`, `f.make_inventory_item`)
para los tests, reutilizado tal cual sin fixture nuevo (clarificación de la Historia 2 en `spec.md`)

**Testing**: `unittest` vía `python3 -m unittest` (convención existente del proyecto, verificada:
no hay `pytest.ini` en el repo) — los 16 characterization tests de `test_inventory_stock.py`
(`RecordMovementTests`: 7, `ApplyAdjustmentTests`: 6, `LockItemsTests`: 3), sin golden master nuevo
(ver Decisión 4 de research.md), y el nuevo test/script de equivalencia comparativa de la Historia 2

**Target Platform**: Linux server (`pos-backend`, API FastAPI en producción)

**Project Type**: extracción de módulo interno dentro de un servicio backend único (no hay
frontend involucrado — `pos-heladeria` no importa código de `inventory/stock.py`, solo consume la
API HTTP de `inventory/router.py`, que no cambia en esta spec)

**Performance Goals**: sin objetivo nuevo — el requisito es cero regresión: mismo número de
consultas SQL por función, mismo punto exacto donde se aplica `.with_for_update()` (FR-002), cero
cambio en el número de round-trips a `db`

**Constraints**: cero líneas modificadas en los cuatro ficheros consumidores hasta la Historia 3,
salvo el ajuste mínimo de import en `inventory/service.py` (FR-006, FR-008, SC-004); el patrón
`SELECT ... FOR UPDATE` no se reordena ni se envuelve en abstracción nueva (FR-002, SC-006); ninguna
aserción de los 16 characterization tests se modifica (FR-007); los tres sub-hallazgos de A-35 en
alcance se reproducen sin corregir (FR-003)

**Scale/Scope**: 3 funciones, ~121 líneas movidas desde 1 fichero legado a un paquete nuevo de 2
módulos (`__init__.py` + `stock.py`); 4 ficheros consumidores de producción, de los cuales solo
`inventory/service.py` puede tocarse (import únicamente); 16 characterization tests como red de
regresión permanente; 1 batería comparativa temporal de 100-200 casos (Historia 2, se retira o
archiva tras la Historia 3, mismo tratamiento que dio la spec 014 a la suya)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. El Comportamiento Sigue Siendo Sagrado (por Defecto)** | La spec no introduce ningún cambio de comportamiento — los tres sub-hallazgos `PENDIENTE`/`ACCIDENTAL` de A-35 en alcance se reproducen tal cual (FR-003), y el sub-hallazgo 4 (costo unitario, `inventory/service.py`) queda explícitamente fuera (FR-004) porque corregirlo exigiría una decisión de negocio que esta spec no toma. No se necesita ninguna entrada nueva en el registro de anomalías porque no hay cambio de comportamiento que autorizar. | PASS |
| **II. Los Characterization Tests son el Árbitro** | Los 16 characterization tests se ejecutan sin modificar ni una aserción (FR-007); cualquier test en rojo bloquea la conmutación y se revierte, nunca se ajusta (FR-011). Ningún test con prefijo `"CONGELA comportamiento actual:"` se toca en esta spec — el docstring del módulo `test_inventory_stock.py` ya lleva ese prefijo (verificado: `"""CONGELA app/api/v1/inventory/stock.py..."""`). | PASS |
| **III. Estrangulamiento antes que Reescritura** | Es exactamente un módulo (el motor de stock de inventario) el que se extrae; verificado que la spec 014 (motor de catálogo) ya completó su Historia 3 (existe `app/catalog_engine/` con fachadas activas) — no hay dos módulos en reescritura simultánea. Los cuatro ficheros consumidores no se tocan salvo el import mínimo de `inventory/service.py` (FR-006, FR-008). | PASS |
| **IV. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia nueva. El generador determinista de la Historia 2 usa `random.Random(seed)` de la biblioteca estándar, mismo patrón que `catalog_engine_equivalence_gate.py` de la spec 014 — no hace falta justificación porque no hay nada que aprobar. | PASS (no aplica) |
| **V. Ningún Cambio Retroactivo** | Esta spec no toca facturación ni datos ya persistidos; el motor de stock mueve `current_stock` y registra kardex en el momento de la operación, no reprocesa histórico ni facturas emitidas. | PASS (no aplica) |
| **VI. Todo en Español de Colombia** | Esta spec, su plan y los artefactos que genera (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que la spec de origen. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/018-extraccion-motor-inventario/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — incluye la revisión manual de A-35 (Decisión 4)
├── data-model.md         # Fase 1 (/speckit-plan)
├── quickstart.md         # Fase 1 (/speckit-plan)
├── contracts/
│   └── module-api.md    # Fase 1 (/speckit-plan) — contrato de la API pública del paquete
└── tasks.md              # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorio `../pos-backend`, sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en el repositorio sibling
`pos-backend` (`../pos-backend` relativo a `pos-specs`, según la Constitución §Alcance). Rutas
listadas relativas a la raíz de `pos-backend`.

```text
app/
├── inventory_engine/                # NUEVO — paquete de destino de esta extracción
│   ├── __init__.py                  # API pública: reexporta lock_items, record_movement,
│   │                                 # apply_adjustment
│   └── stock.py                     # Las tres funciones, movidas sin dividir núcleo/adaptador
│                                     # (las tres reciben db: Session — no hay función pura que
│                                     # aislar, a diferencia de catalog_engine/core.py)
│
├── api/v1/inventory/
│   ├── stock.py                     # Historia 3: pasa a ser fachada — reexporta las tres
│   │                                 # funciones desde app.inventory_engine
│   ├── router.py                    # Sin cambios (solo consumidor, importa apply_adjustment)
│   ├── schemas.py                   # Sin cambios
│   └── service.py                   # Único cambio permitido: ajuste mínimo de import de
│                                     # record_movement si apunta directo a app.inventory_engine
│                                     # en vez de vía la fachada (FR-008) — ningún otro cambio
│
├── api/v1/sales/consumption.py      # Consumidor — SIN CAMBIOS (importa lock_items,
│                                     # record_movement)
├── api/v1/orders/consumption.py     # Consumidor — SIN CAMBIOS (importa lock_items,
│                                     # record_movement)
│
└── characterization_tests/
    ├── test_inventory_stock.py                  # Historia 1: se le apunta a
    │                                             # app.inventory_engine, sin modificar ninguna
    │                                             # aserción (FR-007)
    ├── fixtures.py                               # Reutilizado tal cual por la Historia 2
    │                                             # (f.make_inventory_item, f.new_session) —
    │                                             # sin fixture nuevo
    └── inventory_engine_equivalence_gate.py      # NUEVO, Historia 2 — generador determinista
                                                    # (semilla fija, 100-200 casos) + comparador
                                                    # legado-vs-app.inventory_engine, mismo patrón
                                                    # que catalog_engine_equivalence_gate.py de la
                                                    # spec 014. Gate temporal: se retira/archiva
                                                    # tras verificar la Historia 3
```

**Structure Decision**: paquete nuevo `app/inventory_engine/` al mismo nivel que `app/models/`,
`app/core/` y `app/catalog_engine/`, tal como anticipa la sección Assumptions de spec.md (verificado
sin colisión: `find app -iname "*inventory_engine*"` no devuelve nada hoy). A diferencia de
`catalog_engine/`, que separa `core.py` (sin `sqlalchemy`) de `pricing.py`/`consumption.py` (con
`db: Session`) porque la spec 014 exigía esa frontera verificable por inspección estática, esta spec
**no tiene** un requisito equivalente — las tres funciones de `stock.py` son impuras hoy y lo siguen
siendo (FR-002 exige preservar `.with_for_update()`, no aislarlo). Introducir una división
núcleo/adaptador aquí sería una abstracción no solicitada por ningún requisito (regla de "no
diseñar para lo hipotético"), así que el paquete queda con un único módulo `stock.py` que conserva
el nombre y el contenido del fichero legado, más un `__init__.py` de reexport. El fichero original
`app/api/v1/inventory/stock.py` no se elimina: sobrevive como fachada (Historia 3, FR-005) porque
`router.py` y los tres ficheros consumidores restantes siguen importando de esa ruta.

## Complexity Tracking

*Sin violaciones de la Constitution Check — tabla vacía, no aplica.*
