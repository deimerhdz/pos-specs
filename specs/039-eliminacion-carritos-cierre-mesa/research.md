# Research: Eliminación de Carritos al Liberarse la Mesa

Fase 0 de `/speckit-plan`. `spec.md` no dejó ningún `[NEEDS CLARIFICATION]` (checklist 16/16), así
que este documento no resuelve ambigüedades del contrato funcional — investiga el código real de
`../pos-backend` para decidir **dónde y cómo** enganchar el comportamiento que `spec.md` ya define,
y deja registro de qué alternativas se descartaron y por qué.

## Decisión 1 — Punto único de enganche: el call-site, no `close_table_sessions`

**Decisión**: no se modifica `close_table_sessions` (`checkout.py:635-663`) ni `close_participants`
(`checkout.py:604-632`) para que borren nada. Se agrega una función nueva y separada,
`delete_orphan_carts(db, sessions)`, invocada explícitamente en cada uno de los 5 call-sites —
`try_release_if_empty`, `close_session`, `release_paid_session`, `release_table`, `_sweep_schema` —
en el punto exacto donde ese call-site ya sabe que la mesa quedó `libre`.

**Por qué**: `close_table_sessions` es hoy el único punto compartido por los 5 caminos (lo llaman
directo o vía `checkout.close_table_sessions`), y ya devuelve `list[TableSession]` — a primera vista,
el lugar obvio para centralizar el borrado. Pero investigar `_sweep_schema`
(`app/core/scheduler.py:88-165`) muestra que **ese orden no es seguro**: ahí `close_table_sessions`
se llama primero (línea ~140), y **después** se decide si la mesa realmente queda `libre`
comprobando si queda algún `CustomerOrder` no terminal huérfano de la misma mesa física sin
`table_session_id` (RN-SCHED-04, líneas ~146-160). Si `delete_orphan_carts` viviera dentro de
`close_table_sessions`, se ejecutaría siempre que hay sesiones que cerrar — incluso en el caso en
que la mesa **no** termina `libre` (spec.md Historia 3, Escenario 2) — violando FR-003 directamente
en el único camino donde esa distinción importa.

`close_participants` tiene el mismo problema por una razón distinta: se llama sola, sin liberar la
mesa, cuando `_sweep_schema` encuentra una sesión vencida **con** algo por cobrar (RN-SCHED-03,
`scheduler.py` rama `if has_billable_orders(...)`) — ese es exactamente el escenario 1 de Historia 3
(spec.md). Poner el borrado ahí también lo dispararía en el caso negativo explícito.

**Alternativas consideradas**:
- *Borrar dentro de `close_table_sessions`*: rechazada por lo anterior — requeriría, además,
  pasarle a `close_table_sessions` una bandera "¿va a quedar libre?" que en `_sweep_schema` todavía
  no se conoce en ese punto de la ejecución, complicando su firma sin necesidad.
- *Borrar dentro de `close_participants`*: rechazada por el mismo motivo — es la función que
  `_sweep_schema` invoca precisamente en el caso en que la mesa **no** se libera.
- *Un listener de SQLAlchemy sobre el evento de cambio de `DiningTable.status`*: rechazada — no
  existe precedente de este patrón en el proyecto, dificulta ver en el código dónde se decide el
  borrado (rompe la trazabilidad que exige el Principio XII), y no aporta nada frente a una llamada
  explícita de una línea en cada call-site, que ya son solo 5 y ya están identificados.

## Decisión 2 — Mecanismo de borrado: `DELETE` masivo, no cascada ORM fila a fila

**Decisión**: `delete_orphan_carts` ejecuta una única sentencia `DELETE` (SQLAlchemy Core,
`db.execute(delete(Cart).where(Cart.participant_id.in_(subquery_participant_ids)))`), no un `for` que
cargue cada `Cart` y llame `db.delete(cart)`.

**Por qué**: `cart_items.cart_id` y `cart_item_options.cart_item_id` ya declaran
`ondelete="CASCADE"` a nivel de base de datos (`app/models/cart_item.py:19,58`) — el mismo mecanismo
que la spec 038 documentó y verificó para su propio borrado físico (`data-model.md` de esa spec).
Ese `ondelete="CASCADE"` es una restricción de la base de datos: se dispara con cualquier `DELETE`
sobre `carts`, sea un `db.delete(obj)` de un objeto cargado por el ORM o una sentencia `DELETE`
masiva — no depende de que SQLAlchemy cargue el objeto en memoria. El `cascade="all, delete-orphan"`
que `Cart.items` declara a nivel de ORM (`app/models/cart.py:31-33`) es el mecanismo que sí exige
cargar el objeto (solo se dispara al hacer `db.delete()` sobre una instancia ya trackeada por la
sesión), pero es redundante aquí precisamente porque el `ondelete="CASCADE"` de la base de datos ya
cubre el mismo resultado sin necesitar cargar nada.

**Alternativas consideradas**:
- *Cargar cada `Cart` de los participantes y llamar `db.delete(cart)` en un `for`*: funcionalmente
  equivalente, pero implica una consulta `SELECT` adicional para traer objetos que ningún otro
  código de esta spec necesita en memoria, y no aprovecha que la clave de selección
  (`participant_id IN (...)`) ya es exactamente lo que un `DELETE ... WHERE ... IN (...)` expresa
  en una sola sentencia. El edge case de spec.md ("dos `TableSession` cerrándose sobre la misma
  mesa en la misma operación") puede implicar varios participantes con varios carritos cada uno —
  el `DELETE` masivo escala igual sin importar cuántos sean, sin loop en Python.
- *`Session.query(Cart).filter(...).delete(synchronize_session=False)` (API 1.x)*: mismo resultado,
  pero el archivo (`checkout.py`) ya usa exclusivamente sentencias Core estilo 2.0 (`select`) — se
  mantiene consistencia de estilo usando `delete()` de `sqlalchemy`, no la API de `Query`.

## Decisión 3 — Ubicación y firma de `delete_orphan_carts`

**Decisión**: la función nueva vive en `app/api/v1/orders/checkout.py`, junto a
`close_participants`/`close_table_sessions`, con la firma:

```python
def delete_orphan_carts(db: Session, sessions: list[TableSession]) -> None:
```

**Por qué**: `checkout.py` ya importa `Cart` (línea 24) y `SessionParticipant`, y ya es donde viven
las otras dos funciones de este mismo flujo de cierre. Recibir `sessions: list[TableSession]` (el
mismo tipo que `close_table_sessions` ya devuelve) en vez de una lista de IDs evita que cada
call-site tenga que re-derivar esa lista por su cuenta — se reusa exactamente lo que
`close_table_sessions` ya calculó, sin una segunda fuente de verdad sobre "qué sesiones se
cerraron". `table_sessions/service.py` ya importa `checkout` como módulo (`from app.api.v1.orders
import checkout`), así que no hace falta ningún import nuevo ahí; `scheduler.py` ya importa
`close_participants, close_table_sessions` desde `checkout` con la sintaxis `from ... import
(...)` — se agrega `delete_orphan_carts` a esa misma tupla.

**Alternativas consideradas**: colocarla en `table_sessions/service.py` — rechazada porque
`release_table` y `_sweep_schema` (los otros 2 call-sites) no importan ese módulo hoy y tendrían
que empezar a hacerlo, mientras que los 3 call-sites de `table_sessions/service.py` ya importan
`checkout`. Colocarla en un módulo nuevo (`app/api/v1/cart/cleanup.py` o similar) — rechazada por
Principio V: no hay ninguna necesidad de un módulo nuevo para una función de ~6 líneas que ya tiene
un hogar natural junto a las funciones que produce su único insumo (`sessions`).

## Decisión 4 — Verificación: ningún consumidor depende de un `Cart` huérfano tras liberar la mesa

**Decisión**: se confirma, leyendo el código (no solo repitiendo la Assumption de `spec.md`), que
ningún consumidor de `Cart` fuera de `app/api/v1/cart/service.py` se ve afectado.

**Evidencia** (búsqueda de todos los usos de `Cart` en `../pos-backend`, excluyendo
`characterization_tests` y el propio módulo `cart`):
- `app/core/qr_context.py:82-89` — filtra `Cart.status == "abierto"` para un `participant` cuyo
  token todavía es válido; un participante de una sesión ya cerrada no puede llegar aquí con un
  token válido (la validación de sesión activa ocurre antes).
- `app/api/v1/orders/checkout.py:657` (dentro de `close_participants`, sin cambios) — filtra
  `Cart.status == "abierto"` de participantes que **está cerrando en esa misma llamada**; corre
  **antes** de que `delete_orphan_carts` se invoque en el call-site (esta spec no reordena
  `close_participants` respecto a sí misma, solo agrega un paso después de que el call-site termina
  de cerrar sesiones).
- `app/api/v1/orders/consolidation.py:110-116` — filtra `Cart.status == "abierto"` de participantes
  de una sesión que el propio flujo de consolidación exige `active` (mesero consolidando una mesa
  en curso); no se ejecuta sobre una sesión ya cerrada.
- `app/scripts/test_table_sessions.py:111` — script manual (`python -m app.scripts...`), no forma
  parte de la suite `unittest`; fuera de alcance modificarlo (no es un characterization test).

Ningún consumidor lee `Cart` por su `status` **después** de que su sesión ya cerró — todos filtran
por `'abierto'` sobre sesiones que asumen activas. Esto confirma que sustituir "el carrito queda
`abandonado`/`confirmado` para siempre" por "el carrito deja de existir" no cambia ningún resultado
observable en esos tres puntos.

## Decisión 5 — Los otros 3 tests `CONGELA` de cierre de mesa no requieren cambio

**Decisión**: se verifica, no se asume, que `test_close_participants_cierra_activos_y_devuelve_
conteo`, `test_close_table_sessions_no_valida_pendientes_rn_ord_31` y
`test_release_table_409_con_ordenes_activas_y_libera_sin_ellas` (los tres en
`test_orders_checkout.py`) siguen protegiendo exactamente lo mismo que protegen hoy.

**Evidencia**:
- `test_close_participants_cierra_activos_y_devuelve_conteo` (líneas 435-456): crea un carrito
  (`cart_ana`) y llama `checkout.close_participants(db, ts)` **directamente**, sin que la mesa se
  libere en el test — la sesión de mesa (`ts`) ni siquiera se cierra en este escenario. Su
  aserción (`cart_ana.status == "abandonado"`) sigue siendo válida porque `close_participants` no
  cambia en esta spec.
- `test_close_table_sessions_no_valida_pendientes_rn_ord_31` (líneas 460-490): no crea ningún
  `Cart` en su fixture — ejercita el aislamiento de validación de `close_table_sessions` frente a
  pedidos pendientes, sin comensal con carrito de por medio. Nada que este cambio pueda afectar.
- `test_release_table_409_con_ordenes_activas_y_libera_sin_ellas` (líneas 494-519): crea una
  `CustomerOrder` pero ningún `SessionParticipant` ni `Cart`. `release_table` sí libera la mesa en
  la segunda mitad del test, pero como no hay ningún carrito de por medio, `delete_orphan_carts`
  no tiene ninguna fila que tocar — el test sigue pasando sin cambios.

**Conclusión**: los 3 se dejan intactos; se verifican en verde como parte de `quickstart.md`, no se
tocan como parte de la implementación.

## Decisión 6 — Cómo se reescribe el único test `CONGELA` afectado

**Decisión**: `test_leave_session_cierra_participante_abandona_carrito_y_libera_mesa`
(`test_cart_service.py:466-481`) se reescribe **con el mismo nombre** (a diferencia de spec 038, que
renombró sus dos tests CONGELA): el nombre ya describe el comportamiento correcto ("abandona
carrito y libera mesa" sigue siendo cierto como paso intermedio — `close_participants` todavía
marca `abandonado` antes de que `delete_orphan_carts` borre la fila); lo único que cambia es la
aserción final, de `self.assertEqual(cart.status, "abandonado")` a
`self.assertIsNone(db.get(Cart, resp.cart_id))`, citando esta spec en el docstring (mismo patrón
que spec 038 usó para citar sus cambios).

**Por qué no renombrar**: spec 038 renombró porque el comportamiento *que el test describe en su
nombre* cambiaba de raíz (de "confirma y dispara evento" a "elimina carrito y abre uno nuevo"). Aquí
el nombre describe correctamente un paso intermedio que **sigue ocurriendo** (`close_participants`
no cambia); solo el estado final de la fila cambia. Cambiar el nombre sin necesidad sería
alcance no pedido (Principio V).

## Decisión 7 — Tests nuevos: dónde viven, uno por historia de usuario

**Decisión**:
- **US1** (escenarios 2-3: dos comensales con carritos en distinto `status`; cancelación del
  último pedido activo) → `test_table_sessions_service.py`, junto a
  `test_try_release_if_empty_libera_y_no_libera` (ya existente, línea 126).
- **US2** (escenario 1: `close_session`; escenario 2 sobre `release_paid_session`) →
  `test_table_sessions_service.py`, junto a `test_close_session_unified_camino_feliz` (260) y
  `test_release_paid_session_libera_la_mesa_cuando_todo_esta_pagado_y_listo` (528).
- **US2** (escenario 2 sobre `release_table`, "Liberar Mesa" de staff) → `test_orders_checkout.py`,
  junto a `test_release_table_409_con_ordenes_activas_y_libera_sin_ellas` (494).
- **US2** (escenario 3, barrido) y **US3** (escenarios 1-2, negativos sobre el barrido) →
  `test_scheduler.py`, fichero nuevo (Decisión 8).
- **US3** (escenario 3, rollback ante fallo genérico) → `test_table_sessions_service.py`, forzando
  una excepción dentro de `close_session` (p. ej. mockeando `_close_unified` para lanzar) y
  verificando que el `Cart` sigue existiendo tras el `rollback()`.
- **US4** (aislamiento entre dos mesas) → `test_table_sessions_service.py`, sobre
  `try_release_if_empty` con dos mesas/sesiones/participantes distintos.

**Por qué**: cada test nuevo se agrega al fichero que ya ejercita la función que ese test cubre —
ninguna historia de usuario introduce un endpoint o módulo nuevo que justifique un fichero de test
adicional, salvo `_sweep_schema`, que no tiene ninguno hoy (Decisión 8).

## Decisión 8 — `test_scheduler.py` nuevo, no migrar el script manual existente

**Decisión**: se crea `app/characterization_tests/test_scheduler.py`, un fichero `unittest.TestCase`
nuevo que ejercita `app.core.scheduler._sweep_schema` directamente contra SQLite en memoria (mismo
patrón que el resto de `characterization_tests`). No se toca `app/scripts/test_table_release.py`
(el script manual que hoy es la única forma de probar este código, ejecutado con `python -m
app.scripts.test_table_release`).

**Por qué**: `_sweep_schema` no tiene, hoy, ningún test `unittest` que corra como parte de la suite
automatizada (`python -m unittest discover`) — es el único de los 5 caminos de liberación sin esa
cobertura. Los escenarios 3 de US2 y 1-2 de US3 de `spec.md` exigen verificar exactamente ese
código (Principio X, Verificación Obligatoria), y no hay manera de hacerlo sin agregar el test. Se
elige un fichero nuevo, no extender uno existente, porque ningún fichero de
`characterization_tests` importa hoy `app.core.scheduler` — no hay un lugar natural donde
"agregarlo junto a lo que ya existe". Migrar `app/scripts/test_table_release.py` a `unittest` sería
una mejora técnica válida por sí misma, pero no la exige ninguna FR de esta spec y mezclarla aquí
violaría el Principio V (Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas) — se dejan
ambos (el script manual y el nuevo `test_scheduler.py`) coexistiendo, cada uno con su propósito.

## Decisión 9 — `make_cart` nuevo en `table_sessions_fixtures.py`

**Decisión**: se agrega un helper `make_cart(db, participant, **kw)` a
`app/characterization_tests/table_sessions_fixtures.py`, como espejo directo del que ya existe en
`app/characterization_tests/cart_fixtures.py:248-258`.

**Por qué**: los tests nuevos de US1, US2, US3 (rollback) y US4 (Decisión 7) necesitan sembrar
carritos sobre participantes usando el módulo de fixtures que `test_table_sessions_service.py` y
`test_orders_checkout.py` ya usan (`table_sessions_fixtures as fx`), que hoy no tiene ningún helper
para crear un `Cart`. Duplicar la lógica de `cart_fixtures.make_cart` en cada test nuevo (`Cart(
participant_id=..., status=...)` + `db.add` + `db.flush`) sería repetir código que el propio
proyecto ya resolvió una vez en el fixture equivalente del otro módulo — se copia esa misma función,
sin alterar `cart_fixtures.py` (que sigue sirviendo a `test_cart_service.py`/`test_cart_router.py`
como hasta ahora).

**Alternativa considerada**: importar `cart_fixtures.make_cart` directamente desde
`table_sessions_fixtures.py` o desde los tests nuevos — rechazada porque `cart_fixtures.make_cart`
construye sobre convenciones propias de ese módulo (su propio `make_participant`, sus propios
valores por defecto) que no coinciden necesariamente con las de `table_sessions_fixtures.py`; los
dos módulos de fixtures ya son independientes entre sí hoy (ninguno importa del otro), y esta spec
no es el lugar para unificarlos (Principio V).
