# Ciclo de vida del pedido de mesa: del QR a la factura

**Fecha de reconocimiento**: 2026-08-15
**Alcance**: `../pos-backend` y `../pos-heladeria`, siguiendo el método y los principios de la
[Constitución](../../.specify/memory/constitution.md) (evidencia obligatoria, comportamiento
actual sagrado, hallazgos registrados sin corregir).
**Método**: lectura directa de código, siguiendo cada camino desde el endpoint hasta el `commit`.
Toda afirmación cita fichero y línea. Donde el código no basta se marca `SUPOSICIÓN` con la
pregunta que la resolvería.

---

## 0. Resumen ejecutivo

El sistema expone **cuatro** funciones de backend que insertan una `CustomerOrder` o le añaden
líneas, no tres:

| # | Función | Endpoint | Quién lo dispara | ¿Tiene botón en `pos-heladeria`? |
|---|---|---|---|---|
| 1 | `cart.service.submit_cart` (`app/api/v1/cart/service.py:471-531`) | `POST /cart/submit` | Comensal, desde el QR | **Sí** — `public-menu.component.ts:748` |
| 2 | `orders.service.create_order` (`app/api/v1/orders/service.py:37-137`) | `POST /orders` | Staff, comanda completa de una vez | **No** — ver §3.2 |
| 3 | `orders.consolidation.add_item_to_table` (`app/api/v1/orders/consolidation.py:180-234`) | `POST /orders/tables/{id}/items` | Staff, ítem a ítem desde la terminal | **Sí** — `pos-terminal.store.ts:869-871` |
| 4 | `orders.consolidation.consolidate_table` (`app/api/v1/orders/consolidation.py:106-169`) | `POST /orders/tables/{id}/consolidate` | Staff, agrupa carritos de comensales sin enviar | **No** — sin caller en `pos-heladeria/src/app` (`grep -rn "consolidate" src/app --include=*.ts` → 0 resultados) |

De estos, los que el usuario pidió cubrir explícitamente son el **1** (el flujo QR pedido, que
es el hilo principal de este documento), el **2** ("creación manual desde el menú" — existe en
el backend con la forma de un alta de comanda completa, pero se verificó que no tiene ningún
disparador en el frontend actual) y el **3** ("desde la terminal de mesas" — es el que
realmente usa la UI de producción). El **4** se documenta porque existe y crea/reutiliza líneas
de una `CustomerOrder`, pero no es un alta desde cero (reempaqueta carritos ya creados por el
camino 1) y tampoco tiene UI: se incluye por completitud, no como uno de los "tres caminos".

**Hallazgo transversal más importante**: los endpoints legados de cobro por pedido
(`POST /orders/{id}/block`, `POST /orders/{id}/pay`) tampoco tienen ningún caller en
`pos-heladeria` (`grep -rn "orders/\${.*}/block\|orders/\${.*}/pay\b" src/app` → 0 resultados).
El cobro real en producción es `POST /table-sessions/{id}/close`
(`table-session.service.ts:64`), que salta el estado `bloqueada` por completo — pasa
`abierta → pagada` directo (`table_sessions/service.py:249`). El propio código lo confirma:
`sales/builder.py:1-15` habla de "los dos caminos de cobro" y luego, dentro de la misma
función, de "cuatro formas de cobrar (mostrador, cierre unificado, cierre dividido y el
`pay_order` legacy)" (`sales/builder.py:92-95`) — el propio autor documentó que ese cuarto
camino es legado. Se detalla en §9.

---

## 1. Punto de entrada: escanear el QR

1. El QR físico de la mesa codifica la URL `/menu/t/:token`
   (`pos-heladeria/src/app/app.routes.ts:57`, ruta pública sin guard), donde `token` es un JWT
   firmado con `typ="qr"`, sin expiración, que lleva `tenant_id` + `table_id`
   (`pos-backend/app/core/qr_token.py:77-93`). Se firma en
   `orders/router.py` al crear la mesa (import `mint_qr_token`, `router.py:14`).
2. El componente `PublicMenuComponent.ngOnInit` (`public-menu.component.ts:537-575`) toma el
   token de la URL (`:538`) y llama `DinerService.resolveByToken` (`diner.service.ts:78-83`),
   que hace `GET /menu/qr-token/{token}` (`:80`).
3. En el backend, `menu_by_signed_qr` (`app/api/v1/menu/router.py:193-213`) aplica rate-limit
   por IP y luego por mesa (`:199-201`), abre `open_qr_context(token)` (`core/qr_context.py`)
   que verifica la firma y resuelve el tenant **desde el token**, no desde `x-tenant-host`
   (comentario propio, `menu/router.py:1-5`), y devuelve mesa + negocio + menú
   (`_build_menu`, `:80-179`) en una sola respuesta.
4. El frontend guarda mesa/negocio/menú (`public-menu.component.ts:548-554`) y decide la vista
   siguiente: si no hay `session_token` local (`tokenStore.token()`, `:562`) pide el nombre
   (`view.set('name')`, `:563`); si ya hay uno, intenta restaurar carrito + pedidos
   (`cart.load()`, `refreshOrders()`, `:567`) y si falla (401, mesa ya cobrada) vuelve a pedir
   el nombre (`expireSession`, `:573`).
5. Al escribir el nombre, `confirmName()` (`:615-645`) llama
   `DinerService.openSession(token, name)` (`diner.service.ts:91-100`) → `POST /cart/sessions`
   (`cart/router.py:47-67`). El backend valida rate-limit por IP y luego por mesa
   (`cart/router.py:56,59`), abre `open_qr_context(qr_token)` (verifica firma) y llama
   `cart.service.open_session` (`cart/service.py:93-150`):
   - `_get_or_create_table_session` (`:58-74`) reutiliza la `table_session` `active` de la
     mesa si ya existe (comentario: escanear una mesa ocupada **une**, no abre una segunda,
     `:59-63`), protegido por el índice parcial `idx_active_session_per_table`.
   - Crea `SessionParticipant` (`:101-109`, tabla `session_participants`) con
     `display_label` desambiguado (`unique_display_label`, `:77-90`, "Ana" → "Ana (2)").
   - Crea `Cart` vacío en estado `abierto` (`:112-113`, tabla `carts`).
   - Marca la mesa `ocupada` si no lo estaba (`:115-117`) y publica `table_status_changed`
     tras el commit (`:128-132`).
   - Firma y devuelve un `session_token` (`typ="sess"`, con `exp`, `mint_session_token`,
     `qr_token.py:98-120`) que el frontend guarda en `DinerTokenStore` (`diner.service.ts:98`)
     y que el interceptor inyecta como cabecera `x-session-token` en toda petición posterior
     (`auth-token.interceptor.ts`, citado en `mapa-sistema.md:448-449`).

A partir de aquí el comensal ve el menú y arma su carrito — sección siguiente.

---

## 2. El carrito del comensal (previo a la creación del pedido)

El carrito **no crea un pedido todavía**; es el borrador que luego se convierte en
`CustomerOrder` al enviarse (§3.1). Entradas y cálculos:

- `POST /cart/items` (`cart/router.py:75-86`) → `cart.service.add_item`
  (`cart/service.py:276-312`). Entrada: `product_variant_id` **o** `combo_id` (exclusivos,
  `cart/schemas.py:47-53`), `quantity`, `option_ids[]`, `notes`.
  - Si es un combo, delega en `_add_combo` (`:315-348`), que expande el combo en sus
    componentes reales a precio normal vía `promotions.expand_combo`
    (`promotions/service.py:365-391`) — el ahorro del combo **no** se calcula aquí, se aplica
    al cobrar (comentario `cart/service.py:316-318`).
  - Si es un producto normal: `load_valid_options` (`catalog/line_pricing.py:43-66`) carga y
    valida las opciones (activas, y contra los grupos de la variante vía
    `validate_option_selection`, `:108-188`, que exige el **máximo** — no el mínimo — en
    grupos que descuentan inventario y son obligatorios, `_exige_maximo`, `:94-105`).
  - Chequeo preventivo de disponibilidad: `_cart_consumption` (consumo ya en el carrito) +
    `required_consumption` de la línea nueva, contra `check_availability`
    (`catalog/line_pricing.py:199-220`) — **no bloquea ni reserva stock**, solo lee
    `current_stock` sin lock (comentario `line_pricing.py:5-8`); el chequeo real y atómico es
    el de la confirmación (§4).
  - Precio de línea: `compute_line_price(variant, options)` = `variant.price + Σ option.extra_price`
    (`catalog/line_pricing.py:191-196`) — snapshot guardado en `CartItem.unit_price`.
- `PATCH /cart/items/{id}` y `DELETE /cart/items/{id}` (`cart/router.py:89-96,99-103`) —
  `update_item`/`remove_item` (`cart/service.py:351-395,534-541`), mismo chequeo de
  disponibilidad recalculado excluyendo la propia línea.
- `GET /cart` (`serialize_cart`, `cart/service.py:204-263`) muestra, además del total, un
  **preview** de descuento línea a línea con `promotions.best_line_discount`
  (`:229-234`) para percent/fixed — los combos no se les aplica esto porque ya llevan su
  propio ahorro (`:227-228`). Este preview es solo informativo: el descuento real se recalcula
  íntegro al cobrar (§9).

---

## 3. Los caminos de creación del pedido

### 3.1 Camino 1 — Comensal por QR (`POST /cart/submit`)

- **Entrada**: ninguna en el body; toma el `x-session-token` de la cabecera
  (`SessionContext`, `cart/router.py:140-159`). Rate-limit por mesa (`:157`).
- **Módulos atravesados, en orden**: `cart/router.py:140` → `cart.service.submit_cart`
  (`cart/service.py:471-531`) → (sin tocar `catalog` ni `inventory`: el precio y las opciones
  ya estaban fijados en el carrito).
- **Qué hace exactamente** (`cart/service.py:471-531`):
  1. Carga el carrito abierto del comensal (`_load_open_cart`, `:482`); rechaza si está vacío
     (`:483-484`, `409`).
  2. Chequeo preventivo de disponibilidad agregado (`check_availability`, `:486`) — sigue sin
     lock, es UX, no la validación real.
  3. Crea `CustomerOrder` (`:489-497`) con `channel="qr"`, **`status="recibida"`**,
     `user_id=None` (comentario: "lo envió el comensal, no un usuario del sistema", `:496`).
  4. Por cada línea del carrito, crea `OrderItem` con el `unit_price` **copiado tal cual del
     carrito** (snapshot, `:507`, comentario explícito) y `estado_cocina="pendiente"`
     (`:510`), y sus `OrderItemOption` (`:514-515`).
  5. Marca el carrito `confirmado` (`:517`) — así `_get_or_create_open_cart` le abrirá uno
     nuevo si el comensal pide otra ronda.
  6. `db.commit()` (`:518`). **No se descuenta inventario en ningún punto de este camino.**
- **Cálculo aplicado**: ninguno nuevo — los precios ya vienen del carrito (`compute_line_price`
  se ejecutó al añadir cada línea, §2). Cero evaluación de promociones aquí (el preview del
  carrito ya se mostró; el descuento real se recalcula al cobrar, §9).
- **Descuento de inventario**: cero. El comentario del propio módulo lo dice tres veces
  (`cart/service.py:1-4`, `:475-478`).
- **Tablas escritas, en orden**: `customer_orders` (1 fila) → `order_items` (N filas) →
  `order_item_options` (M filas) → `carts` (`UPDATE status='confirmado'` sobre la fila
  existente). Sin `inventory_movements`.
- **Estado resultante**: `recibida`. El router publica `order_created` **después** del commit
  del servicio (`cart/router.py:149-158`, comentario explícito: "si la transacción fallara no
  puede haber salido un evento anunciando un pedido que no existe").
- **Frontend**: `PublicMenuComponent.sendOrder()` (`public-menu.component.ts:743-761`) llama
  `DinerService.submitCart()` (`diner.service.ts:129-131`), limpia el carrito local
  (`cart.clear()`) y refresca "mis pedidos".

### 3.2 Camino 2 — Manual staff, comanda completa (`POST /orders`) — sin uso confirmado en la UI

- **Entrada**: `OrderCreate` (`orders/schemas.py:116-122`) — `channel` (default `COUNTER`),
  `participant_id?`, `dining_table_id?`, `customer_name?`, `notes?`, y **`items[]` obligatorio
  (mínimo 1)**: la comanda entera se manda de una sola vez, a diferencia del camino 3.
- **Módulos atravesados**: `orders/router.py:405-418` → `orders.service.create_order`
  (`orders/service.py:37-137`) → `catalog/line_pricing.py` (precio, opciones) →
  `promotions.service.expand_combo` (si hay combo) → `orders/consumption.deduct_order_items`
  → `inventory/stock.py`.
- **Qué hace exactamente** (`orders/service.py:37-137`):
  1. Si viene `participant_id`, lo resuelve y toma su mesa/nombre (`:42-49`); si no, valida
     que la mesa exista (`:51-52`).
  2. Crea `CustomerOrder` (`:55-70`) con **`status="abierta"` directamente** — nace ya
     confirmada, nunca pasa por `recibida` (comentario propio: "Nace confirmada... es el mismo
     punto de descuento que la confirmación, por la otra puerta", `:1-8`). Si la mesa no tenía
     `table_session`, la abre (`get_or_create_table_session_id`, `:59-63`).
  3. Por cada línea: si es combo, expande vía `promotions.expand_combo` (`:80`) y crea un
     `OrderItem` por componente a precio normal (`:81-92`); si es producto, valida variante
     activa (`:96-97`), carga/valida opciones (`:102`), crea el `OrderItem` y calcula
     `unit_price = compute_line_price(variant, options)` (`:117`).
  4. `deduct_order_items(db, entries, user_id, reference_id=order.id)` (`:124`) — **descuenta
     inventario en la misma transacción de creación**. Si falta un insumo, la excepción
     revierte toda la comanda (`:127-133`, ninguna comanda parcial).
  5. `db.commit()` (`:126`).
- **Cálculo de precio**: idéntico a §2 (`compute_line_price`), pero re-ejecutado aquí en vez
  de heredado de un carrito, porque este camino no pasa por un carrito.
- **Descuento de inventario**: inmediato, vía la misma función de bajo nivel que usa el
  camino 3 (`deduct_order_items` → `deduct_order_item` → `plan_line_consumption` →
  `record_movement(..., type="out", reason=VENTA, reference_type=REF_ORDER)`,
  `orders/consumption.py:52-102`).
- **Promociones**: cero evaluación de percent/fixed aquí — solo la expansión de combo
  (`expand_combo`) a precio normal; el ahorro del combo, igual que en el resto de caminos, se
  calcularía al cobrar.
- **Tablas escritas, en orden**: `table_sessions` (solo si no existía) → `customer_orders` (1
  fila, `status='abierta'` desde el alta) → `order_items` (N) → `order_item_options` (M) →
  `inventory_movements` (una fila `'out'` por línea de consumo, con lock de fila en
  `inventory_items` vía `SELECT...FOR UPDATE`, `inventory/stock.py:66-77`).
  **Sin** `carts` ni `cart_items`: este camino no pasa por el carrito del comensal.
- **Estado resultante**: `abierta` (nunca `recibida`).
- **Verificación de uso en frontend**: `grep -rn "OrderCreate" pos-backend/app --include=*.py`
  solo aparece en `orders/service.py`, `orders/schemas.py` y `orders/router.py` — ningún otro
  módulo backend lo referencia. En `pos-heladeria`, ningún fichero bajo `src/app` hace
  `POST` a `${...}/orders` sin sufijo (`DiningSessionService` solo expone `GET /orders`,
  `dining-session.service.ts:28-31`; `TableService.ordersUrl` solo se usa para `/move`,
  `/merge` y `/group/.../bill`, `table.service.ts:20,117,125,133`). Tampoco aparece en los 12
  scripts de `app/scripts/test_*.py`. **Este endpoint existe y funciona, pero no tiene ningún
  disparador conocido en el sistema actual** — ver Hallazgo H1 (§11).

### 3.3 Camino 3 — Manual staff, ítem por ítem en la terminal de mesas (`POST /orders/tables/{id}/items`)

Este es el que la terminal usa de verdad para que el mesero registre pedidos sin pasar por el
QR del comensal.

- **Entrada**: `OrderItemIn` (`orders/schemas.py:100-113`) — **una sola línea** por
  llamada: `product_variant_id` **o** `combo_id`, `quantity`, `option_ids[]`, `notes`. La
  terminal arma varias líneas en memoria (`draftLines`,
  `pos-terminal.store.ts:201,747-770,772-790` para producto/combo respectivamente, tras
  seleccionarlas del catálogo embebido en la terminal vía `product-select.component.ts` /
  `combo-select.component.ts`) y las manda **una petición HTTP por línea**, en un bucle
  (`saveOrder()`, `pos-terminal.store.ts:862-889`, líneas `869-880`).
- **Módulos atravesados**: `orders/router.py:203-216` → `orders.consolidation.add_item_to_table`
  (`consolidation.py:180-234`) → `orders.consolidation.get_or_create_open_order` (`:69-103`) →
  `catalog/line_pricing.py` / `promotions.expand_combo` → `orders/consumption.deduct_order_items`
  → `inventory/stock.py`.
- **Qué hace exactamente** (`consolidation.py:180-234`):
  1. Si `combo_id`, expande a componentes reales a precio normal (`:190-194`, mismo mecanismo
     que el camino 2); si no, valida variante activa y opciones (`:196-200`).
  2. `get_or_create_open_order(db, table_id, user.id)` (`:203`, definido en `:69-103`):
     busca una `CustomerOrder` **`status='abierta'` y `channel='waiter'`** de esa mesa,
     ordenada por `created_at` (`:78-87`) y la reutiliza; si no hay ninguna, crea una nueva con
     `channel="waiter"` explícito (`:90-96`). También abre/reutiliza la `table_session` de la
     mesa (`get_or_create_table_session_id`, `:88`, mismo helper que usa el camino 2).
     Comentario propio: "ya no existe el índice único de una orden abierta por mesa... la mesa
     puede tener varios pedidos a la vez (uno por comensal)" (`:70-77`) — así que el pedido del
     mesero convive con los pedidos QR de los comensales en la misma mesa, cada uno en su
     propia fila de `customer_orders`.
  3. Crea el/los `OrderItem` con `participant_id=None` explícito (comentario: "ítem agregado
     por el mesero, sin comensal asignado", `:209`) y `estado_cocina="pendiente"` (`:215`).
  4. `deduct_order_items(...)` (`:223`) — descuenta inventario en la misma transacción, **igual
     que el camino 2**.
  5. `db.commit()` (`:225`).
- **Cálculo, descuento de inventario y promociones**: idénticos en mecanismo al camino 2
  (mismas funciones: `compute_line_price`, `expand_combo`, `deduct_order_items` →
  `plan_line_consumption` → `record_movement`). La diferencia no está en el cálculo sino en la
  forma de invocarlo: una línea por HTTP request aquí, todas de una vez allá (§11 detalla las
  consecuencias).
- **Tablas escritas, en orden, por cada línea**: `table_sessions` (solo la primera vez) →
  `customer_orders` (solo si no había una `abierta`/`waiter` reutilizable) → `order_items` (1)
  → `order_item_options` (si aplica) → `inventory_movements` (1 por línea de consumo).
- **Estado resultante**: `abierta` desde el alta (igual que el camino 2 — nunca pasa por
  `recibida`).
- **Recepción visual**: la terminal recarga tras `saveOrder()` (`pos-terminal.store.ts:882`,
  `await this.reload()`) y selecciona la orden recién tocada (`:883`); no depende del
  mecanismo de "campana" de §5 porque quien la crea es la misma persona que la está mirando.

### 3.4 Nota — `consolidate_table`, un cuarto camino sin UI

`POST /orders/tables/{id}/consolidate` (`orders/router.py:188-200`) →
`consolidation.consolidate_table` (`:106-169`) agrupa **todos** los carritos `abierto` con
ítems de los comensales de una mesa (no solo uno) dentro de una `CustomerOrder` del mesero
(reutilizando `get_or_create_open_order`), copiando el `unit_price` tal cual del `CartItem`
(`:140`, snapshot — sin recalcular precio) y descontando inventario en bloque
(`deduct_order_items`, `:158`). Marca cada carrito consolidado como `confirmado` (`:156`). No
es un alta "desde cero": opera sobre carritos que el comensal **ya llenó pero no envió** (si
no hay carritos con ítems, `409`, `:121-124`). `grep -rn "consolidate" pos-heladeria/src/app
--include=*.ts` no devuelve ningún resultado: **tampoco tiene disparador en el frontend
actual**.

---

## 4. Confirmación del staff — el único punto de descuento del camino QR

El camino 1 (QR) es el único que llega a `recibida` sin haber tocado inventario. El paso que lo
convierte en `abierta` y descuenta stock es:

- **Endpoint**: `POST /orders/{order_id}/confirm` (`orders/router.py:165-184`) →
  `checkout.confirm_order` (`orders/checkout.py:307-352`).
- **Disparador en la UI**: `PendingOrdersPanelComponent`
  (`pending-orders-panel.component.ts:1-24`) lista `GET /orders?status=recibida`
  (comentario propio: "es el paso que compromete el inventario... por eso no salen en el KDS y
  hay que pedirlos aparte", `:18-24`) y el botón "Confirmar" llama
  `DiningSessionService.confirmOrder` (`dining-session.service.ts:38-47`).
- **Qué hace** (`checkout.py:307-352`):
  1. Carga la orden **con `SELECT...FOR UPDATE`** (`:320`) y exige `status == 'recibida'`
     (`:324-328`).
  2. `deduct_order_items(db, entries, user.id, reference_id=order.id)` (`:339`) — misma función
     de bajo nivel que usan los caminos 2 y 3; aquí es la validación real y atómica, a
     diferencia del chequeo preventivo del carrito (comentario `:311-315`).
  3. `order.status = "abierta"`; `order.version += 1` (`:341-342`); commit.
- Si falta stock, la transacción entera revierte y el pedido **sigue en `recibida`**, listo
  para reintentar tras reponer (`checkout.py:313-315`; el frontend muestra el error sin
  quitarlo de la bandeja, `pending-orders-panel.component.ts:104-109`).
- Alternativa: "Rechazar" en el mismo panel llama `cancelOrder`
  (`dining-session.service.ts:50-54`) → `checkout.cancel_order` con el pedido aún en
  `recibida` → como nunca se descontó (`_NOT_DEDUCTED = ("recibida",)`, `checkout.py:51`), la
  cancelación no genera ningún movimiento de inventario (§7).

---

## 5. Aviso al personal ("recibir la orden")

- Backend: tras el commit de `submit_cart`, el router publica `events.order_created`
  (`cart/router.py:149-158`) sobre Redis Streams (`core/event_bus.py`).
- Frontend: `PosTerminalStore.connectRealtime()` (`pos-terminal.store.ts:591-612`) escucha
  `order.created` (entre otros) y dispara una recarga debounced (`scheduleReload`,
  `:624-632`) en vez de sonar directamente — el comentario explica por qué: sonar desde el
  evento haría que un replay tras reconectar repitiera la campana por pedidos que el cajero ya
  vio (`:582-589`).
- `announcePending()` (`:651-658`) es quien realmente decide sonar: compara los ids de pedidos
  `recibida` contra los ya vistos (`seenPending`) y solo llama `sound.bell()` si hay alguno
  nuevo **y** ya se sembró el conjunto (`pendingSeeded`, para no sonar al abrir la pantalla con
  pedidos viejos pendientes, `:648-650`).

---

## 6. Preparación en cocina (estado por ítem)

Independiente del `status` del pedido, cada `OrderItem` tiene su propio `estado_cocina`
(`pendiente → en_preparacion → listo`, o `anulado`), gobernado por `_ALLOWED`
(`orders/kitchen.py:30-33`) — **siempre hacia adelante**, y con el salto directo
`pendiente → listo` permitido a propósito ("quien toma el pedido es quien lo prepara",
`:27-29`).

- `PATCH /orders/items/{id}/kitchen` (`orders/router.py:220-251`) →
  `kitchen.transition_kitchen` (`kitchen.py:43-60`). Disparado desde
  `DiningSessionService.updateItemKitchen` (`dining-session.service.ts:56-68`).
- `POST /orders/{id}/ready` (`orders/router.py:254-278`) →
  `kitchen.mark_order_ready` (`kitchen.py:63-90`) marca todos los ítems en curso de una vez —
  existe para que el cajero no tenga que ir ítem por ítem antes de cobrar
  (`dining-session.service.ts:70-78`).
- `POST /orders/items/{id}/void` (`orders/router.py:281-302`) → `kitchen.void_item`
  (`kitchen.py:93-176`): anula un ítem y, si estaba `pendiente` (cocina no lo tocó), revierte
  su inventario (`reverse_order_items`, `:132-136`); si venía en curso, no se revierte (ya se
  consumió). Admite reemplazo simultáneo (`data.replacement`) que crea un `OrderItem` nuevo con
  `void_de` apuntando al anulado y descuenta su propio inventario (`:143-161`).

---

## 7. Cancelación de un pedido

Dos entradas distintas, mismo motor:

- **Staff, cualquier estado no terminal**: `POST /orders/{id}/cancel`
  (`orders/router.py:353-371`) → `checkout.cancel_order(..., user=user, participant=None)`.
- **Comensal, solo antes de que cocina empiece**: `POST /cart/orders/{id}/cancel`
  (`cart/router.py:175-193`) → `cart.service.cancel_my_order` (`cart/service.py:410-456`), que
  valida la restricción de estado (`recibida` siempre; `abierta` solo si **todos** los ítems
  siguen `pendiente`, `:415-419,436-445`) y delega en `checkout.cancel_order(..., user=None,
  participant=participant)`.

`checkout.cancel_order` (`checkout.py:357-461`) es **asimétrico**, no una reversa simétrica
(comentario explícito, `:364-380`):
- Si la orden nunca se confirmó (`recibida`, `_NOT_DEDUCTED`), cero movimientos de inventario.
- Ítem `pendiente`: se revierte (`reverse_order_items` → movimiento `'in'`).
- Ítem `en_preparacion`/`listo`: **no vuelve al stock** (el insumo ya se combinó); se registra
  como pérdida en `audit_logs` vía `record_audit` (`:437-450`), no como movimiento de
  inventario (evita doble descuento).
- Ítem `anulado`: ya resuelto por `void_item`, se ignora aquí.

---

## 8. Ciclo de vida completo: estados y quién los cambia

### 8.1 `CustomerOrder.status`

Documentado en el propio modelo (`customer_order.py:14-28`):

```
recibida → abierta → bloqueada → pagada
         ↘  cancelada (terminal, desde cualquier estado no terminal)
```

| Transición | Quién la dispara | Endpoint / función | Efecto en inventario |
|---|---|---|---|
| (alta) → `recibida` | Comensal (QR) | `POST /cart/submit` → `submit_cart` | Ninguno |
| (alta) → `abierta` | Staff (comanda directa o ítem de mesero) | `POST /orders` → `create_order`, o `POST /orders/tables/{id}/items` → `add_item_to_table` | Descuenta al crear |
| `recibida` → `abierta` | Staff, desde la bandeja de pendientes | `POST /orders/{id}/confirm` → `confirm_order` | Descuenta aquí |
| `abierta` → `bloqueada` | Staff (camino **legado**, sin UI — ver §9) | `POST /orders/{id}/block` → `block_order` | Ninguno |
| `bloqueada` → `pagada` | Staff (camino **legado**, sin UI) | `POST /orders/{id}/pay` → `pay_order` | Ninguno (ya descontado) |
| `abierta` → `pagada` | Staff, desde el cierre de sesión (camino **real**) | `POST /table-sessions/{id}/close` → `close_session` | Ninguno (ya descontado); salta `bloqueada` |
| cualquier no terminal → `cancelada` | Staff (cualquier estado) o comensal (restringido) | `POST /orders/{id}/cancel` / `POST /cart/orders/{id}/cancel` → `cancel_order` | Asimétrico, ver §7 |

El comentario del propio router deja constancia de por qué no existe un `PATCH .../status`
genérico: "asignaba cualquier estado sin validar la transición... permitía pasar un pedido de
'recibida' a 'abierta' esquivando `confirm_order`... y dejaba el inventario sobrestimado sin
que nadie se enterara" (`orders/router.py:443-452`) — cada transición legítima tiene su propio
endpoint con sus propias reglas.

### 8.2 `OrderItem.estado_cocina`

`pendiente → en_preparacion → listo`, o `anulado` (§6). Vive en la fila del ítem, no del
pedido, e independiente del `status` de pago (`order_item.py:49-51`).

---

## 9. Cobro y factura en la terminal de mesas

### 9.1 El camino legado (`block` → `pay`), verificado sin uso

- `POST /orders/{id}/block` (`orders/router.py:315-325`) → `checkout.block_order`
  (`checkout.py:71-122`): toma `SELECT...FOR UPDATE` sobre la orden (`:74`), exige
  `status='abierta'` y coincidencia de `version` (lock optimista, `:78-86`), rechaza si hay
  ítems `EN_CURSO` sin terminar (`:88-109`), y pasa a `bloqueada` (`:111-112`).
- `POST /orders/{id}/pay` (`orders/router.py:328-350`) → `checkout.pay_order`
  (`checkout.py:250-302`): exige `status='bloqueada'` (`:252-256`), arma las líneas
  (`order_sale_lines`), evalúa promociones (`promotions.evaluate` + `combo_discount_for_lines`,
  `:266-267`) y llama `build_sale` **sin pasar `invoice_prefix`**
  (`checkout.py:271-285` — compárese con la firma de `build_sale`, que acepta
  `invoice_prefix: str = ""`, `sales/builder.py:83`, y con las dos llamadas de
  `table_sessions/service.py:574,669`, que sí lo pasan). Marca `order.status = "pagada"`
  (`:286`) sin volver a descontar inventario (`:287`).
- **Verificación de uso**: `grep -rn "orders/\${.*}/block\|orders/\${.*}/pay\b|blockOrder|payOrder"
  pos-heladeria/src/app --include=*.ts` → 0 resultados. Los propios comentarios del frontend lo
  confirman: `table-session.service.ts:33` ("Sustituye al ciclo `block` → `pay` por orden") y
  `pos-terminal.store.ts:1101` ("en vez del antiguo `block` → `pay` → `release` por orden").

### 9.2 El camino real: `POST /table-sessions/{id}/close`

- **Disparador UI**: panel de cobro de la terminal → `TableSessionService.close`
  (`table-session.service.ts:64`) → `POST /table-sessions/{id}/close`.
- **Entrada** (`CloseSessionIn`, `table_sessions/schemas.py:107-132`): `cash_shift_id`,
  `billing_mode` (`unified` | `split`), y según el modo — `unified`: `payments[]`,
  `discount`, `tax`, `tip`, `customer_name?` en la raíz; `split`: `splits[]`, cada uno con su
  propio `payments[]`/`discount`/`tax`/`tip`/`participant_id` (los de la raíz se rechazan si
  vienen poblados en modo `split`, `service.py:592-597`, para que una propina en el sitio
  equivocado no se pierda en silencio).
- **Módulos atravesados**: `table_sessions/router.py:120-138` →
  `table_sessions.service.close_session` (`:212-300`) → `_close_unified`/`_close_split`
  (`:539-671`) → `promotions.evaluate` + `promotions.combo_discount_for_lines` →
  `sales.builder.build_sale` → `invoices.service.issue_for_sale`.
- **Qué hace `close_session`** (`:212-300`):
  1. Carga la sesión con `FOR UPDATE` (`:228`) — evita el doble cobro de dos `POST /close`
     concurrentes (comentario `:41-44`).
  2. Exige turno de caja abierto (`ensure_open_shift`) y que la sesión tenga pedidos cobrables
     (`_billable_orders`, excluye `cancelada`/`pagada`, `:126-136`).
  3. `_assert_closable` (`:184-209`): rechaza si hay pedidos `recibida` sin confirmar o ítems
     `EN_CURSO` en cualquier orden de la sesión.
  4. Según `billing_mode`, arma 1 venta (`unified`) o N ventas —una por comensal con consumo—
     (`split`).
  5. **Todos** los `orders` de la sesión pasan a `pagada` directamente (`:248-249`) — no hay
     paso intermedio por `bloqueada`.
  6. Cierra la `table_session`, sus comensales y sus carritos abandonados
     (`checkout.close_table_sessions`, reutilizado del módulo `orders`, `:252`), libera la mesa
     (`:254-256`) y hace un único `commit` (`:258`) para toda la cascada.
- **Cálculo de descuento por venta** (idéntico en `_close_unified`, `:554-558`, y por cada
  bloque de `_close_split`, `:652-655`):
  `discount = data.discount (manual) + promo_discount (percent/fixed vía promotions.evaluate)
  + combo_discount (vía promotions.combo_discount_for_lines)`.
  `promotion_id` final: si las líneas de la venta usan **un solo** combo, ese `combo_id` manda
  sobre el `promo_id` de percent/fixed (`combo_ids_used`, `:557-558` / `:654-655`) — es una
  simplificación de trazabilidad: `Sale.promotion_id` es un solo campo aunque hayan aplicado
  varias promociones a la vez (el desglose completo vive en `PromotionResult.lines`,
  `promotions/service.py:176-190`, pero no se persiste en `Sale`).
- **`build_sale`** (`sales/builder.py:67-179`), compartido por los cuatro caminos de cobro
  (comentario propio, `:1-15,92-95`): crea `Sale` (`:100-116`), sus `SaleItem` (`:119-130`),
  valida que el pago cubra el total y que ningún pago no-efectivo supere el total (`:149-163`,
  para que el vuelto solo salga del efectivo, relevante para el arqueo de `cash`), fija
  `paid_amount`/`change_given` (`:170-171`) y **emite la factura en la misma transacción**
  (`issue_for_sale`, `:176-178`) — nunca hace `commit` por sí mismo, se une a la transacción
  del caller.
- **Frontend, tras el cobro**: `PosTerminalStore.onCharged(closed)`
  (`pos-terminal.store.ts:1127-1135`) → `loadReceipts(closed)` (`:1143-1151`) trae cada
  `Sale` por `GET /sales/{id}` y la convierte a un objeto imprimible (`saleToReceipt`,
  `receipt.util.ts`); `printReceipt()` (`:1170-1177`) genera el HTML del ticket
  (`buildReceiptHtml`) y lo imprime por un iframe oculto (`printReceiptHtml`,
  `receipt.util.ts:1-9` según `mapa-sistema.md:504-507`) — esta es la "generación de factura en
  la terminal de mesas" que pidió el usuario: la factura ya existe en base de datos desde el
  `commit` del cierre; lo que hace la terminal aquí es solo **recuperarla e imprimirla**.

### 9.3 Numeración de factura

- **Función exacta**: `invoices.service._next_number(db, prefix)` (`invoices/service.py:30-42`)
  — bloquea la fila de `InvoiceCounter` para ese `prefix` (`with_for_update`, `:34`), la crea
  con `next_number=1` si no existe (`:36-39`), y devuelve/incrementa el consecutivo. El lock
  serializa las ventas del mismo prefijo mientras dura la transacción (comentario `:50-53`:
  "es el precio de un consecutivo sin huecos").
- **Origen del prefijo**: `Tenant.invoice_prefix` (`core/models.py:65`), editable en
  `PATCH /tenant` (`tenant/router.py:68-71`, normalizado a mayúsculas). Se pasa como
  `invoice_prefix=tenant.invoice_prefix or ""` desde:
  - `table_sessions/router.py:136` (el camino real, §9.2) — **sí** usa el prefijo del tenant.
  - `sales/router.py:53` (venta de mostrador) — igual, sí lo usa.
  - `orders/checkout.py:271-285` (`pay_order`, legado) — **no** lo pasa (queda en `""`), como
    se detalló en §9.1.
- `issue_for_sale` (`invoices/service.py:45-76`) es idempotente: si la venta ya tiene factura
  (`Invoice.sale_id` es único), devuelve la existente en vez de fallar (`:54-58`) — protección
  contra reintentos del mismo `build_sale`.
- Comentario propio del módulo: antes de este diseño hubo "20 ventas reales, cero facturas" por
  depender de un botón manual (`invoices/service.py:1-9`, ya citado también en
  `mapa-sistema.md:308-310`); ahora se emite dentro de la misma transacción del cobro.

---

## 10. Aplicación de promociones y descuento de inventario — resumen por camino

| | Camino 1 (QR) | Camino 2 (`POST /orders`) | Camino 3 (`.../items`) | Confirmación (`confirm_order`) | Cobro (`close_session`) |
|---|---|---|---|---|---|
| **Precio de línea** | `compute_line_price` al añadir al carrito (§2) | `compute_line_price` en el propio alta | `compute_line_price` en el propio alta | No recalcula | No recalcula (usa `unit_price` ya fijado en `order_items`) |
| **Combo (selección explícita)** | `expand_combo` al añadir al carrito | `expand_combo` en el alta | `expand_combo` en el alta | — | `combo_discount_for_lines` calcula el ahorro real |
| **Percent/fixed automático** | Solo *preview* (`best_line_discount` en `GET /cart`) | No se evalúa | No se evalúa | No se evalúa | `promotions.evaluate` (real, vía `evaluate_detailed`) |
| **Descuento de inventario** | Ninguno | `deduct_order_items` en el alta | `deduct_order_items` en el alta | `deduct_order_items` (único punto para el camino QR) | Ninguno (ya descontado antes) |
| **Función de bajo nivel** | — | `plan_line_consumption` → `record_movement('out')` | igual | igual | — |

El punto de fondo: **las promociones automáticas (percent/fixed) nunca se aplican en el
momento de crear o confirmar el pedido, en ningún camino** — solo se calculan al cobrar
(`promotions.evaluate` dentro de `pay_order`/`_close_unified`/`_close_split`). Lo que el
comensal ve en el carrito y lo que el cajero ve en la cuenta (`compute_bill`,
`table_sessions/service.py:139-181`, que llama a las mismas funciones) son *previews*; el
número que efectivamente se cobra sale de la re-evaluación al cerrar.

---

## 11. Puntos donde dos caminos hacen "lo mismo" con código distinto

1. **`create_order` (camino 2) y `add_item_to_table` (camino 3) duplican, casi línea por
   línea, la secuencia "expandir combo o validar variante+opciones → crear `OrderItem` →
   `deduct_order_items`"** (`orders/service.py:74-124` vs `orders/consolidation.py:190-223`).
   No comparten una función común más allá de las piezas de más bajo nivel
   (`compute_line_price`, `expand_combo`, `deduct_order_items`); el ensamblaje del `OrderItem`
   y el manejo de errores están escritos dos veces.
2. **Canal por defecto distinto entre los dos caminos de staff.** `create_order` (camino 2)
   usa `OrderChannel.COUNTER` por defecto si no se especifica (`orders/schemas.py:117`),
   mientras que `add_item_to_table` (camino 3) siempre fuerza `channel="waiter"`
   (`consolidation.py:93`). Como `get_or_create_open_order` — el helper que decide si
   reutilizar una orden abierta de la mesa o crear una nueva — filtra explícitamente por
   `CustomerOrder.channel == 'waiter'` (`consolidation.py:83`), una comanda creada por el
   camino 2 sobre una mesa (con `dining_table_id` pero sin channel explícito → queda
   `counter`) **no sería encontrada ni reutilizada** por una llamada posterior del camino 3
   sobre la misma mesa: cada uno crearía su propia fila de `customer_orders`, aunque ambos
   caminos, si se usaran juntos, pretenden atender la misma mesa. No se pudo verificar en la
   práctica porque el camino 2 no tiene caller conocido (§3.2) — se marca `SUPOSICIÓN` sobre
   si esta interacción llegó a probarse alguna vez: *¿se probó crear una comanda vía
   `POST /orders` con `dining_table_id` sobre una mesa que luego el mesero también atendiera
   desde la terminal?*
3. **Numeración de factura inconsistente entre los cuatro caminos de cobro.**
   `table_sessions/service.py` (unified y split) y `sales/router.py` (mostrador) pasan
   `invoice_prefix=tenant.invoice_prefix`; `orders/checkout.py:pay_order` no lo pasa y factura
   siempre con prefijo vacío (§9.1, §9.3). Como `pay_order` no tiene caller en la UI, esto no
   afecta la operación actual, pero el propio motor de facturación (`invoices/service.py`)
   trataría ambos prefijos como series independientes si algún día se volviera a usar ese
   endpoint — dos facturas con el mismo número pero prefijo distinto (una con el prefijo del
   negocio, otra sin él) conviven sin colisión aparente en la tabla, lo cual podría confundir a
   la gestoría si esa vía se reactivara sin arreglarlo.
4. **El chequeo de disponibilidad de inventario existe en dos "niveles de verdad" a
   propósito**: el preventivo sin lock (`check_availability`, usado por el carrito) y el real
   con lock por fila (`record_movement` dentro de `deduct_order_items`, usado por los tres
   caminos de creación de staff y por `confirm_order`). Esto **no** es una duplicación
   accidental — está documentado como decisión de diseño (`catalog/line_pricing.py:5-8`) — pero
   sí es un punto donde "parece que se valida dos veces lo mismo" con resultados que pueden
   divergir (el carrito dijo que había stock; la confirmación, con datos más frescos, puede
   rechazar).
5. **`compute_bill` (previsualización de cuenta) y `_close_split`/`_close_unified` (cobro
   real) recalculan promociones con las mismas funciones pero en llamadas separadas**
   (`table_sessions/service.py:139-181` vs `:539-671`) — no hay riesgo de inconsistencia de
   fórmula (mismas funciones, `promotions.evaluate` + `combo_discount_for_lines`), pero sí de
   *momento*: el preview se calcula cuando el cajero abre el panel; el cobro real, cuando
   confirma — si algo cambió en el medio (un ítem se anuló, una promoción expiró), el número
   final puede diferir del preview, sin que haya ningún mecanismo que fuerce un refresco
   automático antes de cobrar (`billStale` en el frontend solo *marca* como desactualizado, no
   recarga solo — comentario explícito `pos-terminal.store.ts:1085-1094`).

---

## 12. Hallazgos adicionales (registrados, no corregidos — Principio III)

**H1. `POST /orders` (creación manual de comanda completa) no tiene ningún disparador conocido
en el sistema actual.** Verificado por `grep` cruzado en ambos repos (§3.2): ni la UI de
`pos-heladeria`, ni los 12 scripts de test de `pos-backend`, lo invocan. El endpoint funciona
(tiene su propio flujo completo de precio, combo e inventario) pero es, hasta donde el código
permite confirmar, alcanzable solo por API directa (Swagger/Postman/integración externa no
versionada). `SUPOSICIÓN`: podría ser una superficie pensada para una integración futura (POS
externo, pedidos telefónicos) o un resto de una pantalla de staff que existió y se retiró sin
limpiar el backend — pregunta abierta al negocio: *¿existe o existió algún cliente (interno o
de terceros) que use `POST /orders` directamente?*

**H2. `POST /orders/{id}/block` y `POST /orders/{id}/pay` (el ciclo de cobro legado por
pedido) están confirmados sin caller en la UI**, y los propios comentarios del frontend
(`table-session.service.ts:33`, `pos-terminal.store.ts:1101`) documentan que fueron
reemplazados por `POST /table-sessions/{id}/close`. A diferencia de H1, aquí sí hay evidencia
textual de que se trata de una migración deliberada, no de un endpoint que nunca se usó. El
estado `bloqueada` del enum `CustomerOrder.status` (`customer_order.py:82`) sigue siendo un
valor válido en la base de datos pero, con el flujo actual, ningún pedido llega a él.

**H3. El bug de prefijo de factura en `pay_order` (§9.1, §11.3) es inofensivo en la práctica
actual** porque el endpoint que lo contiene no tiene caller — pero si el negocio reactivara ese
camino (por ejemplo, para permitir cobro pedido-a-pedido sin cerrar toda la sesión) empezaría a
emitir facturas sin el prefijo configurado del tenant, sin ningún error visible que lo delate.

**H4. `consolidate_table` (camino 4, §3.4) tampoco tiene disparador en la UI actual.** A
diferencia de `add_item_to_table` (que sí tiene botón), no se encontró ningún flujo de la
terminal que ofrezca "consolidar los carritos sin enviar de esta mesa" como acción explícita.
`SUPOSICIÓN`: puede ser una función pensada para un caso de uso que la UI resolvió de otra
forma (el mesero simplemente espera a que el comensal envíe, o agrega el ítem él mismo por el
camino 3) — pregunta abierta: *¿se dejó de necesitar `consolidate_table` cuando se agregó
`add_item_to_table`, o falta conectarlo a algún botón?*

---

## 13. Qué se graba en qué tablas y en qué orden — vista consolidada

Para el camino completo QR → factura (caminos 1 + confirmación + cierre unificado), en el
orden real de escritura a través de toda la vida del pedido:

1. `table_sessions` (al escanear el QR o al primer `add_item_to_table` de la mesa) — 1 fila.
2. `session_participants` (al escanear el QR) — 1 fila por comensal.
3. `carts` (al escanear el QR) — 1 fila por comensal, `status='abierto'`.
4. `cart_items` / `cart_item_options` (al añadir cada línea) — N filas.
5. `customer_orders` (al enviar el pedido, `POST /cart/submit`) — 1 fila, `status='recibida'`.
6. `order_items` / `order_item_options` (mismo momento, copiados del carrito) — N/M filas.
7. `carts.status` → `'confirmado'` (`UPDATE`, mismo commit que el punto 5).
8. `inventory_movements` (al confirmar, `POST /orders/{id}/confirm`) — 1 fila `'out'` por
   línea de consumo (receta + opciones), con `inventory_items.current_stock` decrementado en
   la misma transacción (`UPDATE`, bajo lock de fila).
9. `customer_orders.status` → `'abierta'`, `version += 1` (`UPDATE`, mismo commit que el 8).
10. (repite 5-9, o 3.2/3.3, por cada pedido adicional de la sesión — QR de otro comensal, o
    ítems directos del mesero.)
11. `sales` (al cerrar la sesión, `POST /table-sessions/{id}/close`) — 1 fila (`unified`) o N
    filas (`split`, una por comensal con consumo).
12. `sale_items` (mismo commit) — copia de las líneas cobrables de los `order_items`.
13. `payments` (mismo commit) — uno por método de pago recibido.
14. `invoices` (mismo commit, `issue_for_sale`) — 1 fila por `Sale`, con el número tomado de
    `invoice_counters` bajo lock.
15. `invoice_counters.next_number` → incrementado (`UPDATE`, mismo commit).
16. `customer_orders.status` → `'pagada'` para todos los pedidos de la sesión (`UPDATE`, mismo
    commit).
17. `table_sessions.status` → `'closed'`, `session_participants.status` → `'closed'`,
    `carts.status` → `'abandonado'` para los que quedaran abiertos (`UPDATE`, mismo commit).
18. `dining_tables.status` → `'libre'` (`UPDATE`, mismo commit).

Los pasos 11-18 ocurren en **una sola transacción** (`table_sessions/service.py:242-265`): si
un pago no cubre su parte, nada de esto se escribe (`db.rollback()` completo).
