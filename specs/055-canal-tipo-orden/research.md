# Research: Estandarización de canal y tipo de orden — habilitación de pedidos "Para Llevar"

Decisiones técnicas para implementar `specs/055-canal-tipo-orden/spec.md`. El código vive en dos
repositorios hermanos de este (`pos-specs`): `../pos-backend` (FastAPI + SQLAlchemy + Alembic,
multi-tenant por schema Postgres) y `../pos-heladeria` (Angular 21, signals).

## Decisión 1 — `String` + `CheckConstraint` + `Index`, no un `ENUM` nativo de Postgres

- **Decisión**: tanto `channel` (estandarizado) como el nuevo `order_type` se implementan como
  `Mapped[str]` con `String(N)`, un `CheckConstraint` con los valores permitidos, y un `Index`
  dedicado — exactamente el mismo patrón que ya usa `status` en `CustomerOrder`
  (`app/models/customer_order.py:98-104`) y el resto del modelo de datos del proyecto (`status`,
  `type` en otras tablas de la baseline). Cada valor sigue existiendo como un enum real a nivel de
  aplicación: `OrderChannel(str, Enum)` (`orders/schemas.py:12-15`, ya existe, se le cambian los
  valores) y un nuevo `OrderType(str, Enum)` en el mismo archivo.
- **Rationale**: es el mismo patrón que este modelo de datos ya usa en todas partes — confirmado
  (spec 042, research.md) que el proyecto **no usa `sa.Enum` nativo de Postgres en ningún lado**.
  Introducir un tipo nativo aquí sería la única tabla con dos convenciones distintas para el mismo
  tipo de restricción, sin ninguna ganancia funcional (el `CheckConstraint` + índice ya cumple
  exactamente lo pedido: "deben ser un enum e índice para poder filtrar").
- **Alternatives considered**: `sa.Enum` nativo de Postgres (`CREATE TYPE ... AS ENUM`) — descartado
  por inconsistencia con el resto del modelo y porque alterar un `ENUM` nativo después (agregar un
  valor) es una operación de esquema más rígida en Postgres que ampliar un `CheckConstraint`.

## Decisión 2 — Mapeo de `channel` histórico y preservación de la distinción interna `counter`/`waiter`

- **Hallazgo crítico** (no anticipado en spec.md, encontrado al auditar todos los lugares que leen
  `channel`): `channel == 'waiter'` no es un valor sin uso real, como parecía a primera vista —
  `app/api/v1/orders/consolidation.py:83` (`get_or_create_open_order`, usada por
  `consolidate_table` y `add_item_to_table`, ambas del modo híbrido de Terminal de Mesas, spec 028)
  filtra **específicamente** por `channel == "waiter"` para encontrar/reusar "la comanda que el
  mesero ya viene llenando en esta mesa", y la crea con `channel="waiter"` (línea 93) por la misma
  razón. Esto es intencional y está protegido por characterization tests: `test_orders_consolidation.py:241,263`
  (`"""CONGELA comportamiento actual..."""`) hacen `self.assertEqual(order.channel, "waiter")`.
- **Por qué importa**: `channel == 'counter'` (la comanda que arma el cajero desde
  `manual-order-page.component.ts`, vía `orders.service.create_order`) también puede terminar en
  `status == 'abierta'` — confirmado en `orders/service.py:40-58`: `checkout_and_send`/
  `_confirm_order_impl` dejan la orden **deliberadamente** en `'abierta'` incluso después de
  cobrada ("la venta ya emitida"). Es decir: una comanda `counter` ya pagada puede quedar con
  exactamente la misma forma (`status='abierta'`, `dining_table_id=<mesa>`) que
  `get_or_create_open_order` busca para el mesero. El único motivo por el que hoy **no** se
  confunden es que esa función exige además `channel == 'waiter'`. Si `counter` y `waiter` se
  funden en el mismo valor físico `POS` sin ningún otro cambio, `get_or_create_open_order`
  empezaría a poder "reabrir" y seguir agregando ítems a una comanda de mostrador **ya cobrada**
  (venta ya emitida) — un bug de facturación real, no cosmético.
- **Decisión**: agregar una columna técnica nueva, **no expuesta** en ningún schema de API ni en el
  "canal" de negocio que pide la spec: `is_consolidation_order: Mapped[bool]` (`Boolean`,
  `NOT NULL`, `server_default='false'`). Con esto:
  - `channel` se estandariza exactamente como pide la spec: `qr` → `QR_MENU`, **`counter` y
    `waiter` → `POS`** (FR-001, FR-013) — el "canal" que ve el negocio para filtrar/reportar deja
    de tener esta distinción, que nunca fue una distinción de canal de negocio real (ambos son
    "creado directamente desde el cajero/mesero"), sino una necesidad técnica interna de
    `consolidation.py`.
  - `orders/consolidation.py::get_or_create_open_order` deja de filtrar/crear por `channel`, y pasa
    a filtrar/crear por `is_consolidation_order = true` (además de `channel == 'POS'` y
    `status == 'abierta'`, sin cambio en esas dos condiciones). `orders/service.py::create_order`
    (comanda armada desde `manual-order-page`, y cualquier futuro llamador de `OrderCreate`) siempre
    crea con `is_consolidation_order = false`.
  - **Resultado**: el conjunto de comandas que `get_or_create_open_order` encuentra/crea es
    exactamente el mismo antes y después del cambio — es un *rename* del criterio técnico
    (`channel == 'waiter'` → `is_consolidation_order == true`), no un cambio de comportamiento.
    Migración histórica: `is_consolidation_order = (channel_original = 'waiter')`.
- **Characterization tests a actualizar** (Principio III: requiere spec — esta, 055 — y evidencia de
  que ningún otro comportamiento protegido se ve afectado; el cambio es un rename 1:1, verificado
  arriba): `test_orders_consolidation.py:241,263` pasan de `assertEqual(order.channel, "waiter")` a
  `assertEqual(order.channel, "POS")` + `assertTrue(order.is_consolidation_order)`. Los fixtures
  genéricos `orders_fixtures.py:268` y `table_sessions_fixtures.py:250`
  (`kw.setdefault("channel", "waiter")`) pasan a `kw.setdefault("channel", "POS")` — son fábricas
  de "una orden de staff genérica" en muchos tests que no hacen ninguna aserción sobre el canal
  exacto, así que no hace falta que además fijen `is_consolidation_order` salvo que el test
  específico lo necesite. El resto de usos de `"waiter"`/`"counter"` como literal en tests
  (`test_orders_service.py`, `test_orders_checkout.py`, `test_orders_kitchen.py`,
  `test_cart_single_active_order.py`, `test_scheduler.py`, `test_orders_timezone.py`,
  `scripts/test_split_blindaje.py`) son valores de canal usados para setup, no aserciones sobre el
  valor en sí (confirmado por grep — ninguno de esos otros archivos hace
  `assertEqual(*.channel, "waiter"/"counter")`); se actualizan a `"POS"` sin más implicación.
- **Otros lugares que leen `channel` y solo necesitan el *rename* 1:1** (sin necesidad de
  `is_consolidation_order`, porque no distinguen `counter` de `waiter` entre sí, solo "staff" de
  "QR"):
  - `app/api/v1/cart/service.py:589` — `channel.in_(("counter", "waiter"))` → `channel == "POS"`.
  - `app/api/v1/orders/service.py:121,155` — comparaciones contra `OrderChannel.QR` → 
    `OrderChannel.QR_MENU` (mismo enum, nuevo nombre de miembro).
  - Frontend (`pos-heladeria`), todas comparan contra `'qr'` exclusivamente (no contra
    `'counter'`/`'waiter'`), así que solo cambian el literal: `dining.interface.ts:234`
    (`getSidebarMode`), `pos-terminal.store.ts:162,425,450`.
- **Alternatives considered**: (a) no fusionar `counter`/`waiter` y exponer 5 valores de canal en la
  API (`POS_COUNTER`, `POS_WAITER`, `QR_MENU`, `WHATSAPP`, `API`) — descartado porque contradice
  literalmente el catálogo de 4 valores que pidió el usuario y expondría al negocio una distinción
  interna sin valor de reporte real. (b) resolver la colisión con una consulta más compleja en
  `get_or_create_open_order` (p. ej. excluir órdenes con `Sale` ya emitida, usando
  `order_has_sale()` que ya existe en `orders/service.py`) en vez de una columna nueva — descartado:
  cambia el criterio de "qué comanda es mía" a algo más permisivo de lo que es hoy (aceptaría
  cualquier comanda de mostrador abierta y no pagada, no solo las creadas por consolidación), lo
  cual sí sería un cambio de comportamiento observable nuevo, no autorizado por spec.md.

## Decisión 3 — `order_type`: nulable en la base de datos, siempre poblado por la aplicación desde ahora

- **Decisión**: `order_type: Mapped[Optional[str]]` (`String(10)`, `CheckConstraint`, `Index`,
  **nulable** a nivel de columna). `OrderCreate.order_type: OrderType = OrderType.DINE_IN` en el
  schema de request (mismo patrón que `channel: OrderChannel = OrderChannel.COUNTER` ya
  existente) — todo llamador nuevo de `POST /orders` obtiene un valor por defecto sensato sin tener
  que enviarlo explícitamente, y el valor real para "En Mesa"/"Para Llevar" lo fija
  `manual-order-page.component.ts` (Decisión 6).
- **Backfill histórico** (clarificación de spec.md, sesión 2026-08-29): `UPDATE customer_orders SET
  order_type = 'DINE_IN' WHERE dining_table_id IS NOT NULL` — el resto queda en `NULL`. Reproduce
  exactamente la decisión de negocio ya tomada (FR-014).
- **Por qué nulable y no `NOT NULL` con default**: forzar un valor no nulo obligaría a inventarle un
  tipo de orden a pedidos históricos sin mesa que no lo tienen realmente definido (la clarificación
  del usuario fue explícita: esos casos quedan sin asignar). FR-015 ("todo pedido nuevo siempre
  tiene tipo de orden") se garantiza en la capa de aplicación (default del schema + los tres puntos
  de creación de orden lo fijan siempre explícitamente — Decisión 4), no en la columna.
- **Alternatives considered**: `NOT NULL` con `server_default='DINE_IN'` y sin distinguir los
  pedidos sin mesa — descartado, contradice la clarificación explícita del usuario sobre el
  backfill.

## Decisión 4 — Dónde se valida la combinación canal × tipo de orden (FR-006)

- **Decisión**: la validación de combinaciones permitidas vive **únicamente** en
  `orders/service.py::create_order` (el único punto donde `channel` y `order_type` llegan como
  datos arbitrarios de un llamador vía `OrderCreate`). Los otros dos puntos que crean
  `CustomerOrder` directamente construyen su propia combinación fija y ya válida por construcción,
  así que no necesitan (ni deben) pasar por esta validación:
  - `cart/service.py::submit_cart` (flujo QR): siempre `channel="qr"` (→ `QR_MENU`) +
    `order_type="DINE_IN"` — el menú QR de esta versión del sistema es inherentemente de mesa.
  - `orders/consolidation.py::get_or_create_open_order`: siempre `channel="waiter"` (→ `POS`,
    `is_consolidation_order=true`) + `order_type="DINE_IN"` — opera siempre sobre una mesa física.
- **Regla** (spec.md FR-006, ya con los nombres finales de los miembros del enum):
  ```text
  POS:      DINE_IN, TAKEAWAY, DELIVERY   (los tres)
  QR_MENU:  DINE_IN                       (únicamente)
  WHATSAPP: TAKEAWAY, DELIVERY            (nunca DINE_IN)
  API:      TAKEAWAY, DELIVERY            (nunca DINE_IN)
  ```
  Implementada como un `dict[OrderChannel, frozenset[OrderType]]` a nivel de módulo en
  `orders/service.py`, consultado al inicio de `create_order` (mismo lugar que la validación
  existente de `hold_for_payment` + `channel is QR_MENU`, `service.py:120-125`) — `HTTPException`
  `400` con mensaje explícito si la combinación no está en el conjunto permitido (FR-007).
- **Rationale**: valida exactamente donde el dato es realmente variable (limita el radio de
  cambio), y dos de los tres canales/flujos de creación (QR_MENU, y la parte "waiter" de POS) ya
  eran, antes de esta spec, inherentemente de mesa — no hay ningún escenario real hoy en que
  necesiten pasar por la validación general.
- **Alternatives considered**: validar en un único punto compartido por los tres caminos de
  creación (p. ej. un `validator` a nivel de modelo/evento de SQLAlchemy) — descartado por
  desproporcionado: los otros dos caminos nunca pueden producir una combinación inválida por
  construcción, así que envolverlos en la misma validación solo agrega código sin reducir riesgo.

## Decisión 5 — `dining_table_id` y `order_type`: TAKEAWAY/DELIVERY nunca llevan mesa

- **Decisión**: en `create_order`, si `data.order_type` es `TAKEAWAY` o `DELIVERY` y
  `data.dining_table_id` no es `None`, se rechaza con `422` ("un pedido para llevar o a domicilio
  no puede tener una mesa asociada"). Es la contraparte defensiva de FR-011 ("sin mesa asociada")
  — la spec no lo pide como un escenario de aceptación explícito, pero es la validación de
  contrato mínima para que ese requisito no dependa por completo de que el frontend nunca envíe
  `dining_table_id` por error.
- **Confirmado sin cambio necesario**: el resto de la lógica de `create_order` (`table_id`,
  `table_session_id`, el chequeo de conflicto QR↔staff de la Decisión 4 en `service.py:155`) ya
  maneja `dining_table_id=None` correctamente hoy (`table_session_id` queda `None`, se saltan todos
  los chequeos atados a sesión de mesa) — no requiere ningún cambio adicional para TAKEAWAY, más
  allá del rechazo explícito de mesa+TAKEAWAY/DELIVERY de este punto.

## Decisión 6 — Frontend: reusar `PosTerminalStore.orderTypeTab`, no un signal paralelo

- **Decisión**: `manual-order-page.component.ts` conecta las pestañas "🍽️ En Mesa" / "🛍️ Para
  Llevar" al signal `orderTypeTab` que `PosTerminalStore` ya expone (`pos-terminal.store.ts:99,319`,
  spec 036) vía el método ya existente `setOrderTypeTab()` (`pos-terminal.store.ts:1017-1019`), en
  vez de introducir un signal local nuevo. Mapeo: `'mesas'` ⇄ `DINE_IN`, `'para-llevar'` ⇄
  `TAKEAWAY` (`'domicilios'` sigue sin usarse desde esta pantalla — FR-012). Esta instancia del
  store es propia del componente (`providers: [PosTerminalStore]`, `manual-order-page.component.ts:29`,
  no singleton de la app), así que no hay ningún efecto sobre `pos-tables-panel.component.ts`
  (Terminal de Mesas general), que usa el mismo tipo pero su propia instancia del store — sus
  pestañas "Para Llevar"/"Domicilio" siguen mostrando el mensaje "todavía no hay una vía de
  creación" exactamente igual que hoy (fuera de alcance de esta spec).
- **Cambios en `PosTerminalStore.createManualOrderFromDraft()`** (`pos-terminal.store.ts:1056-1091`,
  único llamador: `manual-order-page.component.ts:300`): 
  - El guard de entrada `if (!tableId || draftLines().length === 0) return false` pasa a exigir
    `tableId` **solo** cuando `orderTypeTab() === 'mesas'`; con `'para-llevar'` solo exige que el
    carrito no esté vacío.
  - El payload que arma `api.createManualOrder(...)` pasa de `channel: 'counter'` fijo a
    `channel: 'POS'` (siempre — ambas pestañas están en el punto de venta) + `order_type:
    orderTypeTab() === 'para-llevar' ? 'TAKEAWAY' : 'DINE_IN'` + `dining_table_id: orderTypeTab()
    === 'para-llevar' ? null : tableId`.
- **Campo "Cliente" (spec 054) sin cambio de comportamiento, solo de cuándo se muestra/aplica**: el
  bloque ya existente (`manual-order-page.component.ts:147-169`, `applyDefaultCustomerName()`) se
  muestra también con `'para-llevar'` seleccionado (FR-009/FR-010 de esta spec ya piden
  exactamente el mismo comportamiento que "En Mesa"); solo cambia que `applyDefaultCustomerName()`
  debe llamarse también al cambiar a la pestaña "Para Llevar" (hoy solo se llama desde `ngOnInit`,
  `selectTable()` y `onClienteBlur()`/`confirm()` — ninguno de esos disparadores existe todavía
  para un simple cambio de pestaña sin mesa).
- **Bloque "Mesas" (buscador de mesa, spec 053)**: se oculta con `@if (store.orderTypeTab() ===
  'mesas')` cuando la pestaña activa es "Para Llevar" (FR-009 — no se exige ni se muestra
  selección de mesa).
- **Botón "Confirmar y Enviar"**: el `[disabled]` pasa de
  `store.cartEmpty() || store.submitting() || !store.selectedTableId()` a
  `store.cartEmpty() || store.submitting() || (store.orderTypeTab() === 'mesas' &&
  !store.selectedTableId())`.
- **Contrato de red del lado frontend** (`dining.interface.ts`): `OrderChannel` pasa de
  `'qr' | 'counter' | 'waiter'` a `'POS' | 'QR_MENU' | 'WHATSAPP' | 'API'`; se agrega `OrderType =
  'DINE_IN' | 'TAKEAWAY' | 'DELIVERY'`; `OrderCreatePayload.order_type?: OrderType` nuevo;
  `DiningOrder.channel` (hoy `channel: string` sin tipar al enum, línea 189) puede mantenerse como
  `string` o estrecharse a `OrderChannel` — se decide en `tasks.md`, no cambia ningún
  comportamiento. Los tres literales `'qr'` de comparación (`dining.interface.ts:234`,
  `pos-terminal.store.ts:162,425,450`) pasan a `'QR_MENU'` (rename 1:1, Decisión 2).
- **Rationale**: reusar `orderTypeTab` evita un segundo signal redundante para el mismo concepto
  dentro del mismo componente, y es el mismo patrón que ya usa `pos-tables-panel.component.ts`
  desde spec 036 — consistencia dentro del propio código base.
- **Alternatives considered**: signal local `orderType` propio de `manual-order-page.component.ts`
  — descartado por duplicar un concepto que el store ya modela, sin ninguna ventaja (esta pantalla
  ya usa una instancia de store dedicada, así que no hay riesgo de interferencia con otras
  pantallas que sí justificara mantenerlo separado).

## Decisión 7 — Migración Alembic: patrón `@for_each_tenant_schema` + `_has_table`

- **Decisión**: una única migración nueva, encadenada al head real de `alembic/versions/` al
  momento de implementar (confirmar con `alembic heads` — al momento de esta planeación el
  historial tiene múltiples heads sueltos, p. ej. `c8ff3a5551cb_product_variants_display_order.py`;
  se resuelve al implementar, no aquí), que en un único `upgrade()` por schema de tenant:
  1. Agrega las columnas `order_type` (nulable) e `is_consolidation_order` (`NOT NULL DEFAULT
     false`) a `customer_orders`.
  2. Backfill de `order_type` (Decisión 3) e `is_consolidation_order` (Decisión 2) con `UPDATE`.
  3. Reemplaza los valores de `channel` (`UPDATE customer_orders SET channel = CASE channel WHEN
     'qr' THEN 'QR_MENU' WHEN 'counter' THEN 'POS' WHEN 'waiter' THEN 'POS' END`).
  4. Reemplaza el `CheckConstraint` de `channel` (drop + create con los 4 valores nuevos) y agrega
     el `CheckConstraint` de `order_type` + los dos índices nuevos (FR-002, FR-004).
  - Sigue el mismo guard `_has_table(schema, "products")` ya usado en
    `d2e3f4a5b6c7_active_order_per_participant.py` para saltar schemas de tenant aún no
    inicializados (scratch).
- **Rationale**: una sola migración, orden de pasos que nunca deja la tabla en un estado
  intermedio inconsistente (columnas antes que backfill, backfill antes que constraints nuevos).
- **Downgrade**: revierte los 4 pasos en orden inverso — quita constraints/índices nuevos, revierte
  `channel` a los 3 valores originales (`CASE` inverso: `'QR_MENU'→'qr'`, y `'POS'` se revierte
  usando `is_consolidation_order` para decidir `'waiter'` vs `'counter'`, antes de borrar esa
  columna), y elimina `order_type`/`is_consolidation_order`. Documentado explícitamente porque
  Principio VIII exige estrategia de rollback declarada.
