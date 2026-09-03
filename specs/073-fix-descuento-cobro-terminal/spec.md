# Feature Specification: Corrección — la Terminal de mesas cobra sin aplicar el descuento por promoción

**Feature Branch**: `073-fix-descuento-cobro-terminal`

**Created**: 2026-09-02

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección**, con una decisión de negocio nueva adherida
(FR-009, ver más abajo). Igual que las specs
[019](../019-correccion-cuenta-mesas-fusionadas/spec.md),
[029](../029-correccion-cobro-cierre-mesa/spec.md),
[050](../050-correccion-liberar-mesa-pedido-cancelado/spec.md),
[069](../069-fix-creacion-tenant/spec.md) y
[072](../072-fix-deteccion-turno-caja/spec.md), cita nombres de archivo, función y línea del
código actual (`pos-heladeria`, `pos-backend`) porque son el contrato observable que se está
corrigiendo, no una fuga de detalles de implementación.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-09-02, reproduciendo el bug con un pedido real
(1 cono de helado a $8.000, promoción del 50% llevando 2). En la ronda de aclaración previa a
esta spec, el dueño decidió: (1) el descuento se muestra como una **fila agregada**
Subtotal / Descuento / Total, no prorrateado línea por línea; (2) el alcance cubre las
**cuatro** superficies (cobro en terminal de mesas, cobro de para llevar y domicilio, pantalla
de armado de orden manual, catálogo de productos de la terminal); (3) el defecto del **valor
del domicilio** entra en esta misma spec; (4) **el descuento que manda es el vigente al tomar
el pedido, no al cobrarlo**.

**Autorización adicional (2026-09-03, tercera ronda de aclaración)**: tras implementar la spec, el
dueño reprodujo el bug todavía presente en el panel **"Pagos por confirmar"** (el cajero pulsa
"Confirmar efectivo" o "Aprobar comprobante" para un pedido de comensal por QR) — evidencia
adjunta: la tarjeta muestra `$ 8.000` pero al confirmar $10.000 en efectivo el sistema responde
*"El monto recibido (10000) es menor al total de la orden (16000.00)"*. Decisión: esa revisión de pago del cajero entra como
**quinta superficie** en alcance (FR-021 y ss.); tanto el chequeo previo del "monto recibido" como
el total, el vuelto y el desglose que ese panel muestra DEBEN usar la misma cuenta autoritativa
—descuento por promoción, valor de domicilio e instante congelado— que construye la venta. Queda
**derogada** la fila "Comensal por QR — Correcto" del análisis del defecto de abajo. No abre
anomalía nueva: es el mismo defecto de FR-001/FR-002 (un total sin descuento calculado fuera de la
autoridad de la venta), solo que no se había detectado en esta superficie.

**Anomalía asociada**: la decisión (4) deroga una regla de negocio deliberada y vigente del
backend — hoy `auto_discount(db, lines, now)` se evalúa siempre con la hora **del cobro**
(`app/api/v1/orders/checkout.py:293`, `:486`, `:899`, `:1028`,
`app/api/v1/table_sessions/service.py:186` y `app/api/v1/orders/tables_advanced.py:145` — lista
completa en [research.md](./research.md) D3). Requiere una entrada nueva
`A-70 — [DECISIÓN DE NEGOCIO — spec 073]` en
[registro-de-anomalias.md](../000-reconocimiento/registro-de-anomalias.md). Las decisiones (1),
(2) y (3) **no** abren anomalía: restauran el comportamiento pretendido. El **alcance de A-70**
quedó precisado en la ronda de aclaración posterior a la redacción (ver **Clarifications**): cubre
también la cuenta de mesa que reúne varias rondas (FR-012a) y la cuenta consolidada de mesas
fusionadas (FR-018a), y **no** incluye congelar el estado de la promoción (FR-009a), que se sigue
leyendo vivo.

**Input**: User description (verbatim): "encontre un problema el cual hay que solucionar,
actualmente es que en la terminal de mesas cuando un prodcuto tiene descuento al momento de
cobrar no detecta ese descuento, esto lo hice creando una orden con pago en efectivo, si
unproducto en desceunto da 8000 en la terminal de mesas muestra el precio pero al registrar el
efectivo si el valo es menor al precio total sin desceunto me meustra mensaje de error por
ejemplo 1 cono de helado cuesta 8000 y hay una promocion del 50% por 2 conos, cuando voy a
registrar en la terminal de mesas ingreso que el cliente me dio 8000 en efectivo me manda error
diciendo que el precio a pagar con 16000. revisa si este mismo error se ve en la opcion de apgo
por transferencia y creacion de ordenes manuales para llevar y domicilio."

---

## Contexto del defecto (verificado sobre el código actual)

### Causa raíz — el importe que se valida lo calcula el navegador, no el backend

El panel de cobro de la Terminal valida el pago contra un total que arma el propio navegador:

- `PosCheckoutPanelComponent.issue` (`src/app/modules/tables/components/pos-checkout-panel.component.ts:320-322`)
  valida con `store.totals().total`, y ese mismo valor viaja como `[total]` a
  `app-payment-input` (línea 173).
- `PosTerminalStore.totals` (`src/app/modules/tables/services/pos-terminal.store.ts:883-896`)
  fija **`const discount = 0`** de forma literal, y suma las líneas de `cartView()`, cuyo precio
  unitario sale de `itemUnitPrice()` (`pos-terminal.store.ts:395-402`):
  `discounted_unit_price ?? unit_price`.
- **`discounted_unit_price` solo lo escribe el flujo del carrito del comensal (QR)**
  (`app/api/v1/cart/service.py:275-286` y `:601-610`). Los pedidos creados desde la Terminal o
  desde la pantalla de orden manual se crean en `app/api/v1/orders/service.py:245-260`, que
  puebla `unit_price` y **nunca** `discounted_unit_price`. Queda siempre `NULL`, así que
  `itemUnitPrice()` cae siempre al precio pleno.
- El backend, en cambio, **sí** aplica la promoción al cobrar: `checkout_and_send` llama a
  `auto_discount()` (`app/api/v1/orders/checkout.py:486`) y construye la venta con ese
  descuento.

Resultado: la pantalla exige $16.000 y la venta real vale $8.000.

### El mismo defecto en los demás flujos (respuesta a la pregunta del reporte)

| Flujo | Método | Comportamiento observado |
|---|---|---|
| Terminal de mesas, pedido en `recibida` | Efectivo | **Bloqueo en el navegador**: `paymentIssue()` (`services/payment-draft.util.ts:60-77`) devuelve "Faltan $8.000 para cubrir la cuenta." y deshabilita "Cobrar". La única salida es cobrar $16.000 → el cliente paga de más y la venta registra un vuelto de $8.000 que nadie entregó, descuadrando el arqueo del turno. |
| Terminal de mesas | Transferencia (no efectivo) | **Peor: no hay salida.** `PaymentInputComponent.setMethod()` (`components/payment-input.component.ts:103-105`) precarga el importe con el total mostrado y el campo queda **deshabilitado** (línea 56). Al pulsar "Cobrar", el backend responde 422 *"Los pagos que no son en efectivo (16000) no pueden superar el total (8000): el vuelto solo sale del efectivo."* (`app/api/v1/sales/builder.py:169-175`). El pedido **no se puede cobrar de ninguna manera**. |
| Órdenes manuales **para llevar** y **domicilio** | Ambos | Se cobran por el mismo `PosCheckoutPanelComponent` → mismo `store.totals()` → mismo fallo. |
| Pantalla de armado de orden manual | — | `manual-order-page.component.ts:273-286` muestra Subtotal/Total desde el mismo `store.totals()`. El cajero le canta al cliente $16.000 antes de guardar el pedido. |
| Catálogo de productos de la Terminal | — | Solo pinta una insignia local ("-50%", `pos-terminal.store.ts:405-441`); no muestra ni la condición de la promoción ni el equivalente por unidad, aunque el backend ya los publica. |
| Pedido de mesero ya enviado a cocina | Ambos | **Correcto**: usa `SessionBillPanelComponent` con el total de `compute_bill` (`app/api/v1/table_sessions/service.py:159-212`), que sí aplica `auto_discount`. |
| Comensal por QR — menú y carrito | — | **Correcto**: el carrito del comensal ya escribe `discounted_unit_price`/`discounted_line_total` (`app/api/v1/cart/service.py:601-618`). |
| **Comensal por QR — revisión de pago del cajero** ("Pagos por confirmar": "Confirmar efectivo" / "Aprobar comprobante") | Efectivo | **Defecto confirmado (evidencia 2026-09-03).** El chequeo previo `confirm_cash_payment_attempt` → `_order_total` (`app/api/v1/orders/checkout.py:939-955`) suma `unit_price × cantidad + domicilio` y **nunca resta el descuento**, así que rechaza con 422 *"El monto recibido (10000) es menor al total de la orden (16000.00)"* aunque la venta que construye a continuación (línea 1163-1180) sí aplica `auto_discount` con el instante congelado. El frontend repite el fallo en `PaymentAttemptReviewPanelComponent.orderTotal()` (`payment-attempt-review-panel.component.ts:266-282`), que calcula la vista previa del vuelto sobre el precio pleno. La tarjeta contenedora `PaymentValidationBlockComponent.total()` (`payment-validation-block.component.ts`) muestra `$ 8.000` porque lee el `discounted_line_total` **congelado del carrito**, que puede diferir del recálculo por instante congelado / cambio de ítems / promoción pausada. |
| **Comensal por QR — revisión de pago del cajero** | Transferencia | `approve_payment_attempt` (`checkout.py:990-1082`) ya construye la venta por el total con descuento (línea 1041-1044) tras la implementación de esta spec, así que hoy no rompe; pero el cajero **no ve** ese total antes de pulsar "Aprobar" — si la promoción cambió y el total sube, aprueba a ciegas. |

### Defecto adyacente confirmado — el valor del domicilio tampoco entra en el total del cobro

`PosTerminalStore.totals` suma el domicilio solo si `orderTypeTab() === 'domicilios'` **y** la
señal `deliveryFee()` está cargada (`pos-terminal.store.ts:893`). Ambas pertenecen al
**borrador** de la pantalla de armado, que tiene su **propia instancia** del store
(`manual-order-page.component.ts:27-34`, `providers: [PosTerminalStore]`) y navega de vuelta a
la Terminal al confirmar (línea 407). En la Terminal, `orderTypeTab` vale su valor por defecto
`'mesas'` (`pos-terminal.store.ts:337`) y `deliveryFee()` es `null`, así que el total mostrado
**omite el domicilio**, mientras `build_sale` sí lo suma (`app/api/v1/sales/builder.py:144`) →
422 *"El pago (X) no cubre el total (Y)"*. Es independiente de las promociones, pero es el
mismo síntoma para el cajero y toca exactamente el mismo cálculo.

### Naturaleza del motor de descuentos (restringe lo que se puede mostrar)

El motor es **por conjunto de variantes con umbral de cantidad**: cada regla tiene `min_qty`
(`app/models/promotion.py:137`) y `evaluate_variant_sets()` trocea las unidades elegibles en
bloques completos de `min_qty` (`app/api/v1/promotions/service.py:176-183`), dejando el
remanente a precio pleno. Un descuento **no** es una propiedad de una unidad suelta: depende de
cuántas unidades del conjunto tenga el pedido completo. Esto es lo que hace inviable pintar un
"precio con descuento" fijo por producto en el catálogo (ver FR-017), y lo que obliga a
recalcular el descuento cada vez que cambian los ítems del pedido (ver FR-010).

---

## Clarifications

### Session 2026-09-03

- Q: ¿La corrección debe cubrir también el panel "Pagos por confirmar" de la Terminal (el cajero
  pulsa "Confirmar efectivo" o "Aprobar comprobante" para un pedido de comensal por QR), de modo
  que el chequeo del "monto recibido" y el total/vuelto/desglose que muestra usen la misma cuenta
  autoritativa con descuento, domicilio e instante congelado que emite la venta? → A: Sí, como
  quinta superficie en alcance. El chequeo previo del backend y lo que muestra el panel usan la
  misma cuenta autoritativa que construye la venta; queda derogada la fila "Comensal por QR —
  Correcto" del análisis del defecto.
- Q: El panel "Pagos por confirmar" solo existe para pedidos de comensal por QR (siempre de
  mesa); los pedidos para llevar y a domicilio se cobran directo en el panel de cobro de la
  Terminal. ¿A qué te refieres con que el arreglo cubra "para llevar, desde la mesa y a
  domicilio"? → A: La pantalla QR es solo de mesa: se corrige ahí y, además, se añaden escenarios
  de aceptación por tipo de orden (mesa / para llevar / domicilio, con y sin promoción y con
  domicilio) en el panel de cobro de la Terminal, para dejar constancia de que el arreglo aplica
  a los tres. No hay otra pantalla de confirmación de pago para llevar/domicilio.
- Q: En el panel "Pagos por confirmar", si al confirmar el pago el total autoritativo resulta
  distinto del que vio el comensal al declarar el pago (p. ej. la promoción se pausó entre el
  pedido y el cobro y el total sube), ¿qué debe ver y hacer el cajero? → A: El panel muestra el
  desglose autoritativo en vivo (Subtotal / Descuento / Domicilio / Total) y, si difiere del que
  la tarjeta venía mostrando, lo marca y exige que el cajero lo confirme antes de emitir la venta
  — misma regla que FR-007/FR-007a.

### Session 2026-09-02

- Q: Cuando una mesa acumula varias rondas de pedidos creadas a horas distintas, ¿contra qué
  instante se evalúa la vigencia de las promociones al cobrar la cuenta completa? → A: Un solo
  instante para toda la cuenta de la mesa, el del pedido más antiguo pendiente de cobro,
  conservando la agrupación de líneas actual (todas las líneas del mismo comensal, de todas las
  rondas, se evalúan juntas).
- Q: Si el administrador pausa, borra o desactiva una promoción después de tomado el pedido pero
  antes de cobrarlo, ¿el pedido conserva el descuento? → A: No. Solo se congela la vigencia
  **temporal** (fecha, día y franja horaria); el estado de la promoción se lee vivo en cada
  cálculo, sin historial de estados.
- Q: ¿Cuánto puede tardar el sistema en mostrarle al cajero el total con descuento antes de que se
  considere una falla? → A: ≤ 1 segundo en el 95% de las veces; por encima de eso, estado visible
  de "calculando" y botón "Cobrar" deshabilitado hasta que llegue el total.
- Q: Si la franja de una promoción vence mientras el cajero arma una orden manual, ¿el precio ya
  cantado se sostiene o cambia al confirmar? → A: El instante congelado sigue siendo el de crear
  el pedido (FR-008), pero si el total cambió respecto al mostrado, el sistema presenta el total
  nuevo y pide una segunda confirmación antes de crear el pedido.
- Q: Cuando una venta quede emitida con un descuento cuya promoción ya estaba vencida a la hora
  del cobro, ¿qué debe quedar registrado para poder explicarlo después? → A: La venta emitida
  guarda el instante congelado con el que se evaluaron sus promociones, junto a las promociones
  aplicadas que ya registra hoy, consultable desde el detalle de la venta.

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Cobrar en efectivo un pedido con promoción (Priority: P1)

El cajero atiende una mesa que pidió 2 conos de helado ($8.000 c/u) con una promoción vigente
del 50% llevando 2. La Terminal debe exigirle $8.000, no $16.000: cuando el cliente le entrega
$8.000, el cobro se registra sin ningún mensaje de importe insuficiente.

**Why this priority**: es el defecto reportado, ocurre en el flujo más usado del negocio y su
única salida actual (cobrar el precio pleno) **le cobra de más al cliente** y descuadra el
arqueo del turno con un vuelto que nunca se entregó.

**Independent Test**: crear un pedido de mesa con un producto cubierto por una promoción
vigente, cobrarlo en efectivo con el importe exacto con descuento, y verificar que la venta se
emite al primer intento por ese mismo importe.

**Acceptance Scenarios**:

1. **Given** un pedido de mesa con 2 conos a $8.000 y una promoción vigente del 50% llevando 2,
   **When** el cajero abre el panel de cobro, **Then** ve `Subtotal $16.000`,
   `Descuento −$8.000` y `Total $8.000`.
2. **Given** ese mismo panel, **When** el cajero registra $8.000 en efectivo, **Then** no
   aparece ningún mensaje de importe faltante, el vuelto se muestra en $0 y el botón "Cobrar"
   queda habilitado.
3. **Given** ese mismo panel, **When** el cajero registra $10.000 en efectivo, **Then** el
   vuelto mostrado es $2.000 (calculado sobre $8.000, no sobre $16.000).
4. **Given** ese mismo panel, **When** el cajero registra $5.000 en efectivo, **Then** el
   sistema indica que faltan $3.000 y no permite cobrar.
5. **Given** el cobro registrado por $8.000, **When** se consulta la venta emitida, **Then** su
   total es $8.000 y el vuelto registrado es $0.

---

### User Story 2 - Cobrar por transferencia un pedido con promoción (Priority: P1)

El mismo pedido, cobrado por un método que no es efectivo. Hoy el importe queda fijado al precio
pleno con el campo deshabilitado, así que el cobro es **imposible**: el sistema lo rechaza
siempre. Debe fijarse al total real y registrarse sin error.

**Why this priority**: es un bloqueo duro, no una molestia — el pedido queda sin poder cobrarse
y sin factura, obligando al cajero a rechazarlo o a cobrar por fuera del sistema.

**Independent Test**: crear un pedido con promoción vigente y cobrarlo por transferencia,
verificando que se registra al primer intento sin ningún error del servidor.

**Acceptance Scenarios**:

1. **Given** un pedido de mesa con promoción vigente cuyo total real es $8.000, **When** el
   cajero selecciona un método que no es efectivo, **Then** el importe se precarga en $8.000
   (no en $16.000).
2. **Given** ese estado, **When** el cajero pulsa "Cobrar", **Then** la venta se emite por
   $8.000 y no aparece ningún mensaje sobre pagos que no son en efectivo superando el total.
3. **Given** un pedido a domicilio con promoción vigente y valor de domicilio, **When** se cobra
   por transferencia, **Then** el importe precargado es `subtotal − descuento + domicilio` y el
   cobro se registra al primer intento.

---

### User Story 3 - Cobrar órdenes para llevar y a domicilio (Priority: P2)

Los pedidos manuales para llevar y a domicilio se cobran por el mismo panel y arrastran el mismo
defecto, más el del valor del domicilio: el total mostrado omite el domicilio aunque el sistema
lo cobre.

**Why this priority**: mismo síntoma y misma causa que P1, sobre volumen menor de pedidos, pero
el defecto del domicilio hace que **todo** pedido a domicilio con valor de envío falle al
cobrarse, tenga o no promoción.

**Independent Test**: crear una orden manual a domicilio con valor de envío y un producto con
promoción vigente, cobrarla, y verificar que el total mostrado y el cobrado coinciden.

**Acceptance Scenarios**:

1. **Given** una orden para llevar con promoción vigente, **When** el cajero abre el panel de
   cobro, **Then** el total mostrado incluye el descuento y el cobro se registra por ese
   importe.
2. **Given** una orden a domicilio con valor de envío $5.000 y sin promoción, **When** el cajero
   abre el panel de cobro, **Then** ve una fila `Domicilio $5.000` y el total la incluye.
3. **Given** una orden a domicilio con valor de envío $5.000 y una promoción que descuenta
   $8.000, **When** el cajero abre el panel de cobro, **Then** ve `Subtotal $16.000`,
   `Descuento −$8.000`, `Domicilio $5.000` y `Total $13.000`.
4. **Given** cualquiera de los anteriores, **When** el cobro se registra, **Then** el total de
   la venta emitida coincide al peso con el total que mostraba la pantalla.
5. **Given** un pedido con la misma promoción vigente creado, por separado, como pedido **de
   mesa**, como pedido **para llevar** y como pedido **a domicilio** (este último con valor de
   envío), **When** el cajero cobra cada uno por el panel de cobro de la Terminal con cualquier
   método de pago, **Then** los tres se cobran al primer intento y en los tres el total mostrado
   coincide al peso con el de la venta emitida — el arreglo no depende del tipo de orden.

---

### User Story 4 - El descuento prometido al tomar el pedido es el que se cobra (Priority: P2)

Una promoción vigente cuando se tomó el pedido debe seguir aplicándose al cobrarlo, aunque su
franja horaria haya terminado en el intervalo. El cliente pidió dentro de la promoción; el
precio que se le prometió es el que se le cobra.

**Why this priority**: cambia una regla de negocio (hoy el descuento se evalúa a la hora del
cobro), así que necesita decisión explícita del dueño — ya tomada. Sin esto, una mesa que pide a
las 7:59 pm y paga a las 8:05 pm recibe una cuenta más alta que la que le cantaron, y el cajero
no tiene forma de corregirla (no existe descuento manual, spec 029 Historia 2).

**Independent Test**: crear un pedido dentro de la franja de una promoción, esperar a que la
franja termine, cobrarlo, y verificar que el descuento se aplicó igual.

**Acceptance Scenarios**:

1. **Given** una promoción vigente solo hasta las 20:00 y un pedido creado a las 19:59,
   **When** el cajero lo cobra a las 20:05, **Then** el descuento se aplica y el total es el
   mismo que se mostró al tomar el pedido.
2. **Given** una promoción que **empieza** a las 20:00 y un pedido creado a las 19:59, **When**
   el cajero lo cobra a las 20:05, **Then** el descuento **no** se aplica (no estaba vigente
   cuando se tomó el pedido).
3. **Given** el pedido del escenario 1, **When** el cajero le agrega un tercer cono a las 20:05
   antes de cobrar, **Then** el descuento se recalcula sobre los tres conos usando la vigencia
   congelada del pedido (las 19:59), y el remanente que no completa un bloque queda a precio
   pleno.
4. **Given** una venta ya emitida antes de este cambio, **When** se consulta o se reimprime,
   **Then** su importe y su desglose no cambian.
5. **Given** una mesa con una primera ronda creada a las 19:59 (1 cono) y una segunda ronda creada
   a las 20:05 (1 cono), con una promoción del 50% llevando 2 vigente solo hasta las 20:00,
   **When** el cajero cobra la cuenta completa a las 20:10, **Then** los dos conos se evalúan
   juntos contra las 19:59 y el descuento se aplica.
6. **Given** dos mesas fusionadas, una con un pedido creado a las 19:59 (dentro de una promoción
   vigente hasta las 20:00) y otra con un pedido creado a las 20:05, **When** el cajero mira la
   cuenta consolidada y luego cobra cada pedido, **Then** el descuento del primer pedido se aplica
   (vigencia de las 19:59), el del segundo no, y el total consolidado que se mostró coincide al
   peso con lo que se cobra pedido por pedido (FR-018a).

---

### User Story 5 - Ver el descuento mientras se arma la orden manual (Priority: P3)

Mientras el cajero agrega productos en la pantalla de orden manual, el total que ve debe ser el
que va a cobrar, para poder decírselo al cliente antes de confirmar el pedido.

**Why this priority**: no bloquea ningún cobro (P1–P3 ya garantizan que el importe correcto se
cobra), pero evita que el cajero le cante al cliente un precio que después baja, que es la
segunda cara del mismo reporte.

**Independent Test**: armar una orden manual agregando unidades de un producto con promoción y
verificar que el total mostrado cambia al cruzar el umbral de la promoción.

**Acceptance Scenarios**:

1. **Given** la pantalla de armado con 1 cono a $8.000 y una promoción del 50% llevando 2,
   **When** el cajero mira el resumen, **Then** ve `Total $8.000` sin fila de descuento (un solo
   cono no completa el bloque).
2. **Given** ese estado, **When** el cajero agrega un segundo cono, **Then** el resumen pasa a
   `Subtotal $16.000`, `Descuento −$8.000`, `Total $8.000`.
3. **Given** ese estado, **When** el cajero agrega un tercer cono, **Then** el resumen muestra
   `Subtotal $24.000`, `Descuento −$8.000`, `Total $16.000` (el tercero queda a precio pleno).
4. **Given** que el descuento no se puede consultar (sin conexión o error del servidor),
   **When** el cajero mira el resumen, **Then** ve el subtotal sin descuento junto con un aviso
   de que el descuento se confirma al cobrar, y **puede** confirmar el pedido igual.
5. **Given** un borrador que muestra `Total $8.000` con una promoción vigente hasta las 20:00,
   **When** el cajero pulsa "Confirmar pedido" a las 20:01 y el total real pasa a $16.000,
   **Then** el sistema le muestra el total actualizado y le pide confirmar de nuevo antes de
   crear el pedido, en lugar de guardarlo en silencio con otro precio.

---

### User Story 6 - Ver la condición de la promoción en el catálogo de la Terminal (Priority: P3)

El catálogo de la Terminal muestra hoy una insignia suelta ("-50%") que no le dice al cajero qué
tiene que llevar el cliente para obtenerla. Debe mostrar la misma condición legible que ya ve el
comensal en el menú QR.

**Why this priority**: es información, no corrección de importes. Reduce la consulta del cajero
al cliente ("¿le cuento la promo?") pero ningún cobro depende de ella.

**Independent Test**: abrir el catálogo de la Terminal con una promoción vigente del 50%
llevando 2 sobre un producto de $8.000 y verificar que el texto de la condición aparece en la
tarjeta del producto.

**Acceptance Scenarios**:

1. **Given** una promoción vigente del 50% llevando 2 sobre un cono de $8.000, **When** el
   cajero abre el catálogo de la Terminal, **Then** la tarjeta del cono muestra la condición
   legible con su equivalente por unidad (por ejemplo `2 x -50% · ≈ $4.000 c/u`).
2. **Given** una regla de precio de paquete de 3 unidades por $20.000, **When** el cajero abre
   el catálogo, **Then** la tarjeta muestra la condición de esa regla, no un precio unitario
   descontado suelto.
3. **Given** un producto sin promoción vigente, **When** el cajero abre el catálogo, **Then** su
   tarjeta se ve exactamente igual que hoy.
4. **Given** una promoción vigente cuya regla exige 1 unidad (umbral mínimo), **When** el cajero
   abre el catálogo, **Then** la tarjeta puede mostrar el precio unitario con descuento, porque
   en ese caso sí corresponde a una unidad suelta.

---

### User Story 7 - Confirmar el pago de un pedido de comensal por QR con promoción (Priority: P1)

Un comensal pidió 2 conos ($8.000 c/u) por QR con una promoción vigente del 50% llevando 2 y
declaró pago en efectivo. El cajero abre "Pagos por confirmar", registra el efectivo recibido y
pulsa "Confirmar efectivo". Hoy el sistema rechaza cualquier monto menor a $16.000 aunque la
venta valga $8.000, así que el pedido no se puede confirmar ni llegar a cocina.

**Why this priority**: es la regresión de la evidencia del 2026-09-03 — el mismo defecto que
P1/P2, en la única superficie de cobro que la spec había dado por correcta. Bloquea el envío del
pedido a cocina: sin un intento de pago confirmado, el pedido del comensal se queda en `recibida`
indefinidamente (spec [024](../024-pagos-ordenes-mesa/spec.md)).

**Independent Test**: crear un pedido de comensal por QR con un producto cubierto por una
promoción vigente y pago en efectivo declarado, confirmarlo desde "Pagos por confirmar" con el
efectivo exacto con descuento, y verificar que la venta se emite al primer intento por ese
importe y el pedido pasa a cocina.

**Acceptance Scenarios**:

1. **Given** un pedido QR con 2 conos a $8.000 y una promoción vigente del 50% llevando 2,
   **When** el cajero abre "Pagos por confirmar", **Then** ve el desglose `Subtotal $16.000`,
   `Descuento −$8.000`, `Total $8.000`.
2. **Given** ese pedido, **When** el cajero registra $10.000 en efectivo y pulsa "Confirmar
   efectivo", **Then** la venta se emite por $8.000, el vuelto registrado es $2.000 y el pedido
   pasa a cocina — sin el mensaje "El monto recibido es menor al total de la orden".
3. **Given** ese pedido, **When** el cajero registra $8.000, **Then** el vuelto mostrado es $0 y
   el cobro se registra al primer intento.
4. **Given** ese pedido, **When** el cajero registra $5.000, **Then** el sistema indica que
   faltan $3.000 (calculado sobre $8.000, no sobre $16.000) y no permite confirmar.
5. **Given** un comprobante de transferencia por un pedido QR con promoción cuyo total real es
   $8.000, **When** el cajero pulsa "Aprobar", **Then** ve el total autoritativo $8.000 antes de
   aprobar y la venta se emite por $8.000, sin el error de "pago no efectivo supera el total".
6. **Given** un pedido QR creado a las 19:59 dentro de una promoción vigente hasta las 20:00,
   **When** el cajero confirma el pago a las 20:05, **Then** el descuento se aplica igual
   (instante congelado del pedido, FR-009) y el total confirmado es $8.000.
7. **Given** un pedido QR cuya promoción fue pausada por el administrador entre el pedido y el
   cobro, de modo que el total real sube de $8.000 a $16.000, **When** el cajero abre "Pagos por
   confirmar", **Then** el panel muestra el total actualizado $16.000, marca que cambió respecto
   al $8.000 que mostraba la tarjeta y exige que el cajero lo confirme antes de emitir la venta
   (FR-009a + regla de FR-007).

---

### Edge Cases

- **Remanente fuera del bloque**: 3 conos con una promoción de 2 → 2 conos descontados, 1 a
  precio pleno. El total mostrado y el cobrado deben coincidir en ese reparto.
- **La cuenta cambia mientras el cajero teclea**: llega un ítem nuevo o se anula uno mientras
  hay un importe escrito. El comportamiento actual (marcar la cuenta como obsoleta y dejar que
  el cajero decida recargarla con "Actualizar", `session-bill-panel.component.ts:248-260`) se
  mantiene; el importe se reinicia al recargar, nunca se cobra contra un total viejo.
- **La promoción se pausa, se borra o se agota entre tomar el pedido y cobrarlo**: la vigencia
  congelada cubre el vencimiento por franja/fecha (FR-009). Si la promoción deja de existir o
  pasa a un estado no vigente por acción del administrador, el descuento del pedido se recalcula
  con las reglas que existan en ese momento y el cajero ve el total actualizado antes de cobrar,
  nunca un error del servidor al pulsar "Cobrar" (FR-009a: el estado se lee vivo, no se congela).
- **Pedido creado antes de este cambio**: sin instante de vigencia congelado. Mantiene el
  comportamiento actual (evaluar con la hora del cobro), sin recálculo retroactivo.
- **Rondas sucesivas en una misma mesa**: manda el instante del pedido más antiguo pendiente de
  cobro (FR-012a) y las líneas se siguen agrupando como hoy, así que dos unidades pedidas en rondas
  distintas siguen completando un umbral de 2. Cualquier desviación de este criterio favorece al
  cliente: nunca se le cobra más de lo que se le prometió en la primera ronda.
- **Pedido a domicilio sin valor de envío**: la fila `Domicilio` no se muestra y el total no
  cambia.
- **Ítems anulados en cocina**: no entran ni en el subtotal, ni en el conteo de unidades para el
  umbral de la promoción, ni en el total a cobrar.
- **Descuento que iguala o supera el subtotal**: el total a cobrar nunca es negativo; el mínimo
  es $0 más el valor del domicilio si lo hay.
- **Redondeo**: el descuento se reparte y se redondea a pesos una sola vez, de forma que
  `subtotal − descuento + domicilio` mostrado coincide **al peso** con el total de la venta.
- **Sin conexión al consultar el total del cobro**: el panel no permite cobrar contra un total
  no verificado; muestra el estado de error y ofrece reintentar.
- **La tarjeta de "Pagos por confirmar" muestra un total desactualizado**: el total que hoy pinta
  la tarjeta del pedido QR viene del descuento calculado cuando el comensal armó su carrito, que
  puede haber quedado desfasado (promoción pausada, ítems cambiados). Al abrir el panel de
  revisión, ese número se reemplaza por el desglose autoritativo en vivo (FR-021/FR-022); si
  difiere, se marca y se exige confirmación del cajero antes de emitir (FR-024).
- **La promoción se pausa entre el pedido QR y la confirmación del cajero**: mismo criterio que
  el panel de cobro — el descuento se recalcula con las reglas vivas contra el instante congelado
  del pedido (FR-009a), el cajero ve el total actualizado y lo confirma antes de emitir, nunca un
  422 del servidor al pulsar "Confirmar efectivo" o "Aprobar".

## Requirements *(mandatory)*

### Functional Requirements

#### Importe a cobrar (Historias 1, 2 y 3)

- **FR-001**: El importe que la Terminal muestra, exige y valida al cobrar un pedido DEBE ser
  idéntico al importe que el sistema registra en la venta resultante, para todos los métodos de
  pago y todos los tipos de orden (mesa, para llevar, domicilio).
- **FR-002**: Ese importe DEBE incluir los descuentos por promoción que apliquen al pedido, y
  DEBE calcularlo la misma autoridad que los calcula al emitir la venta — el navegador no
  replica el motor de descuentos (se mantiene la decisión de la spec
  [063](../063-promociones-por-variante/spec.md), FR-023).
- **FR-003**: Ese importe DEBE incluir el valor del domicilio del pedido cuando el pedido es a
  domicilio, tomado del pedido mismo y no del borrador de la pantalla de armado.
- **FR-004**: El panel de cobro DEBE mostrar el desglose `Subtotal`, `Descuento`, `Domicilio` y
  `Total`, con el mismo formato agregado que ya presenta la cuenta de mesa. `Descuento` y
  `Domicilio` se muestran solo cuando su valor es mayor que cero.
- **FR-005**: Con un método en efectivo, el sistema DEBE indicar cuánto falta únicamente cuando
  el efectivo recibido es menor que el total real, y DEBE calcular el vuelto sobre el total
  real.
- **FR-006**: Con un método que no es efectivo, el importe precargado DEBE ser el total real del
  pedido, de modo que un pedido con promoción o con domicilio nunca quede imposible de cobrar.
- **FR-007**: Si el total real cambia entre que se muestra y que se intenta cobrar, el sistema
  DEBE presentar el total actualizado y pedir la confirmación del cajero, en lugar de fallar con
  un mensaje técnico del servidor.
- **FR-007a**: Mientras el sistema consulta el total autoritativo, el panel de cobro DEBE mostrar
  un estado visible de "calculando" y mantener deshabilitado el botón "Cobrar" hasta recibirlo.
  NUNCA DEBE presentar un total provisional que el cajero pueda confundir con el definitivo.

#### Revisión de pago del cajero para pedidos QR (Historia 7)

- **FR-021**: La revisión de pago del cajero para un pedido de comensal por QR —el panel "Pagos
  por confirmar", con las acciones "Confirmar efectivo" y "Aprobar comprobante"— DEBE tratarse
  como una superficie de cobro más: el importe que muestra, el que valida en el chequeo del
  "monto recibido" y el que registra en la venta DEBEN ser idénticos entre sí y calcularse con
  la misma autoridad que emite la venta (FR-001/FR-002), incluyendo el descuento por promoción,
  el valor de domicilio (FR-003) y el instante congelado del pedido (FR-008/FR-009). Deroga la
  fila "Comensal por QR — Correcto" del análisis del defecto.
- **FR-022**: Ese panel DEBE mostrar el desglose `Subtotal`, `Descuento`, `Domicilio`, `Total`
  con el mismo formato agregado de FR-004, calcular el vuelto en efectivo sobre el `Total` real
  (nunca sobre el subtotal sin descuento), y tomar ese `Total` de la cuenta autoritativa
  recalculada — no de un total guardado cuando el comensal armó su carrito, que puede haber
  quedado desactualizado.
- **FR-023**: El chequeo previo que hoy impide confirmar el efectivo cuando "el monto recibido
  es menor al total de la orden" DEBE comparar contra el `Total` real con descuento y domicilio.
  NUNCA DEBE rechazar un monto que sí cubre el `Total` real, ni permitir emitir una venta por un
  total distinto del que el panel mostró.
- **FR-024**: Si al abrir el panel o al confirmar el pago el `Total` real difiere del que la
  tarjeta del pedido venía mostrando (por lectura viva del estado de la promoción — FR-009a — o
  por cambio de ítems — FR-010), el panel DEBE presentar el total actualizado, marcar el cambio
  y exigir una confirmación explícita del cajero antes de emitir la venta (misma regla que
  FR-007). Mientras consulta el total autoritativo DEBE mostrar el estado "calculando" y
  mantener deshabilitadas las acciones de confirmación (misma regla que FR-007a).

#### Vigencia del descuento (Historia 4)

- **FR-008**: Cada pedido DEBE conservar el instante que define qué promociones se le aplican:
  el momento en que el pedido fue creado.
- **FR-009**: El descuento por promoción de un pedido DEBE evaluarse contra la vigencia
  **temporal** de ese instante congelado — rango de fechas, día de la semana y franja horaria —,
  no contra la del momento del cobro. **Deroga el comportamiento actual** y requiere entrada
  `A-70` en el registro de anomalías.
- **FR-009a**: El **estado** de la promoción (activa, pausada, eliminada) NO se congela: se lee
  vivo en cada cálculo. Si el administrador pausa, desactiva o borra una promoción, los pedidos
  aún no cobrados pierden ese descuento y el cajero ve el total actualizado antes de cobrar. El
  sistema NO DEBE guardar historial de estados de promociones para responder esta pregunta.
- **FR-010**: Si los ítems del pedido cambian después de creado (se agrega o se anula un ítem),
  el descuento DEBE recalcularse sobre los ítems vigentes manteniendo el instante congelado —
  se congela el instante de evaluación, no el monto. Lo exige el motor por conjunto: el
  descuento depende de cuántas unidades del conjunto tenga el pedido completo.
- **FR-011**: Las ventas y facturas ya emitidas NO DEBEN recalcularse ni cambiar de importe ni
  de representación (Principio VII de la Constitución).
- **FR-011a**: La venta emitida DEBE registrar el instante congelado con el que se evaluaron sus
  promociones, junto a las promociones aplicadas que ya guarda hoy (spec
  [063](../063-promociones-por-variante/spec.md), FR-021), de modo que su detalle explique por sí
  solo un descuento de una promoción que hoy aparece vencida, sin tener que cruzarlo con el
  pedido. Este registro NO es retroactivo: las ventas emitidas antes de este cambio no lo tienen
  y no se migran (FR-011).
- **FR-012**: Los pedidos creados antes de este cambio, que no tienen instante congelado, DEBEN
  mantener el comportamiento actual (evaluar la vigencia con la hora del cobro), sin migración
  retroactiva.
- **FR-012a**: Cuando la cuenta que se cobra reúne varios pedidos de una misma mesa (rondas
  sucesivas), la vigencia DEBE evaluarse contra un único instante: el del **pedido más antiguo
  pendiente de cobro** de esa cuenta. La agrupación de líneas para alcanzar el umbral de cantidad
  NO cambia respecto al comportamiento actual — las líneas del mismo comensal se siguen evaluando
  juntas sobre todas las rondas —, de modo que ninguna promoción por umbral deje de alcanzarse por
  haberse pedido en dos rondas distintas. Si la cuenta mezcla pedidos con instante congelado y
  pedidos anteriores a este cambio (FR-012), manda igualmente el instante del pedido más antiguo
  que sí lo tenga.

#### Armado de orden manual (Historia 5)

- **FR-013**: Mientras el cajero arma una orden manual, el total mostrado DEBE reflejar el
  descuento por promoción que aplicaría al conjunto de productos del borrador, y DEBE
  actualizarse al agregar, quitar o cambiar la cantidad de una línea.
- **FR-014**: Esa pantalla DEBE usar el mismo desglose de FR-004.
- **FR-015**: Si el descuento del borrador no se puede consultar, la pantalla DEBE mostrar el
  subtotal sin descuento con un aviso de que el descuento se confirma al cobrar, y NO DEBE
  impedir confirmar el pedido.
- **FR-015a**: Si al confirmar la orden manual el total real difiere del que la pantalla venía
  mostrando, el sistema DEBE presentar el total actualizado y pedir una segunda confirmación
  antes de crear el pedido — la misma regla que FR-007 aplica en el panel de cobro. El pedido
  creado conserva como instante congelado el momento de su creación (FR-008), no el del borrador.

#### Catálogo de la Terminal (Historia 6)

- **FR-016**: Cada producto del catálogo de la Terminal cubierto por una promoción vigente DEBE
  mostrar la condición legible y el equivalente por unidad que el sistema ya publica para el
  menú del comensal (spec [066](../066-promociones-legibles-menu/spec.md), FR-007/FR-008), en
  lugar de la insignia suelta actual.
- **FR-017**: El catálogo NO DEBE presentar un precio unitario con descuento como si fuera el
  precio de una unidad suelta cuando la regla exige más de una unidad; en ese caso solo se
  muestra la condición y el equivalente por unidad, señalado como equivalente.

#### Alcance protegido

- **FR-018**: El cobro de la cuenta de mesa (pedido de mesero ya enviado a cocina), la cuenta
  consolidada de mesas fusionadas (RF-053, spec
  [019](../019-correccion-cuenta-mesas-fusionadas/spec.md)) y el flujo del comensal por QR en su
  menú y su carrito DEBEN conservar su comportamiento actual salvo por el cambio de vigencia de
  FR-009, que les aplica igual. La **revisión de pago del cajero** para pedidos QR ("Pagos por
  confirmar") NO queda protegida: la corrigen FR-021 a FR-024.
- **FR-018a**: En la cuenta consolidada de mesas fusionadas, cada pedido del grupo evalúa la
  vigencia de sus promociones contra **su propio** instante congelado (o la hora del cobro si es
  anterior a este cambio, FR-012) — NO un instante único para todo el grupo. Los pedidos de mesas
  fusionadas se cobran individualmente, así que el desglose consolidado que se muestra DEBE
  coincidir con lo que se cobra pedido por pedido (FR-001). Se diferencia de FR-012a: las rondas
  de una misma mesa se cobran juntas y por eso comparten un instante; los pedidos de mesas
  fusionadas no.
- **FR-019**: No se introduce descuento manual: el cajero sigue sin poder alterar el importe
  (spec [029](../029-correccion-cobro-cierre-mesa/spec.md), Historia 2). Los impuestos siguen
  en $0 (spec [036](../036-terminal-mesas-rediseno-layout/spec.md), FR-011).
- **FR-020**: Se conserva un único método de pago por cobro (spec
  [046](../046-dividir-cuenta-pago-pendiente/spec.md), FR-004/FR-007) y la regla de que un pago que no es
  en efectivo no puede superar el total.

### Key Entities

- **Pedido**: la comanda que se cobra. Gana un atributo nuevo: el instante que fija su vigencia
  de promociones (FR-008). Ya conserva el descuento agregado y la lista de promociones aplicadas
  (spec 063, FR-021).
- **Ítem de pedido**: línea del pedido, con su precio unitario pleno. Su descuento no es una
  propiedad propia: depende del conjunto completo del pedido y del umbral de la regla.
- **Regla de promoción**: el conjunto de variantes, el tipo (porcentaje o precio de paquete), el
  valor y la cantidad mínima de unidades que activa el descuento. La vigencia (estado, fechas,
  días, franja) es de la promoción dueña, compartida por todas sus reglas. De esa vigencia solo se
  congela la parte **temporal** —fechas, días y franja— (FR-009); el estado se lee vivo en cada
  cálculo (FR-009a).
- **Cuenta del cobro**: el desglose autoritativo de un pedido — subtotal, descuento, domicilio y
  total —, calculado por la misma lógica que emitirá la venta. Es lo que muestran y validan tanto
  el panel de cobro de la Terminal como la revisión de pago del cajero para pedidos QR ("Pagos
  por confirmar").
- **Venta**: el registro emitido. Inmutable una vez emitida (FR-011). Gana un atributo nuevo: el
  instante congelado con el que se evaluaron sus promociones (FR-011a), que se suma al descuento y
  a la lista de promociones aplicadas que ya guardaba.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los pedidos con promoción vigente se cobran al primer intento, sin
  mensajes de importe insuficiente y sin errores del servidor, en los tres tipos de orden y con
  cualquier método de pago.
- **SC-002**: El total que muestra el panel de cobro coincide al peso con el total de la venta
  emitida en el 100% de los cobros, medido sobre todas las combinaciones de promoción,
  domicilio y tipo de orden.
- **SC-002a**: El total que muestra la revisión de pago del cajero para pedidos QR ("Pagos por
  confirmar") y el que valida su chequeo de "monto recibido" coinciden al peso con el total de la
  venta emitida en el 100% de los casos; ningún pedido QR con promoción vigente queda bloqueado
  por el mensaje "el monto recibido es menor al total de la orden".
- **SC-003**: Ningún pedido a domicilio queda bloqueado por diferencia entre el total mostrado y
  el cobrado.
- **SC-004**: Ninguna venta registra un vuelto derivado de un total inflado — es decir, ningún
  cliente paga de más ni el arqueo del turno arrastra un sobrante que nadie pueda explicar.
- **SC-005**: El total que el cajero **confirma** al crear una orden manual coincide con el que se
  le cobra al cliente en el 100% de los casos, incluso si la franja de la promoción termina entre
  ambos momentos. Si el total cambia entre el resumen del borrador y la confirmación, el cajero lo
  ve y lo acepta **antes** de que el pedido exista, nunca después.
- **SC-006**: Un cajero que abre el catálogo de la Terminal puede decir qué hay que llevar para
  obtener una promoción vigente sin consultar el módulo de promociones.
- **SC-007**: Cero cambios en el importe o el desglose de cualquier venta o factura emitida
  antes de esta corrección.
- **SC-008**: El desglose con descuento aparece en el panel de cobro en **1 segundo o menos en el
  95%** de las aperturas. En el 5% restante el cajero ve el estado de "calculando" —nunca un total
  provisional ni una pantalla congelada— y el cobro sigue siendo posible en cuanto llega el total.
- **SC-009**: El 100% de las ventas emitidas con descuento por promoción permiten explicar, desde
  su propio detalle y sin cruzar otras pantallas, contra qué instante se evaluó la promoción — de
  modo que un descuento de una promoción hoy vencida se distinga de una falla del sistema.

## Assumptions

- **El backend sigue siendo la única autoridad del monto.** Esta spec no reintroduce el cálculo
  local de descuentos en el navegador que la spec 063 retiró deliberadamente (FR-023,
  `research.md` D10); lo que corrige es que la Terminal **no estaba consultando** ese monto para
  los pedidos que no vienen del carrito QR.
- **"Congelar la vigencia" significa congelar el instante de evaluación, no el monto.** Es la
  única lectura coherente con un motor por conjunto con umbral de cantidad: si el pedido cambia
  de ítems, el bloque de la promoción cambia y el monto debe recalcularse (FR-010). Congelar el
  monto dejaría descuentos imposibles al anular un ítem.
- **El desglose es agregado, no prorrateado por línea** (decisión del dueño, 2026-09-02). No se
  pide al sistema repartir el descuento entre las líneas para pintarlas tachadas, aunque el
  motor internamente sí lo reparta para construir la venta.
- **El texto legible de la promoción ya existe y se reutiliza tal cual.** La spec 066 ya define y
  publica la condición y el equivalente por unidad para el menú del comensal, y el catálogo de
  la Terminal consume el mismo origen de datos; FR-016 no define un formato nuevo.
- **No se toca el modelo de promociones.** Conjuntos de variantes, tipos, umbrales, solapes y
  administración quedan como los dejaron las specs 063, 066 y 071.
- El cajero opera con un turno de caja abierto; la detección del turno es asunto de la spec
  [072](../072-fix-deteccion-turno-caja/spec.md) y no cambia aquí.
- Los importes se manejan en pesos colombianos enteros, sin decimales, como en todo el sistema.

## Dependencias

- Spec [063](../063-promociones-por-variante/spec.md) — motor de descuentos por conjunto de
  variantes; es la autoridad del monto que esta spec conecta a la Terminal.
- Spec [066](../066-promociones-legibles-menu/spec.md) — condición legible y equivalente por
  unidad que FR-016 reutiliza.
- Spec [056](../056-domicilio-orden-manual/spec.md) — valor del domicilio, cuyo defecto de
  arrastre al panel de cobro se corrige aquí.
- Spec [058](../058-panel-cobro-pedido-manual/spec.md) — panel de cobro del pedido manual, la
  superficie principal que se corrige.
- Spec [046](../046-dividir-cuenta-pago-pendiente/spec.md) — método único por cobro y regla del pago no
  efectivo, que se conservan (FR-020).
- Spec [024](../024-pagos-ordenes-mesa/spec.md) y spec
  [025](../025-revision-pago-antes-envio/spec.md) — intento de pago del comensal por QR y su
  revisión por el cajero ("Confirmar efectivo" / "Aprobar comprobante"); superficie que
  FR-021–FR-024 corrigen.
- Spec [028](../028-terminal-mesas-modo-hibrido/spec.md) — la revisión de pago del cajero genera
  la venta/factura en la misma llamada; el chequeo del "monto recibido" y esa emisión deben usar
  el mismo total (FR-021/FR-023).
