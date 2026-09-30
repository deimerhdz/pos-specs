# Research — Spec 089

**Fecha**: 2026-09-30 · **Spec**: [spec.md](./spec.md) · **Plan**: [plan.md](./plan.md)

Cada decisión sale de leer el código de `pos-backend` y `pos-heladeria` (ramas `develop`, árbol limpio). No quedó ningún `NEEDS CLARIFICATION`; las dos cosas que la spec delegó al plan (cómo distinguir líneas nuevas de históricas, y la causa de la hoja en blanco) se resuelven en D1 y D9.

---

## A. Adicionales del Menú QR

### D1 — Cómo distinguir la regla nueva de la histórica sin reescribir datos

**Hallazgo**: hoy `unit_price` guarda "precio de **una** unidad" = precio de la presentación + Σ(extra × cantidad de opción) (`catalog_engine/core.py::compute_line_price`), y **once** sitios lo multiplican por la cantidad del producto: `cart/service.py` (`serialize_cart`, `submit_cart`), `cart/router.py:161`, `orders/checkout.py` (`compute_bill`, `order_sale_lines`), `table_sessions/service.py` (`compute_bill`), `sales/builder.py::SaleLine`, `orders/tables_advanced.py`. Además el frontend recalcula `unit_price × quantity` en `pos-terminal.store.ts`, `payment-attempt-review-panel.component.ts` y `public-menu.component.ts`.

**Decisión**: dos columnas aditivas con valor por defecto que reproduce el comportamiento actual.

| Columna | Tabla(s) | Tipo | Default | Significado |
|---|---|---|---|---|
| `addons_total` | `cart_items`, `order_items` | `NUMERIC(12,2) NOT NULL` | `0` | Σ(precio del adicional × cantidad elegida), cobrado **una vez por línea** |
| `per_line` | `cart_item_options`, `order_item_options` | `BOOLEAN NOT NULL` | `false` | La opción es un adicional de la regla nueva |

Fórmula única para **toda** línea: `line_total = unit_price × quantity + addons_total`.

- Línea histórica: `addons_total = 0`, `unit_price` sigue incluyendo los extras por unidad → el total es **el mismo número de siempre** (SC-003, Principio VII) sin backfill ni recálculo.
- Línea nueva del Menú QR: `unit_price` = solo precio de la presentación; `addons_total` = adicionales una vez.
- Línea de la terminal POS / pedido manual: no cambia (`per_line = false`, `addons_total = 0`).

**Por qué `per_line` en la fila de la opción y no derivarlo de `option_groups.pricing_type`**: el consumo de inventario se calcula al descontar **y de nuevo al revertir** (`reverse_order_item`, cancelación/anulación). Si el administrador cambia el tipo de un grupo entre ambos momentos, la reversa devolvería una cantidad distinta de la descontada: descuadre permanente del kardex (el propio docstring de `consumption.py` advierte esto). La marca por fila es un snapshot inmutable de qué regla se aplicó.

**Por qué `addons_total` en el ítem y no derivarlo de `Option.extra_price × quantity`**: el precio de una opción puede editarse después; el total de una línea ya pedida no puede moverse (Principio VII). (`base_unit_price` de la spec 083 sí lee `extra_price` vigente — ver D3 — pero solo alimenta el descuento porcentual, no el total.)

**Alternativas descartadas**:
- *`unit_price` fraccionario* (`(precio×cant + adicionales)/cant`): con `NUMERIC(12,2)` el redondeo rompe `unit_price × cantidad` (3 hamburguesas + $3.000 → $1.000 exacto, pero $3.000,50 → no) y contaminaría el motor de promociones.
- *Bandera por ítem `addons_per_line`*: no resuelve qué opciones de la línea son adicionales y cuáles "incluidas" en una línea nueva.
- *Reescribir histórico con `UPDATE`*: viola Principio VII.
- *Columna en `sale_items`*: innecesaria. `SaleItem.line_total` ya es autoritativo y guardado; `unit_price` sigue siendo "por unidad de producto". El snapshot JSONB de `options` gana `"per_line": true` para poder describir la línea. Las facturas emitidas no se tocan.

### D2 — Qué opciones son "adicionales" y quién decide la regla

**Decisión**: son las opciones de grupos con `pricing_type = 'con_recargo'` (Clarification 1 de la spec). Solo `cart/service.py` (`add_item`, `update_item`) las marca `per_line = true`. `ChosenOption` (NamedTuple `option, quantity`) gana un tercer campo `per_line: bool = False` — el default mantiene compatible a todo llamador existente (POS, mostrador, checkout, kitchen).

Un grupo `con_recargo` con precio $0 (topping gratis) también es adicional: su **consumo** pasa a ser por línea aunque no sume dinero (coherente con "cantidad elegida, no multiplicada").

**Bandera de emergencia**: `QR_ADDONS_PER_LINE` (bool, default `true`) en `core/config.py`. Solo gobierna la **creación** de líneas nuevas: con `false`, `add_item`/`update_item` producen líneas con la regla histórica. Toda lectura, consumo, reversa y cobro honra siempre las marcas de fila, así que apagarla no invalida líneas ya creadas. No es una dependencia nueva (Principio IX) ni exige despliegue.

### D3 — Motor de promociones y descuento (spec 083, FR-005/FR-027)

- `_cart_promo_lines` y `SaleLine.base_unit_price` restan del `unit_price` los "toppings" para obtener la base del descuento `percent`. Con la regla nueva `unit_price` ya no contiene adicionales, así que **solo se restan las opciones con `per_line = false`** (para líneas nuevas esa suma es $0 por definición: los grupos incluidos fuerzan $0). Resultado: `base_unit_price == unit_price` en líneas nuevas, y la promoción nunca descuenta adicionales (FR-005).
- El motor (`evaluate_variant_sets`) sigue recibiendo `unit_price` por unidad y `quantity`; no cambia. Sus "unidades gratis" (2×1) descuentan `unit_price` = presentación, nunca un adicional.
- `_cart_line_discount` calcula `discounted_line_total = line_total − descuento`; `discounted_unit_price` se define ahora como `(discounted_line_total − addons_total) / quantity` para conservar su significado "precio por unidad de producto". Sin adicionales, idéntico al actual.

### D4 — Consumo de inventario y comanda de cocina

`plan_line_consumption` (única definición del consumo, usada por descuento, reversa y chequeo de disponibilidad): para una `ChosenOption` con `per_line = true` el consumo es `per_unit × chosen.quantity` (**sin** `× qty`); para el resto, igual que hoy. Receta y grupos incluidos siguen `× qty`.

Como `deduct`, `reverse` y `required_consumption` pasan por esta función, las tres cuentas cuadran por construcción (FR-007). `_cart_consumption` y el reconstructor de opciones en `consolidation.py`, `cart/service.py::update_item` y `orders/kitchen.py::_item_options` deben **propagar `per_line` desde la fila** (hoy reconstruyen `ChosenOption(option, quantity)` a partir de `OrderItemOption.quantity`, perdiendo la marca). Se centraliza en un helper `chosen_from_rows(...)` que reemplaza las dos copias de `_item_options` (`orders/checkout.py:131`, `orders/kitchen.py:38`) y las reconstrucciones sueltas.

**Comanda**: la línea ya muestra opciones con su `quantity` (spec 087, FR-013, `formatQuantifiedLabel`); como `quantity` de la opción no cambia con la del producto, la comanda dice "2 hamburguesas · Tocino x1" sin cambio de presentación. Lo que corrige FR-006 es el **consumo** y el **precio**, no el texto.

### D5 — Copia de líneas entre tablas y "repartir por unidades"

Los sitios que copian una línea deben copiar `addons_total` y `per_line`:
- `orders/consolidation.py::consolidate_table` (carrito → pedido, `ci.unit_price`) y `cart/service.py::submit_cart` (mismo copiado).
- `table_sessions/service.py::set_assignments` (~l.590-610, "repartir por unidades"): hoy clona `unit_price` y las opciones **sin su `quantity`** (defecto previo, ver research D5-b). Para una línea con `addons_total > 0` los adicionales se cobran **una vez**, así que la fila original conserva `addons_total` y sus opciones `per_line`; las filas nuevas nacen con `addons_total = 0` y **sin** opciones `per_line` (los adicionales no se duplican ni se reparten). Las opciones no-`per_line` se siguen clonando como hoy.
  - *D5-b*: el clon actual pierde `OrderItemOption.quantity` (queda en 1). Es un defecto preexistente ajeno a esta spec (Principio V): **no se corrige aquí**; se anota en el registro como hallazgo y la clonación de opciones no-`per_line` queda igual.
- `orders/service.py::create_order`, `orders/kitchen.py` (reemplazo), `orders/consolidation.py::add_item_to_order` (mesero) y `sales/service.py::checkout` (mostrador): usan `compute_line_price` con `per_line = false` → sin cambio (FR-008, decisión de negocio "solo Menú QR").

### D6 — Edición de adicionales (US3)

**El backend ya lo soporta**: `PATCH /cart/items/{id}` (`cart/service.py::update_item`) acepta `options` y `notes`, revalida con `load_valid_options(..., variant=variant)`, recalcula el precio y reemplaza las filas de opciones. Solo cambia el cálculo (`compute_line_price` → precio base + `addons_total`, D1) y el marcado `per_line`. Añade además una salvaguarda de FR-013: la consulta ya filtra por el carrito **abierto del participante** (`_load_open_cart(db, participant_id)`), por lo que un comensal no puede editar líneas ajenas ni ya enviadas (el carrito se elimina al enviar).

**Frontend**: `ProductSelectComponent` ya tiene `@Input() initialSelection` (modo edición, botón "Guardar cambios") usado por la terminal POS. Se reutiliza en el Menú QR con dos ajustes: (a) un `@Input() addonsPerLine` para que el total del pie use la regla nueva en el Menú QR (la terminal POS lo deja apagado); (b) `existingQtyFor`/paso de promoción (spec 081) se ignoran al editar (la cantidad no cambia). Edge cases de la spec (adicional inactivo/sin stock, línea con producto desactivado) se resuelven con la respuesta del backend (422/409) más la marca "no disponible" ya calculable desde `menu-lookup`.

**Cantidad del producto ≠ adicional (Clarificación)**: el selector ya tiene un `quantity` de producto y un `selected[group][option]` por opción; no se auto-escala nada. La regla nueva cambia el **cálculo** del pie del selector, no la mecánica de selección.

### D7 — Reversa/rollback de la regla

Con `QR_ADDONS_PER_LINE=false` se deja de crear líneas nuevas; lo ya creado sigue correcto. **No se soporta** volver a un binario anterior una vez existan líneas con `addons_total > 0` (el binario viejo ignoraría la columna y **subcobraría**). Para ese caso se documenta en `data-model.md` el procedimiento de emergencia y se entrega el script `fold_line_addons.py` (solo `cart_items`, es decir, carritos aún no enviados, que se pueden re-plegar de forma exacta cuando `addons_total` es divisible entre la cantidad; los pedidos ya confirmados no se tocan y se listan). Ver `data-model.md` §Reversión.

---

## B. Cierre de mesa y sesión del Menú QR

### D8 — Un solo punto de emisión y qué falta

**Hallazgo** (confirma A-95): `close_table_sessions` (`orders/checkout.py`) es el único que pasa `TableSession.status` a `closed` y a cada comensal a `closed`. Emiten `session.closed` **tras el commit**: `table_sessions/service.py` (cobro/cierre, dos veces) y `core/scheduler.py::_sweep_schema`. **No** lo emiten: `orders/router.py::release_table` (botón "Liberar mesa", cajero y mesero) y `try_release_if_empty` (cierre de sesión sin órdenes, llamado desde `cart/service.py` —dos sitios, entre ellos `leave_session`— y `core/qr_context.py::_abandon_expired`).

**Decisión**: no mover la emisión dentro de `close_table_sessions` (corre antes del commit; un evento antes del commit anunciaría un cierre que puede hacer rollback — la razón por la que hoy se emite fuera). En su lugar:
1. Nuevo helper `notify_sessions_closed(tenant_id, sessions, reason)` en `table_sessions/service.py` que publica `events.session_closed` por sesión, envuelto en el mismo `try/except` best-effort de `events.publish`.
2. `release_table` (router) lo llama tras el commit con `reason="released"`; el router ya tiene `tenant`.
3. `try_release_if_empty` deja de devolver `bool` y devuelve `list[TableSession]` (vacía = no cerró); los tres llamadores hacen commit y luego notifican con `reason="empty"`. En `qr_context._abandon_expired` el `tenant` está en el contexto. Quien solo usaba el truthiness sigue funcionando (`if try_release_if_empty(...)`).
4. `events.session_closed` documenta los valores de `reason`: `paid | swept | released | empty`.

Cero cambio de contrato HTTP y de modelo. El SSE ya enruta a `CH_STAFF` **y** `session_channel(id)`, así que los comensales conectados lo reciben.

### D9 — La pantalla siempre es "gracias", incluso en 401

**Hallazgo**: `public-menu.component.ts` tiene tres caminos a `expireSession`: el evento (`'La mesa se cerró…'`), el 401 del sondeo y el 401 de cualquier acción. Los tres terminan en la vista `'name'` con un mensaje, salvo que el comensal haya salido voluntariamente (vista `'exited'`).

**Decisión**:
- Vista nueva `'closed'` con el texto exigido (`¡Gracias por tu visita! Esperamos verte pronto.`, con logo y nombre del negocio como `'exited'`), sin botones.
- `expireSession` recibe un motivo `'closed' | 'expired'`. `session.closed` → `'closed'`. Un 401 → `'closed'` cuando el backend lo marca así (ver D10), `'expired'` (comportamiento actual: pedir nombre) cuando es vencimiento por inactividad/duración/firma.
- Al entrar a `'closed'`: `stopPolling`, `disconnectRealtime`, `tokenStore.endAccess(token de la URL)` (sustituye a `tokenStore.clear()`; ver D9b), `cart.clear()`, `cart.clearDiner()`, `myOrders.set([])`.
- Si el comensal ya salió voluntariamente (`isExited`) se conserva su marca (ver D9b).
- **Corrección (clarificación 2 de /speckit-clarify, FR-015a/b/c)**: la primera versión de esta decisión dejaba que recargar tras el cierre creara un ingreso nuevo (token borrado ⇒ vista `'name'`). Eso es un defecto: el enlace del QR es fijo por mesa, así que el F5 abría una sesión nueva sin escanear de nuevo. Ahora el cierre deja la **misma marca por pestaña que "Salir"** (D9b).

### D9b — Un solo borrado y una sola marca para "Salir" y cierre de mesa (FR-015a/b/c, FR-017a, SC-011)

**Hallazgo** (`diner-token.store.ts`, `public-menu.component.ts`): `exit()` hace `markExited(this.token)` + `tokenStore.clear()`; `expireSession()` solo hace `clear()` y **no** marca. `markExited` ya guarda en `sessionStorage` (por pestaña) el **token público de la URL de la mesa** (`this.token`, el `:token` de la ruta), no el de sesión — exactamente la "marca mínima" que exige la spec. Las claves de almacenamiento del comensal comparten el prefijo `pos.diner.` (`session_token`, `exited_token`, `checkout_progress.*`); la app no escribe cookies propias del comensal.

**Decisión**:
1. `DinerTokenStore.endAccess(tableToken)` (nuevo, único punto): (a) `clear()` del token; (b) elimina de `localStorage` y de `sessionStorage` **toda clave con prefijo `pos.diner.`** (token de sesión, progreso de checkout, cualquier dato de comensal) y expira defensivamente cualquier cookie `pos.diner*`; (c) al final escribe **solo** `markExited(tableToken)`. `exit()` y `expireSession('closed')` lo llaman; no hay dos copias (FR-015a). Falla silenciosa con almacenamiento bloqueado: la purga en memoria (`token.set(null)`, `cart.clear()`…) igual ocurre.
2. **Dos pantallas, una sola de "acceso denegado"**:
   - `'closed'` = pantalla de gracias **en el momento** (evento `session.closed`, 401 durante el uso, reconexión y también tras pulsar "Salir"): "¡Gracias por tu visita! Esperamos verte pronto.".
   - `'exited'` se conserva como la pantalla **estática de acceso denegado** (recarga, "Atrás"/"Adelante" con la marca presente, y token anterior rechazado al cargar): "Por favor, escanea nuevamente el código QR de la mesa para ingresar al menú". Reemplaza el texto "Acceso finalizado, vuelve a escanear el código QR de tu mesa"; sin campo de nombre, sin menú, sin botones. Se renombra en la plantilla a `data-testid="vista-acceso-denegado"` y se actualizan sus pruebas.
   - `exit()` deja de mostrar `'exited'` al instante y pasa a `'closed'` (la spec: la gracias "solo se ve en el momento del cierre o de Salir, sin recargar").
3. **Carga inicial (`ngOnInit`)**: orden actual (resolver menú → ¿marca? → ¿token?) se conserva. Cambia el `catch` de `cart.load()/refreshOrders()`: un 401 **con** `X-Session-State: closed` en la carga inicial (token de una sesión anterior en `localStorage` o en `?s=`) es acceso denegado, no gracias: `endAccess(this.token)` + vista `'exited'`. Un 401 de vencimiento sigue yendo al ingreso por nombre. Un 401 **durante el uso** (sondeo, acción, reconexión) sigue yendo a `'closed'` (+ `endAccess`).
4. **Backend (FR-017a)**: `open_session_context` ya responde 401 con `X-Session-State: closed` para un token cuyo participante no está `open` o cuya `table_session` ya no es la activa de la mesa, tanto con la mesa libre, cerrada como reocupada (verificado en `qr_context.py`). **Sin cambio de código**; se congela con pruebas de las tres situaciones (mesa libre, cerrada, reocupada). `GET /menu/qr-token/{token}` **no** exige sesión y no cambia: es la entrada de un escaneo nuevo en una mesa libre (la spec lo exige).
5. **Límite aceptado**: la marca es por pestaña; una pestaña nueva o un escaneo nuevo entran normal (FR-015c). Con `sessionStorage` bloqueado no hay marca y una recarga pide nombre (igual que "Salir" hoy); cualquier token anterior sigue rechazado por el backend.
6. **Declaración del riesgo de regresión**: cambia el comportamiento de "Salir" (mismo borrado ampliado, texto y pantalla intermedia). Es decisión del negocio de la spec (clarificación 2), se registra al ampliar **A-95** y se actualizan sus pruebas (`public-menu.component.spec.ts`, `diner-token.store.spec.ts`) citándola (Principio III).

### D10 — Cómo distinguir "sesión cerrada" de "sesión vencida" sin romper el contrato

`open_session_context` responde 401 con `detail` de **texto** en seis casos; los tests de frontend (`diner.service.spec.ts`, `auth-token.interceptor.spec.ts`) fijan esos textos. Cambiar `detail` a objeto arriesga ambos.

**Decisión**: cabecera de respuesta `X-Session-State: closed` en los dos 401 de cierre (`participant.status != "open"` y `table_session.status != "active"` / sesión distinta); ausente en los de vencimiento/duración/firma. `expose_headers` de `CORSMiddleware` gana `"X-Session-State"`. `DinerService.call()` la lee y lanza `DinerSessionExpiredError` con `closed = true`. Cuerpo y mensajes intactos (los tests actuales siguen verdes).

### D11 — Reabrir la mesa no reactiva la sesión (FR-017)

Verificado: ningún camino de código vuelve un `SessionParticipant` de `closed` a `open` (la única creación con `status="open"` es `table_sessions/service.py:479`, alta de comensal nuevo; `f2a3b4c5d6e7` reabrió `TableSession` una sola vez, como migración de datos, y no toca comensales). Y `open_session_context` exige `participant.table_session_id == sesión activa de la mesa`, por lo que aunque se reabra u ocupe la mesa, el token viejo recibe 401. **No hay cambio de backend para FR-017**; se congela con un test de regresión (US4-6).

**Caso a documentar**: cuando el scheduler vence una sesión **con pedidos por cobrar** (`_sweep_schema`, rama `has_billable_orders`) solo cierra comensales y **no** la sesión ni emite evento. El comensal se entera por el sondeo (`GET /cart/orders` → 401 con `X-Session-State: closed`, dentro de los 10 s del sondeo visible) → pantalla de gracias. Es coherente con "cerrar mesa = ya no hay clientes"; no se cambia el barrido (Principio V).

### D12 — Realtime y respaldo (SC-005/SC-006)

Canal principal: SSE (spec 077, `connectRealtime` ya registra `session.closed`). Respaldo ya existente: sondeo cada 10 s solo con la pestaña visible + `resync`/`reconnected` → `refreshOrders()` → 401. Con D10 ese 401 cae en `'closed'`. **Sin dependencias nuevas.**

---

## C. Impresión del cierre de caja

### D13 — Causa de la hoja en blanco (FR-024): hipótesis a confirmar reproduciendo

No se ha impreso nada aún; la spec exige reproducir antes de corregir. Lo que el código permite afirmar:

1. `dashboard-layout.component.ts` envuelve todo en `div.flex.h-screen.overflow-hidden` y el contenido (`router-outlet`) vive en `main.overflow-y-auto` dentro de otro `div.overflow-hidden`. Al imprimir, el navegador pagina un contenedor de **altura de una pantalla con `overflow: hidden`**: el contenido que excede la primera "pantalla" se recorta y el resto de hojas sale vacío. Ocultar el `app-sidebar`/`app-header` (spec 087, US4) no quita esa altura fija.
2. `cash-report.component.ts` no fuerza colores de impresión: usa clases claras (`text-gray-900` sobre fondo del shell `bg-gray-50`); `tagClass`/`diffClass` usan colores de estado. Si el tema oscuro invierte `body` a texto claro sobre fondo oscuro y el navegador omite fondos al imprimir (`print-color-adjust: economy` es el valor por defecto), queda texto claro sobre blanco. En el repo no hay `@media print` global ni reglas `dark` en `styles.css`/`app.css` (verificado por búsqueda): el tema oscuro, si existe, vive en clases de Tailwind por componente.
3. `sidebar` es `fixed`: ocultarlo no libera espacio, pero el `lg:ml-64` del contenedor **sí** sigue aplicando al imprimir (margen izquierdo de 16 rem → columna estrecha).

**Causa confirmada (T067, reproducida con Chrome real — CDP `Emulation.setEmulatedMedia print` + `Page.printToPDF`; ver `implementation-notes.md` Fase 7)**: no era ninguna de las tres hipótesis tal cual, aunque la (1) también aplicaba:
- **Hoja en blanco (causa principal)**: el navegador pagina a ~816 px de ancho (< `lg`), así que el telón del menú móvil `fixed inset-0 z-30 lg:hidden` (renderizado porque `sidebarOpen()` es `true` en escritorio) deja de estar oculto y se pinta a página completa **encima** del reporte en cada hoja; ocurre también con un turno sin movimientos. Aislar `app-cash-report` imprime bien; quitar sidebar/header no lo arregla.
- **Sin paginar**: `h-screen overflow-hidden` + `main overflow-y-auto` (hipótesis 1) solo explican el recorte de lo que excede la primera hoja.
- **Colores / tema oscuro (hipótesis 2)**: descartada como causa — el tema oscuro no existe en la app (cero `dark:`); aun así se fuerzan negro sobre blanco en impresión por robustez. **`lg:ml-64` (hipótesis 3)**: no fue la causa.
- **Corrección aplicada**: `print:hidden` en el telón; `.shell-root/.shell-content/.shell-main` liberados en `@media print` acotado a `body.printing-cash-report`; barra superior del módulo de caja `print:hidden`; `cash-report` con negro/blanco forzado, `print-color-adjust: exact`, `break-inside: avoid` y `thead` repetido.

**Método original** (tarea de diagnóstico, se ejecuta primero): Chrome (vista previa de impresión + `Emulate CSS media type: print`) y Firefox, con tema claro y oscuro, turno cerrado con y sin movimientos; capturar el árbol computado de `app-dashboard-layout > div`, `main` y `app-cash-report`. Registrar en `implementation-notes.md` cuál de (1)–(3) o combinación es la causa antes de tocar CSS.

**Corrección prevista (sujeta a la confirmación)**, acotada por `body.printing-cash-report` para no afectar recibos, cuenta de mesa ni hoja de QR (FR-023): en `@media print` liberar `height`, `overflow` y `margin-left` de los contenedores del shell; `position: static` y `display: block` en `main`; fondo `#fff` y color `#000` forzados a `app-cash-report` y descendientes con `print-color-adjust: exact`; `break-inside: avoid` en filas de tablas. Se añade el estilo en `dashboard-layout.component.ts` (donde ya vive el `:host-context(body.printing-cash-report)` de la spec 087) y en el `styles` de `cash-report.component.ts`.

**Verificación**: manual en vista previa (Chrome y Firefox, claro/oscuro, 1 y ≥2 hojas) porque ningún test unitario ejecuta la impresión real — se añade solo un test de clase/estilo del componente y se documenta el resultado en `quickstart.md`.

---

## D. Total inmediato en el POS

### D14 — De dónde sale el modal y qué se retira

Hay **tres** modales "El total cambió" (spec 073), todos con la misma forma: `loadPreview()` → comparar → `confirm.ask({title:'El total cambió'})`:
1. `pos-checkout-panel.component.ts::checkout()` — cobro de una orden abierta (mesa/domicilio/para llevar).
2. `payment-attempt-review-panel.component.ts::reconfirmIfTotalChanged()` — "Pagos por confirmar". Además tiene su propio mecanismo `totalChanged`/`totalChangeAck` (FR-024 de la spec 073) que **bloquea** las acciones hasta que el cajero "reconozca" el cambio.
3. `manual-order-page.component.ts::confirm()` — alta de pedido manual (FR-015a).

Además `pos-checkout-panel` muestra un aviso **no modal** "El total cambió · Actualizar" cuando `checkoutPreviewStale()` (evento `session.bill_changed` del SSE); esa barra se conserva (ver D16).

### D15 — Causa raíz: el preview no se refresca al guardar

`pos-terminal.store.ts::saveOrder()` agrega las líneas (`addOrderItems`, un POST por línea), hace `reload()` y **no** recarga `checkoutPreview`, así que el panel de cobro sigue mostrando el total anterior hasta que `checkout()` lo vuelve a pedir y el modal lo descubre. Es lo que corrige esta historia: refrescar en el propio `saveOrder()`.

### D16 — Diseño de la actualización inmediata

Estados de la cifra en pantalla (Key Entity de la spec): **estimado** y **confirmado**. Solo el confirmado habilita Cobrar.

1. **Estimación local (instantánea, < 0,2 s)**: `saveOrder()` calcula, antes de la primera llamada, `estimado = total confirmado vigente + Σ(subtotal de draftLines)` (la terminal POS usa precio por unidad; `draftLines` ya trae `unitPrice` y `subtotal`) y lo publica en un signal `checkoutPreviewEstimate` (`{subtotal, total}`); el panel lo muestra con un indicador discreto "actualizando…". **Cobrar queda deshabilitado mientras `estimate != null`.** Es una estimación: promociones e impuestos que el cliente no conoce los reconcilia el servidor.
2. **Confirmación**: al terminar el último POST, `saveOrder()` llama `loadCheckoutPreview(orderId)` (`GET /orders/{id}/checkout-preview`, ya existente) y limpia el estimado; la cifra visible pasa a ser la del servidor **sin diálogo**.
3. **Fallo**: si un POST falla, se limpia el estimado y se **recarga** el preview real (el total vuelve al confirmado, incluyendo las líneas que sí se guardaron antes del fallo, porque `saveOrder` es una secuencia de POST no transaccional). Sin servidor: se conserva el error habitual, no se cobra (FR-031) y el estimado nunca se deja como confirmado.
4. **Varias altas seguidas**: cada `saveOrder()` se serializa con el flag `submitting` ya existente; el estimado es aditivo y se recalcula desde el último confirmado, sin cifras intermedias que se queden.
5. **Cobrar (FR-027/FR-028)**: `checkout()` vuelve a pedir el preview; si difiere de lo visible, **no** abre modal: actualiza la cifra, muestra un aviso no bloqueante **dentro del panel** ("El total cambió: ahora es $X. Revisa y vuelve a pulsar Cobrar") y retorna sin cobrar; el siguiente Cobrar cobra el importe visible. Se conserva la regla de fondo: nunca se cobra lo que el cajero no vio.
6. **Barra `checkoutPreviewStale`** (evento externo): se conserva como está — es el aviso no bloqueante que ya cumple FR-028 para cambios de causa externa, y no es un modal.

**Alternativa descartada**: recalcular todo el total (promociones, domicilio, impuesto) en el cliente. Duplicaría el motor de promociones (spec 063/083) en TypeScript; el estimado local solo suma las líneas nuevas y el servidor sigue siendo la autoridad (Supuesto de la spec).

### D17 — "Pagos por confirmar" y pedido manual

- `payment-attempt-review-panel`: `reconfirmIfTotalChanged()` deja de usar `confirm.ask`; si el total cambió, actualiza `checkoutPreview`, muestra el aviso no bloqueante en la propia tarjeta y **retorna sin resolver el pago**. Se elimina el bloqueo por `totalChangeAck` como *gate* previo (la garantía de "no cobrar lo no visto" pasa a ser el "segundo clic"). El aviso informativo `totalChanged()` (el total difiere del que declaró el comensal) se **conserva** como texto, sin exigir acuse: el cajero ve la diferencia y decide (FR-030).
- `manual-order-page::confirm()`: mismo tratamiento (FR-015a de la spec 073 → FR-028 aquí, A-96): si el borrador cambió al volver a pedir el preview, actualiza y no crea el pedido hasta un segundo clic.

### D18 — Producto agotado con nombre (FR-029)

**Hallazgo**: agregar líneas a una orden POS pasa por `orders/service.py`/`consolidation.py::add_item_to_order` → `deduct_order_items` → `record_movement` → `InsufficientStockError`, cuyo mensaje nombra el **insumo** (`"Stock insuficiente de 'Queso'…"`), no el producto. El chequeo preventivo del Menú QR (`check_availability`) ya devuelve un objeto con `insumo`.

**Decisión** (única parte de backend de esta historia): `add_item_to_order`, `create_order` y el reemplazo de `kitchen.py` capturan el error del descuento y lo re-lanzan con `detail = {"error": "«<producto · presentación>» está agotado: falta <insumo>", "producto": <etiqueta>, "insumo": <nombre>}` usando `variant_label` (ya existente). Como `extractError` del frontend ya lee `detail.error`, el mensaje aparece sin cambios de cliente. Se conserva el status actual de `InsufficientStockError` (400); el frontend no depende del status para el texto.

**Revertir el total**: al fallar, `saveOrder()` aplica D16.3.

---

## E. Verificación

### D19 — Matriz de pruebas

| Área | Prueba | Tipo |
|---|---|---|
| Precio de línea | `2 × $15.000 + 1 × $3.000 = $33.000`; subir a 3 → $48.000; adicional ×2 en 1 producto; promoción + adicional; opción incluida por unidad | unittest (nuevos en `test_catalog_line_pricing`, `test_cart_service`) |
| Cinco superficies | Mismo total en carrito, terminal, checkout, "Pagos por confirmar", detalle de venta; línea histórica + nueva en un pedido | unittest + Karma |
| Consumo | Adicional descuenta 1 (no 2); grupo incluido sigue por unidad; deduct == reverse tras cambiar `pricing_type` | unittest (`test_catalog_consumption_plan`, `test_orders_service`) |
| Histórico | Filas sin `addons_total` (default 0) devuelven el total de siempre | unittest |
| Migración | Alembic upgrade/downgrade en esquema de pruebas; columnas con default | script + manual |
| Cierre de sesión | `release_table` y `try_release_if_empty` emiten `session.closed`; 401 con `X-Session-State`; token viejo sigue 401 tras reabrir | unittest (`test_table_sessions_router/service`) |
| Frontend QR | Vista `'closed'` por evento y por 401; carrito limpio; sin botón; `'exited'` intacto; editar adicionales (guardar/cancelar/vacío/inválido) | Karma |
| POS | Total estimado → confirmado; agotado nombra producto y revierte; sin modal en las tres superficies; segundo Cobrar | Karma |
| Impresión | Vista previa Chrome/Firefox claro/oscuro, 1 y ≥2 hojas | manual (quickstart §6) |
| Regresión | Suite completa de `pos-backend` y `ng test` de `pos-heladeria` | ambas |

Tests `CONGELA` afectados (se actualizan con cita a A-94/A-96 en el mismo commit): `test_catalog_line_pricing`, `test_catalog_consumption_plan`, `test_cart_service`, `test_orders_consolidation`, `test_orders_service`; en frontend `pos-checkout-panel.component.spec.ts`, `pos-terminal.store.spec.ts` (los casos que esperan el modal) y `payment-attempt-review-panel.component.spec.ts`.
