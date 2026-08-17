# Implementation Plan: Red de characterization tests para `orders` (`service.py`, `checkout.py`, `consolidation.py`, `kitchen.py`, `tables_advanced.py`)

**Branch**: `017-caracterizacion-orders` | **Date**: 2026-08-17 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/017-caracterizacion-orders/spec.md`

## Summary

Escribir cinco ficheros nuevos de characterization tests (`test_orders_consolidation.py`,
`test_orders_checkout.py`, `test_orders_kitchen.py`, `test_orders_tables_advanced.py`,
`test_orders_service.py`) bajo `app/characterization_tests/` de `pos-backend` que congelen, sin
modificar ni una línea de `app/api/v1/orders/{service,checkout,consolidation,kitchen,
tables_advanced}.py`, el comportamiento hoy observable de las 23 funciones públicas de esos cinco
ficheros — A-01 (caminos B y C), A-04, A-16, A-25 [PROTEGIDA], A-26, A-29 (parcial) y A-38
(parcial, RN-ORD-31/RN-ORD-32) incluidas. El enfoque técnico tiene cuatro piezas: (1) un fixture
module nuevo, autónomo, `orders_fixtures.py`, que reutiliza las factories de catálogo de
`fixtures.py` (10 tablas) y replica —sin importarlos, mismo patrón de independencia que ya
adoptó `table_sessions_fixtures.py` frente a `cart_fixtures.py`— el superconjunto de tablas y
factories que ya existen repartidas entre `cart_fixtures.py` (carts/cart_items,
order_cancel_logs, audit_logs) y `table_sessions_fixtures.py` (mesas/sesión/comensales/pedidos,
promociones, caja/ventas/factura), más una tabla y una factory nuevas que ninguno de los dos
fixtures existentes necesitó: `order_item_void_logs` (`kitchen.void_item`); (2) para A-26
(RN-ORD-60, el manejador huérfano de `IntegrityError` en `move_order`), un
`unittest.mock.patch("app.api.v1.orders.tables_advanced.Session.flush", side_effect=IntegrityError(...))`
de alcance acotado al bloque `with`, documentado como forzado artificial (Clarifications,
sesión 2026-08-17) porque el índice único que lo disparaba ya no existe en el modelo; (3) para el
no determinismo de `merge_orders` (RN-ORD-63), un test que siembra dos grupos preexistentes,
invoca la función una vez, y asegura que el resultado es **uno de los dos** IDs de grupo válidos
sin fijar cuál — el comportamiento no determinista en sí es lo que se congela, no un valor
concreto; (4) migración de los tres scripts legado (`test_cancel_inventory.py`,
`test_receta_obligatoria.py`, `test_session_ttl.py`, ninguno corriendo hoy en CI, A-27) a métodos
`unittest` dentro de los ficheros nuevos, reescritos contra el código real, no copiados tal cual.
`checkout.pay_order` se ejercita contra `sales.builder.build_sale`/`ensure_open_shift` reales, sin
mocks — incluida la emisión de factura que `build_sale` hace internamente (`issue_for_sale`), la
misma razón por la que `table_sessions_fixtures.py` necesitó `invoices`/`invoice_counters` y este
fixture también los necesita. Ningún cambio de CI: `.github/workflows/deploy.yml` ya instala
`requirements.txt` completo y ya descubre `test_*.py` bajo `app/characterization_tests/` desde
`specs/015-caracterizacion-cart/` (Historia 6 de esta spec es solo verificación, no modificación).

## Technical Context

**Language/Version**: Python 3.12 (versión del runner de CI, `.github/workflows/deploy.yml`; el
entorno local de `pos-backend` corre 3.14 vía `env/`, ambos compatibles con el código del proyecto)

**Primary Dependencies**: `unittest` (biblioteca estándar) para los tests; `SQLAlchemy` 2.0.50 ya
en uso por el código legado de `orders`; `unittest.mock` para el forzado artificial de
`IntegrityError` en `move_order` (A-26/RN-ORD-60) y para nada más — no hace falta ningún espía ni
doble de evento en esta spec, a diferencia de `table_sessions` (`orders` no dispara ningún evento
de `app.core.events`); `fastapi.HTTPException`/`status` para las aserciones de código de estado.
Cero dependencias nuevas (Constitución, Principio IV — no aplica justificación porque no se añade
nada).

**Storage**: PostgreSQL 16 en producción, sin cambios (esta spec no lo toca); SQLite en memoria
vía un fixture module nuevo, `app/characterization_tests/orders_fixtures.py`, que reutiliza las
factories de catálogo de `fixtures.py` y replica (sin importarlas) las factories de
mesas/sesión/comensales/pedidos/carrito/promociones/caja/ventas/factura/auditoría que ya existen,
repartidas, en `cart_fixtures.py` y `table_sessions_fixtures.py`.

**Testing**: `unittest` vía `python -m unittest discover -s app/characterization_tests
-p 'test_*.py'` (convención ya establecida, y ya corriendo en CI desde
`specs/015-caracterizacion-cart/`); esta spec añade cinco ficheros nuevos a ese discover sin tocar
el workflow.

**Target Platform**: Linux server (`pos-backend`, GitHub Actions `ubuntu-latest` para CI)

**Project Type**: suite de tests interna sobre un servicio backend único (`pos-backend`); no hay
frontend ni mobile involucrados — `pos-heladeria` no se toca en esta spec.

**Performance Goals**: sin objetivo nuevo — la suite completa debe correr en el tiempo típico de
un job de CI (segundos): SQLite en memoria evita I/O de red en el camino feliz de cada test; el
forzado de `IntegrityError` en `move_order` es un `mock.patch` puntual, no un retry con espera.

**Constraints**: cero líneas modificadas en `service.py`, `checkout.py`, `consolidation.py`,
`kitchen.py` y `tables_advanced.py` (FR-017, SC-005); cada test determinista — sin reloj real
relevante para el resultado (los fixtures de promoción/combo se siembran sin ventana horaria,
mismo patrón que ya validó `table_sessions_fixtures.make_promotion`), sin Postgres real, sin
depender del orden de ejecución (FR-013), incluyendo el caso de `merge_orders` (no determinista en
el comportamiento observado, pero el test es estable en cada corrida — FR-013); ningún test
`"CONGELA comportamiento actual:"` se ajusta para que pase, si falla contra el código actual sin
modificar el defecto está en el test (FR-015, Principio II); A-04 recibe el mayor cuidado de la
spec, con caso de contraste directo contra `service.create_order` (FR-003).

**Scale/Scope**: 5 ficheros de test nuevos (Historia 1: `consolidation.py`, 5 funciones públicas,
A-04 con el mayor número de casos; Historia 2: `checkout.py`, 10 funciones públicas, la más
grande, incluye A-01(B), A-29, A-38; Historia 3: `kitchen.py`, 3 funciones públicas, A-16, A-25;
Historia 4: `tables_advanced.py`, 4 funciones públicas, A-26 (tres hallazgos), A-01(C); Historia 5:
`service.py`, 1 función pública; del orden de 35-45 métodos de test en total, incluyendo los tres
casos migrados de los scripts legado); 1 fixture module nuevo (`orders_fixtures.py`, del orden de
420-480 líneas: `new_session()` con ~21 tablas nuevas sobre las 10 de catálogo + ~18 factories +
un helper de forzado de `IntegrityError`); cero líneas tocadas en los cinco ficheros de
`orders/`; cero líneas tocadas en `.github/workflows/deploy.yml` (ya cubierto).

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. El Comportamiento Sigue Siendo Sagrado (por Defecto)** | Esta spec no cambia ningún comportamiento observable: es exclusivamente construcción de tests sobre código existente. A-01 (B y C), A-04, A-16, A-25, A-26, A-29 (parcial) y A-38 (parcial) se documentan tal cual, sin corregirlos (FR-016); no hace falta ninguna entrada nueva en el registro de anomalías porque no hay decisión de negocio que registrar — el propio registro ya documentaba las siete antes de esta spec. | PASS |
| **II. Los Characterization Tests son el Árbitro** | Esta spec **crea** los tests que se vuelven el árbitro de una futura extracción de `orders`; no modifica ningún test `"CONGELA comportamiento actual:"` existente (los de `catalog`, `cart`, `table_sessions`, etc. quedan intactos). Todo test nuevo lleva ese prefijo (FR-001) y, si falla contra el código actual sin modificar, se corrige el test, nunca el código de producción (FR-015). | PASS |
| **III. Estrangulamiento antes que Reescritura** | Es el prerrequisito que el Principio III exige antes de que `orders` pueda entrar en una spec de extracción — no hay extracción aquí. Verificado que ningún otro módulo de `pos-specs` está en proceso de reescritura simultánea (`014-extraccion-motor-catalogo`, `015-caracterizacion-cart` y `016-caracterizacion-table-sessions` ya están concluidas y son, todas, caracterización o extracción ya cerrada, no trabajo en curso). | PASS |
| **IV. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia nueva a `requirements.txt`. El forzado de `IntegrityError` en `move_order` (A-26/RN-ORD-60) se construye con `unittest.mock` de la biblioteca estándar. | PASS (no aplica) |
| **V. Ningún Cambio Retroactivo** | Esta spec no toca facturación ni datos ya persistidos; construye tests sobre mesas/sesiones/pedidos/ventas/facturas en memoria (SQLite), nunca contra Postgres de producción. | PASS (no aplica) |
| **VI. Todo en Español de Colombia** | Esta spec, su plan, los artefactos de diseño (research.md, data-model.md, contracts/, quickstart.md) y los nombres/docstrings de los tests nuevos se escriben en español de Colombia, igual que el resto de `app/characterization_tests/`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/017-caracterizacion-orders/
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
├── api/v1/orders/
│   ├── service.py                    # SIN CAMBIOS (FR-017, SC-005)
│   ├── checkout.py                   # SIN CAMBIOS (FR-017, SC-005)
│   ├── consolidation.py              # SIN CAMBIOS (FR-017, SC-005) — incluye A-04, sin fix
│   ├── kitchen.py                    # SIN CAMBIOS (FR-017, SC-005)
│   ├── tables_advanced.py            # SIN CAMBIOS (FR-017, SC-005)
│   ├── consumption.py                # SIN CAMBIOS — se consume como dependencia real
│   │                                  # (deduct_order_items/reverse_order_items), no se
│   │                                  # re-caracteriza (ya cubierta indirectamente por
│   │                                  # specs/014-extraccion-motor-catalogo/)
│   ├── router.py                     # SIN CAMBIOS — fuera de alcance (Assumptions)
│   └── schemas.py                    # SIN CAMBIOS — solo se leen los tipos de request/schema
│
├── api/v1/{sales,promotions,invoices}/*.py  # Dependencias reales de `checkout.pay_order` —
│                                      # SIN CAMBIOS, se ejercitan tal cual (FR-010, FR-011):
│                                      # `sales.builder.build_sale`/`ensure_open_shift`,
│                                      # `promotions.service.evaluate`/
│                                      # `combo_discount_for_lines`/`expand_combo`,
│                                      # `invoices.service.issue_for_sale` (invocado
│                                      # internamente por `build_sale`, sin mock)
│
└── characterization_tests/
    ├── __init__.py                   # Sin cambios funcionales; puede sumar una línea al
    │                                  # docstring listando el alcance nuevo (no es código de
    │                                  # producción, no lo alcanza SC-005)
    ├── fixtures.py                   # SIN CAMBIOS — se reutiliza tal cual (make_category,
    │                                  # make_product, make_variant, make_option_group,
    │                                  # make_option, link_variant_group, make_recipe_item,
    │                                  # make_inventory_item, make_unit)
    ├── cart_fixtures.py              # SIN CAMBIOS — módulo hermano, no se toca ni se importa
    ├── table_sessions_fixtures.py    # SIN CAMBIOS — módulo hermano, no se toca ni se importa
    ├── orders_fixtures.py            # NUEVO — Fase 1: new_session() con el esquema ampliado
    │                                  # (catálogo + mesas/sesión/comensales/pedidos/carrito/
    │                                  # promociones/caja/ventas/factura/order_item_void_logs/
    │                                  # order_cancel_logs/audit_logs) + factories +
    │                                  # forzado de IntegrityError (A-26). Autónomo: no importa
    │                                  # de cart_fixtures.py ni table_sessions_fixtures.py (ver
    │                                  # contracts/test-harness-api.md).
    ├── test_orders_consolidation.py           # NUEVO — Historia 1: 5 funciones públicas de
    │                                  # consolidation.py, A-04 con el mayor número de casos
    │                                  # (FR-003, SC-002)
    ├── test_orders_checkout.py                # NUEVO — Historia 2: 10 funciones públicas de
    │                                  # checkout.py, incluye A-01(B) (FR-004), A-29 (FR-008),
    │                                  # A-38/RN-ORD-31/RN-ORD-32 (FR-009), integración real con
    │                                  # build_sale/ensure_open_shift (FR-010)
    ├── test_orders_kitchen.py                 # NUEVO — Historia 3: 3 funciones públicas de
    │                                  # kitchen.py, A-16 (FR-005), A-25 [PROTEGIDA] (FR-006)
    ├── test_orders_tables_advanced.py         # NUEVO — Historia 4: 4 funciones públicas de
    │                                  # tables_advanced.py, A-26 en sus tres hallazgos (FR-007),
    │                                  # A-01(C) (FR-004)
    └── test_orders_service.py                 # NUEVO — Historia 5: create_order, contraste
                                       # directo con A-04 de consolidation.py (FR-003)

.github/workflows/deploy.yml          # SIN CAMBIOS — Historia 6 (FR-014, SC-004) ya satisfecha
                                       # por el paso `python -m unittest discover
                                       # -s app/characterization_tests -p 'test_*.py'` que
                                       # specs/015-caracterizacion-cart/ dejó instalado; esta
                                       # spec solo verifica (quickstart.md) que los cinco ficheros
                                       # nuevos quedan cubiertos por el mismo patrón de nombre.
```

**Structure Decision**: cinco ficheros de test, uno por fichero de producción en alcance —mismo
mapeo 1:1 que ya siguió `cart` (`015-caracterizacion-cart/`), en vez de una única Historia P1
como `table_sessions` (que aisló A-15 en su propio fichero por ser el invariante [PROTEGIDA] de
mayor prioridad). Aquí no hace falta ese aislamiento adicional: A-04 (el hallazgo central de esta
spec) ya tiene su propio fichero natural (`test_orders_consolidation.py`, Historia 1, P1) porque
vive en `consolidation.py`, sin mezclarse con las otras cuatro anomalías. El fixture module nuevo
(`orders_fixtures.py`) se mantiene autónomo, sin importar de `cart_fixtures.py` ni de
`table_sessions_fixtures.py`, replicando en su lugar el superconjunto de sus tablas/factories:
mezclarlos crearía una dependencia cruzada entre tres fixture modules hermanos que hoy son
independientes (research.md §1), y el propio precedente de `table_sessions_fixtures.py` ya
documentó que la idempotencia del registro `@compiles(JSONB, "sqlite")` permite que varios
fixtures autónomos definan el mismo shim sin conflicto si conviven en el mismo proceso de test.
`fixtures.py` no se modifica; `orders_fixtures.py` importa sus factories de catálogo reutilizables
en vez de duplicarlas, igual que hacen los otros dos.

## Complexity Tracking

*Sin violaciones de la Constitution Check — tabla vacía, no aplica.*
