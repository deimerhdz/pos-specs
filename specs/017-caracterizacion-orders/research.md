# Research: Red de characterization tests para `orders`

Todos los "NEEDS CLARIFICATION" del Technical Context se resuelven aquí antes de Fase 1. La única
pregunta de negocio de esta spec ya se resolvió en `spec.md` (Clarifications, sesión 2026-08-17,
sobre el mecanismo para forzar el `IntegrityError` de `move_order`); lo que sigue son decisiones
de implementación explícitamente delegadas a esta fase por las Assumptions de `spec.md`.

## 1. Por qué `orders_fixtures.py` es autónomo (no importa de `cart_fixtures.py` ni de
`table_sessions_fixtures.py`)

**Decisión**: `orders_fixtures.py` es un cuarto fixture module, hermano de los otros tres,
autónomo: importa únicamente las factories de catálogo de `fixtures.py` (igual que los otros
dos) y **replica** —no importa— las factories de mesas/sesión/comensales/pedidos/carrito/
promociones/caja/ventas/factura que ya existen, repartidas, en `cart_fixtures.py`
(carrito, `order_cancel_logs`, `audit_logs`) y `table_sessions_fixtures.py`
(mesas/sesión/comensales/pedidos, promociones, caja/ventas/factura).

**Rationale**: verificado leyendo los tres módulos existentes (`fixtures.py`, 210 líneas;
`cart_fixtures.py`, 472 líneas; `table_sessions_fixtures.py`, 502 líneas) que ninguno de los dos
últimos importa del otro — cada uno registra su propio compilador `@compiles(JSONB, "sqlite")` de
forma independiente y documenta explícitamente esa idempotencia (`table_sessions_fixtures.py:14-
17`). `orders` necesita el superconjunto exacto de ambos (carrito de `cart`, más caja/ventas/
factura/promociones de `table_sessions`, porque `checkout.pay_order` cobra igual que
`close_session`) más una tabla que ninguno de los dos necesitó: `order_item_void_logs`
(`kitchen.void_item`, `OrderItemVoidLog`, `app/models/order_item_void_log.py`). Importar de ambos
crearía la primera dependencia cruzada entre fixture modules hermanos — hoy los tres son
independientes entre sí (cada uno se puede ejecutar solo, como demuestra su bloque
`if __name__ == "__main__":` de test de humo) y encadenarlos rompería esa propiedad sin necesidad:
`orders_fixtures.py` puede replicar el puñado de factories que necesita (existen literalmente:
son las mismas ~15 líneas de cada `make_*` en los dos ficheros existentes) sin ningún costo real
de mantenimiento nuevo, porque las tres son *characterization test infrastructure*, no código de
producción con lógica de negocio que diverja con el tiempo — son constructores de filas.

**Alternatives considered**:
- Importar `orders_fixtures.py` desde `table_sessions_fixtures.py` (el superconjunto más
  parecido) y añadir solo lo que falta (`cart_items`, `order_item_void_logs`): crearía la primera
  dependencia entre fixture modules hermanos del proyecto, y forzaría a cualquier test de
  `orders` a arrastrar el ciclo de importación completo de `table_sessions_fixtures.py`
  (incluyendo su espía `spy_load` y su doble `spy_bill_changed`, que `orders` no usa — `orders`
  no dispara ningún evento de `app.core.events`). Se descarta por acoplamiento innecesario.
- Un cuarto fixture "compartido" (`shop_fixtures.py`) del que los tres (`cart`, `table_sessions`,
  `orders`) importaran, refactorizando los dos existentes: fuera de alcance — tocar
  `cart_fixtures.py`/`table_sessions_fixtures.py` no es necesario para esta spec y arriesga romper
  tests ya congelados de dos specs ya concluidas sin ningún requisito que lo pida. Se descarta.

## 2. A-26 (RN-ORD-60): cómo forzar el `IntegrityError` huérfano de `move_order`

**Decisión** (ya fijada en Clarifications de `spec.md`, sesión 2026-08-17): dentro del bloque
`with`, se parchea `sqlalchemy.orm.Session.flush` con
`unittest.mock.patch.object(db, "flush", side_effect=IntegrityError("stmt", {}, Exception("...")))`
—un parche de **instancia**, no de clase, para no afectar otros `db.flush()` que la propia sesión
de test necesite fuera del bloque `with move_order(...)`— envolviendo únicamente la llamada a
`tables_advanced.move_order`, y se verifica que la función sigue respondiendo `409` con el mensaje
`"La mesa destino ya tiene una orden abierta"` (`tables_advanced.py:59-63`).

**Rationale**: se confirmó leyendo `app/api/v1/orders/tables_advanced.py:56-63` que el único
`db.flush()` dentro del `try/except IntegrityError` de `move_order` es la línea que persiste el
cambio de `order.dining_table_id`/`target.status` — no hay ningún índice único parcial en el
modelo actual (`DiningTable`, `CustomerOrder`) que colisione con esa escritura: la única
restricción de unicidad relacionada, `idx_active_session_per_table` (sobre `table_sessions`, no
sobre `customer_orders`/`dining_tables`), no la toca este `flush`. Forzar el error a nivel de
`Session.flush` (en vez de intentar sembrar datos reales que la disparen, que no existen) es la
única forma de ejercitar el bloque `except IntegrityError` sin falsificar qué es lo que se
congela: que el manejador **sigue presente y traduce a 409**, no que la restricción de base de
datos exista. Es el mismo espíritu que ya usó `table_sessions` para A-17 (R12, espiar en vez de
reproducir un efecto que SQLite no puede dar), pero aquí el objetivo es forzar una excepción, no
solo observar un argumento — de ahí `side_effect`, no `wraps`.

**Alternatives considered**:
- Sembrar dos `CustomerOrder` en la misma mesa destino y esperar que la propia inserción falle:
  no hay ningún índice único parcial hoy en el modelo que lo dispare (verificado, `git log` del
  hallazgo original en el registro de anomalías cita justamente que el índice se retiró) — el
  test fallaría siempre en rojo contra el código actual sin modificar, violando FR-015. Se
  descarta.
- Parchear el método de clase `sqlalchemy.orm.Session.flush` globalmente (no de instancia): más
  amplio de lo necesario y arriesga interferir con cualquier `flush()` interno de SQLAlchemy que
  el propio `commit()`/`refresh()` del test dispare fuera del bloque bajo prueba. Se descarta por
  alcance innecesario, ya resuelto por `patch.object(db, "flush", ...)` sobre la instancia
  concreta de sesión del test.

## 3. El no determinismo de `merge_orders` (RN-ORD-63/A-26): cómo congelarlo sin fijar un valor

**Decisión**: el test siembra dos `CustomerOrder` con `merged_group_id` distintos y preexistentes
(dos grupos de fusión previos, cada uno con al menos otra orden ya asociada para que el `SELECT`
sin `ORDER BY` tenga dos candidatos reales), invoca `merge_orders` una sola vez con ambas, y
afirma `resultado["merged_group_id"] in {group_a, group_b}` — nunca `== group_a` ni `== group_b`
en concreto.

**Rationale**: `merge_orders` (`tables_advanced.py:85`) resuelve el grupo ganador con
`next((o.merged_group_id for o in orders if o.merged_group_id), uuid.uuid4())` sobre `orders`, la
lista que devuelve `select(CustomerOrder).where(CustomerOrder.id.in_(order_ids))` — sin
`ORDER BY`. SQLite en memoria, dentro de una misma ejecución de proceso y sin cambios de
esquema/datos entre corridas, tiende a devolver las filas en un orden estable (orden de inserción
física), lo que en la práctica haría que el test observara siempre el mismo resultado en este
motor concreto — pero ese orden estable es un accidente de implementación de SQLite, no un
contrato del código (Postgres, el motor real de producción, no lo garantiza sin `ORDER BY`
explícito). Afirmar `in {group_a, group_b}` en vez de un valor fijo es lo que hace que el test
documente la propiedad real ("no determinista, cualquiera de los dos es válido") sin acoplarse al
comportamiento incidental de SQLite ni arriesgar un test frágil si una versión futura de SQLAlchemy
o SQLite cambia ese orden interno — ambos casos son "el test pasa" bajo esta aserción, que es
exactamente lo que exige FR-013 (estable en cada corrida, aceptando cualquiera de los resultados
válidos).

**Alternatives considered**:
- Forzar un orden observable con `mock.patch` sobre la consulta interna para que el test sea
  "determinista" en qué grupo gana: falsificaría el hallazgo — RN-ORD-63 es justamente que el
  código de producción **no** fija ningún orden, y mockear la consulta ocultaría eso en vez de
  congelarlo. Se descarta.
- Repetir la llamada muchas veces dentro del mismo test esperando observar ambos resultados en
  distintas corridas para "demostrar" el no determinismo: no es necesario y sería un test lento y
  potencialmente inestable (SQLite puede no reordenar nunca dentro del mismo proceso); la
  aserción de conjunto ya documenta la propiedad sin depender de observarla dos veces. Se
  descarta.

## 4. `checkout.pay_order` contra `build_sale`/`ensure_open_shift` reales: alcance de tablas

**Decisión**: `orders_fixtures.py` incluye `cash_registers`, `cash_shifts`, `payment_methods`,
`payments`, `sales`, `sale_items`, `invoices`, `invoice_counters` — el mismo cierre transitivo que
ya resolvió `table_sessions_fixtures.py` para su propio `close_session`, porque
`checkout.pay_order` llama a `sales.builder.build_sale` (`checkout.py:271`), y `build_sale`
**siempre** emite factura internamente vía `from app.api.v1.invoices.service import
issue_for_sale; issue_for_sale(db, sale, ...)` (`app/api/v1/sales/builder.py:169-171`) — no hay
forma de cobrar sin facturar en este código, así que omitir `invoices`/`invoice_counters` del
fixture haría que cualquier test de `pay_order` fallara con un error de tabla inexistente, no con
la aserción de negocio que se quiere observar.

**Rationale**: verificado leyendo `app/api/v1/sales/builder.py` completo — `build_sale` no acepta
ningún flag para saltarse la emisión de factura; es incondicional. `checkout.pay_order` no pasa
`invoice_prefix` explícito (usa el default `""` de `build_sale`), a diferencia de
`table_sessions.close_session` que sí resuelve `tenant.invoice_prefix` — diferencia de
comportamiento real entre los dos caminos de cobro, no un defecto de este fixture: se documenta
tal cual en `test_orders_checkout.py` (el número de factura de una orden de mesa vía `pay_order`
nace siempre sin prefijo), sin corregirlo (fuera de alcance, ninguna de las siete anomalías de
esta spec lo menciona).

**Alternatives considered**:
- Mockear `issue_for_sale` para no necesitar `invoices`/`invoice_counters`: violaría FR-010
  (ejercitar `build_sale` real sin mocks) — `issue_for_sale` es parte del camino real que
  `build_sale` ejecuta siempre, no una dependencia opcional que se pueda recortar. Se descarta.

## 5. Migración de los tres scripts legado (A-27)

**Decisión**: cada uno de los tres scripts (`app/scripts/test_cancel_inventory.py`, 238 líneas;
`app/scripts/test_receta_obligatoria.py`, 238 líneas; `app/scripts/test_session_ttl.py`, 130
líneas) se lee como fuente de casos ya pensados por el equipo, y su escenario relevante para
`orders` se reescribe como método(s) `unittest` dentro del fichero nuevo que corresponde a la
función que ejercita:
- `test_cancel_inventory.py` (política de reversa al cancelar) → casos dentro de
  `test_orders_checkout.py::TestCancelOrder`, verificados contra el código real de
  `checkout.cancel_order` (Historia 2, escenario 5 de `spec.md`).
- `test_receta_obligatoria.py` (guarda de `deduct_order_items` contra variantes sin receta) → un
  caso en cada uno de los tres ficheros cuyos caminos la invocan: `test_orders_service.py`
  (`create_order`), `test_orders_checkout.py` (`confirm_order`), `test_orders_consolidation.py`
  (`add_item_to_table`) — Historias 1, 2 y 5, escenario correspondiente de cada una.
- `test_session_ttl.py` (barrido automático) → solo la porción que invoca
  `checkout.close_table_sessions`/`close_participants` bajo el disparador del barrido
  (`scheduler.py:140`, `closed_by=None`) se migra, dentro de
  `test_orders_checkout.py::TestReleaseTable` (Historia 2, escenario 6). La aritmética de
  `_should_refresh` en `qr_context.py` no se migra (fuera de alcance, ya documentado en Edge
  Cases de `spec.md`).

Ninguno de los tres se ejecuta ya como script aparte (SC-007): no se importa código de
`app/scripts/` desde los tests nuevos, ni se referencia como dependencia de ejecución.

**Rationale**: los tres scripts predatan la convención `unittest` de
`app/characterization_tests/` y no corren en CI (A-27, ya documentado en el registro de
anomalías) — el mismo riesgo que ya resolvieron `cart` y `table_sessions` para sus propios
scripts legado. Reescribir en vez de copiar tal cual asegura que cada caso migrado se verifica
contra el comportamiento real observado hoy (Principio II), no contra una expectativa que el
script legado pudo haber tenido escrita hace tiempo y que ya no coincide con el código actual.

**Alternatives considered**: ninguna — es el mismo patrón ya validado por `cart`
(`015-caracterizacion-cart/`) y `table_sessions` (`016-caracterizacion-table-sessions/`) para sus
propios scripts legado, sin motivo para reabrirlo.

## 6. `with_for_update` en `block_order`/`confirm_order`: por qué esta spec no repite el
mecanismo de espía de A-17 (R12) de `table_sessions`

**Decisión**: `checkout.block_order` (`:74`) y `checkout.confirm_order` (`:320`) usan
`with_for_update()`/`with_for_update(of=CustomerOrder)`, pero ninguna anomalía del registro
asignada a esta spec (A-01, A-04, A-16, A-25, A-26, A-29, A-38) depende de observar ese lock —a
diferencia de A-17 (R12), que sí es una anomalía propia de `table_sessions` ya congelada en la
spec 016. Esta spec no añade ningún espía sobre el lock de `orders`: los tests de `block_order` y
`confirm_order` verifican el contrato observable (transición de status, versión, reversión en
rollback), no el mecanismo de bloqueo en sí.

**Rationale**: FR-002 exige un characterization test por función pública, no una aserción sobre
cada detalle interno de implementación de cada una; ningún Acceptance Scenario de `spec.md`
(Historia 2, escenarios 1 y 4) pide observar el argumento de lock. Añadir un espía sin que ningún
requisito lo exija sería infraestructura sin uso — la misma razón por la que `table_sessions`
descartó replicar `frozen_now` de `cart` "por si acaso" (`016.../research.md` §5).

**Alternatives considered**:
- Replicar el espía `spy_load` de `table_sessions_fixtures.py` para `checkout.block_order`/
  `confirm_order`: código sin ningún test que lo necesite hoy — se descarta por la misma
  disciplina de no añadir infraestructura sin requisito, documentada ya como precedente del
  proyecto.
