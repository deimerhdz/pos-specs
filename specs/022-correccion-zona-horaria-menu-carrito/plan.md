# Implementation Plan: Corrección de zona horaria en vigencia de promociones del menú y carrito QR (A-08)

**Branch**: `022-correccion-zona-horaria-menu-carrito` | **Date**: 2026-08-18 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/022-correccion-zona-horaria-menu-carrito/spec.md`

## Summary

`_build_menu` (`app/api/v1/menu/router.py:82`) y `serialize_cart` (`app/api/v1/cart/service.py:205`)
hoy calculan `now = datetime.now(timezone.utc).replace(tzinfo=None)` / `now = _now()` (que hace lo
mismo) y se lo pasan a `promotions.active_discount_promotions`. `promotions.local_now()` recibe ese
`datetime` **naive** y lo trata como si YA estuviera en hora local del tenant, en vez de
convertirlo — con `TENANT_TIMEZONE=America/Bogota` (UTC-5), esto reproduce en estos dos puntos el
mismo bug de zona horaria que A-07 corrigió en los cuatro caminos de cobro real
(`checkout.py`/`table_sessions/service.py`/`sales/service.py`, que pasan `datetime.now(
timezone.utc)` **aware**).

Esta spec corrige ambos puntos para que pasen un `datetime` aware (FR-001/FR-002), sin tocar
`local_now`/`active_discount_promotions` en sí (spec 012, dueña de A-07, protegida) ni la función
compartida `_now()` de `cart/service.py` (usada también para `expires_at` de la sesión del
comensal, FR-004 — una anomalía distinta que esta spec explícitamente no toca, Principio III).

## Technical Context

**Language/Version**: Python 3.14 (venv de `pos-backend`, `env/pyvenv.cfg`)

**Primary Dependencies**: FastAPI + SQLAlchemy (ya en uso); `zoneinfo` (stdlib, ya usado por
`promotions.service._tz`) — ningún import nuevo, solo se deja de pasar `.replace(tzinfo=None)` en
dos líneas (Constitución, Principio IV: no aplica justificación porque no se añade nada)

**Storage**: PostgreSQL 16 schema-per-tenant en producción (sin cambio de esquema — el fix es de
qué `datetime` se pasa en tiempo de lectura, no de datos persistidos); SQLite en memoria vía
`app/characterization_tests/fixtures.py`/`cart_fixtures.py` para los tests, con el reloj fijado vía
`cart_fixtures.frozen_now` (ya existe, patchea `datetime.now` del módulo indicado)

**Testing**: `unittest` vía `python -m unittest` (mismo patrón que el resto de
`app/characterization_tests/`). El lado `cart` ya tiene un test `CONGELA` dedicado a A-08
(`test_open_session_y_serialize_cart_a08_zona_horaria_no_aplicada`,
`test_cart_service.py:137-160`, spec 015) que esta spec **modifica** citando A-08 en el commit
(Principio II). El lado `menu` no tiene hoy ningún characterization test — no existe
`test_menu_*.py` en `app/characterization_tests/` (verificado); esta spec crea el primero
(`test_menu_router.py`), acotado a `_build_menu` y la anomalía A-08

**Target Platform**: Linux server (`pos-backend`, API FastAPI en producción)

**Project Type**: corrección puntual dentro de dos módulos backend ya existentes (no hay frontend
involucrado — el menú público y el carrito ya consumen los mismos endpoints; el único cambio
observable es *cuándo*, en el límite horario, una promoción aparece vigente o no)

**Performance Goals**: sin objetivo nuevo — la corrección solo deja de aplicar `.replace(
tzinfo=None)` a un `datetime` ya calculado, sin ninguna operación adicional

**Constraints**: no se toca `promotions.local_now`/`active_discount_promotions`/
`best_line_discount` (`app/api/v1/promotions/service.py`, spec 012, A-07 protegida) — Principio
III, un módulo a la vez; no se toca `_now()` de `cart/service.py` (líneas 52-53) ni su uso en
`open_session` (línea 107, `expires_at`) — FR-004; no se recalcula ninguna promoción, pedido ni
factura ya generados antes del cambio (FR-005)

**Scale/Scope**: 1 línea en `menu/router.py:82` (`_build_menu`) + 1 línea en
`cart/service.py:205` (`serialize_cart`); 1 test existente a modificar con cita de decisión
(`test_cart_service.py:137-160`) + 1 fichero de test nuevo (`test_menu_router.py`) + una pequeña
generalización de `cart_fixtures.frozen_now` para aceptar el módulo a parchear (hoy hardcodeado a
`cart.service`, ver research.md Decisión 2); ningún fichero de `pos-heladeria` ni migración de base
de datos

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. El Comportamiento Sigue Siendo Sagrado (por Defecto)** | El cambio de comportamiento está autorizado por escrito en `registro-de-anomalias.md`, entrada A-08 (ACCIDENTAL por contraste directo con A-07, "tratamiento acordado: corregir en fase de modernización... Decisión de negocio pendiente: ninguna"), y ya estaba anticipado como corrección pendiente en la propia spec 007 (`FR-030`). Citado en `spec.md` §Autorización de negocio. Ningún otro comportamiento cambia por criterio técnico. | PASS |
| **II. Los Characterization Tests son el Árbitro** | `test_open_session_y_serialize_cart_a08_zona_horaria_no_aplicada` (prefijo `"CONGELA comportamiento actual"` en su docstring, `test_cart_service.py:137`) congela hoy el defecto del lado `cart` — se modifica citando A-08 en el commit, como exige el propio Principio II. Se crea además un test nuevo para el lado `menu` (no existía ninguno). Ningún otro test `CONGELA` de `pos-backend` se toca — en particular `app/scripts/test_promotions_rules.py` (único script en CI) no importa `cart` ni `menu`, verificado por sus imports; sigue vigente tal cual. | PASS (modificación autorizada y citada) |
| **III. Estrangulamiento antes que Reescritura** | Dos módulos exactos en juego: `menu.router._build_menu` y `cart.service.serialize_cart`, una línea cada uno — ambos citados literalmente por A-08. `promotions.service` (spec 012) y el resto de `cart.service` (en particular `_now()`/`open_session`, FR-004) quedan explícitamente sin tocar (Out of Scope de `spec.md`). No hay otra extracción/reescritura en curso que se solape. | PASS |
| **IV. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia ni ningún import — `datetime`/`timezone` ya están importados en ambos ficheros; el cambio es eliminar `.replace(tzinfo=None)`, no agregar código. | PASS (no aplica) |
| **V. Ningún Cambio Retroactivo** | FR-005 lo exige explícitamente; `_build_menu`/`serialize_cart` son funciones de lectura sin persistencia de su propio resultado — el cambio solo afecta lo que se muestra **después** del despliegue. Ninguna promoción, pedido ni factura ya generados se toca. | PASS |
| **VI. Todo en Español de Colombia** | Esta spec, este plan y los artefactos que genera (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que las specs de origen (007, 015) y el resto de `pos-specs`. | PASS |

Sin violaciones. La tabla de Complexity Tracking al final de este documento queda vacía.

## Project Structure

### Documentation (this feature)

```text
specs/022-correccion-zona-horaria-menu-carrito/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — por qué no tocar _now(), cómo fijar el reloj en menu
├── data-model.md          # Fase 1 (/speckit-plan)
├── quickstart.md           # Fase 1 (/speckit-plan)
├── contracts/
│   ├── menu-endpoint.md         # Fase 1 (/speckit-plan) — GET /menu, sin cambio de forma
│   └── cart-endpoint.md          # Fase 1 (/speckit-plan) — endpoints que llaman serialize_cart
└── tasks.md               # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorio `../pos-backend`, sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en el repositorio sibling
`pos-backend` (`../pos-backend` relativo a `pos-specs`, según la Constitución §Alcance). Rutas
listadas relativas a la raíz de `pos-backend`.

```text
app/
├── api/v1/menu/
│   └── router.py               # CAMBIA — _build_menu:82 pasa datetime.now(timezone.utc) aware
│                                  # (FR-001); el resto de la función (disponibilidad de opciones,
│                                  # armado de categorías/productos) SIN CAMBIOS
│
├── api/v1/cart/
│   └── service.py                # CAMBIA — serialize_cart:205 pasa datetime.now(timezone.utc)
│                                    # aware, SIN usar _now() en este punto (FR-002); _now() en sí
│                                    # (líneas 52-53) y su uso en open_session:107 (expires_at)
│                                    # SIN CAMBIOS (FR-004)
│
├── api/v1/promotions/service.py    # SIN CAMBIOS — active_discount_promotions/local_now/
│                                      # best_line_discount, contrato ya fijado por la spec 012
│                                      # (A-07 protegida); esta delta solo cambia qué le pasan
│                                      # sus dos callers
│
└── characterization_tests/
    ├── test_cart_service.py         # Se MODIFICA el test existente T012-A08 (cita A-08 en el
    │                                   # commit, Principio II): pasa de congelar el defecto a
    │                                   # congelar el comportamiento corregido
    ├── test_menu_router.py           # NUEVO — primer characterization test de menu/router.py,
    │                                    # acotado a _build_menu y A-08
    └── cart_fixtures.py               # Se GENERALIZA frozen_now para aceptar el módulo a
                                         # parchear (research.md Decisión 2) — comportamiento por
                                         # defecto sin cambio para los tests existentes de cart
```

**Structure Decision**: dos líneas de producción cambiadas en dos ficheros ya existentes
(`menu/router.py:82`, `cart/service.py:205`), sin paquete ni módulo nuevo — no es una extracción
(Principio III no exige "estrangulamiento" para corregir el `datetime` que se le pasa a una función
ya fijada por la spec 012). Los únicos artefactos nuevos son el fichero de test
`test_menu_router.py` (porque hoy no existe ninguna caracterización de `menu`) y la pequeña
generalización de `cart_fixtures.frozen_now` que ambos ficheros de test comparten.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
