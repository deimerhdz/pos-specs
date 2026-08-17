# Implementation Plan: Extracción del motor de catálogo a `app/catalog_engine/`

**Branch**: `014-extraccion-motor-catalogo` | **Date**: 2026-08-17 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/014-extraccion-motor-catalogo/spec.md`

## Summary

Mover las trece funciones (y el dataclass `ConsumptionLine`) hoy repartidas entre
`app/api/v1/catalog/line_pricing.py` (220 líneas) y `consumption_plan.py` (226 líneas) del
repositorio `pos-backend` a un paquete nuevo, `app/catalog_engine/`, separando físicamente el
núcleo puro (`compute_line_price`, `_exige_maximo`) de las once funciones adaptadoras que reciben
`db: Session`. Es una extracción de código por estrangulamiento (Constitución, Principio III), no
una reescritura: cero reglas de negocio cambian, las dos invariantes `[PROTEGIDA]` (A-02, A-05) y
las anomalías documentadas (A-03, A-06, A-32, A-33, A-36 parcial, A-47 parcial) se reproducen tal
cual, y A-04 queda explícitamente fuera. El enfoque técnico tiene tres pasos secuenciales que
corresponden a las tres historias de usuario: (1) crear `app/catalog_engine/` con las trece
funciones verificadas contra los 41 characterization tests y el golden master existentes sin
modificarlos; (2) correr una batería comparativa generada con semilla fija (100-200 casos,
reutilizando el fixture ya existente) como gate único antes de conmutar; (3) convertir
`line_pricing.py` y `consumption_plan.py` en fachadas que reexportan desde `app/catalog_engine/`,
sin tocar ninguno de los siete ficheros consumidores.

## Technical Context

**Language/Version**: Python 3.14 (venv de `pos-backend`, `env/pyvenv.cfg`)

**Primary Dependencies**: FastAPI 0.136 (`HTTPException`, `status`), SQLAlchemy (ORM — `Session`,
`select`) ya en uso por el código legado; ninguna dependencia nueva (Constitución, Principio IV —
no hace falta justificar nada porque no se añade ninguna)

**Storage**: PostgreSQL 16 schema-per-tenant en producción (sin cambios: el motor no gana ni
pierde acceso a datos); SQLite en memoria vía `app/characterization_tests/fixtures.py` para los
tests, reutilizado tal cual (sin fixture nuevo, según clarificación de la Historia 2)

**Testing**: `unittest` vía `python3 -m unittest` (convención existente del proyecto; no hay
`pytest.ini` ni configuración de pytest en el repo) — los 41 characterization tests
(`test_catalog_line_pricing.py`: 25, `test_catalog_consumption_plan.py`: 16), el golden master
(`test_golden_master_pricing_consumption.py` + `golden_master_core.py` +
`pricing_consumption.master.json`), y el nuevo test/script de equivalencia comparativa de la
Historia 2

**Target Platform**: Linux server (`pos-backend`, API FastAPI en producción)

**Project Type**: extracción de módulo interno dentro de un servicio backend único (no hay
frontend/mobile involucrados en esta spec — `pos-heladeria` no tiene puerto propio del motor,
verificado en `specs/000-reconocimiento/consumidores-motor.md` §4.4)

**Performance Goals**: sin objetivo nuevo — el requisito es cero regresión: mismas consultas SQL,
mismo número de round-trips a `db`, ninguna función adaptadora cambia qué o cómo consulta (FR-004)

**Constraints**: cero líneas modificadas en los siete ficheros consumidores hasta la Historia 3
(FR-010, SC-004); el núcleo puro no debe importar `sqlalchemy` ni nada que dependa de él (FR-003,
SC-006); el golden master no se regenera (FR-012); ninguna aserción de los 41 characterization
tests se modifica (FR-011)

**Scale/Scope**: 13 funciones + 1 dataclass (`ConsumptionLine`), ~446 líneas movidas desde 2
ficheros legado a un paquete nuevo de 4 módulos; 7 ficheros consumidores de producción sin tocar;
41 characterization tests + 1 golden master (8-12 casos encadenados) como red de regresión
permanente; 1 batería comparativa temporal de 100-200 casos (Historia 2, se retira tras Historia 3)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. El Comportamiento Sigue Siendo Sagrado (por Defecto)** | La spec no introduce ningún cambio de comportamiento — las dos invariantes `[PROTEGIDA]` (A-02, A-05) y las anomalías `PENDIENTE`/`ACCIDENTAL` documentadas se reproducen tal cual (FR-005, FR-006). A-04 queda explícitamente fuera de esta spec (FR-007) porque corregirlo exigiría una decisión de negocio que esta spec no toma. No se necesita ninguna entrada nueva en el registro de anomalías porque no hay cambio de comportamiento que autorizar. | PASS |
| **II. Los Characterization Tests son el Árbitro** | Los 41 characterization tests y el golden master se ejecutan sin modificar ni una aserción (FR-011, FR-012); cualquier test en rojo bloquea la conmutación y se revierte, nunca se ajusta (FR-014). Ningún test con prefijo `"CONGELA comportamiento actual:"` se toca en esta spec. | PASS |
| **III. Estrangulamiento antes que Reescritura** | Es exactamente un módulo (el motor de catálogo) el que se extrae; verificado contra `specs/` que ningún otro módulo de `pos-specs` está en proceso de extracción/reescritura simultánea (las specs 001-013 son de la fase de documentación ya cerrada). Los siete ficheros consumidores no se tocan (FR-010). | PASS |
| **IV. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia nueva. El generador determinista de la Historia 2 usa `random.Random(seed)` de la biblioteca estándar, no una librería de testing basado en propiedades — no hace falta justificación porque no hay nada que aprobar. | PASS (no aplica) |
| **V. Ningún Cambio Retroactivo** | Esta spec no toca facturación ni datos ya persistidos; el motor de catálogo calcula precio de línea y plan de consumo en el momento de la venta/pedido, no reprocesa histórico. | PASS (no aplica) |
| **VI. Todo en Español de Colombia** | Esta spec, su plan y los artefactos que genera (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que la spec de origen. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/014-extraccion-motor-catalogo/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan)
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
├── catalog_engine/                  # NUEVO — paquete de destino de esta extracción
│   ├── __init__.py                  # API pública: reexporta las 13 funciones + ConsumptionLine
│   ├── core.py                      # Núcleo puro: compute_line_price, _exige_maximo,
│   │                                 # ConsumptionLine (dataclass frozen) — CERO imports de sqlalchemy
│   ├── pricing.py                   # Adaptadores ex-line_pricing.py: load_valid_options,
│   │                                 # check_availability, validate_option_selection,
│   │                                 # grupos_que_descuentan (reciben db: Session)
│   └── consumption.py               # Adaptadores ex-consumption_plan.py: load_recipe,
│                                     # load_variant_groups, plan_line_consumption,
│                                     # required_consumption, group_discounts, variant_label,
│                                     # ensure_lines_consume_inventory (reciben db: Session)
│
├── api/v1/catalog/
│   ├── line_pricing.py              # Historia 3: pasa a ser fachada — reexporta desde
│   │                                 # app.catalog_engine, incluido el reexport de
│   │                                 # ConsumptionLine/load_variant_groups/plan_line_consumption/
│   │                                 # required_consumption que hoy usa cart/service.py:31-36
│   ├── consumption_plan.py          # Historia 3: pasa a ser fachada — reexporta desde
│   │                                 # app.catalog_engine
│   ├── router.py                    # Sin cambios
│   ├── schemas.py                   # Sin cambios
│   └── service.py                   # Sin cambios
│
├── api/v1/{sales,orders,cart}/*.py  # Los 7 ficheros consumidores — SIN CAMBIOS en ninguna historia
│                                     # (sales/service.py, sales/consumption.py, orders/service.py,
│                                     #  orders/consolidation.py, orders/kitchen.py,
│                                     #  orders/consumption.py, cart/service.py)
│
└── characterization_tests/
    ├── test_catalog_line_pricing.py            # Historia 1: se le apunta a app.catalog_engine
    ├── test_catalog_consumption_plan.py        # sin modificar ninguna aserción (FR-011)
    ├── golden_master_core.py                   # Historia 1: se le apunta a app.catalog_engine
    ├── test_golden_master_pricing_consumption.py  # sin regenerar el master (FR-012)
    ├── golden_master/pricing_consumption.master.json  # Sin cambios
    ├── fixtures.py                              # Reutilizado tal cual por la Historia 2 (sin
    │                                             # fixture nuevo, según clarificación de spec.md)
    └── catalog_engine_equivalence_gate.py        # NUEVO, Historia 2 — generador determinista
                                                    # (semilla fija, 100-200 casos) + comparador
                                                    # legado-vs-app.catalog_engine. Gate temporal:
                                                    # se retira/archiva tras verificar la Historia 3
                                                    # (ver Clarifications de spec.md, sesión 2026-08-17)
```

**Structure Decision**: paquete nuevo `app/catalog_engine/` al mismo nivel que `app/models/` y
`app/core/`, tal como anticipa la sección Assumptions de spec.md. Dentro del paquete, la frontera
núcleo/adaptador de FR-003 se traduce en dos ficheros separados (`core.py` sin `sqlalchemy`,
`pricing.py`/`consumption.py` con `db: Session`) en vez de un único módulo plano, para que la
ausencia de imports de `sqlalchemy` en el núcleo (SC-006) sea verificable por inspección estática
de un solo fichero. Se conservan dos ficheros de adaptadores (no uno solo) que preservan el
agrupamiento original `line_pricing` / `consumption_plan`, porque ninguna de las dos historias
exige fusionarlos y mantener el agrupamiento reduce el diff conceptual entre "dónde vivía" y
"dónde vive". El paquete original bajo `app/api/v1/catalog/` no se elimina: `line_pricing.py` y
`consumption_plan.py` sobreviven como fachadas (Historia 3, FR-008) porque `router.py` y
`service.py` de ese mismo directorio, y los siete ficheros consumidores, siguen importando de esa
ruta.

## Complexity Tracking

*Sin violaciones de la Constitution Check — tabla vacía, no aplica.*
