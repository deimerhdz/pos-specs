# Implementation Plan: Eliminación de Carritos al Liberarse la Mesa

**Branch**: `039-eliminacion-carritos-cierre-mesa` | **Date**: 2026-08-26 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/039-eliminacion-carritos-cierre-mesa/spec.md`

## Summary

Los cinco caminos que hoy liberan una `DiningTable` (`try_release_if_empty`, `close_session`,
`release_paid_session`, `release_table`, `_sweep_schema` — todos citados en spec.md "Naturaleza de
esta spec") convergen, sin excepción, en una única función compartida:
`checkout.close_table_sessions` (`app/api/v1/orders/checkout.py:664-693`), que a su vez llama a
`close_participants` (`checkout.py:635-663`) — la que hoy solo marca `Cart.status = 'abandonado'`
sin borrar nada. `close_table_sessions` ya devuelve `list[TableSession]` con las sesiones que
efectivamente cerró en esa llamada.

Esta spec introduce una única función nueva, `checkout.delete_orphan_carts(db, sessions)`
(research.md Decisión 1), que hace un `DELETE` masivo de `Cart` por `participant_id` de esas
sesiones, apoyándose en el `ondelete="CASCADE"` que `cart_items.cart_id`/`cart_item_options.
cart_item_id` ya declaran (mismo mecanismo que la spec 038 documentó para su borrado físico —
research.md Decisión 2). No se llama desde dentro de `close_table_sessions`: se invoca, en cada uno
de los 5 call-sites, exactamente en el punto donde ya se sabe que la mesa quedó `libre` — nunca
antes de saberlo (research.md Decisión 1). Esto es obligatorio para el único camino donde la mesa
podría NO quedar libre después de cerrar las sesiones (`_sweep_schema`, RN-SCHED-04): decidir ahí
adentro de `close_table_sessions` violaría FR-003.

Sin migración de esquema (Principio VIII no aplica — ver [data-model.md](./data-model.md)). Sin
dependencia nueva (Principio IX no aplica). El único test `"CONGELA comportamiento actual:"`
afectado, ya identificado en spec.md, se actualiza citando esta spec; se agregan tests nuevos por
historia de usuario, incluyendo — por primera vez — cobertura `unittest` automatizada de
`_sweep_schema` (research.md Decisión 8), que hoy solo tiene un script manual
(`app/scripts/test_table_release.py`, fuera de la suite de `characterization_tests`).

## Technical Context

**Language/Version**: Python 3.12 (imagen Docker) / 3.14 (venv local `pos-backend/env`). Sin
frontend involucrado — esta spec es puramente de backend (ningún endpoint cambia de forma, request
o response; solo cambia un efecto secundario ya interno a esos endpoints).

**Primary Dependencies**: FastAPI 0.136.3, SQLAlchemy 2.0.50 (sync, `Mapped`/`mapped_column`,
sentencias Core `select`/`delete`), Alembic 1.18.4 (sin migración nueva en esta spec). **Ninguna
dependencia nueva** — todo lo que se necesita (`sqlalchemy.delete`, `ondelete="CASCADE"` ya
declarado) ya existe en el proyecto.

**Storage**: PostgreSQL 16, schema por tenant (`{"schema": "tenant"}` en `Cart`, `CartItem`,
`CartItemOption`, `SessionParticipant`, `TableSession`, `DiningTable`). **Sin migración**: el
borrado es comportamiento de aplicación (`DELETE` en tiempo de ejecución), no un cambio de esquema
— `cart_items.cart_id` y `cart_item_options.cart_item_id` ya tienen `ondelete="CASCADE"` desde antes
de esta spec (verificado en `app/models/cart_item.py:19,58`, mismo mecanismo que usó la spec 038).

**Testing**: `unittest` vía `python -m unittest` (sin pytest, sin `conftest.py`). DB de
characterization tests: SQLite en memoria. Se reescribe, citando esta spec, el único test `CONGELA`
identificado en `spec.md` (`test_leave_session_cierra_participante_abandona_carrito_y_libera_mesa`,
`app/characterization_tests/test_cart_service.py:466-481`). Se agregan tests nuevos en
`test_table_sessions_service.py` (US1 escenarios 2-3, US2 escenarios 1-2, US4), en
`test_orders_checkout.py` (US2 escenario 2, sobre `release_table`), y en un fichero nuevo
`app/characterization_tests/test_scheduler.py` (US2 escenario 3, US3 escenarios 1-2 — primera
cobertura `unittest` de `_sweep_schema`, research.md Decisión 8). `table_sessions_fixtures.py` gana
un helper `make_cart` (espejo del que ya existe en `cart_fixtures.py`, research.md Decisión 9) —
las pruebas nuevas de esta spec son su único consumidor.

**Target Platform**: Linux server (`pos-backend` en producción). Sin cambio en `pos-heladeria`: no
hay contrato HTTP ni payload que el frontend consuma distinto (ver [contracts/](./contracts/)).

**Project Type**: Web application (backend FastAPI). Esta spec toca un único repositorio,
`../pos-backend` (sibling de `pos-specs`).

**Performance Goals**: Sin objetivo nuevo. El `DELETE` masivo agregado corre una sola vez por
liberación de mesa, sobre a lo sumo unas pocas filas de `Cart` (una por comensal de la sesión) —
orden de magnitud despreciable frente a las consultas de cierre/cobro que cada camino ya ejecuta.

**Constraints**:
- FR-002 exige que el borrado ocurra en la misma transacción que libera la mesa: en los 5
  call-sites, `delete_orphan_carts` se invoca **antes** del único `db.commit()`/`db.flush()` final
  de esa operación, nunca en una transacción separada (research.md Decisión 1).
- FR-003 exige que, si la mesa no queda `libre`, no se borre ningún carrito: el único camino donde
  `close_table_sessions` puede ejecutarse sin que la mesa termine `libre` es `_sweep_schema`
  (RN-SCHED-04) — ahí `delete_orphan_carts` solo se llama dentro de la rama `if quedo_libre:`.
- FR-004 exige aislamiento por sesión/mesa/tenant: `delete_orphan_carts` recibe exactamente la
  lista de `TableSession` que `close_table_sessions` devolvió para **esa** mesa — nunca una consulta
  abierta por tenant completo.
- FR-005 prohíbe tocar las condiciones de liberación: ningún `if`/`return` existente de los 5
  caminos se modifica; solo se agrega una llamada nueva en la rama donde la mesa ya queda `libre`.

**Scale/Scope**: 1 función nueva (`checkout.delete_orphan_carts`), 5 puntos de llamada modificados
(uno por camino de liberación, ver Project Structure), 1 característica de import ajustada en
`scheduler.py` (agregar el nombre a un `import` ya existente), 1 test `CONGELA` reescrito, 1 fichero
de test nuevo (`test_scheduler.py`), 1 helper de fixture nuevo (`make_cart` en
`table_sessions_fixtures.py`), tests nuevos por historia de usuario en 3 ficheros existentes. Cero
migraciones, cero endpoints nuevos, cero cambios de esquema.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principio | Evaluación | Estado |
|---|---|---|
| **I. Las Nuevas Funcionalidades Nacen de un Spec** | `spec.md` existe, aprobado, con 4 historias priorizadas, 5 FRs y checklist de calidad (16/16) completado antes de este plan. | PASS |
| **II. El Comportamiento Existente Sigue Protegido** | El cambio de comportamiento (borrado físico de `Cart` en vez de dejarlo `abandonado`/`confirmado` huérfano al liberar mesa) está explícitamente documentado en `spec.md` §"Naturaleza de esta spec", con la decisión de negocio tomada ahí mismo ("quien encarga esta spec... ejerciendo el rol de negocio", Principio XI). **Pendiente**: el registro formal en `registro-de-anomalias.md` como entrada **A-54** — `spec.md` §"Out of Scope" y §"Naturaleza de esta spec" ya lo declaran explícitamente como paso previo a implementar, no a planear; la última entrada hoy es A-53 (spec 038). Este plan no lo crea (no es un artefacto de `/speckit-plan`), pero deja constancia: **la implementación (`/speckit-implement`) no debe comenzar hasta que exista A-54**, tal como exige el Principio II sin condicional. | PASS (planeación) / **BLOQUEO PENDIENTE para implementar** |
| **III. Los Characterization Tests Protegen el Comportamiento Heredado** | El único test `CONGELA` afectado (`test_leave_session_cierra_participante_abandona_carrito_y_libera_mesa`, `test_cart_service.py:466-481`) se reescribe citando esta spec, seguido el mismo patrón que usó la spec 038 (research.md Decisión 6). Los otros tres tests `CONGELA` que spec.md examina explícitamente (`test_close_participants_cierra_activos_y_devuelve_conteo`, `test_close_table_sessions_no_valida_pendientes_rn_ord_31`, `test_release_table_409_con_ordenes_activas_y_libera_sin_ellas`) se verifican en Fase 0 (research.md Decisión 7): ninguno crea un escenario donde la mesa quede `libre` con un carrito de por medio, así que siguen protegiendo exactamente lo que protegen hoy sin necesitar cambio. | PASS |
| **IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento** | El borrado físico al liberar mesa es comportamiento nuevo definido por `spec.md` (FR-001) — no se exige equivalencia con el pasado en ese punto, solo conformidad con la spec y ausencia de regresión en el resto de los 5 caminos de liberación. | PASS |
| **V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas** | El único código "extra" que este plan agrega — el helper `make_cart` en `table_sessions_fixtures.py` (research.md Decisión 9) — es infraestructura de test que las nuevas pruebas de esta spec necesitan directamente, no una limpieza no relacionada. No se toca `app/scripts/test_table_release.py` (el script manual existente) ni se lo migra a `unittest`: crear `test_scheduler.py` es cobertura nueva exigida por US2/US3, no una migración de ese script (research.md Decisión 8). | PASS |
| **VI. Evolución Incremental** | El alcance se divide igual que las historias de `spec.md`: US1 (camino automático, `try_release_if_empty`), US2 (los tres caminos de staff/barrido), US3 (garantía negativa, sin cambio de condiciones), US4 (aislamiento entre mesas). Cada punto de llamada es un cambio de 2-3 líneas aislado; no se mezcla con ningún cambio de arquitectura o migración de datos (no hay ninguna en esta spec). | PASS |
| **VII. Compatibilidad con Datos Históricos** | No se toca `Sale`/`Payment`/`SaleInvoice` ni ninguna venta o factura ya emitida. Los carritos huérfanos que ya existen hoy en mesas ya `libre` (de antes de desplegar esta spec) no se purgan retroactivamente (spec.md "Out of Scope") — esta spec solo actúa hacia adelante. | PASS |
| **VIII. Evolución del Modelo de Datos** | No aplica: cero cambios de esquema. `data-model.md` lo deja explícito por entidad (`Cart`/`CartItem`/`CartItemOption` sin cambio de columnas ni restricciones) y documenta que el mecanismo de cascada (`ondelete="CASCADE"`) que este borrado explota ya existía antes de esta spec. | PASS (no aplica) |
| **IX. Dependencias Nuevas Permitidas con Justificación** | No se añade ninguna dependencia (Technical Context). | PASS (no aplica) |
| **X. Verificación Obligatoria** | Cada historia de usuario de `spec.md` tiene su "Independent Test"; `quickstart.md` los traduce a comandos `unittest` ejecutables. Se agrega, por primera vez, cobertura automatizada de `_sweep_schema` (antes solo un script manual) — sin esa cobertura, US2 (escenario 3) y US3 (escenarios 1-2) no tendrían forma de verificarse en la suite. | PASS |
| **XI. Decisiones de Negocio Frente a Decisiones Técnicas** | La decisión de negocio (borrar físicamente en vez de conservar huérfano) está en `spec.md`. Cómo implementarla — un único helper compartido en vez de duplicar el `DELETE` cinco veces, invocarlo en el call-site y no dentro de `close_table_sessions`, usar `DELETE` masivo en vez de cascada ORM fila a fila — son decisiones técnicas documentadas en research.md, cada una con su alternativa descartada. | PASS |
| **XII. Trazabilidad** | Cadena completa: `spec.md` (Necesidad+Spec+Decisión, citando los 5 caminos y el test CONGELA real) → este `plan.md`/`research.md` (decisión técnica del punto único de enganche) → `tasks.md` (Fase 2, no generado por este comando) → implementación → test CONGELA reescrito + tests nuevos + verificación explícita de no-regresión sobre los otros 3 CONGELA de cierre de mesa → `quickstart.md` (Verificación). Pendiente de cerrar antes de implementar: entrada A-54 en `registro-de-anomalias.md` (ver Principio II). | PASS (planeación) |
| **XIII. Todo en Español de Colombia** | Este plan y todos sus artefactos (research.md, data-model.md, contracts/, quickstart.md) se escriben en español de Colombia, igual que `spec.md`. | PASS |

Sin violaciones que requieran justificación en Complexity Tracking. El único ítem abierto es
administrativo (registro A-54), no una violación de diseño — no bloquea este plan, sí debe
resolverse antes de `/speckit-implement` (Principio II, sin condicional).

## Project Structure

### Documentation (this feature)

```text
specs/039-eliminacion-carritos-cierre-mesa/
├── plan.md              # Este fichero (/speckit-plan)
├── research.md          # Fase 0 (/speckit-plan) — decisiones técnicas y alternativas descartadas
├── data-model.md         # Fase 1 (/speckit-plan) — sin cambio de esquema, ciclo de vida de Cart
├── quickstart.md          # Fase 1 (/speckit-plan) — validación ejecutable por historia de usuario
├── contracts/              # Fase 1 (/speckit-plan) — efecto secundario sobre endpoints existentes
│   └── liberacion-mesa.md
├── checklists/
│   └── requirements.md   # Ya existente, 16/16
└── tasks.md                # Fase 2 (/speckit-tasks — NO se crea en este comando)
```

### Source Code (repositorio sibling de `pos-specs`)

Esta spec vive en `pos-specs`, pero el código que describe está en `../pos-backend` (Constitución
§Alcance). Rutas relativas a la raíz de ese repo. Ningún cambio en `../pos-heladeria` (ver
[contracts/liberacion-mesa.md](./contracts/liberacion-mesa.md)).

```text
# pos-backend
app/
├── api/v1/orders/
│   └── checkout.py                   # MODIFICADO — nueva función `delete_orphan_carts(db,
│                                        sessions)` (research.md D1/D2), junto a
│                                        `close_participants`/`close_table_sessions` (líneas
│                                        635-693 hoy: `close_participants` 635-663,
│                                        `close_table_sessions` 664-693). `release_table`
│                                        (líneas 697-736) la invoca
│                                        tras `close_table_sessions`, dentro del mismo `try`.
│                                        Import nuevo: `delete` de `sqlalchemy` (ya importa
│                                        `select`).
│
├── api/v1/table_sessions/
│   └── service.py                    # MODIFICADO — `try_release_if_empty` (líneas 88-130),
│                                        `close_session` (líneas ~193-260) y
│                                        `release_paid_session` (líneas ~335-398) capturan el
│                                        `list[TableSession]` que ya devuelve
│                                        `checkout.close_table_sessions` y llaman
│                                        `checkout.delete_orphan_carts(db, sessions)` justo donde
│                                        cada una asigna `table.status = "libre"`. Sin import
│                                        nuevo (ya importa `checkout`).
│
├── core/
│   └── scheduler.py                  # MODIFICADO — `_sweep_schema` (líneas 88-165) captura el
│                                        retorno de `close_table_sessions` y llama
│                                        `delete_orphan_carts(db, sessions)` solo dentro de la
│                                        rama `if quedo_libre:` — nunca en la rama RN-SCHED-04
│                                        (pedido huérfano de la misma mesa). Import ajustado:
│                                        `delete_orphan_carts` se agrega al `from
│                                        app.api.v1.orders.checkout import (...)` ya existente.
│
├── models/
│   # cart.py / cart_item.py / session_participant.py / table_session.py / dining_table.py:
│   # SIN CAMBIOS de esquema — ver data-model.md.
│
└── characterization_tests/
    ├── test_cart_service.py          # MODIFICADO — el único test CONGELA que spec.md identifica
    │                                    (líneas 466-481) se reescribe citando esta spec:
    │                                    `db.get(Cart, resp.cart_id)` pasa a `None` en vez de
    │                                    `status == 'abandonado'`.
    ├── test_table_sessions_service.py # MODIFICADO — tests nuevos para US1 (escenarios 2-3,
    │                                    `try_release_if_empty` con carritos huérfanos de
    │                                    comensales ya cerrados / carrito `confirmado`), US2
    │                                    (escenarios 1-2, `close_session`/`release_paid_session`),
    │                                    US3 (escenario 3, rollback ante fallo), US4 (aislamiento
    │                                    entre dos mesas). Usa el `make_cart` nuevo de
    │                                    `table_sessions_fixtures.py`.
    ├── table_sessions_fixtures.py    # MODIFICADO — helper nuevo `make_cart(db, participant,
    │                                    **kw)` (espejo de `cart_fixtures.make_cart`,
    │                                    research.md D9); ningún fixture existente cambia.
    ├── test_orders_checkout.py       # MODIFICADO — test nuevo para US2 (escenario 2, sobre
    │                                    `release_table` con carrito huérfano); se verifica que
    │                                    los 3 tests CONGELA de este fichero (`close_participants`,
    │                                    `close_table_sessions`, `release_table`) siguen en verde
    │                                    sin ninguna edición (research.md D7).
    └── test_scheduler.py             # NUEVO — primera cobertura `unittest` de `_sweep_schema`:
                                         US2 (escenario 3, barrido sin nada por cobrar borra el
                                         carrito huérfano), US3 (escenarios 1-2, barrido con pedido
                                         facturable pendiente o con `CustomerOrder` huérfano de la
                                         misma mesa física NO borra ningún carrito). No reemplaza
                                         `app/scripts/test_table_release.py` (script manual,
                                         fuera de alcance tocarlo, research.md D8).
```

**Structure Decision**: todo el cambio de comportamiento vive detrás de una única función nueva
(`checkout.delete_orphan_carts`) invocada desde exactamente los 5 puntos donde el código ya decide
que la mesa queda `libre` — no se crea ningún módulo/paquete nuevo, no se toca `cart/service.py` (
`try_release_if_empty` ya es el único punto de enganche que sus dos llamadores —`leave_session`,
`cancel_my_order`— necesitan, sin que ellos mismos cambien), ni `core/qr_context.py` ni
`orders/consolidation.py` (verificado en research.md Decisión 4: ambos solo leen `Cart.status ==
'abierto'` de sesiones todavía activas, ajenos al borrado de una sesión ya cerrada). El único
fichero de producción sin test previo que gana cobertura nueva es `core/scheduler.py`, porque es el
único de los 5 caminos sin ningún `unittest` existente que ejercite `_sweep_schema` hoy.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

Sin violaciones — tabla vacía.
