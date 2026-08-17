# Implementation Plan: Red de characterization tests para `cart` (`router.py` + `service.py`)

**Branch**: `015-caracterizacion-cart` | **Date**: 2026-08-17 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/015-caracterizacion-cart/spec.md`

## Summary

Escribir dos ficheros nuevos de characterization tests (`test_cart_service.py`, `test_cart_router.py`)
bajo `app/characterization_tests/` de `pos-backend` que congelen, sin modificar ni una línea de
`app/api/v1/cart/service.py` ni de `app/api/v1/cart/router.py`, el comportamiento hoy observable de
las 11 funciones públicas del service y los 9 endpoints del router — A-08 y A-17 (R16) incluidas — y
sumar esos ficheros al job `test` de `.github/workflows/deploy.yml`. El enfoque técnico tiene tres
piezas: (1) un fixture module nuevo (`cart_fixtures.py`) que amplía el motor SQLite-en-memoria ya
existente en `fixtures.py` con los modelos de mesas/carrito/pedidos/promociones que `cart` necesita,
neutralizando ahí mismo dos índices únicos parciales que SQLite compila como totales (bloquearían
sembrar el estado de A-17 y romperían el flujo normal de "un carrito nuevo tras `submit_cart`"); (2)
para el router, invocar las funciones de endpoint directamente pasándoles un `SessionContext`
construido a mano (sin pasar por `Depends`) y sustituyendo, solo donde el propio endpoint abre el
contexto internamente (`open_session`, `leave`), `open_qr_context`/`open_session_context` por un doble
de prueba — evita depender de Postgres real, que es lo que esas funciones tocan hoy vía
`resolve_tenant_by_id`; (3) para Redis (rate limiting y eventos), aprovechar que ambos caminos ya son
*fail-open* en el código de producción y sustituir el cliente por un doble mínimo en el único
escenario que sí necesita observar el 429. Cambio adicional, no de test: el paso de CI existente
instala solo 4 paquetes (`sqlalchemy pydantic pydantic-settings fastapi`), insuficiente incluso para
los characterization tests que ya existen hoy (necesitan `alembic`, `psycopg`, `bcrypt`, `PyJWT`,
`redis`, ...) — es la causa raíz de A-27 y esta spec la corrige instalando `requirements.txt`
completo (ya congelado en el repo, cero dependencias nuevas).

## Technical Context

**Language/Version**: Python 3.12 (versión del runner de CI, `.github/workflows/deploy.yml`; el
entorno local de `pos-backend` corre 3.14 vía `env/`, ambos compatibles con el código del proyecto)

**Primary Dependencies**: `unittest` (biblioteca estándar) para los tests; `SQLAlchemy` 2.0.50 y
`FastAPI` 0.136.3 ya en uso por el código legado de `cart`; `PyJWT` (vía `app.core.qr_token`, ya
dependencia existente) solo si algún test decide ejercitar el minteo/verificación real del token de
QR — no es obligatorio porque el harness de router sustituye el punto que resuelve el tenant. Cero
dependencias nuevas (Constitución, Principio IV — no aplica justificación porque no se añade nada).

**Storage**: PostgreSQL 16 en producción, sin cambios (esta spec no lo toca); SQLite en memoria vía
un fixture module nuevo, `app/characterization_tests/cart_fixtures.py`, que reutiliza las factories de
catálogo/inventario de `fixtures.py` y añade las de mesas/carrito/pedidos/promociones.

**Testing**: `unittest` vía `python -m unittest discover -s app/characterization_tests -p 'test_*.py'`
(convención ya establecida por el resto de `app/characterization_tests/`); esta spec añade
`test_cart_service.py` y `test_cart_router.py` a ese discover, sin fichero de configuración nuevo.

**Target Platform**: Linux server (`pos-backend`, GitHub Actions `ubuntu-latest` para CI)

**Project Type**: suite de tests interna sobre un servicio backend único (`pos-backend`); no hay
frontend ni mobile involucrados — `pos-heladeria` no se toca en esta spec.

**Performance Goals**: sin objetivo nuevo — el único requisito de rendimiento es que la suite completa
corra en el tiempo típico de un job de CI (segundos, no minutos): SQLite en memoria y dobles de Redis
evitan I/O de red real en el camino feliz de cada test.

**Constraints**: cero líneas modificadas en `cart/service.py` y `cart/router.py` (FR-010, FR-012,
SC-005); cada test determinista — sin reloj real, sin Postgres/Redis real, sin depender del orden de
ejecución (FR-008); ningún test `"CONGELA comportamiento actual:"` se ajusta para que pase, si falla
contra el código actual el defecto está en el test (FR-010, Principio II); el paso de CI nuevo se
suma al de `test_promotions_rules`, no lo reemplaza (FR-009).

**Scale/Scope**: 2 ficheros de test nuevos (11 funciones de service + 9 endpoints de router, con al
menos un caso por unidad más los casos de A-08/A-17 — del orden de 35-45 métodos de test en total);
1 fixture module nuevo (`cart_fixtures.py`, ~250-320 líneas: `new_session()` ampliado + ~12 factories
nuevas + dobles de Redis + helper de reloj fijado); 2 líneas de diff conceptual en
`.github/workflows/deploy.yml` (instalar `requirements.txt` completo, sumar el paso de discover);
cero líneas tocadas en `cart/service.py` y `cart/router.py`.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. El Comportamiento Sigue Siendo Sagrado (por Defecto)** | Esta spec no cambia ningún comportamiento observable del sistema: es exclusivamente construcción de tests sobre código existente. A-08 y A-17 (R16) se documentan tal cual, sin corregirlos (FR-011); no hace falta ninguna entrada nueva en el registro de anomalías porque no hay decisión de negocio que registrar. | PASS |
| **II. Los Characterization Tests son el Árbitro** | Esta spec **crea** los tests que se vuelven el árbitro de una futura extracción de `cart`; no modifica ningún test `"CONGELA comportamiento actual:"` existente. Todo test nuevo lleva ese prefijo (FR-001) y, si falla contra el código actual sin modificar, se corrige el test, nunca el código de producción (FR-010). | PASS |
| **III. Estrangulamiento antes que Reescritura** | Es exactamente el prerrequisito que el Principio III exige antes de que `cart` pueda entrar en una spec de extracción — no hay extracción en esta spec. Verificado que ningún otro módulo de `pos-specs` está en proceso de extracción simultánea (`014-extraccion-motor-catalogo` ya está concluida). | PASS |
| **IV. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia nueva a `requirements.txt`; el cambio de CI instala el fichero ya existente completo, no agrega paquetes. Los dobles de Redis y el reloj fijado se construyen con `unittest.mock` de la biblioteca estándar. | PASS (no aplica) |
| **V. Ningún Cambio Retroactivo** | Esta spec no toca facturación ni datos ya persistidos; construye tests sobre carrito/pedidos en memoria. | PASS (no aplica) |
| **VI. Todo en Español de Colombia** | Esta spec, su plan, los artefactos de diseño (research.md, data-model.md, contracts/, quickstart.md) y los nombres/docstrings de los tests nuevos se escriben en español de Colombia, igual que el resto de `app/characterization_tests/`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/015-caracterizacion-cart/
├── plan.md                      # Este fichero (/speckit-plan)
├── research.md                  # Fase 0 (/speckit-plan)
├── data-model.md                # Fase 1 (/speckit-plan)
├── quickstart.md                # Fase 1 (/speckit-plan)
├── contracts/
│   └── test-harness-api.md      # Fase 1 (/speckit-plan) — contrato del fixture module nuevo
└── tasks.md                     # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorio `../pos-backend`, sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código y los tests que describe están en el repositorio
sibling `pos-backend` (`../pos-backend` relativo a `pos-specs`, según la Constitución §Alcance).
Rutas listadas relativas a la raíz de `pos-backend`.

```text
app/
├── api/v1/cart/
│   ├── router.py                     # SIN CAMBIOS (FR-012, SC-005)
│   ├── service.py                    # SIN CAMBIOS (FR-012, SC-005)
│   └── schemas.py                    # SIN CAMBIOS — solo se leen los tipos de respuesta
│
├── api/v1/{catalog,promotions,orders}/*.py  # Dependencias reales de `cart` — SIN CAMBIOS,
│                                              # se ejercitan tal cual (FR-006, FR-007)
│
└── characterization_tests/
    ├── __init__.py                   # Sin cambios funcionales; puede sumar una línea al
    │                                  # docstring listando el alcance nuevo (no es código de
    │                                  # producción, no lo alcanza SC-005)
    ├── fixtures.py                   # SIN CAMBIOS — se reutiliza tal cual (make_category,
    │                                  # make_product, make_variant, make_option_group,
    │                                  # make_option, link_variant_group, make_recipe_item,
    │                                  # make_inventory_item, make_unit)
    ├── cart_fixtures.py              # NUEVO — Fase 1: new_session() ampliado (mesas, sesiones,
    │                                  # participantes, carritos, pedidos, promociones) +
    │                                  # factories nuevas + FakeRedis (rate limit/eventos) +
    │                                  # frozen_now() + build_session_context()/patched contexts
    │                                  # de router. Ver contracts/test-harness-api.md.
    ├── test_cart_service.py          # NUEVO — Historia 1: 11 funciones públicas de service.py,
    │                                  # incluye los casos A-08 (FR-004) y A-17/R16 (FR-005)
    └── test_cart_router.py           # NUEVO — Historia 2: 9 endpoints de router.py, reutiliza
                                       # cart_fixtures.py

.github/workflows/deploy.yml          # Historia 3 — job `test`: instala requirements.txt
                                       # completo (antes 4 paquetes sueltos) y suma el paso
                                       # `python -m unittest discover -s app/characterization_tests
                                       # -p 'test_*.py'` junto al de test_promotions_rules
                                       # (FR-009, no lo reemplaza)
```

**Structure Decision**: dos ficheros de test (no uno) que reflejan la frontera que la propia spec ya
traza entre Historia 1 (service) e Historia 2 (router) — permite correr cada uno de forma
independiente tal como piden los "Independent Test" de la spec, y separa el fixture de router
(dobles de `open_qr_context`/`open_session_context`/Redis) del de service (que no los necesita en
absoluto). El fixture module nuevo (`cart_fixtures.py`) se mantiene **separado** de `fixtures.py` en
vez de ampliar el existente in-place: `fixtures.py` ya sirve a 8 ficheros de test que no tocan
mesas/carrito/promociones, y la mutación de `Base.metadata` necesaria para neutralizar los dos
índices únicos parciales (ver research.md) es más segura de razonar y de revertir si vive en un
módulo que solo importan los dos ficheros de esta spec, en vez de en el fixture compartido por toda
la red. `fixtures.py` no se modifica; `cart_fixtures.py` importa sus factories reutilizables en vez
de duplicarlas.

## Complexity Tracking

*Sin violaciones de la Constitution Check — tabla vacía, no aplica.*
