# Feature Specification: Red de characterization tests para `orders` (`service.py`, `checkout.py`, `consolidation.py`, `kitchen.py`, `tables_advanced.py`)

**Feature Branch**: `017-caracterizacion-orders`

**Created**: 2026-08-17

**Status**: Draft

**Input**: User description: "Construcción de la red de characterization tests para orders
(app/api/v1/orders/{service,checkout,consolidation,kitchen,tables_advanced}.py) de pos-backend,
como prerrequisito de una futura extracción strangler fig. Esta spec no extrae ni reescribe nada:
solo congela, mediante tests, el comportamiento actual de orders tal como existe hoy — anomalías
incluidas, A-04 entre ellas (no se corrige aquí; su fix de una línea en consolidation.py:199 queda
para una spec delta posterior)."

**Naturaleza de esta spec**: **characterization, no extracción** (Principio II de la
[Constitución](../../.specify/memory/constitution.md)). No mueve ni reescribe una sola línea de
`orders/service.py`, `checkout.py`, `consolidation.py`, `kitchen.py` ni `tables_advanced.py`:
construye la red de tests que **congela** su comportamiento observable actual — anomalías
incluidas — como el prerrequisito que el Principio III (Estrangulamiento antes que Reescritura)
exige antes de que `orders` pueda entrar en una spec de extracción, igual que
`specs/015-caracterizacion-cart/` y `specs/016-caracterizacion-table-sessions/` exigieron primero
su propia red antes de que `cart` y `table_sessions` pudieran moverse.

## Contexto — qué existe hoy y qué protege esta spec

`app/api/v1/orders/` tiene ocho ficheros de producción; esta spec cubre cinco, con 23 funciones
públicas en total:

- **`service.py`** (137 líneas, 1 función pública): `create_order` — la comanda directa del
  staff (mostrador/mesero), que nace ya `abierta` (confirmada) y descuenta inventario al crearse,
  porque no vuelve a pasar por `confirm_order`.
- **`checkout.py`** (567 líneas, 10 funciones públicas): `block_order`, `compute_bill`,
  `order_sale_lines`, `promo_lines_for`, `pay_order`, `confirm_order`, `cancel_order`,
  `close_participants`, `close_table_sessions`, `release_table` — el ciclo completo de cobro y
  cierre de una orden de mesa: bloqueo con lock optimista, cuenta, pago (vía `build_sale`),
  confirmación (único punto de descuento del flujo QR), cancelación con reversa asimétrica según
  estado de cocina, y liberación de mesa en cascada.
- **`consolidation.py`** (234 líneas, 5 funciones públicas): `active_table_session_id`,
  `get_or_create_table_session_id`, `get_or_create_open_order`, `consolidate_table`,
  `add_item_to_table` — la ruta de alta directa del mesero desde la terminal (Fase 5), sin pasar
  por el carrito QR del comensal.
- **`kitchen.py`** (176 líneas, 3 funciones públicas): `transition_kitchen`, `mark_order_ready`,
  `void_item` — el ciclo de vida de preparación y anulación de cada ítem, independiente del
  status de pago de la orden.
- **`tables_advanced.py`** (114 líneas, 4 funciones públicas): `set_table_status`, `move_order`,
  `merge_orders`, `group_bill` — mover una orden de mesa, fusionar mesas por grupo, y la cuenta
  agrupada de un grupo fusionado.

`orders/router.py` (la capa de endpoints HTTP) **no** está en el alcance de esta spec: el encargo
lista explícitamente los cinco ficheros de lógica de negocio, no la capa de rutas.

`orders/consumption.py` queda **deliberadamente fuera** de esta lista: aunque `orders` como
candidato no tiene tests propios, `consumption.py` ya tiene cobertura indirecta vía el golden
master del motor de catálogo (`specs/014-extraccion-motor-catalogo/`), que lo encadena en su
flujo real. Esta spec lo consume como dependencia real (`deduct_order_items`,
`reverse_order_items`) sin re-caracterizarlo — es, según el propio análisis base, el siguiente
eslabón natural una vez extraído el motor, y se trata en una spec aparte.

`orders` no es una isla: `checkout.pay_order` construye la venta llamando a
`app.api.v1.sales.builder.build_sale` y `ensure_open_shift` (integración con caja/ventas) — la
red de tests de esta spec incluye ese camino real, sin mockear `build_sale` ni reordenar la
dependencia. `checkout.py` también consume `app.api.v1.promotions.service`
(`evaluate`, `combo_discount_for_lines`, `expand_combo` vía `consolidation.add_item_to_table`)
con fixtures mínimos, sin caracterizar sus reglas propias.

`orders` y `table_sessions` están mutuamente entrelazados: `table_sessions/service.py`
(`specs/016-caracterizacion-table-sessions/`, ya congelada) consume `checkout.TERMINAL`,
`checkout.close_table_sessions`, `checkout.order_sale_lines` y `checkout.promo_lines_for` como
dependencias reales con fixtures mínimos, **sin volver a caracterizar `checkout.py` en
profundidad** — ese trabajo quedó explícitamente pendiente para la spec de `orders`. Esta spec es
la que cierra ese hueco: caracteriza `checkout.py` en profundidad por primera vez. De forma
simétrica, `cart/service.py` (`specs/015-caracterizacion-cart/`, ya congelada) consume
`checkout.cancel_order` en `cancel_my_order` con fixtures mínimos, también sin profundizar — esta
spec tampoco reordena esa dependencia (`cart` y `table_sessions` siguen llamando a `checkout`,
nunca al revés), solo cierra el hueco de caracterización propia que ambas dejaron abierto.

Tres scripts legado sirven de punto de partida de casos, sin correr hoy en CI (**A-27**):
`app/scripts/test_cancel_inventory.py` (política de reversa de inventario al cancelar, ya
migrada en espíritu por el propio código de `cancel_order`), `app/scripts/test_receta_obligatoria.py`
(guarda de `deduct_order_items` contra variantes sin receta, ejercitada aquí a través de los tres
caminos de `orders` que la invocan: `create_order`, `confirm_order`, `add_item_to_table`) y
`app/scripts/test_session_ttl.py` (su aritmética de TTL vive en `qr_context.py`, fuera del
alcance de esta spec — pero protege el barrido automático que invoca
`checkout.close_table_sessions`/`close_participants` vía `scheduler.py:140`; esta spec migra solo
la porción de su escenario que ejercita esas dos funciones bajo ese disparador, no la aritmética
de TTL en sí). Los tres deben **migrarse** al formato `unittest` de
`app/characterization_tests/`, no incorporarse tal cual.

Siete anomalías del registro caen, en su totalidad o en parte, dentro de estos cinco ficheros y
esta spec las cubre explícitamente:

- **A-01, caminos B y C** (`checkout.compute_bill:127-180` y `tables_advanced.group_bill:92-114`):
  de las tres implementaciones que responden "cuánto se le debe cobrar", solo la de
  `table_sessions.compute_bill` (camino A, ya congelada en la spec 016) es correcta. El camino B
  (`checkout.compute_bill`) no aplica descuentos y no tiene caller confirmado — código muerto
  pero peligroso si se reactiva. El camino C (`tables_advanced.group_bill`), en cambio, **sí está
  en uso real** para mesas fusionadas: no filtra por status de orden (incluye `pagada` y
  `cancelada` en el total) ni aplica descuentos. Esta spec congela ambos caminos tal cual, sin
  unificarlos con el camino A.
- **A-04** (`consolidation.py:199`, dentro de `add_item_to_table`): `load_valid_options` solo
  valida `min_select`/`max_select`/pertenencia de grupo cuando se le pasa `variant`;
  `add_item_to_table` (el botón real que usa el mesero desde la terminal, el camino de uso
  diario) no se lo pasa, mientras que `service.create_order:102` sí lo hace. Es el hallazgo con
  prueba directa de `git log` de cómo se rompió (regresión de fusión entre una rama de corrección
  y una de combos). **Esta spec lo congela tal cual, sin corregirlo** — su fix de una línea
  (`variant=variant`) queda para una spec delta posterior autorizada por el registro.
- **A-16** (`kitchen.py`): `transition_kitchen` (`:43-60`) y `void_item` (`:93-176`) no comprueban
  el `status` de la `CustomerOrder` padre — funcionan igual aunque la orden esté `pagada`,
  `cancelada` o `bloqueada`. `mark_order_ready` (`:63-90`) sí valida, pero solo bloquea los
  estados terminales de pago, no `bloqueada`. Ninguna de las tres usa `with_for_update` (a
  diferencia de `checkout.confirm_order`).
- **A-25, [PROTEGIDA]**: no existe, entre las funciones públicas de estos cinco ficheros, ninguna
  vía de transición libre de `status`/`estado_cocina` — cada función que muta un estado
  (`block_order`, `confirm_order`, `pay_order`, `cancel_order`, `transition_kitchen`,
  `mark_order_ready`, `void_item`) impone su propia transición validada. Esta spec congela ese
  invariante como caso de referencia, sin reintroducir la vía genérica que ya se retiró
  deliberadamente (`orders/router.py`, fuera de alcance).
- **A-26** (`tables_advanced.py`): tres hallazgos del mismo módulo — `move_order` (`:45-73`)
  exige que la mesa destino esté completamente libre de órdenes activas, más estricto que el
  modelo general que permite varias órdenes por mesa; el mismo `move_order` (`:59-63`) captura un
  `IntegrityError` que traduce a 409, pero el modelo ya no tiene ningún índice único que lo
  dispare — manejador huérfano; `merge_orders` (`:75-89`), si las órdenes a fusionar ya
  pertenecían a grupos distintos preexistentes, conserva el primero según un `SELECT` sin
  `ORDER BY` — no determinista.
- **A-29, parcial** (`checkout.py:268-269`, dentro de `pay_order`): cuando las líneas cobradas
  usan más de un combo distinto (o ninguno), `promotion_id` no registra ningún combo — el
  descuento monetario se suma correctamente, pero se pierde la trazabilidad por promoción en
  reportes. Es uno de los tres caminos de cobro que comparten el mismo mecanismo (los otros dos,
  `table_sessions` y `sales/service.py`, quedan fuera — el de `table_sessions` ya está congelado
  en la spec 016).
- **A-38, parcial**: de los cinco hallazgos del clúster, esta spec cubre los dos que viven en
  `checkout.py` — **RN-ORD-31** (`close_table_sessions:495-527`, dentro de `release_table`): el
  cierre en cascada de sesiones de mesa no valida por sí mismo que no haya órdenes pendientes,
  delega esa responsabilidad al llamador (`release_table` sí valida antes de invocarlo; el job de
  sesiones huérfanas, no) — y **RN-ORD-32** (`order_sale_lines:191-230`): la descripción de línea
  de venta (`"Producto - Variante"`) puede quedar incompleta o vacía si el producto o la variante
  fueron borrados. Los otros tres hallazgos del clúster (`RN-MESA-13`, `RN-MESA-24`, `RN-ORD-34`)
  ya se trataron en la spec 016 o son observación documental sin impacto de comportamiento.

Y el motivo por el que esta red debe correr en CI, no ser otro script más: la misma razón que ya
documentaron las specs 015 y 016 sobre **A-27** — el pipeline de `pos-backend`
(`.github/workflows/deploy.yml`) solo ejecuta `test_promotions_rules.py` de los ~12 scripts
`app/scripts/test_*.py`. Esta spec suma sus ficheros al mismo paso
(`python -m unittest discover -s app/characterization_tests`) que ya ejecutan las redes de `cart`
y `table_sessions`, sin repetir el patrón de A-27.

## Clarifications

### Session 2026-08-17

- Q: ¿Cómo debe el test del manejador huérfano de `IntegrityError` en `move_order`
  (RN-ORD-60/A-26) ejercitar ese `except`, dado que el índice único que lo disparaba ya no
  existe en el modelo y ninguna secuencia de datos real puede provocar la excepción? → A:
  Forzar la excepción artificialmente (mock/monkeypatch de `db.flush` con `unittest.mock`,
  sin dependencias nuevas) dentro de `move_order`, y verificar que el manejador sigue
  presente y la traduce a 409.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Congelar A-04 y las 5 funciones públicas de `consolidation.py` (Priority: P1)

Como responsable de la modernización, escribo characterization tests que ejercitan cada una de
las 5 funciones públicas de `consolidation.py` — la ruta de alta directa del mesero desde la
terminal — documentando en particular, con el mayor cuidado de toda la spec, el defecto de A-04:
que `add_item_to_table` omite pasar `variant` a `load_valid_options` y por tanto se salta la
validación de `min_select`/`max_select`/pertenencia de grupo en el camino de uso diario real.

**Why this priority**: A-04 es el hallazgo central que motiva explícitamente esta spec — el único
de todo el registro con prueba directa de `git log` de cómo se rompió, y cuyo fix de una línea
queda deliberadamente diferido. Congelarlo con precisión, antes de cualquier otra cosa, es lo que
hace posible que una spec delta futura lo corrija con la certeza de no romper nada más.

**Independent Test**: se puede verificar de forma aislada corriendo
`python -m unittest app.characterization_tests.test_orders_consolidation -v` (o el nombre de
fichero que se use) sin que existan aún los tests de los otros cuatro ficheros.

**Acceptance Scenarios**:

1. **Given** una variante con un grupo de opciones `min_select=1` obligatorio y ninguna opción
   seleccionada, **When** se llama a `add_item_to_table` sin `combo_id`, **Then** el ítem se
   agrega igualmente sin error de validación — congelando A-04 tal cual: el camino real del
   mesero no aplica la regla de selección mínima que sí aplica `create_order` en el mismo
   escenario.
2. **Given** el mismo escenario del punto 1 pero llamando a `service.create_order` en su lugar,
   **When** se pasa la misma variante sin opciones seleccionadas, **Then** la llamada sí falla
   con el error de validación de opciones — documentando el contraste exacto entre los dos
   caminos que motiva A-04.
3. **Given** una mesa sin sesión de mesa abierta ni orden abierta, **When** se llama a
   `add_item_to_table`, **Then** se crea la sesión de mesa y la orden abierta necesarias
   (`get_or_create_table_session_id`, `get_or_create_open_order`) y el ítem queda asociado a
   ellas, congelando la regla de "abrir sobre la marcha" tal como existe hoy.
4. **Given** una orden ya abierta en la mesa con ítems previos, **When** se llama a
   `consolidate_table`, **Then** el resultado consolida los ítems de los carritos de los
   comensales en la orden existente sin duplicar los ya agregados, congelando el comportamiento
   actual de idempotencia (o su ausencia, si el código no la garantiza).
5. **Given** un `combo_id` válido en la petición de `add_item_to_table`, **When** se agrega,
   **Then** el combo se expande en sus componentes reales a precio normal (sin el ahorro del
   combo, que se calcula al cobrar), congelando la regla documentada en el docstring de la
   función.

---

### User Story 2 - Congelar las 10 funciones públicas de `checkout.py`, incluyendo la integración con `build_sale` (Priority: P1)

Como responsable de la modernización, escribo characterization tests que ejercitan cada una de
las 10 funciones públicas de `checkout.py` — bloqueo, cuenta, pago, confirmación, cancelación y
liberación de mesa — incluyendo el camino real hacia `sales.builder.build_sale` en `pay_order`,
sin mockearlo, y cerrando el hueco de caracterización que las specs 015 (`cart`) y 016
(`table_sessions`) dejaron abierto al consumir estas mismas funciones como dependencias externas.

**Why this priority**: junto con la Historia 1, es el núcleo de la spec — `checkout.py` es el
fichero más grande y de mayor superficie de las cinco, y es la pieza que tanto `cart` como
`table_sessions` ya dependen de tener bien congelada. Es P1 porque, sin esta red, ninguna de las
dos specs anteriores tiene realmente cerrada su propia frontera con `orders`.

**Independent Test**: se puede verificar de forma aislada corriendo
`python -m unittest app.characterization_tests.test_orders_checkout -v`, sembrando una orden,
una sesión de mesa y un turno de caja abierto directamente, sin depender de `cart` ni
`table_sessions`.

**Acceptance Scenarios**:

1. **Given** una orden `abierta` sin ítems pendientes en cocina, **When** se llama a
   `block_order` con la versión correcta, **Then** pasa a `bloqueada`; **Given** la misma orden
   con al menos un ítem en `pendiente`/`en_preparacion`, **When** se llama de nuevo, **Then**
   responde 409 con el detalle de los ítems sin terminar — congelando el contrato tal cual.
2. **Given** una orden `bloqueada` y un turno de caja abierto, **When** se llama a `pay_order` con
   una promoción activa y sin combos, **Then** el `Sale` resultante se construye vía `build_sale`
   real (sin mock), con el descuento de la promoción sumado y el turno de caja correcto —
   congelando la integración real con `cash`/`sales`.
3. **Given** líneas cobradas que usan dos combos distintos, **When** se llama a `pay_order`,
   **Then** el `Sale` resultante tiene `promotion_id=None` (ninguno de los dos combos queda
   registrado) aunque el descuento monetario de ambos se sume correctamente — congelando A-29
   (parcial) tal cual, sin corregirlo.
4. **Given** un pedido `recibida` con ítems válidos, **When** se llama a `confirm_order`,
   **Then** pasa a `abierta` y descuenta inventario exactamente una vez (único punto de descuento
   del flujo QR); **Given** stock insuficiente de un insumo, **When** se llama de nuevo, **Then**
   la transacción entera revierte y el pedido sigue `recibida`.
5. **Given** una orden con ítems en distintos estados de cocina (`pendiente`, `en_preparacion`,
   `listo`, `anulado`), **When** se llama a `cancel_order`, **Then** solo los ítems `pendiente`
   generan una entrada real de inventario; los `en_preparacion`/`listo` no vuelven al stock (se
   registran como pérdida en `audit_logs`) — congelando la reversa asimétrica documentada.
6. **Given** una mesa con órdenes activas sin cerrar, **When** se llama a `release_table`,
   **Then** responde 409 con el detalle de las órdenes bloqueantes sin liberar la mesa; **Given**
   la misma mesa sin órdenes activas, **When** se llama de nuevo, **Then** la mesa queda `libre`
   y sus sesiones de mesa `active` se cierran en cascada (`close_table_sessions`,
   `close_participants`), congelando que `close_table_sessions` en sí no valida pendientes por su
   cuenta (RN-ORD-31/A-38, congelado tal cual).
7. **Given** un ítem cuyo producto o variante fue borrado antes de cobrar, **When** se llama a
   `order_sale_lines`, **Then** la descripción de la línea resultante queda incompleta o vacía
   (según cuál de los dos falte) — congelando RN-ORD-32 (A-38) tal cual.
8. **Given** una tabla con órdenes en distintos status (`abierta`, `pagada`, `cancelada`), **When**
   se llama directamente a `checkout.compute_bill` (camino B de A-01, sin caller de producción
   conocido), **Then** el total incluye las órdenes `pagada` y no aplica ningún descuento —
   congelando el comportamiento del código muerto tal cual, sin activarlo ni corregirlo.

---

### User Story 3 - Congelar las 3 funciones públicas de `kitchen.py`, incluyendo A-16 y A-25 (Priority: P2)

Como responsable de la modernización, escribo characterization tests que ejercitan
`transition_kitchen`, `mark_order_ready` y `void_item`, documentando que ninguna de las tres
valida el `status` de la `CustomerOrder` padre salvo la validación parcial de `mark_order_ready`
(A-16), y que sus transiciones internas están cerradas a una lista blanca sin vía genérica de
asignación libre (A-25).

**Why this priority**: depende conceptualmente de que existan órdenes e ítems ya sembrados (igual
mecanismo que las Historias 1 y 2), pero es una capa más acotada — el ciclo de vida de
preparación es independiente del status de pago, así que se puede caracterizar con fixtures
propios sin esperar a que las otras dos terminen.

**Independent Test**: se puede verificar de forma aislada corriendo
`python -m unittest app.characterization_tests.test_orders_kitchen -v`, sembrando una orden y
sus ítems con distintos `estado_cocina` directamente.

**Acceptance Scenarios**:

1. **Given** un ítem `pendiente`, **When** se llama a `transition_kitchen` hacia `listo`
   directamente (el salto de un toque), **Then** se acepta; **Given** un ítem `listo`, **When**
   se intenta transicionar hacia `pendiente` (retroceso), **Then** responde 409 — congelando la
   lista blanca de transiciones (`_ALLOWED`) tal cual, incluyendo que solo van hacia adelante.
2. **Given** una orden `pagada` con ítems aún `pendiente` (estado producible sembrando datos
   directamente), **When** se llama a `transition_kitchen` o `void_item` sobre esos ítems,
   **Then** ambas operaciones se ejecutan igual, sin ningún error por el status de la orden padre
   — congelando A-16 tal cual: ninguna de las dos valida el status de la orden.
3. **Given** la misma orden `pagada` con ítems `pendiente`, **When** se llama a
   `mark_order_ready`, **Then** responde 409 citando que la orden ya es terminal — congelando que
   esta función sí valida, a diferencia de las otras dos (el contraste que documenta A-16).
4. **Given** una orden `bloqueada` (no terminal de pago) con ítems `en_curso`, **When** se llama
   a `mark_order_ready`, **Then** los ítems pasan a `listo` sin ningún error — congelando que la
   validación de `mark_order_ready` bloquea solo `pagada`/`cancelada`, no `bloqueada` (la porción
   pendiente de A-16 documentada tal cual, sin resolver la pregunta de negocio abierta).
5. **Given** un ítem `pendiente` con un reemplazo válido, **When** se llama a `void_item` con
   `data.replacement`, **Then** el ítem original queda `anulado` (con reversa de inventario, por
   ser `pendiente`) y se crea uno nuevo `pendiente` con `void_de` apuntando al original y su
   propio descuento de inventario — congelando el ciclo completo de anular-y-reemplazar.
6. **Given** las siete funciones públicas de estos cinco ficheros que mutan algún estado
   (`block_order`, `confirm_order`, `pay_order`, `cancel_order`, `transition_kitchen`,
   `mark_order_ready`, `void_item`), **When** se inspecciona cada una, **Then** cada una impone
   su propia transición validada y ninguna acepta un `status`/`estado_cocina` arbitrario sin
   pasar por su propia guarda — congelando el invariante [PROTEGIDA] de A-25 como caso de
   referencia.

---

### User Story 4 - Congelar las 4 funciones públicas de `tables_advanced.py`, incluyendo A-26 y A-01 (camino C) (Priority: P2)

Como responsable de la modernización, escribo characterization tests que ejercitan
`set_table_status`, `move_order`, `merge_orders` y `group_bill`, documentando los tres hallazgos
de A-26 (estrictez de `move_order`, manejador huérfano de `IntegrityError`, no-determinismo de
`merge_orders`) y el camino C de A-01 (`group_bill` sin filtro de status ni descuentos, en uso
real para mesas fusionadas).

**Why this priority**: es una capa acotada de "mover, fusionar y agrupar mesas" — funcionalmente
independiente de cocina (Historia 3), aunque comparte el mismo tipo de fixtures de órdenes que
las Historias 1 y 2. Se numera P2 junto con la Historia 3 por ser de menor superficie que el
núcleo de cobro.

**Independent Test**: se puede verificar de forma aislada corriendo
`python -m unittest app.characterization_tests.test_orders_tables_advanced -v`, sembrando
órdenes en distintas mesas y grupos fusionados directamente.

**Acceptance Scenarios**:

1. **Given** una mesa destino con una orden activa ya presente, **When** se llama a `move_order`
   hacia esa mesa, **Then** responde 409 — congelando que `move_order` exige la mesa destino
   completamente libre de órdenes activas, más estricto que el modelo general de "varias órdenes
   por mesa" (RN-ORD-58/A-26, documentado sin corregir).
2. **Given** que el índice único que originaba la colisión ya no existe en el modelo (ninguna
   secuencia de datos real puede disparar `IntegrityError` por sí sola), **When** se fuerza
   artificialmente esa excepción (mock/monkeypatch de `db.flush` con `unittest.mock`) dentro de
   `move_order`, **Then** el manejador sigue presente y la traduce a 409 — congelando el
   manejador huérfano (RN-ORD-60/A-26) tal cual, documentando que ya no hay ningún camino de
   datos reales que lo alcance.
3. **Given** dos órdenes que ya pertenecen a dos `merged_group_id` distintos preexistentes,
   **When** se llama a `merge_orders` con ambas, **Then** el grupo resultante es uno de los dos
   preexistentes, elegido por un `SELECT` sin `ORDER BY` — el test documenta explícitamente que
   el resultado depende del orden de retorno de esa consulta (no determinista), sin exigir un
   valor específico, congelando RN-ORD-63/A-26 tal cual.
4. **Given** un grupo fusionado con órdenes en status `abierta`, `pagada` y `cancelada`, **When**
   se llama a `group_bill`, **Then** el total agregado incluye las tres sin excluir ninguna por
   status, y no aplica ningún descuento — congelando el camino C de A-01 tal cual, tal como lo
   consume hoy `table.service.ts` del frontend.
5. **Given** una mesa con al menos una orden activa, **When** se llama a `set_table_status` con
   `new_status='libre'` o `'reservada'`, **Then** responde 409; **Given** la misma mesa sin
   órdenes activas, **When** se llama de nuevo, **Then** el cambio se acepta.

---

### User Story 5 - Congelar `create_order`, la única función pública de `service.py` (Priority: P3)

Como responsable de la modernización, escribo characterization tests que ejercitan `create_order`
— la comanda directa de mostrador/mesero que nace ya `abierta` y descuenta inventario al crearse
— documentando su contraste explícito con A-04 (esta función sí pasa `variant` a
`load_valid_options`, a diferencia de `add_item_to_table`).

**Why this priority**: es la función pública más pequeña de las cinco en cantidad (una sola), y
su comportamiento correcto ya sirve de caso de contraste dentro de la Historia 1 (A-04); esta
historia completa su propia cobertura dedicada por separado, sin bloquear a las demás.

**Independent Test**: se puede verificar de forma aislada corriendo
`python -m unittest app.characterization_tests.test_orders_service -v`.

**Acceptance Scenarios**:

1. **Given** una variante con receta y opciones válidas, **When** se llama a `create_order`,
   **Then** la orden nace en status `abierta` (no `recibida`) con el inventario ya descontado —
   congelando que este camino no vuelve a pasar por `confirm_order`.
2. **Given** una variante sin receta asociada, **When** se llama a `create_order`, **Then** la
   guarda de `deduct_order_items` rechaza la creación — congelando la regla que
   `test_receta_obligatoria.py` ya verificaba de forma legado, ahora migrada al formato formal.
3. **Given** una mesa sin sesión de mesa activa, **When** se llama a `create_order` con
   `dining_table_id`, **Then** se crea la sesión de mesa vía `consolidation.get_or_create_table_session_id`
   antes de crear la orden — congelando la dependencia real entre `service.py` y
   `consolidation.py`.

---

### User Story 6 - La suite corre en CI de forma determinista (Priority: P3)

Como responsable de la modernización, incorporo los ficheros de test de las historias 1 a 5 al
mismo paso de CI del backend que ya ejecuta las redes de `cart` y `table_sessions`
(`.github/workflows/deploy.yml` o el que lo sustituya), de modo que se ejecuten automáticamente
en cada push, y verifico que la suite completa pasa de forma idéntica en ejecuciones repetidas.

**Why this priority**: sin esto, la red existe pero no es el árbitro que el Principio II exige —
el mismo riesgo que A-27 ya documentó. Es P3 porque depende de que las historias anteriores
existan primero para tener algo que ejecutar.

**Independent Test**: se puede verificar abriendo un pull request trivial contra el branch de
esta spec y confirmando en la ejecución de CI que el paso existente recoge los ficheros nuevos.

**Acceptance Scenarios**:

1. **Given** el workflow de CI del backend, **When** se inspecciona su definición, **Then** el
   paso `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` (o
   equivalente) cubre también los ficheros nuevos de esta spec, sin necesitar un paso adicional
   ni reemplazar el existente.
2. **Given** la suite de esta spec ejecutada tres veces seguidas sin ningún cambio de código,
   **When** se comparan los tres resultados, **Then** los tres son idénticos — ninguno depende
   del reloj real, de una conexión Postgres real, ni del orden de ejecución.
3. **Given** un push que modifica cualquiera de los cinco ficheros de `orders` de forma que
   cambia su comportamiento observable, **When** corre CI, **Then** al menos un test de esta
   suite falla en rojo, demostrando que la red efectivamente detecta el cambio.

### Edge Cases

- ¿Qué pasa si, al escribir un test de esta suite, falla contra el código actual sin modificar?
  → El defecto está en el test, nunca en los cinco ficheros de producción: se corrige el test
  hasta que refleje la realidad observada. Esta spec no autoriza ningún cambio en `orders`.
- ¿Qué pasa si aparece, mientras se escriben estos tests, una anomalía no listada hoy en
  `registro-de-anomalias.md`? → Se documenta ahí (Principio I) y se congela tal cual en el test;
  no se corrige como parte de esta spec.
- ¿Qué pasa con A-04 específicamente si, al escribir su test, alguien tiene la tentación de
  "aprovechar y arreglarlo ya que se ve tan simple"? → Fuera de alcance por diseño explícito: el
  fix de una línea (`variant=variant` en `consolidation.py:199`) queda para una spec delta
  posterior autorizada por el registro de anomalías, exactamente como exige el Principio II.
- ¿Qué pasa con las reglas propias de `promotions` (qué promoción/combo aplica) o de `sales`
  (cómo arma `build_sale` el objeto `Sale` internamente)? → Fuera de alcance: esta spec ejercita
  esas funciones reales con fixtures mínimos controlados para congelar cómo `orders` las usa, no
  para caracterizar exhaustivamente sus propias reglas.
- ¿Qué pasa con el no-determinismo de `merge_orders` (RN-ORD-63/A-26)? → El test lo documenta
  explícitamente como no determinista (ambos resultados posibles son válidos según el orden de
  retorno de la consulta), sin fijar un valor esperado único — congelar "es no determinista" es
  en sí mismo el comportamiento a congelar, no una contradicción con el requisito de determinismo
  de la suite (FR-011), que aplica al resultado del test, no al comportamiento observado.
- ¿Qué pasa con la porción de `test_session_ttl.py` que no toca `orders` (la aritmética de
  `_should_refresh` en `qr_context.py`)? → No se migra en esta spec: no es superficie pública de
  ninguno de los cinco ficheros en alcance. Solo se migra el escenario de invocación de
  `close_table_sessions`/`close_participants` bajo el disparador del barrido, ya cubierto por la
  Historia 2.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE incluir, bajo `app/characterization_tests/`, uno o más ficheros de
  test (prefijo `test_orders_`) que sigan la convención documentada en
  `app/characterization_tests/__init__.py` (docstring "CONGELA comportamiento actual:", Principio
  II) y usen exclusivamente `unittest` de la biblioteca estándar sobre los modelos SQLAlchemy
  reales del proyecto vía SQLite en memoria — cero dependencias nuevas (Principio IV).
- **FR-002**: La suite DEBE incluir al menos un characterization test por cada una de las 23
  funciones públicas de los cinco ficheros en alcance: `service.py` (`create_order`);
  `checkout.py` (`block_order`, `compute_bill`, `order_sale_lines`, `promo_lines_for`,
  `pay_order`, `confirm_order`, `cancel_order`, `close_participants`, `close_table_sessions`,
  `release_table`); `consolidation.py` (`active_table_session_id`,
  `get_or_create_table_session_id`, `get_or_create_open_order`, `consolidate_table`,
  `add_item_to_table`); `kitchen.py` (`transition_kitchen`, `mark_order_ready`, `void_item`);
  `tables_advanced.py` (`set_table_status`, `move_order`, `merge_orders`, `group_bill`).
- **FR-003**: La suite DEBE incluir al menos un caso que congele A-04
  (`consolidation.py:199`, dentro de `add_item_to_table`): la omisión de `variant` en la llamada
  a `load_valid_options` que salta la validación de selección de opciones en el camino real del
  mesero, con un caso de contraste directo contra `service.create_order` (que sí valida) —
  documentado tal cual, sin aplicar el fix de una línea.
- **FR-004**: La suite DEBE incluir al menos un caso por cada uno de los dos caminos de A-01
  cubiertos por esta spec: camino B (`checkout.compute_bill`, sin descuentos, sin caller
  conocido) y camino C (`tables_advanced.group_bill`, sin filtro de status ni descuentos, en uso
  real para mesas fusionadas) — ambos congelados tal cual, sin unificarlos con el camino A ya
  congelado en la spec 016.
- **FR-005**: La suite DEBE incluir al menos un caso que congele A-16: `transition_kitchen` y
  `void_item` no validan el `status` de la `CustomerOrder` padre; `mark_order_ready` sí valida,
  pero solo bloquea los estados terminales de pago (`pagada`/`cancelada`), no `bloqueada` —
  documentado tal cual, incluyendo el contraste entre las tres funciones.
- **FR-006**: La suite DEBE incluir al menos un caso que congele A-25 [PROTEGIDA]: ninguna de las
  siete funciones públicas que mutan `status`/`estado_cocina` (`block_order`, `confirm_order`,
  `pay_order`, `cancel_order`, `transition_kitchen`, `mark_order_ready`, `void_item`) acepta una
  transición fuera de su propia lista blanca — congelado como invariante de referencia.
- **FR-007**: La suite DEBE incluir al menos un caso por cada uno de los tres hallazgos de A-26 en
  `tables_advanced.py`: la estrictez de `move_order` frente al modelo general de varias órdenes
  por mesa, el manejador huérfano de `IntegrityError`, y el no-determinismo de `merge_orders`
  ante grupos preexistentes en colisión — todos documentados tal cual, sin corregirlos.
- **FR-008**: La suite DEBE incluir al menos un caso que congele A-29 (parcial,
  `checkout.py:268-269`, dentro de `pay_order`): con dos o más combos distintos en las líneas
  cobradas, `promotion_id` no registra ningún combo, aunque el descuento monetario se sume
  correctamente.
- **FR-009**: La suite DEBE incluir al menos un caso por cada uno de los dos hallazgos de A-38 que
  viven en `checkout.py`: RN-ORD-31 (`close_table_sessions` no valida por sí mismo órdenes
  pendientes, delega esa responsabilidad al llamador) y RN-ORD-32 (`order_sale_lines` produce una
  descripción de línea incompleta o vacía si el producto o la variante fueron borrados).
- **FR-010**: La suite DEBE ejercitar `sales.builder.build_sale` y `ensure_open_shift` como
  dependencias reales, sin mocks, a través de `checkout.pay_order`, incluyendo al menos un caso
  con una promoción activa y otro con combos — sin reordenar la dependencia de `orders` sobre
  `cash`/`sales`.
- **FR-011**: La suite DEBE ejercitar `promotions.evaluate`, `promotions.combo_discount_for_lines`
  y `promotions.expand_combo` como dependencias reales sobre fixtures mínimos y controlados, sin
  mocks y sin re-derivar sus propias reglas de negocio.
- **FR-012**: La suite DEBE ejercitar `orders.consumption.deduct_order_items` y
  `reverse_order_items` como dependencias reales a través de los tres caminos que los invocan
  (`create_order`, `confirm_order`/`cancel_order`, `add_item_to_table`/`void_item`), sin mocks y
  sin re-caracterizar en profundidad las reglas propias del motor de catálogo/consumo (ya
  cubiertas por `specs/014-extraccion-motor-catalogo/`).
- **FR-013**: Cada test DEBE ser determinista en su resultado (verde/rojo estable): ninguno puede
  depender del reloj real de la máquina que lo ejecuta, de una conexión Postgres real, ni del
  orden de ejecución de otros tests — incluyendo el caso de `merge_orders` (RN-ORD-63/A-26), cuyo
  comportamiento observado es no determinista pero cuyo test debe pasar de forma estable en cada
  ejecución (aceptando cualquiera de los resultados válidos, no exigiendo uno específico).
- **FR-014**: La suite DEBE ejecutarse automáticamente como parte del mismo paso de CI del backend
  que ya ejecutan las redes de `cart` y `table_sessions`
  (`.github/workflows/deploy.yml` o el que lo sustituya) en cada push, sin reemplazar ningún paso
  existente.
- **FR-015**: Ningún test de esta suite DEBE requerir una modificación de `orders/service.py`,
  `checkout.py`, `consolidation.py`, `kitchen.py` ni `tables_advanced.py` para pasar: si un test
  recién escrito falla contra el código actual sin modificar, el defecto está en el test.
- **FR-016**: El sistema NO DEBE corregir, mitigar ni alterar A-01, A-04, A-16, A-25, A-26, A-29 ni
  A-38 como parte de esta spec — cada test debe documentar el comportamiento observado, no el
  comportamiento deseado. En particular, el fix de una línea de A-04
  (`variant=variant` en `consolidation.py:199`) queda explícitamente fuera de esta spec.
- **FR-017**: El sistema NO DEBE extraer, mover, ni reescribir código de ninguno de los cinco
  ficheros en alcance, ni cambiar la interfaz (firmas de funciones) de ninguno de ellos, ni
  reordenar su dependencia de `cash`/`sales` (`build_sale`) ni de `orders.consumption`, como
  parte de esta spec.

### Key Entities *(include if feature involves data)*

- **Función pública de los cinco ficheros de `orders` en alcance**: una de las 23 funciones sin
  prefijo `_` repartidas entre `service.py`, `checkout.py`, `consolidation.py`, `kitchen.py` y
  `tables_advanced.py` — la unidad mínima de cobertura exigida por FR-002.
- **Caso de anomalía citado (A-01 / A-04 / A-16 / A-25 / A-26 / A-29 / A-38)**: un
  characterization test cuyo propósito explícito es documentar el comportamiento de una anomalía
  ya registrada, con su cita correspondiente en el docstring o nombre del test.
- **Dependencia externa fijada**: un módulo que `orders` consume (`sales.builder.build_sale`,
  `promotions.service`, `orders.consumption`) cuyo comportamiento se ejercita real y sin mocks,
  pero no se re-caracteriza en profundidad dentro de esta spec.
- **Frontera con `cart`/`table_sessions`**: la superficie de `checkout.py` (en particular
  `cancel_order`, `TERMINAL`, `close_table_sessions`, `order_sale_lines`, `promo_lines_for`) que
  ya se consumía como dependencia real, sin profundizar, en las specs 015 y 016 — esta spec cierra
  esa cobertura pendiente sin reabrir ni modificar las dos specs anteriores.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las 23 funciones públicas de los cinco ficheros en alcance
  (`service.py`, `checkout.py`, `consolidation.py`, `kitchen.py`, `tables_advanced.py`) tienen al
  menos un characterization test asociado.
- **SC-002**: A-01 (caminos B y C), A-04, A-16, A-25, A-26, A-29 (parcial) y A-38 (parcial,
  RN-ORD-31 y RN-ORD-32) están representados cada uno por al menos un caso, verificable por su
  cita explícita en el docstring o nombre del test correspondiente.
- **SC-003**: La suite completa pasa en verde en tres ejecuciones consecutivas sin ningún cambio
  de código entre ellas, con exactamente el mismo resultado en las tres (incluyendo el caso de
  `merge_orders`, cuyo comportamiento interno es no determinista pero cuyo test es estable).
- **SC-004**: La suite se ejecuta automáticamente en el pipeline de CI del backend — verificable
  inspeccionando la definición del workflow — y no solo de forma manual/local.
- **SC-005**: `service.py`, `checkout.py`, `consolidation.py`, `kitchen.py` y `tables_advanced.py`
  tienen cero líneas modificadas (diff vacío) al concluir esta spec, comparados contra su estado
  inmediatamente anterior a ella.
- **SC-006**: Cero dependencias nuevas añadidas a `requirements.txt` como resultado de esta spec.
- **SC-007**: Los tres scripts legado (`test_cancel_inventory.py`, `test_receta_obligatoria.py`,
  `test_session_ttl.py`) quedan representados en la suite formal por al menos un caso migrado
  cada uno, sin que ninguno de los tres se ejecute ya como script aparte fuera de
  `app/characterization_tests/`.

## Assumptions

- `orders/router.py` no está en el alcance de esta spec (el encargo lo excluye explícitamente):
  el mecanismo de verificación es invocar las funciones de `service.py`, `checkout.py`,
  `consolidation.py`, `kitchen.py` y `tables_advanced.py` directamente, no a través de
  `fastapi.testclient` — decisión de mecanismo concreto que se resuelve en la fase de
  planificación.
- `orders/consumption.py` no se caracteriza en esta spec: se consume como dependencia real
  (`deduct_order_items`, `reverse_order_items`), apoyada en su cobertura indirecta existente vía
  el golden master del motor de catálogo (`specs/014-extraccion-motor-catalogo/`). Su propia red
  de characterization tests, si se necesita, es una spec futura — el propio análisis base lo
  señala como el siguiente eslabón natural.
- `sales.builder.build_sale` y `ensure_open_shift`, así como `promotions.service`, no tienen
  todavía su propia red `characterization_tests` dedicada. Esta spec no llena ese vacío: los
  ejercita como dependencias reales con fixtures mínimos, suficientes para congelar cómo `orders`
  los usa, sin caracterizar sus reglas propias — ese trabajo, si se necesita, es una spec futura
  de esos módulos.
- `cart` (`specs/015-caracterizacion-cart/`) y `table_sessions`
  (`specs/016-caracterizacion-table-sessions/`) ya tienen su propia red de characterization
  tests, concluida, y ya consumían `checkout.py` como dependencia real con fixtures mínimos, sin
  profundizar. Esta spec no reabre ni modifica esas dos specs: solo añade la caracterización
  propia de `checkout.py` que ambas dejaron pendiente, cerrando el hueco de frontera que el
  encargo pide explícitamente coordinar.
- El caso de A-26 (manejador huérfano de `IntegrityError` en `move_order`) se reproduce forzando
  artificialmente la excepción (mock/monkeypatch de `db.flush` con `unittest.mock` de la
  biblioteca estándar, sin dependencias nuevas) dentro de `move_order`, en vez de sembrar datos
  reales: el índice único que originaba la colisión ya no existe en el modelo, así que ninguna
  secuencia de datos real puede disparar `IntegrityError` por sí sola. El objetivo es documentar
  que el manejador de excepción sigue presente y activo (traduce a 409), no verificar que la
  restricción de base de datos existe ni recrearla en el esquema de test.
- El paso de CI de esta spec se suma al mismo paso ya usado por las specs 015 y 016 en
  `.github/workflows/deploy.yml` (el único workflow del backend que existe hoy); si en el momento
  de implementar existe otro mecanismo de CI para el backend, se usa ese en su lugar,
  conservando el requisito sustantivo (FR-014): que la suite corra automáticamente en cada push,
  no solo a mano.
- Los tres scripts legado citados en el encargo (`test_cancel_inventory.py`,
  `test_receta_obligatoria.py`, `test_session_ttl.py`) se leen como fuente de casos ya pensados
  por el equipo, pero cada caso migrado se reescribe en la convención formal y se verifica contra
  el código real antes de aceptarse — no se importan ni se referencian directamente como
  dependencia de ejecución. La porción de `test_session_ttl.py` ajena a `orders` (la aritmética
  de `_should_refresh` en `qr_context.py`) no se migra en esta spec.
