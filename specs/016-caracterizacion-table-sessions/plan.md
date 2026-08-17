# Implementation Plan: Red de characterization tests para `table_sessions` (`router.py` + `service.py`)

**Branch**: `016-caracterizacion-table-sessions` | **Date**: 2026-08-17 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/016-caracterizacion-table-sessions/spec.md`

## Summary

Escribir tres ficheros nuevos de characterization tests
(`test_table_sessions_split_blindaje.py`, `test_table_sessions_service.py`,
`test_table_sessions_router.py`) bajo `app/characterization_tests/` de `pos-backend` que congelen,
sin modificar ni una línea de `app/api/v1/table_sessions/service.py` ni de
`app/api/v1/table_sessions/router.py`, el comportamiento hoy observable de las 9 funciones
públicas del service y los 7 endpoints del router — A-01, A-15 [PROTEGIDA] (con el mayor número
de casos), A-17 (R12), A-29 y A-38 (RN-MESA-13, RN-MESA-24) incluidas. El enfoque técnico tiene
cuatro piezas: (1) un fixture module nuevo (`table_sessions_fixtures.py`) que amplía el motor
SQLite-en-memoria de `fixtures.py` con las 18 tablas nuevas que `table_sessions` necesita
(mesas/sesión/comensales/pedidos, más la cadena completa de cobro: caja, ventas, pagos, factura,
que `cart` nunca llegó a tocar) y el mismo shim de compilador para `JSONB` sobre SQLite que ya
resolvió `cart_fixtures.py`; (2) para el router, invocar las 7 funciones de endpoint directamente
como funciones Python, pasándoles dobles mínimos (`SimpleNamespace`) de `Tenant`/`User` en vez de
resolver `Depends` — más simple que el harness de `cart` porque `table_sessions/router.py` no abre
ningún contexto propio ni depende de `app.core.qr_context`; (3) para A-17 (R12), como se verifica
en research.md que `with_for_update()` compila a una consulta idéntica sin cláusula sobre SQLite
(no hay lock real que observar), el mecanismo es un espía sobre `service._load` que congela el
argumento `lock=` con el que cada función pública lo invoca, no un intento de reproducir bloqueo
real; (4) para Redis, se intercepta explícitamente `app.core.events.bill_changed` (FR-010a, mismo
patrón que usó `cart` para `order_created`) y se confía en el diseño *fail-open* ya existente de
`app.core.events.publish` para los otros tres eventos que dispara `close_session`
(`payment_completed`, `session_closed`, `table_status_changed`), sin necesitar dobles para ellos.
Ningún cambio de CI: `.github/workflows/deploy.yml` ya instala `requirements.txt` completo y ya
descubre `test_*.py` bajo `app/characterization_tests/` desde `specs/015-caracterizacion-cart/`
(Historia 3 de esta spec es solo verificación, no modificación).

## Technical Context

**Language/Version**: Python 3.12 (versión del runner de CI, `.github/workflows/deploy.yml`; el
entorno local de `pos-backend` corre 3.14 vía `env/`, ambos compatibles con el código del proyecto)

**Primary Dependencies**: `unittest` (biblioteca estándar) para los tests; `SQLAlchemy` 2.0.50 ya
en uso por el código legado de `table_sessions`; `fastapi.HTTPException`/`status` para las
aserciones de código de estado. Cero dependencias nuevas (Constitución, Principio IV — no aplica
justificación porque no se añade nada).

**Storage**: PostgreSQL 16 en producción, sin cambios (esta spec no lo toca); SQLite en memoria
vía un fixture module nuevo, `app/characterization_tests/table_sessions_fixtures.py`, que reutiliza
las factories de catálogo de `fixtures.py` y añade las de
mesas/sesión/comensales/pedidos/promociones/caja/ventas/factura.

**Testing**: `unittest` vía `python -m unittest discover -s app/characterization_tests
-p 'test_*.py'` (convención ya establecida, y ya corriendo en CI desde
`specs/015-caracterizacion-cart/`); esta spec añade tres ficheros nuevos a ese discover sin tocar
el workflow.

**Target Platform**: Linux server (`pos-backend`, GitHub Actions `ubuntu-latest` para CI)

**Project Type**: suite de tests interna sobre un servicio backend único (`pos-backend`); no hay
frontend ni mobile involucrados — `pos-heladeria` no se toca en esta spec.

**Performance Goals**: sin objetivo nuevo — la suite completa debe correr en el tiempo típico de
un job de CI (segundos): SQLite en memoria y el espía de `_load` evitan I/O de red y bloqueo real
en el camino feliz de cada test.

**Constraints**: cero líneas modificadas en `table_sessions/service.py` y `table_sessions/router.py`
(FR-012, FR-014, SC-005); cada test determinista — sin reloj real relevante para el resultado
(los tests de promoción/combo evitan ventanas horarias en vez de fijar el reloj, ver research.md
§5), sin Postgres/Redis real, sin depender del orden de ejecución (FR-010); A-15 recibe, de las
cinco anomalías, el mayor número de casos (mínimo 4, uno por hueco de seguridad — FR-005, SC-002);
ningún test `"CONGELA comportamiento actual:"` se ajusta para que pase, si falla contra el código
actual sin modificar el defecto está en el test (FR-012, Principio II).

**Scale/Scope**: 3 ficheros de test nuevos (Historia 1: A-15 con ≥4 casos sobre `close_session`;
Historia 2: 9 funciones públicas de `service.py`, incluye A-01/A-17(R12)/A-29/A-38; Historia 3: 7
endpoints de `router.py`; del orden de 30-38 métodos de test en total); 1 fixture module nuevo
(`table_sessions_fixtures.py`, ~280-350 líneas: `new_session()` ampliado + ~15 factories nuevas +
espía de `_load` + doble de `events.bill_changed`); cero líneas tocadas en `table_sessions/service.py`
y `router.py`; cero líneas tocadas en `.github/workflows/deploy.yml` (ya cubierto).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. El Comportamiento Sigue Siendo Sagrado (por Defecto)** | Esta spec no cambia ningún comportamiento observable: es exclusivamente construcción de tests sobre código existente. A-01, A-15, A-17 (R12), A-29 y A-38 se documentan tal cual, sin corregirlos (FR-013); no hace falta ninguna entrada nueva en el registro de anomalías porque no hay decisión de negocio que registrar. | PASS |
| **II. Los Characterization Tests son el Árbitro** | Esta spec **crea** los tests que se vuelven el árbitro de una futura extracción de `table_sessions`; no modifica ningún test `"CONGELA comportamiento actual:"` existente (los de `cart`, catálogo, etc. quedan intactos). Todo test nuevo lleva ese prefijo (FR-001) y, si falla contra el código actual sin modificar, se corrige el test, nunca el código de producción (FR-012). | PASS |
| **III. Estrangulamiento antes que Reescritura** | Es el prerrequisito que el Principio III exige antes de que `table_sessions` pueda entrar en una spec de extracción — no hay extracción aquí. Verificado que ningún otro módulo de `pos-specs` está en proceso de extracción simultánea (`014-extraccion-motor-catalogo` ya está concluida; `015-caracterizacion-cart` también). | PASS |
| **IV. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia nueva a `requirements.txt`. El espía de `_load` y el doble de `events.bill_changed` se construyen con `unittest.mock` de la biblioteca estándar. | PASS (no aplica) |
| **V. Ningún Cambio Retroactivo** | Esta spec no toca facturación ni datos ya persistidos; construye tests sobre mesas/sesiones/pedidos/ventas en memoria (SQLite), nunca contra Postgres de producción. | PASS (no aplica) |
| **VI. Todo en Español de Colombia** | Esta spec, su plan, los artefactos de diseño (research.md, data-model.md, contracts/, quickstart.md) y los nombres/docstrings de los tests nuevos se escriben en español de Colombia, igual que el resto de `app/characterization_tests/`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/016-caracterizacion-table-sessions/
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
├── api/v1/table_sessions/
│   ├── router.py                     # SIN CAMBIOS (FR-012, FR-014, SC-005)
│   ├── service.py                    # SIN CAMBIOS (FR-012, FR-014, SC-005)
│   └── schemas.py                    # SIN CAMBIOS — solo se leen los tipos de request/response
│
├── api/v1/{orders,promotions,sales,cart,invoices}/*.py  # Dependencias reales de
│                                              # `table_sessions` — SIN CAMBIOS, se ejercitan tal
│                                              # cual (FR-009); `cart.service.unique_display_label`
│                                              # ya congelada por specs/015-caracterizacion-cart/
│
└── characterization_tests/
    ├── __init__.py                   # Sin cambios funcionales; puede sumar una línea al
    │                                  # docstring listando el alcance nuevo (no es código de
    │                                  # producción, no lo alcanza SC-005)
    ├── fixtures.py                   # SIN CAMBIOS — se reutiliza tal cual (make_category,
    │                                  # make_product, make_variant, make_option_group,
    │                                  # make_option, link_variant_group, make_recipe_item,
    │                                  # make_inventory_item, make_unit)
    ├── cart_fixtures.py               # SIN CAMBIOS — módulo hermano, no se toca ni se importa
    │                                  # (table_sessions_fixtures.py no depende de cart_fixtures.py)
    ├── table_sessions_fixtures.py    # NUEVO — Fase 1: new_session() con el esquema ampliado
    │                                  # (catálogo + mesas/sesión/comensales/pedidos/promociones/
    │                                  # caja/ventas/factura) + factories nuevas + espía de
    │                                  # `_load` (A-17/R12) + doble de `events.bill_changed`
    │                                  # (FR-010a). Ver contracts/test-harness-api.md.
    ├── test_table_sessions_split_blindaje.py  # NUEVO — Historia 1: A-15 [PROTEGIDA] sobre
    │                                  # `close_session` (billing_mode='split'), ≥4 casos
    │                                  # (FR-005, SC-002)
    ├── test_table_sessions_service.py         # NUEVO — Historia 2: 9 funciones públicas de
    │                                  # service.py, incluye A-01 (FR-004), A-17/R12 (FR-006),
    │                                  # A-29 (FR-007), A-38 RN-MESA-13/RN-MESA-24 (FR-008)
    └── test_table_sessions_router.py          # NUEVO — Historia 3: 7 endpoints de router.py,
                                       # reutiliza table_sessions_fixtures.py

.github/workflows/deploy.yml          # SIN CAMBIOS — Historia 4 (FR-011, SC-004) ya satisfecha
                                       # por el paso `python -m unittest discover
                                       # -s app/characterization_tests -p 'test_*.py'` que
                                       # specs/015-caracterizacion-cart/ dejó instalado; esta
                                       # spec solo verifica (quickstart.md, Escenario 5) que los
                                       # tres ficheros nuevos quedan cubiertos por el mismo patrón
                                       # de nombre, sin necesitar un paso adicional.
```

**Structure Decision**: tres ficheros de test (no dos) porque la propia spec numera A-15 como su
propia Historia 1 con "Independent Test" citando explícitamente
`test_table_sessions_split_blindaje` — el invariante [PROTEGIDA] de mayor prioridad se aísla en su
propio fichero, ejecutable solo, en vez de mezclarse entre los demás casos de `close_session` en
Historia 2. El fixture module nuevo (`table_sessions_fixtures.py`) se mantiene separado tanto de
`fixtures.py` como de `cart_fixtures.py`: no depende del harness de router de `cart`
(`SessionContext`/`QrContext`/parches de `open_qr_context`) porque `table_sessions/router.py` no
los usa, y necesita un cierre transitivo de tablas bastante mayor (caja, ventas, pagos, factura)
que `cart` nunca llegó a tocar — mezclarlo en `cart_fixtures.py` obligaría a los tests de `cart` a
cargar con ~8 tablas que no usan. `fixtures.py` no se modifica; `table_sessions_fixtures.py`
importa sus factories de catálogo reutilizables en vez de duplicarlas, igual que hace
`cart_fixtures.py`.

## Complexity Tracking

*Sin violaciones de la Constitution Check — tabla vacía, no aplica.*
