# Quickstart: validar Eliminación de Carritos al Liberarse la Mesa

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite firmas ni
columnas ya detalladas en [data-model.md](./data-model.md) y [contracts/](./contracts/) — solo
enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend`, ejecutado desde la raíz de `../pos-backend`
(sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL: los characterization tests usan SQLite en memoria
(`table_sessions_fixtures.py::new_session`, `cart_fixtures.py::new_session`). Sin migraciones que
correr — esta spec no cambia el esquema (`data-model.md`).

## Paso 0 — Confirmar la línea base antes de tocar código

```bash
python -m unittest app.characterization_tests.test_cart_service -v
python -m unittest app.characterization_tests.test_table_sessions_service -v
python -m unittest app.characterization_tests.test_orders_checkout -v
```

**Resultado esperado**: todo en verde, incluido el único test `CONGELA` que esta spec va a
modificar (`test_leave_session_cierra_participante_abandona_carrito_y_libera_mesa`) y los otros
tres que **no** se tocan (`test_close_participants_cierra_activos_y_devuelve_conteo`,
`test_close_table_sessions_no_valida_pendientes_rn_ord_31`,
`test_release_table_409_con_ordenes_activas_y_libera_sin_ellas` — research.md Decisión 5).

## US1 — La mesa se libera sola y sus carritos desaparecen con ella

Fichero: `test_table_sessions_service.py`, junto a `test_try_release_if_empty_libera_y_no_libera`.

1. Mesa con un único comensal, un carrito `'abierto'` sin confirmar → el comensal sale
   (`leave_session`) siendo el último activo sin nada por cobrar → `table.status == 'libre'` y
   `db.get(Cart, cart.id) is None` (Acceptance Scenario 1).
2. Mesa con dos comensales: uno con carrito ya `'abandonado'` (salió antes), otro con un carrito
   `'confirmado'` (el mesero lo consolidó) → el segundo sale siendo el último activo sin nada por
   cobrar → la mesa queda `'libre'` y **ninguno** de los dos `Cart` sigue existiendo (Acceptance
   Scenario 2 — confirma que el borrado no distingue por `status`, edge case de spec.md).
3. Sesión con un único comensal que cancela su último pedido activo (`cancel_my_order`), dejando la
   mesa sin nadie activo y sin nada por cobrar → la mesa se libera y su `Cart` se elimina en la
   misma operación (Acceptance Scenario 3).

```bash
python -m unittest app.characterization_tests.test_cart_service -v
python -m unittest app.characterization_tests.test_table_sessions_service -v
```

**Verificación del test `CONGELA` reescrito**: `test_leave_session_cierra_participante_abandona_
carrito_y_libera_mesa` (`test_cart_service.py`) sigue en verde con su aserción nueva
(`db.get(Cart, resp.cart_id) is None`, research.md Decisión 6) — no con la vieja
(`cart.status == 'abandonado'`).

## US2 — La liberación manual de staff también limpia los carritos

Ficheros: `test_table_sessions_service.py` (escenarios 1-2), `test_orders_checkout.py` (escenario 2
sobre `release_table`), `test_scheduler.py` (escenario 3, nuevo — research.md Decisión 8).

1. Mesa con pedidos facturables y un `Cart` huérfano de un comensal ya cerrado antes del cobro → el
   cajero cobra y cierra (`close_session`) → la mesa queda `'libre'` y ese `Cart` deja de existir.
2. Sesión ya completamente pagada con un `Cart` huérfano → staff la libera sin generar venta nueva
   (`release_paid_session`) → la mesa queda `'libre'` y el `Cart` deja de existir.
2b. Mesa con "Liberar Mesa" (`release_table`, sin órdenes bloqueantes) y un `Cart` huérfano de un
    comensal ya cerrado → la mesa queda `'libre'` y el `Cart` deja de existir; incluye el caso de
    dos `TableSession` activas sobre la misma mesa cerrándose en la misma llamada (edge case) — se
    verifica que se borran los carritos de participantes de **ambas**.
3. Mesa vencida por inactividad sin nada por cobrar y con un `Cart` huérfano → corre el barrido
   (`_sweep_schema`, vía `python -m app.scripts.sweep_sessions` o el test nuevo) → la mesa queda
   `'libre'` y el `Cart` deja de existir.

```bash
python -m unittest app.characterization_tests.test_table_sessions_service -v
python -m unittest app.characterization_tests.test_orders_checkout -v
python -m unittest app.characterization_tests.test_scheduler -v
```

## US3 — Si la mesa no queda libre, ningún carrito se elimina

Fichero: `test_scheduler.py` (escenarios 1-2, primera cobertura automatizada de estas dos ramas —
research.md Decisión 8), `test_table_sessions_service.py` (escenario 3, rollback).

1. Sesión vencida con un pedido `'abierta'` sin cobrar y un `Cart` huérfano de un comensal ya
   cerrado → corre el barrido → solo cierra a los comensales (`close_participants`, sin llamar
   `close_table_sessions`), la mesa sigue `'ocupada'`, el `Cart` huérfano **sigue existiendo** (pasa
   a `'abandonado'` si estaba `'abierto'`, comportamiento sin cambio — RN-SCHED-03).
2. Sesión vencida sin nada por cobrar pero con un `CustomerOrder` no terminal huérfano de la misma
   mesa física (sin `table_session_id`) → corre el barrido → la sesión se cierra
   (`close_table_sessions` sí corre) pero la mesa **no** vuelve a `'libre'` (RN-SCHED-04) →
   `delete_orphan_carts` no se invoca (queda dentro del `if quedo_libre:` que no se cumple) →
   ningún `Cart` de esa sesión se elimina.
3. Forzar un fallo dentro de `close_session` (mockear `_close_unified`/`_close_split` para lanzar
   una excepción) sobre una sesión con un `Cart` huérfano → verificar que, tras el `rollback()`, el
   `Cart` sigue existiendo con el mismo `id` y `status` que tenía antes del intento — ninguna
   eliminación parcial (Acceptance Scenario 3, FR-002).

```bash
python -m unittest app.characterization_tests.test_scheduler -v
python -m unittest app.characterization_tests.test_table_sessions_service -v
```

## US4 — La liberación de una mesa nunca toca los carritos de otra

Fichero: `test_table_sessions_service.py`, sobre `try_release_if_empty` (o `close_session`) con dos
mesas independientes.

1. Dos mesas activas, cada una con su propio `Cart` huérfano de un comensal ya cerrado → se libera
   una de las dos → el `Cart` de la mesa que sigue ocupada permanece exactamente igual (mismo `id`,
   mismo `status`, ninguna fila de más ni de menos).

```bash
python -m unittest app.characterization_tests.test_table_sessions_service -v
```

## Verificación final — no regresión sobre el resto de la suite

```bash
python -m unittest discover -s app/characterization_tests -v
```

**Resultado esperado**: la suite completa pasa, incluidos los 3 tests `CONGELA` de
`test_orders_checkout.py` que esta spec examinó y dejó sin cambios (research.md Decisión 5), y
`test_orders_consolidation.py`/`test_cart_router.py` (ajenos a este cambio, verificados en
research.md Decisión 4 como consumidores de `Cart` que no se ven afectados).

## Antes de dar la spec por completada (Principio II)

Esta implementación **no debe iniciarse** hasta que exista la entrada **A-54** en
`specs/000-reconocimiento/registro-de-anomalias.md` (última entrada hoy: A-53, spec 038) — es un
paso administrativo previo, no una tarea de código, exigido sin condicional por la Constitución
(ver `plan.md` §Constitution Check, Principio II).
