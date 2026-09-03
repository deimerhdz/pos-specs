# Research: Corrección — la Terminal de mesas cobra sin aplicar el descuento por promoción

**Spec**: [spec.md](./spec.md) | **Fecha**: 2026-09-02 (adenda 2026-09-03: D13-D15, quinta superficie US7)

Este documento no investiga tecnologías nuevas (no hay ninguna `NEEDS CLARIFICATION` en el
Technical Context: stack, dependencias y testing ya están fijados por specs anteriores y no
cambian aquí). Investiga y fija el **diseño técnico** de la corrección sobre el código real de
`pos-backend` y `pos-heladeria`, reconocido íntegramente antes de escribir este plan (archivo y
línea citados en cada decisión).

---

## D1 — El campo nuevo: `promotion_evaluated_at`, nulable, sin backfill, en dos tablas

**Decisión**: agregar `promotion_evaluated_at: datetime | None` a `customer_orders` (el instante
congelado, FR-008) y a `sales` (el instante que efectivamente se usó al facturar, FR-011a). Ambas
columnas nulables, **sin** `server_default` y **sin backfill**.

**Por qué nulable y sin backfill, no reutilizar `created_at`**: `CustomerOrder.created_at`
(`app/models/customer_order.py:106-108`) ya existe y tiene valor en el 100% de los pedidos,
viejos y nuevos — si se reutilizara tal cual como instante de vigencia, **todo** pedido histórico
cambiaría de comportamiento (usaría su `created_at` en vez de la hora del cobro), violando
FR-012 ("pedidos creados antes de este cambio... mantienen el comportamiento actual... sin
migración retroactiva"). Se necesita una columna que sea `NULL` en todo lo existente y solo se
puebla para pedidos nuevos — exactamente el patrón ya usado por
`OrderItem.discounted_unit_price` (comentario explícito ahí: *"Nullable sin default: las filas
anteriores a esta spec quedan en NULL, sin backfill"*) y por `delivery_fee`/`delivery_address`
(`customer_order.py:71-77`, migración `d427cd419e79`).

**Alternativa descartada**: una tabla de auditoría aparte para "instante de vigencia". Rechazada:
el dato es 1:1 con el pedido y con la venta, no un historial — una columna nulable es más simple
y es el patrón ya establecido en este mismo modelo para datos "solo presentes desde tal spec".

**Tipo de columna: aware UTC (`DateTime(timezone=True)`), NO naive como `created_at`**: el valor
se pasa a `auto_discount` → `evaluate_variant_sets`/`_valid_now` → `local_now()`
(`promotions/service.py:59`), cuyo contrato es *"acepta aware (convierte a hora local del tenant)
o naive (asume que ya es hora local)"*. Un instante UTC guardado **naive** lo interpretaría como
hora local y desplazaría la evaluación de fecha/día/franja por el offset del tenant (−5 en
Colombia): una promoción "hasta las 20:00" se evaluaría como "hasta las 15:00". Hoy todos los
call sites pasan `datetime.now(timezone.utc)` **aware**, y así debe seguir siendo el instante
congelado. `created_at` no sufre esto porque nunca se pasa a `local_now()`.

## D2 — Dónde se puebla `CustomerOrder.promotion_evaluated_at`: los DOS puntos de creación

**Decisión**: fijar `promotion_evaluated_at = datetime.now(timezone.utc)` en **ambos** lugares
donde hoy se instancia `CustomerOrder` directamente:
1. `app/api/v1/orders/service.py::create_order` (~líneas 214-230) — mostrador/mesero/API, cubre
   las cuatro superficies en alcance.
2. `app/api/v1/cart/service.py` (~líneas 577-586) — confirmación del carrito QR.

**Por qué ambos y no solo el primero**: FR-018 exige que el flujo QR reciba el mismo cambio de
vigencia de FR-009 ("les aplica igual"), y `cart/service.py` construye su propio
`CustomerOrder` de forma independiente — no pasa por `orders/service.py::create_order` (confirmado
por lectura directa de ambos archivos). Sin este segundo punto, el pedido del comensal por QR
quedaría con el campo siempre `NULL` y sin protección de FR-009, incumpliendo FR-018.

**Nota — venta de mostrador directa** (`POST /sales`, `app/api/v1/sales/service.py::checkout`,
~línea 255): no crea ningún `CustomerOrder` — pedido y cobro ocurren en la misma llamada. No hay
ventana entre "tomar el pedido" y "cobrarlo" que congelar: el `now` que ya usa hoy **es** el
instante de creación y de cobro a la vez. No requiere ningún cambio.

## D3 — Un único helper decide qué instante usar: `promotion_evaluation_instant`

**Decisión**: introducir una función pura en `app/api/v1/orders/checkout.py` (junto a
`auto_discount`, que ya vive ahí):

```python
def promotion_evaluation_instant(orders: list[CustomerOrder], *, now: datetime) -> datetime:
    """FR-009/FR-012/FR-012a: el instante contra el que se evalúa la vigencia
    TEMPORAL de las promociones. El más antiguo `promotion_evaluated_at` no nulo
    de `orders` (FR-012a: rondas sucesivas de una misma cuenta usan un único
    instante, el del pedido más antiguo pendiente). Si ninguna orden lo tiene
    (todas anteriores a esta spec, FR-012), cae a `now` — la hora del cobro,
    comportamiento actual sin cambios.

    Todo aware UTC: la columna es `DateTime(timezone=True)` (D1) y `now` en cada
    call site es `datetime.now(timezone.utc)` / `utc_now()`. `min()` y el retorno
    son homogéneos; `local_now()` los convierte a hora local del tenant. Guard
    defensivo por si algún call site futuro pasara naive."""
    now = now if now.tzinfo is not None else now.replace(tzinfo=timezone.utc)
    frozen = [
        (o.promotion_evaluated_at if o.promotion_evaluated_at.tzinfo is not None
         else o.promotion_evaluated_at.replace(tzinfo=timezone.utc))
        for o in orders if o.promotion_evaluated_at is not None
    ]
    return min(frozen) if frozen else now
```

Para `group_bill` (mesas fusionadas) el helper se llama **por pedido**, `promotion_evaluation_instant([o], now=now)`
dentro del bucle que ya evalúa `auto_discount` pedido a pedido — cada pedido contra su propio
instante, no el `MIN` del grupo (FR-018a): esos pedidos se cobran individualmente y el preview
consolidado debe coincidir con el cobro per-pedido.

Y aplicarlo en **todo** call site que hoy calcula `now = datetime.now(timezone.utc)` /
`utc_now()` inmediatamente antes de llamar `auto_discount`/`evaluate_variant_sets` sobre líneas de
uno o más `CustomerOrder` ya persistidos:

| Archivo:línea | Función | Pedidos que agrupa |
|---|---|---|
| `checkout.py:289` | `pay_order` | `[order]` (uno) |
| `checkout.py:481` | `checkout_and_send` | `[order]` (uno) |
| `checkout.py:897` | `approve_payment_attempt` | `[order]` (uno) |
| `checkout.py:1026` | `confirm_cash_payment_attempt` | `[order]` (uno) |
| `table_sessions/service.py:177` (`compute_bill`) | preview de cuenta de sesión | `_billable_orders(...)` (varias rondas) |
| `table_sessions/service.py:665` (`_close_unified`) | cierre unificado | `_billable_orders(...)` |
| `table_sessions/service.py:756` (`_close_split`) | cierre dividido | `_billable_orders(...)` |
| `orders/tables_advanced.py:145` (`group_bill`) | preview de mesas fusionadas (RF-053) | **cada pedido contra su propio instante** — `promotion_evaluation_instant([o], now=now)` dentro del bucle, NO sobre todo el grupo (FR-018a) |

No se toca `evaluate_variant_sets`/`_valid_now`/`active_variant_set_rules`
(`promotions/service.py`): siguen recibiendo `now` como siempre, solo que ese `now` ahora puede
ser el instante congelado en vez de la hora del cobro. El motor de descuentos no sabe ni necesita
saber que existe un "instante congelado" — es responsabilidad exclusiva de quien lo llama.

**Por qué extender también a `table_sessions`/`tables_advanced` y no solo a los 4 call sites de
`checkout.py` que cita la spec**: son el mismo defecto (mismo `auto_discount`, mismo parámetro
`now`) y FR-018 pide explícitamente que "el cobro de la cuenta de mesa... conserve su
comportamiento actual **salvo por el cambio de vigencia de FR-009, que les aplica igual**". No es
una refactorización oportunista (Principio V): es el mismo cambio autorizado por FR-009,
alcanzando cada sitio donde ese mismo parámetro se decide hoy con la hora del cobro. El helper
D3 es la única pieza nueva de lógica; cada call site solo cambia una línea (qué le pasa como
`now`). En particular `tables_advanced.py::group_bill` **entra en alcance de forma obligada**: sus
pedidos se cobran uno a uno por `checkout_and_send`/`pay_order` (que sí cambian), así que si su
preview consolidado siguiera evaluando con la hora del cobro, mostraría un total distinto al que
se termina cobrando (FR-018a) — de ahí la evaluación por pedido dentro del bucle.

## D4 — Endpoint nuevo para un pedido ya creado: `GET /orders/{order_id}/checkout-preview`

**Decisión**: nuevo endpoint de solo lectura, sin efectos secundarios, que devuelve el desglose
autoritativo `{subtotal, discount, delivery_fee, total}` de un pedido **ya persistido** y aún no
cobrado — cubre Historia 1 (mesa), Historia 2 (todos los métodos) e Historia 3 (para llevar y
domicilio), porque los tres tipos de orden son el mismo `CustomerOrder` con distinto
`order_type`, y `order_sale_lines` (`checkout.py:202-240`) ya no filtra por tipo.

**Por qué no reutilizar ninguno de los tres endpoints "bill" existentes**:
- `GET /orders/tables/{table_id}/bill` (`checkout.compute_bill`, `checkout.py:138-191`) **no
  aplica `auto_discount` en absoluto** (suma `unit_price * quantity` a secas, línea 166) — sería
  el candidato más obvio por nombre, pero está roto para este propósito y, confirmado por
  búsqueda en `pos-heladeria` (`grep` sobre `src/app`, sin ninguna coincidencia), **ningún**
  componente del frontend lo llama hoy. No hay ninguna de las 4 superficies en alcance que lo
  consuma, así que corregirlo sería tocar un endpoint sin efecto observable para esta spec —
  queda **fuera de alcance** (Principio V: no se mezcla una corrección no pedida).
- `GET /table-sessions/{id}/bill` (`table_sessions.compute_bill`) es para la **sesión de mesa**
  (varias rondas, varios comensales) del pedido de **mesero** ya enviado a cocina — un flujo que
  la propia spec marca como "correcto hoy" (tabla de la sección "Contexto del defecto") y fuera de
  alcance salvo por D3. Forzarlo a servir también al pedido individual de mostrador mezclaría dos
  conceptos (sesión vs. pedido) que hoy están deliberadamente separados.
- No hay ningún endpoint de preview para pedidos `TAKEAWAY`/`DELIVERY` (sin `table_session_id`).

**Implementación**: `checkout.compute_checkout_preview(db, order_id) -> CheckoutPreview`, que
reutiliza literalmente `order_sale_lines`, D3 (con `orders=[order]`) y la fórmula de D6 — sin
persistir nada, sin lock, sin turno de caja (no cobra, solo muestra). Contrato completo en
[contracts/preview-cobro-pedido.md](./contracts/preview-cobro-pedido.md).

## D5 — Endpoint nuevo para el borrador sin guardar: `POST /orders/draft-preview`

**Decisión**: nuevo endpoint que acepta el mismo `items: list[OrderItemIn]` (+ `delivery_fee`
opcional) que ya usa `OrderCreate` (`orders/schemas.py:124-144`), sin crear ningún `CustomerOrder`,
y devuelve el mismo desglose `{subtotal, discount, delivery_fee, total}` — cubre Historia 5
(pantalla de armado de orden manual, FR-013).

**Por qué no existe ya nada así**: el único lugar que evalúa promociones sin persistir un pedido
es `GET /cart` (`cart/service.py::serialize_cart`), pero exige un `Cart`/`participant_id` real del
flujo QR — no sirve para un borrador de la Terminal que todavía no tiene ni `participant_id` ni
fila en base de datos.

**`now` sin congelar**: como el pedido no existe todavía, no hay ningún `promotion_evaluated_at`
que leer — se evalúa siempre con `datetime.now(timezone.utc)` (FR-008: el instante se congela
recién al **crear** el pedido, no antes). Coherente con el Edge Case de la spec y con FR-015a: el
pedido que finalmente se crea (al confirmar) congela su propio instante en ese momento, no el que
tenía el borrador mientras se armaba.

Contrato completo en
[contracts/preview-borrador-orden-manual.md](./contracts/preview-borrador-orden-manual.md).

## D6 — Una sola fórmula del total, sin duplicarla entre `build_sale` y el preview

**Decisión**: extraer la aritmética que hoy vive inline en
`sales/builder.py:142-144` (`total = subtotal - discount + tax + tip + delivery_fee`, con el
`max(0, ...)`/422 de negativo) a una función pura `compute_total(subtotal, discount, tax, tip,
delivery_fee) -> Decimal`, y que tanto `build_sale` como los dos preview nuevos (D4/D5) la llamen.

**Por qué**: si el preview calculara el total con su propia copia de la fórmula, cualquier cambio
futuro a uno de los dos lugares (p. ej. un impuesto que deje de estar fijo en $0) los
desincroniza en silencio — exactamente el riesgo que `research.md` de la spec 063 (D10) ya
advertía sobre replicar cálculos de descuento en dos sitios. Aquí el riesgo es menor (es solo una
suma/resta), pero el costo de no duplicarla es cero.

## D7 — `Sale.promotion_evaluated_at`: nuevo parámetro opcional de `build_sale`

**Decisión**: `build_sale` (`sales/builder.py:72-91`) gana un kwarg nuevo
`promotion_evaluated_at: datetime | None = None`, que asigna a `sale.promotion_evaluated_at` junto
a donde ya fija `sale.total`/`sale.discount` (línea ~177-183). Cada uno de los 4 call sites de
`checkout.py` (D3) y los 2 de `table_sessions/service.py` le pasan el mismo valor que usaron para
calcular el descuento de esa venta (la salida de D3). Satisface FR-011a: la venta emitida guarda
el instante con el que se evaluaron sus promociones, junto al `applied_promotions`/`promotion_id`
que ya guardaba (spec 063, FR-021).

Ventas que no pasan por ningún pedido congelado (todo lo emitido antes de esta spec, y la venta de
mostrador directa de D2) simplemente no reciben este kwarg → queda `NULL`, coherente con FR-011
(nada retroactivo).

## D8 — FR-009a (estado vivo) no exige ningún cambio: ya funciona así

**Hallazgo**: `active_variant_set_rules` (`promotions/service.py:143-165`) filtra
`Promotion.status == "active"` **en cada llamada**, contra la base de datos en ese instante —
independientemente de qué `now` reciba para el chequeo temporal. Congelar `now` (D1-D3) no toca
en absoluto esta parte: si el administrador pausa/borra/desactiva una promoción entre que se tomó
el pedido y se cobra, la próxima llamada a `auto_discount` (con el `now` que sea) ya no la
encuentra, sin ningún código adicional. FR-009a queda satisfecho **por construcción**, no por una
pieza nueva — es la razón de que esta corrección sea segura de implementar: separa limpiamente
"cuándo" (congelable) de "si sigue existiendo/activa" (siempre vivo), porque el motor ya los
resolvía en dos pasos distintos.

## D9 — Catálogo de la Terminal (Historia 6): cero cambios de backend

**Hallazgo**: `MenuVariantPromotion` (`condition_text`/`short_condition`/`display_text`/
`unit_equivalent_text`, ya con la lógica de FR-017 resuelta vía `unit_equivalent_approx`/
`min_qty`, spec 066) **ya llega intacto** al store de la Terminal a través de
`MenuService`/`store.categories()` — confirmado por lectura de `menu.service.ts:37-46,120-129` y
de `product-select.component.ts:111-127`, componente que **ya comparten** el menú QR y las dos
superficies del cajero.

**Decisión**: FR-016/FR-017 se resuelven **solo en frontend**: `PosTerminalStore.
productDiscountBadge()`/`productDiscountBadges()` (`pos-terminal.store.ts:404-441`, que recalcula
localmente la insignia "-50%" recorriendo `promotionService.activePromotions()`) se elimina, y
`pos-catalog-drawer.component.ts`/`manual-order-page.component.ts` (los dos consumidores de
`productDiscountBadges()`) pasan a leer directo `variant.promotion?.short_condition`/
`display_text` de los datos que el store ya trae. Ningún endpoint nuevo, ninguna migración.

**Alternativa descartada**: exponer promociones en `GET /products` (el catálogo administrativo).
Innecesaria — el dato ya viaja por `MenuService`, que la Terminal ya consume para armar
`store.categories()`.

## D10 — Frontend: reemplazar el cálculo local por consumo del preview, mismo molde que ya usa el store

**Decisión**: `PosTerminalStore.totals()` (`pos-terminal.store.ts:883-896`, hoy con `const
discount = 0` fijo) deja de calcular localmente para el pedido de mostrador/mesa individual
(rama por defecto de `PosCheckoutPanelComponent`) y para el borrador de orden manual. En su lugar:

- Un signal nuevo `checkoutPreview: Signal<CheckoutPreview | null>` + `checkoutPreviewLoading:
  Signal<boolean>` + `checkoutPreviewStale: Signal<boolean>`, poblados por un método
  `loadCheckoutPreview(orderId)` que llama a D4 — **mismo molde exacto** que ya usa
  `sessionBill`/`billLoading`/`billStale`/`loadSessionBill()` (`pos-terminal.store.ts:1620-1679`):
  señal-resultado + señal-loading + señal-stale (se marca, nunca se recarga sola) +
  `try/catch/finally` con `extractError()`.
- Para el borrador de orden manual (su propia instancia de `PosTerminalStore`, D5): un signal
  análogo `draftPreview`, poblado en cada cambio de línea del borrador (agregar/quitar/cambiar
  cantidad, FR-013), consumiendo D5.
- `PaymentInputComponent`/`payment-draft.util.ts` **no cambian**: ya reciben `total` como
  parámetro genérico (`@Input() total`, `paymentIssue(draft, total, methods)`) — no están
  acoplados a `store.totals()`, así que apuntarlos al total del preview es solo cambiar qué
  expresión llega por `[total]` en la plantilla de `PosCheckoutPanelComponent`
  (línea 174: `store.totals().total` → `store.checkoutPreview()?.total`, con el estado
  "calculando" de FR-007a controlando cuándo se pinta ese `[total]` en absoluto).

**Diferencia deliberada entre el panel de cobro y el borrador (FR-007a vs. FR-015)**: el panel de
cobro **nunca** muestra un total provisional — mientras `checkoutPreviewLoading()` es verdadero o
no ha llegado ningún preview, "Cobrar" queda deshabilitado (FR-007a, no negociable: es dinero
real). El borrador de orden manual, en cambio, si el preview falla, **sí** puede mostrar el
subtotal sin descuento con un aviso y dejar confirmar el pedido igual (FR-015: nada bloquea armar
un pedido, solo cobrarlo).

## D11 — Reconfirmación cuando el total cambia (FR-007/FR-015a): doble chequeo antes de someter, no parsear un 422

**Decisión**: tanto antes de `checkoutAndSend()` (cobro) como antes de
`createManualOrderFromDraft()` (confirmar borrador), el store vuelve a pedir el preview
correspondiente una vez más justo antes de la llamada real. Si el total recién obtenido difiere
del último que se le mostró al cajero, la operación se detiene, se muestra el total nuevo, y solo
se somete la operación real tras una segunda confirmación explícita del cajero — nunca en
silencio, y nunca dejando que la request real falle con el mensaje técnico del servidor
(`build_sale`, `sales/builder.py:161-175`).

**Por qué no interceptar el 422 de `build_sale`**: ese error ya existe hoy y su texto
("El pago (X) no cubre el total (Y)" / "Los pagos que no son en efectivo... no pueden superar el
total") es genérico — no distingue "el total cambió entre que se mostró y que se cobró" de
cualquier otro descuadre. Reinterpretarlo con heurísticas del lado del cliente es frágil. El
doble chequeo explícito (repetir la lectura del preview justo antes de someter) es determinista y
no depende de parsear texto de error.

## D12 — Migración Alembic: una sola revisión, dos tablas, esqueleto de `d427cd419e79` (salvo el tipo con zona)

**Decisión**: una migración nueva (`@for_each_tenant_schema`, guardas `_has_table` defensivas,
`downgrade` estrictamente inverso — mismo esqueleto que `d427cd419e79_domicilio_orden_manual.py`)
que agrega `promotion_evaluated_at TIMESTAMP WITH TIME ZONE NULL`
(`sa.DateTime(timezone=True)` — aware UTC, ver D1) a `customer_orders` y a `sales`, sin
`CheckConstraint` (no hay invariante que validar en una columna de solo lectura/nulable) y sin
ningún `UPDATE`/backfill. Ver [data-model.md](./data-model.md) para el detalle exacto de columnas
y el borrador del `upgrade`/`downgrade`.

---

## Adenda 2026-09-03 — quinta superficie: revisión de pago del cajero para pedidos QR (US7)

Tras implementar D1-D12 (Historias 1-6), el dueño reprodujo el mismo defecto en el panel **"Pagos
por confirmar"** — la revisión del cajero de un intento de pago de un pedido de comensal por QR
("Confirmar efectivo" / "Aprobar comprobante"). La spec lo incorpora como **quinta superficie**
(FR-021 a FR-024, Historia 7). **No abre anomalía nueva**: es el mismo defecto de FR-001/FR-002
(un total sin descuento calculado fuera de la autoridad de la venta), y el cambio de vigencia ya
está cubierto por `A-70`, que enumera `confirm_cash_payment_attempt` y `approve_payment_attempt`
entre sus 8 call sites. Estas tres decisiones (D13-D15) se apoyan enteramente en piezas ya
construidas en D1-D12 — no hay endpoint nuevo, ni migración, ni helper nuevo.

## D13 — El chequeo previo `_order_total` de `confirm_cash_payment_attempt` pasa a la cuenta autoritativa

**Hallazgo (verificado sobre el código actual)**: `confirm_cash_payment_attempt`
(`checkout.py:1117-1204`) construye la venta con `promotion_evaluation_instant([order], now=…)` +
`auto_discount` + `build_sale` desde la tarea T028 — la venta emitida **ya sale bien**. Pero el
chequeo previo del "monto recibido" usa una función aparte, `_order_total(db, order_id)`
(`checkout.py:939-955`), que suma `unit_price × cantidad + delivery_fee` y **nunca resta el
descuento**. Resultado: `amount_received < total` rechaza con 422 *"El monto recibido (10000) es
menor al total de la orden (16000.00)"* aunque la venta que se construiría a continuación vale
$8.000. Es exactamente el mismo patrón de la causa raíz de US1 (dos cálculos independientes del
total, uno sin descuento), en otro sitio.

**Decisión**: el chequeo previo pasa a comparar contra `compute_checkout_preview(db,
attempt.order_id).total` (D4) — la **misma** función de solo lectura que ya calcula el total
autoritativo para el panel de cobro y que el frontend de US7 consumirá (D14). `_order_total` no
tiene ningún otro consumidor (`grep` sobre `app/`, única llamada en `checkout.py:1142`), así que
se **elimina** en lugar de dejarlo como una segunda fuente de verdad latente. El `change_amount`
del intento (`attempt.change_amount = amount_received - total`, `checkout.py:1152`) usa ese mismo
`total`, de modo que coincide al peso con el `Sale.change_given` que `build_sale` calcula después
(que ya restaba el descuento) — hoy solo coinciden por casualidad cuando no hay promoción.

**Por qué reusar `compute_checkout_preview` y no extraer un tercer helper**: el pedido QR en
revisión está en estado `recibida` (nunca `pagada`/`cancelada` — `TERMINAL`, `checkout.py:49`), así
que el `409` de `compute_checkout_preview` no se dispara; y llamarla aquí garantiza por
construcción que "lo que el panel muestra" == "lo que el chequeo valida" == "lo que `build_sale`
registra" (FR-021), sin una cuarta aritmética que mantener sincronizada. Es una llamada de solo
lectura extra (`get_or_404` + `order_sale_lines` + `auto_discount`) antes de `ensure_open_shift`,
sin `commit` — coste idéntico al que ya paga el cobro real un instante después.

**`approve_payment_attempt` (transferencia) no necesita cambio de backend**: ya construye la venta
con `sum(line_total) - promo_discount + delivery_fee` e instante congelado (`checkout.py:1033-1044`)
desde T028, y no tiene un chequeo previo de "monto" (el importe es el total exacto, no lo teclea el
cajero). Lo único que le falta es que el cajero **vea** ese total antes de pulsar "Aprobar" — eso
es D14/D15 (frontend). Se deja constancia de que se revisó y no requiere tocar
`approve_payment_attempt`.

## D14 — El panel "Pagos por confirmar" consume `GET /orders/{order_id}/checkout-preview` — cero backend nuevo

**Hallazgo**: el endpoint `GET /orders/{order_id}/checkout-preview` (D4) ya devuelve
`{subtotal, discount, delivery_fee, total, promotion_evaluated_at}` para cualquier `CustomerOrder`
que no esté en `TERMINAL`. Un pedido QR pendiente de confirmación está en `recibida` → el endpoint
funciona **tal cual**, sin cambio de ruta ni de esquema. El servicio de frontend ya tiene el
método: `DiningSessionService.checkoutPreview(orderId)`
(`dining-session.service.ts:85-89`), hoy solo usado por el panel de cobro de la Terminal.

**Decisión**: `PaymentAttemptReviewPanelComponent`
(`payment-attempt-review-panel.component.ts:212`) gana un trío de signals locales
`checkoutPreview` / `checkoutPreviewLoading` / `checkoutPreviewError` + un `loadCheckoutPreview()`
que llama a ese método en `ngOnChanges` (donde ya carga los intentos, línea 240-243). El molde es
el mismo señal-loading-error que el resto de spec 073, pero **local al componente** — este flujo
(revisión QR) no pasa por `PosTerminalStore`, usa `DiningSessionService` directo, y cada tarjeta
del panel es deliberadamente independiente de sus hermanas
(`payment-validation-block.component.ts:20-24`).

Cambios concretos en el componente:
- `orderTotal()` (`:266-272`, hoy `Σ unit_price × quantity` local) → `checkoutPreview()?.total`.
- `cashChangePreview()` (`:278-282`) calcula el vuelto sobre ese `total`, no sobre el bruto.
- Se añade el desglose `Subtotal / Descuento / Domicilio / Total` (FR-022, mismo formato agregado
  de FR-004) en la plantilla del panel; `Descuento`/`Domicilio` solo cuando `> 0`.
- Mientras `checkoutPreviewLoading()` o `checkoutPreview()` es `null`: estado visible
  **"calculando"** y acciones "Confirmar efectivo" / "Aprobar comprobante" deshabilitadas (FR-024
  → misma regla que FR-007a). Nunca se pinta un total provisional.

**`PaymentValidationBlockComponent.total(order)`** (`:136-147`, hoy suma `discounted_line_total`
—el congelado del carrito, el que "disimulaba el fallo"— o el bruto): esa fila de pie
`$ {{ total(order) }}` (`:102-106`) **se elimina** — el total autoritativo ahora lo muestra el
`PaymentAttemptReviewPanelComponent` embebido, con su desglose completo. Se elimina también el
método `total()` y el uso de `discounted_line_total` que quedaba muerto.

**Alternativa descartada**: pasar el preview desde el bloque padre hacia abajo por `@Input`.
Rechazada: rompería la independencia por tarjeta (cada panel resuelve su propio intento sin estado
compartido) y obligaría al bloque a orquestar N cargas; que cada panel cargue su propio preview es
coherente con que ya carga su propio `listPaymentAttempts`.

## D15 — Reconfirmación cuando el total cambió (FR-024): doble chequeo antes de resolver + marca vs. la tarjeta

**Decisión**: mismo patrón que D11 (US1/US5), aplicado a las dos acciones del panel:

1. **Al abrir el panel** (`loadCheckoutPreview()` en `ngOnChanges`): si el `total` autoritativo
   recién traído difiere del que la tarjeta venía mostrando —el `Σ discounted_line_total` del
   carrito, que el bloque calculaba antes de D14—, el panel **marca** ese cambio (aviso visible
   "el total cambió respecto al declarado por el comensal: antes $X, ahora $Y") y exige que el
   cajero lo reconozca antes de habilitar "Confirmar efectivo" / "Aprobar comprobante". Cubre el
   Edge Case "La tarjeta de 'Pagos por confirmar' muestra un total desactualizado" y el caso de
   promoción pausada entre pedido y cobro (FR-009a — el estado se lee vivo, así que el preview ya
   trae el total recalculado sin descuento).
2. **Justo antes de `approve()` / `confirm()`**: se vuelve a pedir `checkoutPreview(order.id)` una
   vez más; si el `total` cambió respecto al último mostrado, se detiene la acción, se presenta el
   total nuevo y se exige una segunda confirmación explícita — **nunca** se deja que el 422 del
   backend (`build_sale` / el chequeo de D13) sea lo que le avise al cajero. Para "Confirmar
   efectivo" esto además re-valida que el `amount_received` tecleado siga cubriendo el total nuevo.

**Por qué no basta con el chequeo de D13 en el backend**: D13 evita la venta incorrecta, pero su
422 es genérico y llega **después** de que el cajero pulsó el botón. FR-024 pide que el cajero vea
el total actualizado y lo confirme *antes* de emitir — es una exigencia de la interacción, no solo
de la corrección del importe. El backend de D13 es la red de seguridad; D15 es el flujo previsto.

**Sin cambio de contrato del intento de pago**: `OrderPaymentAttempt` / `PaymentAttemptResponse` no
ganan ningún campo. El instante congelado ya viaja por `Sale.promotion_evaluated_at` (D7) y el
total autoritativo por `CheckoutPreviewResponse` (D4) — el panel no necesita persistir nada nuevo.
