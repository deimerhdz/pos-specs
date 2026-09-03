---

description: "Task list for feature implementation"
---

# Tasks: Corrección — la Terminal de mesas cobra sin aplicar el descuento por promoción

**Input**: Design documents from `/specs/073-fix-descuento-cobro-terminal/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/), [quickstart.md](./quickstart.md)

**Tests**: incluidos — el plan (Technical Context, Principio X) y los cinco contratos ya
comprometen ficheros de test concretos en ambos repos. No son opcionales aquí.

**Organización**: esta spec toca **dos repositorios** (`../pos-backend/`, `../pos-heladeria/`).
La corrección tiene cuatro piezas verificables por separado (plan.md, Summary):

1. **Autoridad del monto** (US1–US3, US5) — el navegador deja de calcular el descuento y consume
   dos endpoints de solo lectura nuevos.
2. **Vigencia congelada** (US4) — columna nueva `promotion_evaluated_at` + un helper aplicado en
   8 call sites. **Deroga comportamiento vigente → exige `A-70` antes de tocar el helper.**
3. **Catálogo de la Terminal** (US6) — cero backend; se reemplaza un cálculo local por leer el
   dato que ya llega.
4. **Revisión de pago del cajero para pedidos QR** (US7 — añadida 2026-09-03) — el panel "Pagos
   por confirmar" pasa a ser una superficie de cobro más: el chequeo previo del efectivo deja de
   usar `_order_total` (que se elimina) y consume `compute_checkout_preview`; el frontend cablea
   el preview autoritativo. **Cero endpoint/migración/helper nuevos** — reusa las piezas 1 y 2.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: se puede hacer en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: US1, US2, US3, US4, US5, US6 o US7
- Rutas relativas a la raíz del repo indicado (`pos-backend/…` o `pos-heladeria/…`)

---

## Phase 1: Setup

**Purpose**: autorización de proceso (Principio II) y línea base verde antes de tocar nada.

- [x] T001 Registrar la anomalía **A-70** en `pos-specs/specs/000-reconocimiento/registro-de-anomalias.md`, con una entrada nueva justo después de `A-69`, con el mismo formato que las entradas `A-65`–`A-69` (campos **Qué cambia / Por qué cambia / Quién tomó la decisión y cuándo / Funcionalidades afectadas / Clasificación / Tratamiento acordado**). **Hecho** (2026-09-02, entrada `### A-70 — [DECISIÓN DE NEGOCIO — spec 073]`). **Bloquea toda tarea que toque `promotion_evaluation_instant` (T008 y toda la Fase 6).**
- [x] T002 [P] Línea base backend: correr `cd pos-backend && python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` y guardar el conteo en verde (especialmente `test_orders_checkout.py`, `test_table_sessions_service.py`, `test_cart_service.py`, `test_promotions_service.py`) — [quickstart.md §Suites automatizadas](./quickstart.md)
- [x] T003 [P] Línea base frontend: correr `cd pos-heladeria && npm test` y guardar el conteo en verde (especialmente `pos-terminal.store.spec.ts`, `pos-checkout-panel.component.spec.ts`, `payment-input.component.spec.ts`, `manual-order-page.component.spec.ts`)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: infraestructura compartida por US1–US5 — la columna nueva, el helper de instante,
la fórmula única del total y los esquemas de respuesta. Ninguna historia de la pieza "autoridad
del monto" ni de "vigencia congelada" puede empezar sin esto.

**⚠️ CRITICAL**: US1–US5 dependen de esta fase. US6 (catálogo) es independiente y NO depende de ella.

### Migración y modelo (`pos-backend`)

- [x] T004 Crear la revisión Alembic nueva `pos-backend/alembic/versions/<rev>_congela_instante_vigencia_promociones.py`, encadenada tras el head actual `94144eaa60b5` (`down_revision = '94144eaa60b5'`), siguiendo el esqueleto de `d427cd419e79_domicilio_orden_manual.py` (`@for_each_tenant_schema`, guardas `_has_table`, `downgrade` estrictamente inverso): `op.add_column("customer_orders", sa.Column("promotion_evaluated_at", sa.DateTime(timezone=True), nullable=True))` y lo mismo en `"sales"`. **`timezone=True`** (se desvía a propósito del esqueleto en este punto: el instante se pasa a `local_now()`, que trata un naive como hora local del tenant — un UTC naive desplazaría la franja por el offset). Sin `server_default`, sin backfill, sin `CheckConstraint` — [data-model.md §Migración Alembic](./data-model.md)
- [x] T005 [P] Agregar el atributo `promotion_evaluated_at: Mapped[datetime | None]` con `mapped_column(DateTime(timezone=True), nullable=True)` (sin default) a `pos-backend/app/models/customer_order.py`, con el mismo comentario de "sin backfill, NULL en filas anteriores a esta spec" que ya lleva `discounted_unit_price` **más** una nota de por qué `timezone=True` y no `DateTime` como `created_at` — [data-model.md §`customer_orders`](./data-model.md)
- [x] T006 [P] Agregar el atributo `promotion_evaluated_at: Mapped[datetime | None]` con `mapped_column(DateTime(timezone=True), nullable=True)` (sin default) a `pos-backend/app/models/sale.py` — [data-model.md §`sales`](./data-model.md)

### Fórmula única del total y helper de instante (`pos-backend`)

- [x] T007 [P] Extraer la aritmética inline de `pos-backend/app/api/v1/sales/builder.py:144` (`total = subtotal - discount + tax + tip + delivery_fee`, con el `max(0, …)`/422 de negativo) a una función pura `compute_total(subtotal, discount, tax, tip, delivery_fee) -> Decimal`; `build_sale` pasa a llamarla. El comportamiento de `build_sale` NO cambia — los characterization tests de `test_sales_*`/`test_orders_checkout.py` deben seguir en verde sin editarse — [research.md D6](./research.md)
- [x] T008 Añadir la función pura `promotion_evaluation_instant(orders: list[CustomerOrder], *, now: datetime) -> datetime` en `pos-backend/app/api/v1/orders/checkout.py` (junto a `auto_discount`), con la firma y el cuerpo exactos de [research.md D3](./research.md) (`min` de los `promotion_evaluated_at` no nulos, o `now` si ninguno). Mantiene todo **aware UTC**: normaliza cualquier entrada naive con `.replace(tzinfo=timezone.utc)` para que `min()` y el retorno sean homogéneos y `local_now()` los convierta bien a hora local del tenant. **No se aplica todavía a ningún call site existente** — solo se define. Depende de **T001** (A-70) y T005
- [x] T009 [P] Tests unitarios de `promotion_evaluation_instant` en `pos-backend/app/characterization_tests/test_orders_checkout.py`: 1 pedido con instante congelado → lo devuelve; 1 pedido con `NULL` → devuelve `now` (FR-012); N pedidos → devuelve el `min` de los congelados (FR-012a); N pedidos todos `NULL` → devuelve `now`; mezcla congelado + `NULL` → devuelve el congelado más antiguo; el valor devuelto siempre conserva `tzinfo` (aware UTC) aunque la entrada fuera naive — depende de T008

### Esquemas de respuesta compartidos (`pos-backend`)

- [x] T010 [P] Agregar `CheckoutPreviewResponse` (`subtotal`, `discount`, `delivery_fee`, `total`, `promotion_evaluated_at` — forma exacta de [data-model.md §`CheckoutPreview`](./data-model.md)) y `DraftPreviewIn` (`items: list[OrderItemIn]` mínimo 1, `delivery_fee: Decimal | None`, misma validación que `OrderCreate`) a `pos-backend/app/api/v1/orders/schemas.py`

**Checkpoint**: la migración aplica y revierte limpio; `promotion_evaluation_instant` y
`compute_total` existen y están probadas de forma aislada; los esquemas están listos para los
endpoints. Cero cambio de comportamiento observable todavía.

---

## Phase 3: User Story 1 - Cobrar en efectivo un pedido con promoción (Priority: P1) 🎯 MVP

**Goal**: el panel de cobro de la Terminal exige y valida el importe **real** de un pedido de
mesa (con descuento por promoción aplicado), calculado por el backend, no por el navegador. Cubre
además el endpoint de preview del que dependen US2 y US3.

**Independent Test**: crear un pedido de mesa con 2 conos a $8.000 y una promoción vigente del
50% llevando 2; abrir el panel de cobro; verificar `Subtotal $16.000 / Descuento −$8.000 /
Total $8.000`; registrar $8.000 en efectivo y confirmar que la venta se emite al primer intento
por $8.000 con vuelto $0 ([quickstart.md §Historia 1](./quickstart.md)).

### Backend — endpoint de preview del pedido ya creado

- [x] T011 [P] [US1] Tests de `compute_checkout_preview` en `pos-backend/app/characterization_tests/test_orders_checkout.py`: pedido de mesa con promoción → `{subtotal, discount, total}` correctos; pedido sin promoción → `discount = 0`; pedido a domicilio con `delivery_fee` → lo incluye en `total`; ítems `anulado` excluidos; `404` si no existe; `409` si `status` no es cobrable (`pagada`/`cancelada`) — [contracts/preview-cobro-pedido.md §Errores](./contracts/preview-cobro-pedido.md). Deben fallar antes de T012
- [x] T012 [US1] Implementar `compute_checkout_preview(db, order_id) -> CheckoutPreviewResponse` en `pos-backend/app/api/v1/orders/checkout.py` siguiendo los 8 pasos de [contracts/preview-cobro-pedido.md §Implementación](./contracts/preview-cobro-pedido.md): `get_or_404` → `order_sale_lines` (reuso literal) → `raw_subtotal` → `promotion_evaluation_instant([order], now=…)` → `auto_discount(db, lines, instant)` → `order.delivery_fee or 0` → `compute_total(...)`. **Sin `db.commit()`, sin `build_sale`, sin lock, sin turno de caja.** Depende de T009, T010, T011
- [x] T013 [US1] Registrar la ruta `GET /orders/{order_id}/checkout-preview` en `pos-backend/app/api/v1/orders/router.py`, respondiendo `CheckoutPreviewResponse`, con el `409` para estados no cobrables (mismo patrón que `checkout_and_send`/`pay_order`) — depende de T012

### Frontend — señales compartidas del preview en el store

- [x] T014 [US1] Agregar a `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` las señales `checkoutPreview: signal<CheckoutPreview | null>`, `checkoutPreviewLoading: signal<boolean>`, `checkoutPreviewStale: signal<boolean>` y el método `async loadCheckoutPreview(orderId)`, **replicando exactamente el molde de `sessionBill`/`billLoading`/`billStale`/`loadSessionBill()`** (líneas ~1626–1679: señal-resultado + loading + stale que se marca y nunca se recarga sola + `try/catch/finally` con `extractError()`). Añadir el tipo `CheckoutPreview` y la llamada HTTP al servicio correspondiente — [research.md D10](./research.md), [contracts/preview-cobro-pedido.md §Consumo](./contracts/preview-cobro-pedido.md). Depende de T013
- [x] T015 [P] [US1] Tests de `loadCheckoutPreview()` en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts`: éxito → puebla `checkoutPreview()` y baja `checkoutPreviewLoading()`; error HTTP → `checkoutPreview()` en `null` y error expuesto; evento de cuenta obsoleta → `checkoutPreviewStale()` en `true` sin recargar sola — depende de T014

### Frontend — panel de cobro (camino efectivo)

- [x] T016 [US1] En `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.ts`: reemplazar `store.totals().total` por `store.checkoutPreview()?.total` en `[total]` (línea 174) y en `paymentIssue(...)` (línea 321); disparar `store.loadCheckoutPreview(orderId)` en el mismo punto donde hoy se resetea `paymentDraft` (líneas ~324–332) y cuando se marca la cuenta obsoleta — [contracts/preview-cobro-pedido.md §Consumo](./contracts/preview-cobro-pedido.md). Depende de T014
- [x] T017 [US1] En el mismo componente, añadir el desglose `Subtotal / Descuento / Total` (fila `Descuento` solo si `checkoutPreview()?.discount > 0`, FR-004) con el mismo formato agregado que la cuenta de mesa, y el estado visible **"calculando"** (FR-007a) que deshabilita "Cobrar" mientras `checkoutPreviewLoading()` sea `true` o `checkoutPreview()` sea `null` — NUNCA se pinta un total provisional. Texto "calculando…" en español de Colombia (Principio XIII). Depende de T016
- [x] T018 [US1] En el mismo componente, antes de `checkout()` (líneas ~334–337 → `store.checkoutAndSend(...)`), volver a pedir `loadCheckoutPreview` una vez más; si el `total` recién obtenido difiere del último mostrado al cajero, detener el envío, mostrar el total nuevo y exigir una segunda confirmación explícita antes de someter el pago (FR-007, [research.md D11](./research.md)) — **no** parsear el 422 del servidor. Depende de T017
- [x] T019 [US1] Tests en `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.spec.ts`: desglose `16000 / −8000 / 8000` visible (Scenario 1); $8.000 en efectivo → sin faltante, vuelto $0, "Cobrar" habilitado (Scenario 2); $10.000 → vuelto $2.000 sobre $8.000 (Scenario 3); $5.000 → "Faltan $3.000", "Cobrar" deshabilitado (Scenario 4); `checkoutPreviewLoading()` → "calculando" y "Cobrar" deshabilitado (FR-007a); preview devuelve otro total antes de `checkout()` → pide reconfirmación (FR-007). Depende de T018

**Checkpoint**: US1 funciona de punta a punta — un pedido de mesa en efectivo con promoción se
cobra al primer intento por el importe real. El endpoint de preview queda disponible para US2/US3.

---

## Phase 4: User Story 2 - Cobrar por transferencia un pedido con promoción (Priority: P1)

**Goal**: confirmar que el mismo pedido, cobrado por un método que no es efectivo, se precarga
con el total real y se emite sin el 422 "los pagos que no son en efectivo no pueden superar el
total". El importe precargado ya sale del preview tras T016 (`payment-input` recibe `total`
genérico y `PosCheckoutPanelComponent` ya le pasa `checkoutPreview()?.total`) — esta fase lo
verifica y cierra el caso del campo deshabilitado.

**Independent Test**: crear un pedido con promoción vigente (total real $8.000), elegir un método
no efectivo y confirmar que el importe precargado es $8.000 y la venta se emite al primer intento
([quickstart.md §Historia 2](./quickstart.md)).

**Depende de**: US1 (endpoint de preview T013 + cableado del panel T016).

- [x] T020 [US2] Verificar en `pos-heladeria/src/app/modules/tables/components/payment-input.component.ts` que `setMethod()` (línea ~98–99, `amount: methodId ? this.total : 0`) y el `[disabled]` del campo (línea 55) toman `this.total` del preview ya cableado por T016; ajustar únicamente si algún binding intermedio sigue apuntando a `store.totals()`. Sin lógica nueva de cálculo — el total lo manda el backend
- [x] T021 [P] [US2] Tests en `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.spec.ts` (o `payment-input.component.spec.ts`): método no efectivo con preview `total = 8000` → importe precargado $8.000, no $16.000 (Scenario 1); al pulsar "Cobrar" se llama `checkoutAndSend` con $8.000 (Scenario 2) — depende de T020
- [x] T022 [P] [US2] Test backend en `pos-backend/app/characterization_tests/test_orders_checkout.py`: `checkout_and_send` de un pedido con promoción, método transferencia, importe = total con descuento → venta emitida sin `422` "no pueden superar el total"; y el caso domicilio (`subtotal − descuento + domicilio`) del Scenario 3 — depende de T012

**Checkpoint**: US1 y US2 funcionan; ningún pedido con promoción queda imposible de cobrar por
método.

---

## Phase 5: User Story 3 - Cobrar órdenes para llevar y a domicilio (Priority: P2)

**Goal**: los pedidos manuales para llevar y a domicilio se cobran por el mismo panel con el
total correcto, y el valor del domicilio —tomado del **pedido**, no del borrador— aparece en el
desglose y entra en el total.

**Independent Test**: crear una orden manual a domicilio con envío $5.000 y una promoción que
descuenta $8.000 sobre $16.000; abrir el panel de cobro; verificar `Subtotal $16.000 /
Descuento −$8.000 / Domicilio $5.000 / Total $13.000` y que el cobro se registra por $13.000
([quickstart.md §Historia 3](./quickstart.md)).

**Depende de**: US1 (endpoint T013 — ya devuelve `delivery_fee` desde `order.delivery_fee`; el
panel T016/T017).

- [x] T023 [US3] En `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.ts`, añadir la fila `Domicilio` al desglose de T017, visible solo cuando `checkoutPreview()?.delivery_fee > 0` (FR-004), entre `Descuento` y `Total`. El valor sale del preview (pedido), nunca de `store.deliveryFee()` (borrador) — [spec.md §Defecto adyacente](./spec.md). Depende de T017
- [x] T024 [P] [US3] Tests en `pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.spec.ts`: orden para llevar con promoción → total con descuento (Scenario 1); domicilio con envío $5.000 sin promoción → fila `Domicilio $5.000` visible, total la incluye (Scenario 2); domicilio + promoción → `16000 / −8000 / 5000 / 13000` (Scenario 3); el total mostrado == total de la venta emitida (Scenario 4) — depende de T023
- [x] T025 [P] [US3] Test backend en `pos-backend/app/characterization_tests/test_orders_checkout.py`: `compute_checkout_preview` de un pedido `DELIVERY` con `delivery_fee` toma el valor del `CustomerOrder` y `total = subtotal − discount + delivery_fee`; pedido `DELIVERY` sin valor de envío → `delivery_fee = 0`, fila no aplica — depende de T012

**Checkpoint**: US1, US2 y US3 funcionan juntas y por separado; `SC-003` (ningún domicilio
bloqueado) verificable.

---

## Phase 6: User Story 4 - El descuento prometido al tomar el pedido es el que se cobra (Priority: P2)

**Goal**: la vigencia **temporal** de las promociones (fechas, día, franja) se evalúa contra el
instante de creación del pedido —congelado en `promotion_evaluated_at` (aware UTC)— en todos los
call sites de cobro y preview (incluida la cuenta consolidada de mesas fusionadas, por pedido —
FR-018a), y ese instante queda registrado en la venta emitida y **expuesto en su detalle**
(FR-011a/SC-009). El **estado** de la promoción se sigue leyendo vivo (FR-009a, ya funciona así —
[research.md D8](./research.md)).

**Independent Test**: crear un pedido a las 19:59 dentro de una promoción vigente hasta las
20:00, cobrarlo a las 20:05 y verificar que el descuento se aplica igual; repetir con dos rondas
de una misma mesa (19:59 y 20:05) y confirmar que los dos ítems se evalúan juntos contra las
19:59 ([quickstart.md §Historia 4](./quickstart.md)).

**⚠️ Depende de T001 (A-70 registrada)** y de la Fase 2 (T004 migración, T005/T006 modelos,
T008 helper). **Deroga comportamiento vigente** — cualquier characterization test que cubra hoy
el `now` de `auto_discount` se actualiza **en el mismo commit citando `A-70`** (Principio III),
nunca en silencio.

### Poblar el instante congelado al crear el pedido

- [x] T026 [US4] En `pos-backend/app/api/v1/orders/service.py::create_order` (instanciación de `CustomerOrder`, línea ~214), fijar `promotion_evaluated_at = datetime.now(timezone.utc)` (**aware UTC** — la columna es `DateTime(timezone=True)`; NO usar `.replace(tzinfo=None)`, ver [data-model.md](./data-model.md)) una sola vez, en el momento de la inserción — [contracts/vigencia-congelada-promocion.md §El flujo, paso 1](./contracts/vigencia-congelada-promocion.md)
- [x] T027 [P] [US4] En `pos-backend/app/api/v1/cart/service.py` (bloque de creación de `CustomerOrder`, línea ~579), fijar el mismo `promotion_evaluated_at = datetime.now(timezone.utc)` (aware UTC, igual que T026) — segundo punto de creación, exigido por FR-018 ([research.md D2](./research.md))
- [x] T027a [P] [US4] Test en `pos-backend/app/characterization_tests/test_cart_service.py`: confirmar un pedido por carrito QR puebla `CustomerOrder.promotion_evaluated_at` (aware UTC, ≈ hora de la confirmación); cobrar ese pedido evalúa la vigencia contra ese instante, no contra la hora del cobro — paridad de FR-018 para el flujo QR. Depende de T027, T028

### Aplicar el helper en los call sites de cobro y preview

- [x] T028 [US4] En `pos-backend/app/api/v1/orders/checkout.py`, reemplazar el `now` que se pasa a `auto_discount` por `promotion_evaluation_instant([order], now=<el now actual>)` en `pay_order` (~L289/293), `checkout_and_send` (~L481/486), `approve_payment_attempt` (~L897/899) y `confirm_cash_payment_attempt` (~L1026/1028). También en `compute_checkout_preview` (T012) si allí quedó `order.promotion_evaluated_at or now` inline en vez del helper — unificar. Depende de T008, T026
- [x] T029 [US4] En `pos-backend/app/api/v1/table_sessions/service.py`, reemplazar `now = utc_now()` antes de `checkout.auto_discount` por `checkout.promotion_evaluation_instant(orders, now=utc_now())` en `compute_bill` (~L177/187), `_close_unified` (~L665/666) y `_close_split` (~L756/772), pasando la lista `_billable_orders(...)` — FR-012a/FR-018 ([research.md D3](./research.md)). Depende de T008
- [x] T030 [P] [US4] En `pos-backend/app/api/v1/orders/tables_advanced.py::group_bill` (bucle `for o in orders`, ~L145-155), **dentro del bucle** (que ya llama `auto_discount` pedido a pedido), reemplazar el `now` compartido que se pasa a `auto_discount` por `checkout.promotion_evaluation_instant([o], now=now)` — cada pedido del grupo contra **su propio** instante congelado (FR-018a), NO el `MIN` del grupo, de modo que el preview consolidado coincida con el cobro pedido-por-pedido de T028 ([research.md D3](./research.md)). Depende de T008
- [x] T030a [P] [US4] Tests en `pos-backend/app/characterization_tests/test_orders_tables_advanced.py`: grupo de 2 mesas fusionadas, pedido A creado 19:59 (franja hasta 20:00), pedido B creado 20:05; `group_bill` a las 20:10 → descuento de A aplicado (instante 19:59), de B no; el `total` consolidado == suma de los `checkout-preview` individuales de A y B (FR-018a/SC-002). Pedido del grupo anterior a la spec (`promotion_evaluated_at` NULL) → evalúa con la hora del cobro. Depende de T030

### Registrar el instante en la venta emitida

- [x] T031 [US4] Añadir el kwarg `promotion_evaluated_at: datetime | None = None` a `build_sale` (`pos-backend/app/api/v1/sales/builder.py:72`) y asignarlo a `sale.promotion_evaluated_at` junto a donde ya fija `sale.total`/`sale.discount` (~L177–183) — [research.md D7](./research.md). Depende de T006
- [x] T032 [US4] En los 6 call sites que emiten venta con descuento (los 4 de `checkout.py` de T028 + `_close_unified`/`_close_split` de T029), pasar a `build_sale` el mismo `instant` que se usó para calcular el descuento (la salida del helper) como `promotion_evaluated_at` — FR-011a. Depende de T028, T029, T031

### Exponer el instante en el detalle de la venta (FR-011a / SC-009)

- [x] T032a [P] [US4] Agregar `promotion_evaluated_at: UtcDatetime | None = None` a `SaleResponse` (`pos-backend/app/api/v1/sales/schemas.py`, junto a `sold_at`, mismo tipo `UtcDatetime`). `GET /sales/{id}` y `GET /sales` ya responden `SaleResponse` — sin cambio de router — [data-model.md §`sales`](./data-model.md). Depende de T006
- [x] T032b [US4] En `pos-heladeria/src/app/modules/sales/interfaces/sales.interface.ts` agregar `promotion_evaluated_at?: string | null` a `Sale`; en `pos-heladeria/src/app/modules/sales/pages/sales-page.component.ts` (bloque de detalle/recibo, la lista `Subtotal / Descuento / Total`), **solo cuando `+r.discount > 0 && r.promotion_evaluated_at`**, pintar una fila "Promociones evaluadas con la vigencia del {{ r.promotion_evaluated_at | tenantDate: 'dd/MM/yyyy HH:mm' }}" (español de Colombia, Principio XIII) — para distinguir un descuento de una promoción hoy vencida de una falla (SC-009). Depende de T032a
- [x] T032c [P] [US4] Tests: backend en `pos-backend/app/characterization_tests/test_orders_checkout.py` — venta emitida con descuento → `GET /sales/{id}` devuelve `promotion_evaluated_at` = instante usado; venta anterior a esta spec → `null` (FR-011/SC-007). Frontend en `pos-heladeria/src/app/modules/sales/pages/sales-page.component.spec.ts` — detalle con `discount > 0` + `promotion_evaluated_at` muestra la fila; sin descuento no la muestra. Depende de T032 (venta escribe el instante) y T032b

### Tests de la vigencia congelada

- [x] T033 [P] [US4] Tests en `pos-backend/app/characterization_tests/test_orders_checkout.py`: pedido creado dentro de la franja, cobrado después de que vence → descuento aplicado (Scenario 1); promoción que **empieza** después de crear el pedido → no se aplica (Scenario 2); tercer ítem agregado tras vencer la franja → recálculo sobre 3 unidades con la vigencia congelada, remanente a precio pleno (Scenario 3, FR-010); pedido sin `promotion_evaluated_at` (anterior a esta spec) → evalúa con la hora del cobro, sin rama especial (FR-012); `Sale.promotion_evaluated_at` queda persistido con el instante usado (FR-011a — la exposición y el render son T032a-T032c). Depende de T032
- [x] T034 [P] [US4] Tests en `pos-backend/app/characterization_tests/test_table_sessions_service.py`: mesa con ronda 1 a las 19:59 (1 cono) y ronda 2 a las 20:05 (1 cono), promoción del 50% llevando 2 vigente hasta las 20:00 → al cobrar la cuenta completa, los dos conos se evalúan juntos contra las 19:59 y el descuento se aplica (Scenario 5, FR-012a); cuenta que mezcla pedido congelado + pedido anterior a la spec → manda el instante congelado más antiguo. Depende de T029, T032
- [x] T035 [P] [US4] Test de regresión de FR-009a en `pos-backend/app/characterization_tests/test_orders_checkout.py`: promoción pausada/borrada por el admin entre crear el pedido y cobrarlo → el descuento **desaparece** (estado leído vivo), sin error del servidor al cobrar; el instante congelado NO evita esto — [contracts/vigencia-congelada-promocion.md §Invariante 2](./contracts/vigencia-congelada-promocion.md). Depende de T028
- [x] T036 [US4] Revisar `test_orders_checkout.py`, `test_table_sessions_service.py` y `test_cart_service.py` en busca de tests que hoy afirmen el uso de la hora del cobro para la vigencia; actualizar los que cambien **citando `A-70` en el comentario del test y en el mensaje de commit** (Principio III), confirmando que FR-009a (estado vivo) y FR-011/FR-012 (nada retroactivo) siguen intactos. Si ninguno aplica, dejar constancia en el commit de que se verificó. Depende de T033, T034, T035

**Checkpoint**: US4 funciona; `SC-005`, `SC-007` y `SC-009` verificables (SC-009 con el detalle de
venta de T032b); el flujo QR (T027/T027a), la cuenta de mesa y la cuenta consolidada de mesas
fusionadas reciben el mismo cambio (FR-018/FR-018a), con el preview de mesas fusionadas coincidiendo
con el cobro pedido-por-pedido.

---

## Phase 7: User Story 5 - Ver el descuento mientras se arma la orden manual (Priority: P3)

**Goal**: la pantalla de armado de orden manual muestra el total con descuento del borrador
—recalculado en cada cambio de línea— y, si al confirmar el total real cambió, pide una segunda
confirmación antes de crear el pedido.

**Independent Test**: en la pantalla de armado, agregar un segundo cono y verificar que el
resumen pasa a `Subtotal $16.000 / Descuento −$8.000 / Total $8.000`; agregar un tercero y
verificar `Subtotal $24.000 / Descuento −$8.000 / Total $16.000`
([quickstart.md §Historia 5](./quickstart.md)).

**Depende de**: Fase 2 (T010 esquema `DraftPreviewIn`, T007 `compute_total`).

### Backend — endpoint del borrador sin guardar

- [x] T037 [P] [US5] Tests de `compute_draft_preview` en `pos-backend/app/characterization_tests/test_orders_checkout.py` (o `test_orders_service.py`): `items` con 2 conos + promoción → `{subtotal: 16000, discount: 8000, total: 8000}`; subtotal coincide centavo a centavo con el que `create_order` pondría en `OrderItem.unit_price`; `422` si `items` vacío o `product_variant_id` inexistente; `promotion_evaluated_at` = instante de la llamada (sin congelar) — [contracts/preview-borrador-orden-manual.md](./contracts/preview-borrador-orden-manual.md). Deben fallar antes de T038
- [x] T038 [US5] Implementar `compute_draft_preview(db, data: DraftPreviewIn) -> CheckoutPreviewResponse` en `pos-backend/app/api/v1/orders/checkout.py` (o `service.py`), construyendo `SaleLine` directo desde `data.items` con `compute_line_price(variant, options)` (sin `order_sale_lines`), llamando a `auto_discount(db, lines, datetime.now(timezone.utc))` y `compute_total(...)`. Sin persistir nada — [contracts/preview-borrador-orden-manual.md §Implementación](./contracts/preview-borrador-orden-manual.md). Depende de T037
- [x] T039 [US5] Registrar la ruta `POST /orders/draft-preview` en `pos-backend/app/api/v1/orders/router.py`, cuerpo `DraftPreviewIn`, respuesta `CheckoutPreviewResponse`, sin `404`/`409` — depende de T038

### Frontend — resumen del borrador

- [x] T040 [US5] Agregar a `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` las señales `draftPreview: signal<CheckoutPreview | null>`, `draftPreviewLoading`, `draftPreviewError` y `async loadDraftPreview()` que arma el cuerpo desde el borrador actual (mismas líneas que el `POST /orders` real) y llama al endpoint de T039 — mismo molde de señal+loading+error ([research.md D10](./research.md)). Depende de T039
- [x] T041 [US5] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`: disparar `store.loadDraftPreview()` en cada cambio del borrador — `addDraftFromSelection`, quitar línea, cambiar cantidad (FR-013); el resumen (línea ~273, hoy `store.totals()`) pasa a leer `store.draftPreview()` con la fila `Descuento` nueva cuando `discount > 0` (FR-004/FR-014). Depende de T040
- [x] T042 [US5] FR-015: si `loadDraftPreview()` falla, el resumen muestra el subtotal local sin descuento + un aviso "el descuento se confirma al cobrar" (español de Colombia, Principio XIII) y **no** deshabilita "Confirmar pedido" — a diferencia del panel de cobro. Depende de T041
- [x] T043 [US5] FR-015a: antes de `confirm()` (línea ~403–409 → `store.createManualOrderFromDraft()`), volver a pedir el draft-preview; si el `total` cambió respecto al último mostrado, detener, presentar el total nuevo y exigir una segunda confirmación explícita antes de `POST /orders` ([research.md D11](./research.md)). El pedido creado congela su propio instante en ese momento (FR-008), no el del borrador. Depende de T042
- [x] T044 [P] [US5] Tests en `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts`: 1 cono → `Total $8.000` sin fila descuento (Scenario 1); 2 conos → `16000 / −8000 / 8000` (Scenario 2); 3 conos → `24000 / −8000 / 16000` (Scenario 3); `loadDraftPreview()` falla → subtotal sin descuento + aviso, "Confirmar pedido" habilitado (Scenario 4, FR-015); total cambia al confirmar → segunda confirmación (Scenario 5, FR-015a). Depende de T043

**Checkpoint**: US5 funciona; el total que el cajero canta coincide con el que se cobra (`SC-005`).

---

## Phase 8: User Story 6 - Ver la condición de la promoción en el catálogo de la Terminal (Priority: P3)

**Goal**: las tarjetas del catálogo de la Terminal muestran la condición legible + equivalente
por unidad que ya publica el backend para el menú QR, en vez de la insignia local "-50%".

**Independent Test**: abrir el catálogo de la Terminal con una promoción `2 x -50%` sobre un cono
de $8.000 y verificar que la tarjeta muestra `2 x -50% · ≈ $4.000 c/u`, no `-50%`
([quickstart.md §Historia 6](./quickstart.md)).

**Cero backend. Independiente de las Fases 2–7** — `MenuVariantPromotion` (spec 066) ya llega
intacta al store ([research.md D9](./research.md)).

- [x] T045 [US6] En `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`, eliminar `productDiscountBadge()` (líneas ~409–431) y `productDiscountBadges()` (líneas ~432–441), y agregar un helper `cardPromotionText(variants): string | null` que devuelve `variants.find(v => v.promotion != null)?.promotion?.short_condition ?? null` — [contracts/catalogo-condicion-legible.md](./contracts/catalogo-condicion-legible.md)
- [x] T046 [US6] En `pos-heladeria/src/app/modules/tables/components/pos-catalog-drawer.component.ts` (líneas ~69–70), reemplazar `store.productDiscountBadges().get(p.id)` por `store.cardPromotionText(p.variants)`, pintando `short_condition`/`display_text` en lugar del badge `🏷️ -50%` — depende de T045
- [x] T047 [US6] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts` (líneas ~87–89), el mismo reemplazo de fuente de la insignia — depende de T045
- [x] T048 [P] [US6] Tests en `pos-terminal.store.spec.ts` + specs de los dos componentes: `2 x -50%` sobre cono de $8.000 → tarjeta muestra la condición con equivalente (Scenario 1); paquete `3 x $20.000` → esa condición, no precio unitario suelto (Scenario 2); producto sin promoción → `cardPromotionText` devuelve `null`, tarjeta igual que hoy (Scenario 3); `min_qty = 1` → se pinta el texto del backend tal cual, sin rama especial (Scenario 4). Depende de T046, T047

**Checkpoint**: las 6 historias funcionan de forma independiente.

---

## Phase 9: Polish & Cross-Cutting Concerns

- [x] T049 [P] Correr la batería completa de ambos repos (`python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` en `pos-backend`; `npm test` en `pos-heladeria`) y confirmar **0 fallos nuevos** frente a la línea base de T002/T003 — salvo los tests que T036 actualizó citando `A-70`
- [x] T050 [P] En `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`, hacer `grep` de `totals()` y `const discount = 0`: si tras US1/US3/US5 no queda ningún consumidor, eliminar `totals()` (línea ~883) y `itemUnitPrice`/badges muertos; si queda alguno, dejar constancia en el commit de por qué se conserva
- [x] T051 Verificación cruzada de alcance protegido ([quickstart.md §Verificación cruzada](./quickstart.md)): `SessionBillPanelComponent` (pedido de mesero) y flujo QR conservan su comportamiento salvo por FR-009; `checkout.compute_bill` (`GET /orders/tables/{table_id}/bill`) **NO se tocó** (Principio V, [research.md D4](./research.md)); ningún descuento manual disponible (FR-019); un único método de pago por cobro (FR-020)
- [ ] T052 Ejecutar el recorrido manual completo de [quickstart.md](./quickstart.md) (Historias 1–6) contra `pos-backend` + `pos-heladeria` reales con la migración aplicada, confirmando `SC-002` (total mostrado == total de la venta, al peso), `SC-008` (desglose en ≤ 1 s en el 95%, "calculando" por encima) y `SC-009` (el detalle de la venta muestra el instante congelado cuando hubo descuento). **PENDIENTE — requiere ejecución manual**: `alembic upgrade head` en `pos-backend`, ambos servicios corriendo y un tenant con turno de caja abierto (no ejecutable en este entorno). Toda la implementación de código + tests automatizados de las Historias 1–6 está completa y en verde.

---

## Phase 10: User Story 7 - Confirmar el pago de un pedido de comensal por QR con promoción (Priority: P1) — añadida 2026-09-03

**Naturaleza**: **quinta superficie** de cobro, incorporada tras entregar US1–US6 (plan.md rev.
2026-09-03; [research.md D13–D15](./research.md); [contracts/revision-pago-cajero-qr.md](./contracts/revision-pago-cajero-qr.md)).
Es la **cuarta pieza** del Summary. **No abre anomalía nueva** — `A-70` ya enumera
`confirm_cash_payment_attempt` y `approve_payment_attempt` entre sus 8 call sites, y el cambio de
vigencia ya quedó implementado en T028. **Cero endpoint nuevo, cero migración, cero helper nuevo**:
reusa `compute_checkout_preview` (T012/T013) y el método `DiningSessionService.checkoutPreview()`
(ya existe, `dining-session.service.ts:85-89`).

**Goal**: el panel "Pagos por confirmar" trata la revisión de pago del cajero de un pedido QR como
una superficie de cobro más — el chequeo previo del "monto recibido", el desglose que muestra, el
vuelto y el total que registra la venta salen todos de la misma cuenta autoritativa
(`compute_checkout_preview` → `auto_discount` con el instante congelado del pedido). Nunca más un
422 "el monto recibido es menor al total de la orden" con el total inflado.

**Independent Test**: crear un pedido de comensal por QR con 2 conos a $8.000 y una promoción
vigente del 50% llevando 2; abrir "Pagos por confirmar"; verificar
`Subtotal $16.000 / Descuento −$8.000 / Total $8.000`; registrar $10.000 en efectivo, pulsar
"Confirmar efectivo" y confirmar que la venta se emite por $8.000, el vuelto es $2.000 y el pedido
pasa a cocina — sin el mensaje de "monto menor al total" ([quickstart.md §Historia 7](./quickstart.md)).

**Depende de**: Fase 3 (T012 `compute_checkout_preview`, T013 ruta — **ya entregadas**) y Fase 6
(T028 instante congelado en `confirm_cash_payment_attempt`/`approve_payment_attempt`, T027/T027a el
flujo QR congela su instante al confirmar el carrito — **ya entregadas**). Independiente de US2–US6
en código nuevo.

### Backend — el chequeo previo del efectivo pasa a la cuenta autoritativa (FR-023, research.md D13)

- [x] T053 [P] [US7] Tests en `pos-backend/app/characterization_tests/test_orders_payment_gate.py`: pedido QR con 2 conos + promoción vigente del 50% llevando 2 → `confirm_cash_payment_attempt` con `amount_received = 8000` confirma sin 422 y `change_amount = 0` (Scenario 3); con `10000` → `change_amount = 2000` calculado sobre $8.000 (Scenario 2); con `5000` → 422 cuyo total citado es `8000` (faltan $3.000), no `16000` (Scenario 4); pedido QR `DELIVERY` con promoción + domicilio → el chequeo previo compara contra `subtotal − descuento + domicilio`; pedido QR creado 19:59 dentro de una promoción vigente hasta las 20:00, confirmado 20:05 → descuento aplicado, total $8.000 (Scenario 6, instante congelado vía T027); promoción pausada por el admin entre el pedido y el cobro → `compute_checkout_preview(...).total` devuelve $16.000 (estado vivo, FR-009a) y el chequeo previo valida contra ese valor. Los tests **sin** promoción de este fichero deben seguir en verde sin editarse (`discount = 0`). Deben fallar antes de T054 — [contracts/revision-pago-cajero-qr.md §Acceptance Scenarios](./contracts/revision-pago-cajero-qr.md)
- [x] T054 [US7] En `pos-backend/app/api/v1/orders/checkout.py::confirm_cash_payment_attempt` (~L1142), reemplazar `total = _order_total(db, attempt.order_id)` por `total = compute_checkout_preview(db, attempt.order_id).total`. El `attempt.change_amount = amount_received - total` (~L1152) usa ese mismo `total` → coincide al peso con `Sale.change_given` de `build_sale` (FR-022). **Eliminar la función `_order_total`** (~L939–955) — su única llamada era esta ([research.md D13](./research.md), [contracts/revision-pago-cajero-qr.md §Backend](./contracts/revision-pago-cajero-qr.md)). Depende de T053
- [x] T055 [P] [US7] Verificar por `grep -rn "_order_total" pos-backend/app/` que **no queda ninguna referencia**; confirmar que `approve_payment_attempt` (transferencia) **no cambia en backend** — ya emite la venta con `sum(line_total) − promo_discount + delivery_fee` e instante congelado desde T028/T032 — y dejar constancia de esa revisión en el mensaje de commit (invariante 6 de [contracts/revision-pago-cajero-qr.md](./contracts/revision-pago-cajero-qr.md)). Depende de T054

### Frontend — `PaymentAttemptReviewPanelComponent` consume el preview (FR-021/FR-022, research.md D14)

- [x] T056 [US7] En `pos-heladeria/src/app/modules/tables/components/payment-attempt-review-panel.component.ts`, agregar los signals locales `checkoutPreview: signal<CheckoutPreview | null>`, `checkoutPreviewLoading: signal<boolean>`, `checkoutPreviewError: signal<string | null>` y el método `async loadCheckoutPreview()` que llama a `this.api.checkoutPreview(this.order.id)` (ya existe en `DiningSessionService`, `:85-89`), invocado en `ngOnChanges` junto a `this.load()` (`:242`). Molde señal-loading-error como el resto de spec 073, **local al componente** — este flujo no pasa por `PosTerminalStore` ([contracts/revision-pago-cajero-qr.md §Carga del preview](./contracts/revision-pago-cajero-qr.md)). Depende de T013 (ruta, ya entregada)
- [x] T057 [US7] En el mismo componente, `orderTotal()` (`:266-272`, hoy `Σ unit_price × quantity` local) pasa a devolver `checkoutPreview()?.total ?? null`, y `cashChangePreview()` (`:278-282`) calcula el vuelto sobre ese `total` real, nunca sobre el bruto — devuelve `null` mientras no haya preview. El comentario que hoy cita `_order_total` (`:265-267`) se actualiza. Depende de T056
- [x] T058 [US7] En el mismo componente, añadir a la plantilla el desglose `Subtotal / Descuento / Domicilio / Total` (FR-022, mismo formato agregado de FR-004; `Descuento` y `Domicilio` solo cuando `> 0`) y el estado visible **"calculando…"** (español de Colombia, Principio XIII) mientras `checkoutPreviewLoading()` sea `true` o `checkoutPreview()` sea `null`: deshabilita "Confirmar efectivo", "Aprobar" y el botón de confirmar efectivo; NUNCA se pinta un total provisional (FR-024 → regla de FR-007a). Depende de T057
- [x] T059 [US7] FR-024 — al abrir el panel: si `checkoutPreview()?.total` difiere del total que la tarjeta venía mostrando (`Σ (discounted_line_total ?? unit_price × quantity)` de `order.items`), mostrar un aviso "el total cambió respecto al declarado por el comensal: antes $X, ahora $Y" y exigir que el cajero lo reconozca (signal `totalChangeAck`) antes de habilitar "Confirmar efectivo" / "Aprobar" — cubre el Edge Case de tarjeta desactualizada y el de promoción pausada (FR-009a) ([contracts/revision-pago-cajero-qr.md §Reconfirmación](./contracts/revision-pago-cajero-qr.md), [research.md D15](./research.md)). Depende de T058
- [x] T060 [US7] FR-024 — justo antes de `confirmCash()` y de `approve()` (`:330`, `:295`): volver a pedir `this.api.checkoutPreview(this.order.id)` una vez más; si el `total` cambió respecto al último mostrado, detener la acción, presentar el total nuevo y exigir una segunda confirmación explícita — **nunca** dejar que el 422 del backend sea el aviso ([research.md D11/D15](./research.md)). Para `confirmCash()` re-validar además que `amountReceived` siga cubriendo el total nuevo. Depende de T059
- [x] T061 [US7] En `pos-heladeria/src/app/modules/tables/components/payment-validation-block.component.ts`, eliminar la fila de pie `$ {{ total(order) | number: '1.2-2' }}` (`:102-106`), el método `total(order)` (`:136-147`) y el import `DecimalPipe` si queda sin uso — el total autoritativo y su desglose los muestra ahora el `app-payment-attempt-review-panel` embebido ([contracts/revision-pago-cajero-qr.md §PaymentValidationBlockComponent](./contracts/revision-pago-cajero-qr.md)). Depende de T058

### Tests del frontend

- [x] T062 [P] [US7] Tests en `pos-heladeria/src/app/modules/tables/components/payment-attempt-review-panel.component.spec.ts`: desglose `16000 / −8000 / 8000` visible (Scenario 1); $10.000 efectivo → `confirmCashPaymentAttempt` llamado, vuelto $2.000 sobre $8.000 (Scenario 2); $8.000 → vuelto $0 (Scenario 3); $5.000 → "faltan $3.000" sobre $8.000, "Confirmar efectivo" deshabilitado (Scenario 4); transferencia con comprobante → panel muestra `Total $8.000` antes de "Aprobar" (Scenario 5); `checkoutPreviewLoading()` → "calculando" + acciones deshabilitadas (FR-024/FR-007a); `checkoutPreview().total` ≠ total de la tarjeta al abrir → aviso + `totalChangeAck` requerido (Scenario 7, FR-024); preview devuelve otro total justo antes de confirmar/aprobar → segunda confirmación (research.md D15). Depende de T060, T061
- [x] T063 [P] [US7] Tests en `pos-heladeria/src/app/modules/tables/components/payment-validation-block.component.spec.ts`: la fila de pie `$ total` ya no se renderiza; no queda ninguna referencia a `discounted_line_total` ni a `total(order)` en el componente; el bloque sigue renderizando cada pedido con su `app-payment-attempt-review-panel` independiente. Depende de T061

**Checkpoint**: US7 funciona de punta a punta — un pedido QR con promoción se confirma (efectivo o
transferencia) al primer intento por el importe real, el panel muestra el desglose autoritativo y
avisa si el total cambió respecto al que declaró el comensal. `SC-002a` verificable.

---

## Phase 11: Polish & Verificación — User Story 7

- [x] T064 [P] [US7] Correr ambas baterías (`python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` en `pos-backend`; `npm test` en `pos-heladeria`) y confirmar **0 fallos nuevos** frente a la línea base de T002/T003 y al estado post-US6, con especial atención a `test_orders_payment_gate.py`, `payment-attempt-review-panel.component.spec.ts` y `payment-validation-block.component.spec.ts`
- [x] T065 [US7] Verificación cruzada de alcance ([quickstart.md §Verificación cruzada de alcance protegido](./quickstart.md)): confirmar por `grep` que `_order_total` no existe y que `confirm_cash_payment_attempt` usa `compute_checkout_preview(...).total`; confirmar que `OrderPaymentAttempt` / `PaymentAttemptResponse` no ganaron ningún campo ([data-model.md](./data-model.md)); confirmar que `approve_payment_attempt` no cambió en backend
- [ ] T066 [US7] Ejecutar el recorrido manual de [quickstart.md §Historia 7](./quickstart.md) (puntos 1–8) contra `pos-backend` + `pos-heladeria` reales con la migración aplicada y un pedido creado por el carrito del comensal (QR): `SC-002a` (total del panel == total de la venta, al peso; ningún pedido QR con promoción bloqueado por "monto menor al total") y el Scenario 7 (marca de total cambiado + reconfirmación). **PENDIENTE — requiere ejecución manual** (mismo entorno que T052).

---

## Dependencies & Execution Order

### Fases

- **Fase 1 (Setup)**: sin dependencias. **T001 (A-70) bloquea la Fase 6 entera y T008.**
- **Fase 2 (Foundational)**: depende de Fase 1. **Bloquea US1–US5.** No bloquea US6.
- **Fase 3 (US1)**: depende de Fase 2. Entrega el endpoint `checkout-preview` del que dependen US2 y US3.
- **Fase 4 (US2)**: depende de US1 (T013, T016).
- **Fase 5 (US3)**: depende de US1 (T013, T016, T017).
- **Fase 6 (US4)**: depende de T001 + Fase 2 (T004, T005, T006, T008). Independiente de US1–US3/US5 salvo por unificar el helper dentro de `compute_checkout_preview` (T028 menciona T012). Toca también el módulo `sales/` (T032a backend, T032b/T032c frontend) para exponer el instante en el detalle de venta (FR-011a/SC-009) — sin solape con US1/US5.
- **Fase 7 (US5)**: depende de Fase 2 (T007, T010). Independiente de US1–US4.
- **Fase 8 (US6)**: **independiente de todo lo demás** — puede hacerse en cualquier momento tras la Fase 1.
- **Fase 9 (Polish)**: depende de todas las historias en alcance que se vayan a entregar.
- **Fase 10 (US7)**: depende de la Fase 3 (T012/T013 `compute_checkout_preview` + ruta) y de la Fase 6 (T028 instante congelado en `confirm_cash_payment_attempt`, T027/T027a flujo QR) — **todas ya entregadas**. No añade endpoint, migración ni helper. Independiente de US2–US6.
- **Fase 11 (Polish US7)**: depende de la Fase 10.

### Historias — orden de prioridad y grado de independencia

- **US1 (P1)** — MVP. Necesita solo Foundational.
- **US2 (P1)** — construye sobre US1 (mismo endpoint y mismo panel). Verificable por separado con su propio test.
- **US3 (P2)** — construye sobre US1. Verificable por separado.
- **US4 (P2)** — independiente de US1–US3 en código; comparte solo `promotion_evaluation_instant` y la migración (Foundational). Requiere `A-70`.
- **US5 (P3)** — independiente; solo comparte `compute_total` y el esquema con Foundational.
- **US6 (P3)** — totalmente independiente (frontend puro, dato ya disponible).
- **US7 (P1)** — quinta superficie, añadida el 2026-09-03. Reusa el endpoint de US1 y el instante congelado de US4 sin tocarlos; solo cambia el chequeo previo de `confirm_cash_payment_attempt` (elimina `_order_total`) y cablea el preview en el panel "Pagos por confirmar". Verificable por separado con `test_orders_payment_gate.py` y `payment-attempt-review-panel.component.spec.ts`.

### Dentro de cada historia

- Tests antes de implementación donde el contrato lo compromete (T011→T012, T037→T038, etc.).
- Backend (endpoint) antes de frontend (consumo).
- Señales del store antes del componente que las lee.

---

## Parallel Opportunities

- **Fase 1**: T002 y T003 en paralelo (repos distintos).
- **Fase 2**: T005, T006, T007, T009, T010 en paralelo (archivos distintos). T004 primero (o en paralelo, pero T005/T006 lo asumen aplicado para probar). T008 tras T001+T005.
- **Fase 3 (US1)**: T011 en paralelo con nada más de backend; T015 en paralelo con T016+ una vez existe T014.
- **US6 completa** puede correr en paralelo a **US1–US5** con otro desarrollador — no comparte ningún archivo con las demás salvo `pos-terminal.store.ts` y `manual-order-page.component.ts` (coordinar el merge de esos dos con quien haga US5).
- **US4 backend** (T026–T032a) y **US5 backend** (T037–T039) tocan `checkout.py`/`router.py` en zonas distintas — posible en paralelo con cuidado en el merge. T032a toca `sales/schemas.py` (sin solape).
- **US4 frontend del detalle de venta** (T032b/T032c) toca `src/app/modules/sales/` — sin solape con `src/app/modules/tables/` de US1–US3/US5/US6.
- **Fase 10 (US7)**: T053 (tests backend) en paralelo con T056 (signals del panel, frontend). T055 tras T054; T062/T063 tras T060/T061. El backend de US7 toca `checkout.py::confirm_cash_payment_attempt` y elimina `_order_total` — zona distinta de los endpoints de preview de US1/US5. El frontend de US7 toca `payment-attempt-review-panel.component.ts` y `payment-validation-block.component.ts`, sin solape con ningún archivo de US1–US6.

### Ejemplo — Fase 2 en paralelo

```bash
# Tras T001 y T004:
Task T005: "promotion_evaluated_at en customer_order.py"
Task T006: "promotion_evaluated_at en sale.py"
Task T007: "extraer compute_total en sales/builder.py"
Task T010: "CheckoutPreviewResponse + DraftPreviewIn en orders/schemas.py"
# Luego T008 (tras T005) y T009 (tras T008).
```

### Ejemplo — reparto por desarrollador tras la Fase 2

```
Dev A: US1 → US2 → US3   (autoridad del monto en el panel de cobro)
Dev B: US4               (vigencia congelada, tras A-70)
Dev C: US5 + US6         (borrador de orden manual + catálogo)
```

---

## Implementation Strategy

### MVP (solo US1)

1. Fase 1 (Setup) — incluye registrar `A-70` aunque el MVP no la use todavía (barato y desbloquea US4).
2. Fase 2 (Foundational) — migración, modelos, helper, `compute_total`, esquemas.
3. Fase 3 (US1) — endpoint `checkout-preview` + panel de cobro en efectivo.
4. **DETENERSE Y VALIDAR**: cobrar en efectivo un pedido de mesa con promoción, al primer intento, por el importe real (quickstart Historia 1).
5. Desplegar si está listo — ya resuelve el defecto reportado en su forma más común.

### Entrega incremental

1. Setup + Foundational → base lista.
2. + US1 → cobro en efectivo correcto (MVP). Validar y desplegar.
3. + US2 → transferencia deja de ser imposible. Validar y desplegar.
4. + US3 → para llevar y domicilio (incluye el fix del valor del domicilio). Validar y desplegar.
5. + US4 → vigencia congelada (requiere `A-70`). Validar `SC-005`/`SC-007`/`SC-009`. Desplegar.
6. + US5 → total con descuento al armar la orden manual. Desplegar.
7. + US6 → condición legible en el catálogo. Desplegar.
8. + US7 → el panel "Pagos por confirmar" deja de bloquear pedidos QR con promoción; se elimina `_order_total`. Validar `SC-002a`. Desplegar.
9. Fase 9 + Fase 11 (Polish) sobre lo que se haya entregado.

Cada incremento agrega valor sin romper el anterior. US6 puede adelantarse en cualquier punto.
US7 llegó como incremento posterior (2026-09-03), sobre US1–US6 ya entregadas — no reabre ninguna
de las piezas anteriores.

---

## Notes

- `[P]` = archivos distintos, sin dependencia de una tarea sin terminar.
- La etiqueta `[Story]` mapea cada tarea a su historia para trazabilidad (Principio XII).
- Los dos repos se versionan por separado: un commit por repo por tarea (o grupo lógico), citando el `FR`/contrato y —para la Fase 6— `A-70`.
- Verificar que los tests fallan antes de implementar donde el contrato compromete el test primero.
- Detenerse en cada checkpoint para validar la historia de forma aislada.
- Evitar: reintroducir cálculo de descuento en el navegador (spec 063 FR-023); tocar `checkout.compute_bill` (Principio V); congelar el **estado** de la promoción (FR-009a); guardar el instante como `datetime` naive (`local_now()` lo tomaría como hora local del tenant — [data-model.md](./data-model.md)); usar el `MIN` del grupo en `group_bill` (cada pedido de mesas fusionadas va contra su propio instante — FR-018a); dejar `_order_total` como segunda fuente de verdad del total (US7 lo elimina — [research.md D13](./research.md)); añadir campos a `OrderPaymentAttempt`/`PaymentAttemptResponse` (US7 no persiste nada nuevo); tocar `approve_payment_attempt` en backend (ya emite bien desde T028 — solo se revisa).
