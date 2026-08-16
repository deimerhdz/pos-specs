# Feature Specification: Menú público y carrito del comensal (flujo QR)

**Feature Branch**: `007-menu-carrito-qr`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/cart/service.py`, `app/api/v1/menu/router.py` y `app/core/qr_context.py`
(más `app/core/qr_token.py`, invocado desde ahí), para que sirva de contrato formal de cara a la
modernización. Cubre todo lo que ocurre entre que un comensal escanea el QR de su mesa y envía su
pedido a cocina: qué ve en el menú público (precio con descuento, disponibilidad), cómo arma y
edita su carrito, y el ciclo de vida completo de su sesión (TTL deslizante, expiración del token,
unión a sesión existente). Incluye dos reglas protegidas (A-24, la integridad del flujo QR; y por
extensión A-21 y A-47, confirmadas como intencionales en entrevistas de negocio) y tres anomalías
que se documentan **sin especificar como corrección obligatoria** porque su tratamiento acordado es
"documentar sin especificar" (A-08, A-28, A-36), citando su clasificación y evidencia de
`registro-de-anomalias.md`.

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE del
menú público y el carrito del comensal en el flujo de pedido por QR del sistema POS Heladería,
tomado de `reglas-de-negocio.md` (RN-MENU-01 a RN-MENU-09, RN-CART-01 a RN-CART-27) y de
`registro-de-anomalias.md` (A-08, A-21, A-24, A-28, A-36, A-47), para que sirva de contrato en la
modernización. No es una feature nueva."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `cart/service.py`, `menu/router.py` y
  `core/qr_context.py`/`qr_token.py`, no uno deseado. Las anomalías conocidas se marcan inline con
  su tratamiento acordado (registro-de-anomalias.md). A-24 es la regla más importante de esta spec
  (seguridad, PROTEGIDA); A-21 y A-47 son las dos reglas cuya clasificación cambió de PENDIENTE a
  INTENCIONAL confirmado durante las rondas de entrevista de negocio citadas en
  `propuesta-particion-specs.md`.
-->

### User Story 1 - El precio del menú ya incluye el descuento vigente (Priority: P1)

Un comensal escanea el QR de su mesa y navega el menú antes de pedir nada. Para cada presentación
con una promoción porcentual o fija vigente, el precio que ve **ya tiene el descuento aplicado**,
calculado como si pidiera una unidad — porque todavía no existe ningún carrito con una cantidad
real.

**Why this priority**: es lo primero que ve cualquier comensal, y determina qué pide. Un precio de
menú que no reflejara el descuento (o lo reflejara mal) generaría reclamos en el momento de cobrar,
cuando el comensal descubre que pagó distinto a lo que vio.

**Independent Test**: se puede probar consultando `GET /menu/qr-token/{token}` con una variante que
tenga una promoción `percent`/`fixed` activa y sin `min_qty`, y comparando el `price` contra el
`discounted_price` devuelto.

**Acceptance Scenarios**:

1. **Given** «Copa grande» a $15.000 con una promoción del 15% activa (sin `min_qty`), **When** el
   comensal consulta el menú, **Then** el sistema calcula el descuento sobre cantidad 1
   ($15.000 × 15% = $2.250) y devuelve `discounted_price = 12.750,00`, redondeado con "mitad hacia
   arriba" a 2 decimales (`RN-MENU-01`).
2. **Given** «Paleta de agua» a $3.000 con una promoción "lleva 3 paga 2" (`min_qty=3`) activa,
   **When** el comensal consulta el menú, **Then** el precio mostrado es **$3.000 sin descuento** —
   la promoción existe y está vigente, pero no se refleja hasta que el comensal alcance la cantidad
   mínima en su carrito (`RN-MENU-02`).
3. **Given** ninguna promoción activa para una variante, **When** el comensal consulta el menú,
   **Then** `discounted_price` es `null` y solo se muestra `price` (comportamiento base,
   `RN-MENU-01`).

**[Anomalía A-08, `ACCIDENTAL`, documentada sin especificar como corrección obligatoria]**: la
vigencia de la promoción se evalúa con `datetime.now(timezone.utc).replace(tzinfo=None)`
(`menu/router.py:82-83`) — un valor naive que en realidad está en UTC, pero que
`promotions.local_now()` interpreta como si ya estuviera en hora local del tenant. Con
`TENANT_TIMEZONE=America/Bogota` (UTC-5), esto reproduce — únicamente en este punto y en el
carrito (User Story 7) — el mismo bug de zona horaria que la spec 012 (motor de promociones, A-07)
ya corrigió en el resto del sistema. El monto que se cobra al pagar **no se ve afectado** (los
caminos de cobro real usan `datetime` aware); solo la vista previa del menú/carrito puede mostrar
una promoción vigente/no vigente incorrectamente cerca de un límite horario. Tratamiento acordado:
corregir en fase de modernización aplicando el mismo patrón ya usado en
`checkout.py`/`table_sessions/service.py`/`sales/service.py` (pasar `datetime` aware o convertir
explícitamente con `local_now()`), sin riesgo de retroactividad.

---

### User Story 2 - La disponibilidad de una opción se evalúa contra el peor caso de consumo (Priority: P1)

Un comensal navega el menú y ve, para cada opción de un grupo (sabor, topping), si está disponible.
El sistema no calcula esa disponibilidad por combinación variante×opción: compara el stock del
insumo contra la **mayor** cantidad que cualquier presentación que use esa opción necesitaría, sin
importar si el comensal terminará pidiendo la presentación pequeña o la grande.

**Why this priority**: es la regla de disponibilidad central del menú — determina qué ve el
comensal como "agotado" antes de intentar pedir, evitando que choque con un rechazo al armar el
carrito (ver User Story 6).

**Independent Test**: se puede probar consultando el menú con una opción ligada a un insumo cuyo
stock alcanza para la presentación pequeña que la usa pero no para la grande, y verificando que se
marca no disponible.

**Acceptance Scenarios**:

1. **Given** "Fresa" usada en «Ensalada pequeña» (60 g/unidad) y «Ensalada grande» (180 g/unidad),
   con stock de insumo = 100 g, **When** el comensal consulta el menú, **Then** "Fresa" se marca
   **no disponible** globalmente — `100 ≥ 180` es falso, aunque técnicamente alcanzaría para una
   ensalada pequeña (`100 ≥ 60`) (`RN-MENU-03`, falso negativo intencional, "el error seguro").
2. **Given** una opción ligada a un insumo con `item_quantity=0` en todas las presentaciones que la
   usan (no descuenta nada en ninguna), **When** se consulta el menú, **Then** esa opción **nunca**
   aparece agotada por stock — pero si el insumo se desactiva, la opción deja de listarse por
   completo (`RN-MENU-04`, `RN-MENU-05`).
3. **Given** una opción cuyo insumo está desactivado (`InventoryItem.active=False`), **When** se
   consulta el menú, **Then** la opción se muestra **no disponible**, sin importar cuánto stock
   numérico tenga (`RN-MENU-05`).
4. **Given** «Malteada» con dos presentaciones — "Chica" (grupo obligatorio «Sabor» sin ninguna
   opción disponible) y "Grande" (con al menos una opción disponible) —, **When** el comensal
   consulta el menú, **Then** "Chica" se marca no pedible mientras que "Grande" y el producto en su
   conjunto siguen disponibles (`RN-MENU-06`).

---

### User Story 3 - Solo se lista catálogo activo, vía QR firmado con límite de tasa (Priority: P1)

Un comensal escanea el QR físico de su mesa. El sistema resuelve el tenant y la mesa a partir del
token firmado (nunca de un parámetro que el cliente controle — ver User Story 5), aplica límites de
tasa por IP y por mesa, y devuelve solo el catálogo vigente: categorías activas, productos activos y
disponibles, variantes activas.

**Why this priority**: es la puerta de entrada de todo el flujo QR — sin esto no hay menú que
mostrar, y sin los límites de tasa el endpoint público queda expuesto a abuso sin costo.

**Independent Test**: se puede probar consultando `GET /menu/qr-token/{token}` con un producto
desactivado, una variante inactiva, y una mesa desactivada, verificando en cada caso qué se filtra o
qué error se devuelve.

**Acceptance Scenarios**:

1. **Given** una categoría inactiva, un producto inactivo o no disponible, o una variante inactiva,
   **When** el comensal consulta el menú, **Then** ninguno de esos elementos aparece; un producto
   sin ninguna variante activa restante desaparece del todo (`RN-MENU-07`).
2. **Given** una mesa desactivada cuyo QR sigue impreso, **When** un comensal con ese QR consulta el
   menú, **Then** el sistema responde `404 "Mesa no encontrada o inactiva"` (`RN-MENU-09`).
3. **Given** una ráfaga de peticiones al endpoint de menú por token QR, **When** se excede el
   límite de tasa, **Then** el límite **por IP** se aplica primero, antes de verificar la firma del
   token; solo si el token es válido se aplica un segundo límite **por mesa** — así una ráfaga de
   tokens basura se bloquea sin nunca llegar a decodificarse (`RN-MENU-08`).

---

### User Story 4 - Escanear el QR de una mesa ocupada une al comensal a la sesión activa (Priority: P1)

Un segundo comensal (o un tercero, cuarto...) escanea el mismo QR físico de una mesa que ya tiene
una sesión en curso. El sistema no abre una sesión nueva: lo agrega como un comensal más de la
sesión activa, con su nombre desambiguado automáticamente si coincide con uno ya presente.

**Why this priority**: es el mecanismo que hace posible que varias personas en la misma mesa pidan
de forma independiente sin fragmentar la cuenta en sesiones separadas — el corazón operativo del
flujo QR de grupo.

**Independent Test**: se puede probar abriendo sesión dos veces sobre la misma mesa con nombres
iguales y distintos, y verificando que comparten `table_session_id` pero tienen `participant_id`
distintos — cubierto directamente por `test_table_sessions.py`.

**Acceptance Scenarios**:

1. **Given** mesa 5 con una sesión activa de "Ana", **When** "Luis" escanea el mismo QR, **Then**
   se agrega como comensal de esa misma sesión — no se crea una segunda `table_session` para la
   mesa 5 (`RN-CART-01`; el índice parcial `idx_active_session_per_table` garantiza esta unicidad
   incluso ante escaneos simultáneos; verificado en `test_table_sessions.py`, pasos 1-2).
2. **Given** una mesa con "Ana" y "Ana (2)" ya presentes, **When** un tercer comensal escribe
   "Ana", **Then** se le asigna la etiqueta "Ana (3)" — el `display_name` que escribió se conserva
   tal cual, solo el `display_label` visible para cocina/staff se desambigua (`RN-CART-02`;
   verificado en `test_table_sessions.py`, paso 2).
3. **Given** una mesa `libre`, **When** el primer comensal escanea su QR, **Then** la mesa pasa a
   `ocupada` y se publica el evento `table_status_changed` **una sola vez**; comensales
   subsiguientes que se unen a la misma sesión no repiten el evento (`RN-CART-03`).
4. **Given** un comensal recién unido a una sesión, **When** se inspecciona el token de sesión que
   recibió, **Then** el JWT codifica únicamente `tenant_id`, `table_id`, `participant_id` y
   `table_session_id` — nunca el `display_name`; el frontend debe llamar `GET /cart` para recuperar
   el saludo si recarga la página (`RN-CART-04`).

---

### User Story 5 - [Regla protegida A-24] Tenant y mesa siempre vienen del token firmado (Priority: P1)

Cualquier operación del flujo QR (menú, carrito, sesión) resuelve `tenant_id`, `table_id`,
`participant_id` y `table_session_id` exclusivamente a partir de los claims verificados de un JWT
firmado por el servidor. Ningún endpoint público del flujo QR acepta esos identificadores como
parámetro de entrada que el cliente pueda manipular.

**Why this priority**: es el invariante de seguridad central de todo el flujo — sin él, un
comensal podría ver o pedir en una mesa o tenant ajeno con solo cambiar un id en la URL o el body.
Está protegido por doble testimonio (código + historia real de dos endpoints legacy retirados a
propósito).

**Independent Test**: se puede probar auditando la firma de cada endpoint de `cart/router.py` y
`menu/router.py` — ninguno declara un parámetro `tenant_id`, `table_id` ni `table_session_id` en
query, path o body; todos los obtienen de `open_qr_context`/`open_session_context`. El contrato de
tokens en sí está cubierto por `test_qr_token.py` (roundtrip, detección de manipulación,
aislamiento entre tipos de token).

**Acceptance Scenarios**:

1. **Given** el flujo QR completo (menú, apertura de sesión, carrito), **When** se audita cada
   endpoint público, **Then** ninguno recibe `tenant_id`, `table_id`, `participant_id` ni
   `table_session_id` como dato enviado por el cliente — todos se derivan del token firmado
   (`RN-CART-26`).
2. **Given** un token de QR de mesa (`typ="qr"`, sin `exp`), **When** se presenta donde el sistema
   espera un token de sesión de comensal, **Then** se rechaza explícitamente (`SessionInvalidError`
   → 401), y viceversa un token de sesión no sirve como token de QR (`RN-CART-25`; verificado en
   `test_qr_token.py::test_type_isolation`).
3. **Given** un token de personal (con claims `user`/`refresh`), **When** se presenta en cualquier
   endpoint del flujo QR, **Then** se rechaza — los dominios de token de personal y de mesa son
   mutuamente excluyentes aunque compartan el mismo secreto de firma (`RN-CART-25`; verificado en
   `test_qr_token.py::test_type_isolation`).
4. **Given** un token de QR o de sesión manipulado (firma alterada, o firmado con otro secreto),
   **When** se intenta usar, **Then** se rechaza con `QrTokenError`/401, sin exponer ningún dato del
   tenant o la mesa que codifica (`RN-CART-24`, `RN-CART-25`; verificado en
   `test_qr_token.py::test_tamper_detection`).
5. **Given** la apertura de una sesión vía QR, **When** llega una ráfaga de peticiones, **Then** el
   límite de tasa se aplica primero **por IP** (antes de verificar el token) y luego **por mesa**
   (una vez el token es válido) — mismo orden que en el menú (`RN-CART-27`).

**Regla protegida A-24 — `INTENCIONAL [PROTEGIDA]`**: esta spec la especifica **tal cual, sin
tocar el diseño**. Se eliminaron a propósito dos endpoints legacy inseguros — `POST /sessions`
(autenticaba con un UUID plano + header falsificable) y `GET /menu/qr/{qr_token}` (exponía un
`table_id` editable por el cliente) — reemplazados por los endpoints actuales, donde tenant y mesa
viajan firmados dentro del token (`memoria-historica.md` entrada #2, 2026-07-28, commit
`5c1db9ed`, Deimer Hernandez). Cualquier reintroducción de un endpoint que acepte esos ids como
parámetro reproduciría la vulnerabilidad ya corregida.

---

### User Story 6 - El carrito del comensal: una línea por vez, un carrito abierto (Priority: P1)

Un comensal agrega, edita y quita líneas de su carrito (variante + opciones + cantidad) antes de
enviarlo como pedido. En todo momento tiene como máximo un carrito `abierto`; cada línea exige
cantidad mínima 1; los combos se agregan a precio de catálogo normal, sin el ahorro del combo
aplicado todavía.

**Why this priority**: es la operación más frecuente de todo el flujo — el comensal pasa la mayor
parte de su interacción agregando y ajustando líneas antes de decidirse a enviar.

**Independent Test**: se puede probar agregando una línea, editándola sin enviar `option_ids`
(conserva las opciones guardadas sin revalidar), y enviando el carrito para verificar que el
siguiente `add_item` abre un carrito nuevo.

**Acceptance Scenarios**:

1. **Given** un comensal que envía su pedido, **When** intenta agregar un ítem nuevo justo después,
   **Then** el sistema le crea un carrito `abierto` nuevo — nunca reutiliza uno `confirmado`; el
   índice parcial `idx_open_cart_per_participant` garantiza uno solo abierto a la vez (`RN-CART-05`;
   verificado en `test_table_sessions.py`, "cada comensal tiene su propio carrito borrador").
2. **Given** cualquier línea de carrito, **When** se intenta fijar `quantity=0`, **Then** el sistema
   lo rechaza — la cantidad mínima declarada es 1, sin "cantidad 0" como forma de marcar una línea
   para quitar (`RN-CART-07`).
3. **Given** una línea guardada con 3 toppings elegidos cuando el catálogo permitía hasta 3, **When**
   el catálogo cambia después a un máximo de 2 y el comensal edita **solo la cantidad** de esa línea
   (sin enviar `option_ids`), **Then** la edición se acepta conservando los 3 toppings originales
   sin revalidarlos contra el nuevo máximo (`RN-CART-10`).
4. **Given** un combo con dos componentes, **When** el comensal lo selecciona en el carrito,
   **Then** cada componente se guarda como línea normal a precio de catálogo, marcada con el mismo
   `combo_id`, sin descuento aplicado — el ahorro del combo se calcula recién al construir la venta,
   fuera de alcance de esta spec (`RN-CART-11`).
5. **Given** líneas marcadas con `combo_id`, **When** se calcula la vista previa de descuentos del
   carrito, **Then** esas líneas **no** reciben además un descuento por promoción
   porcentual/fija — solo las líneas sin `combo_id` se evalúan contra esas promociones, para no
   descontar dos veces (`RN-CART-12`).
6. **Given** un comensal con carrito abierto, **When** consulta su carrito, **Then** el descuento
   por promoción de cada línea se redondea a 2 decimales con "mitad hacia arriba", igual que en el
   menú (`RN-CART-23`).

---

### User Story 7 - [Anomalía A-47 confirmada intencional] Enviar el carrito no descuenta inventario; el chequeo es best-effort (Priority: P1)

Un comensal envía su carrito como pedido. El sistema crea el pedido en estado `recibida` con sus
líneas, sin tocar el stock — el único punto que descuenta inventario de verdad es la confirmación
de cocina (fuera de alcance de esta spec, specs 008/009). El chequeo de disponibilidad que corre al
enviar es preventivo: no reserva ni bloquea nada, así que en hora pico dos comensales pueden ver el
mismo ítem disponible y solo uno lograr confirmarlo.

**Why this priority**: es el punto de transición del carrito hacia cocina, y la regla que separa
"lo que el comensal ve/decide" de "lo que realmente compromete inventario" — un límite de diseño
central para entender por qué un pedido enviado puede fallar después al confirmarse.

**Independent Test**: se puede probar enviando un carrito vacío (rechazo), y luego uno con líneas,
verificando que no se genera ningún movimiento de kardex al enviarlo.

**Acceptance Scenarios**:

1. **Given** un carrito sin líneas, **When** el comensal intenta enviarlo, **Then** el sistema
   rechaza con `409 "El carrito está vacío"` (`RN-CART-06`).
2. **Given** un carrito con líneas válidas, **When** el comensal lo envía, **Then** se crea una
   `CustomerOrder` en estado `recibida` con sus ítems en `estado_cocina="pendiente"`, **sin generar
   ningún movimiento de inventario** — el chequeo de disponibilidad que corrió antes de crear el
   pedido es preventivo, sin lock ni reserva (`RN-CART-09`).
3. **Given** un comensal que ya envió un pedido en la sesión, **When** decide pedir una segunda
   ronda, **Then** el sistema lo permite sin límite de "rondas" mientras la sesión siga activa; cada
   envío genera una `CustomerOrder` independiente (`RN-CART-08`).
4. **Given** 3 conos de un sabor disponibles, **When** dos comensales de mesas distintas agregan 2
   unidades cada uno de forma casi simultánea, **Then** ambos pasan el chequeo preventivo al enviar
   (nadie reservó nada todavía) — el conflicto real solo se resuelve al confirmar cada pedido en
   cocina, momento en el que uno de los dos puede fallar por falta de stock (`RN-CART-09`, `A-47`).

**Anomalía A-47 — clasificación `INTENCIONAL` (confirmada en entrevista P27-bis, 2/3 testigos:
CÓDIGO + NEGOCIO)**: el chequeo de disponibilidad del carrito es deliberadamente best-effort, sin
reserva ni bloqueo de stock (la regla propia, RN-CAT-26, vive en la spec 003; esta es su
consecuencia visible para el comensal). Preguntado explícitamente si preferiría invertir en
reservar stock preventivamente (más complejo) o mantener el diseño actual (más simple, con el costo
ocasional de un pedido rechazado tarde), el dueño/gerente respondió **"dejarlo como está"**. Esta
spec especifica el comportamiento best-effort tal cual, sin fijar como objetivo de modernización
introducir reserva de stock.

---

### User Story 8 - Cancelación propia del comensal: solo antes de que cocina empiece (Priority: P1)

Un comensal quiere cancelar un pedido que él mismo envió. El sistema solo lo permite si el pedido
sigue `recibida` (nunca se descontó nada) o si está `abierta` con **todos** sus ítems todavía
`pendiente` en cocina. En cuanto un solo ítem avanzó de estado, la cancelación por el comensal se
bloquea por completo — ni siquiera puede cancelar solo la parte que sigue pendiente.

**Why this priority**: es la única acción de "arrepentimiento" disponible para el comensal sin
intervención del personal, y su límite (antes de que cocina empiece) es lo que evita que el
inventario ya comprometido en preparación se pierda sin que el staff decida sobre ello.

**Independent Test**: se puede probar con un pedido `abierta` que tenga un ítem `pendiente` y otro
`en_preparacion`, verificando que la cancelación se rechaza aunque uno de los dos ítems siga
pendiente.

**Acceptance Scenarios**:

1. **Given** un pedido en estado `recibida`, **When** el comensal lo cancela, **Then** el sistema
   lo permite siempre — no se había descontado ningún inventario todavía (`RN-CART-13`).
2. **Given** un pedido `abierta` con helado `pendiente` y malteada `en_preparacion`, **When** el
   comensal intenta cancelarlo, **Then** el sistema rechaza con `409` — incluso el ítem que sigue
   `pendiente` no se cancela solo; a partir de que un ítem avanzó, la decisión pasa a ser del staff
   (`RN-CART-13`).
3. **Given** un pedido `abierta` con **todos** sus ítems todavía `pendiente`, **When** el comensal
   lo cancela, **Then** el sistema lo permite (`RN-CART-13`).
4. **Given** un comensal que cancela el último pedido pendiente de cobrar en su sesión y ya no
   queda nadie más activo en la mesa, **When** se completa la cancelación, **Then** la mesa se
   libera de inmediato, sin esperar al barrido periódico de sesiones abandonadas — cubre el caso
   "pedí, me arrepentí y me fui" (`RN-CART-14`).

---

### User Story 9 - Salir de la mesa nunca falla; liberarla exige dos condiciones (Priority: P2)

Un comensal cierra la pestaña o pulsa "salir" explícitamente. El sistema cierra su carrito y lo
marca `closed`, sin importar si su token ya había expirado — la operación es idempotente y nunca
devuelve un error de autenticación. La mesa completa solo vuelve a `libre` cuando **nadie** sigue
activo **y** no queda nada por cobrar; basta que falte una de las dos condiciones para que la mesa
siga `ocupada`.

**Why this priority**: gobierna la limpieza del estado de la mesa entre grupos de comensales —
menos frecuente que agregar/editar líneas, pero crítica para que el tablero del cajero refleje la
realidad sin intervención manual constante.

**Independent Test**: se puede probar llamando `POST /cart/leave` con un token ya expirado y
verificando que responde sin error; y liberando/no liberando la mesa según se cumplan o no ambas
condiciones.

**Acceptance Scenarios**:

1. **Given** un comensal con token de sesión expirado, **When** llama `POST /cart/leave`, **Then**
   el sistema responde sin error de autenticación — el resultado deseado (dejar de contar como
   presente) ya se cumplía de todos modos (`RN-CART-16`).
2. **Given** Ana y Luis en una mesa, ambos se van sin que nadie cobre el pedido `recibida` de Ana,
   **When** se evalúa si la mesa puede liberarse, **Then** la mesa **NO** se libera — queda
   `ocupada` para que el staff vea el descuadre (`RN-CART-15`, ejemplo A).
3. **Given** el pedido de Ana se cancela pero Luis sigue `open` sin haber pedido nada, **When** se
   evalúa la liberación, **Then** la mesa sigue `ocupada` — falta la condición de "nadie activo"
   (`RN-CART-15`, ejemplo B).

---

### User Story 10 - Ciclo de vida de la sesión: TTL deslizante de 4 horas, tope absoluto de 24 (Priority: P1)

Un comensal interactúa con su carrito a lo largo de la visita. Su sesión tiene una ventana
deslizante de inactividad de 240 minutos (4 horas): cada actividad real la corre hacia adelante,
pero solo se reescribe en base de datos cuando faltan 230 minutos o menos (para no escribir en cada
sondeo). El JWT que lleva en el navegador, en cambio, no se reemite nunca dentro de esa ventana:
expira a un tope absoluto de 1440 minutos (24 horas), fijo desde que se abrió la sesión de mesa,
independiente de la actividad.

**Why this priority**: es el mecanismo que determina cuánto tiempo un comensal puede seguir
interactuando con su carrito sin volver a escanear el QR, y el que protege contra sesiones
zombis que nunca expiran.

**Independent Test**: se puede probar abriendo una sesión, verificando que `expires_at` no cambia
en sondeos tempranos y sí cambia una vez se cruza el umbral de 230 minutos restantes; y confirmando
que el `exp` del JWT permanece fijo a 24 horas de la apertura sin importar la actividad.

**Acceptance Scenarios**:

1. **Given** una sesión abierta a las 19:00 (`expires_at=23:00`), **When** el comensal no vuelve a
   interactuar, **Then** a las 23:01 cualquier operación con su token responde `401 "Sesión
   expirada por inactividad"` (`RN-CART-17`).
2. **Given** la misma sesión abierta a las 19:00, **When** el comensal hace una petición real a las
   19:05 (quedan 235 min, > 230), **Then** `expires_at` **no** se reescribe; **When** hace otra a
   las 19:12 (quedan 228 min, ≤ 230), **Then** `expires_at` sí se corre a las 23:12 — la holgura de
   10 minutos evita un `UPDATE`+`COMMIT` en cada sondeo del menú cada 10 segundos (`RN-CART-19`).
3. **Given** la misma sesión abierta el lunes a las 19:00, **When** se inspecciona el `exp` del
   JWT, **Then** es el martes a las 19:00 (24 horas exactas desde la apertura) — sin importar
   cuánta actividad hubo entremedio; el token **no** se reemite al refrescar la ventana de
   inactividad, esa ventana se controla aparte, solo en base de datos (`RN-CART-20`).
4. **Given** una pestaña con el stream SSE del menú abierto toda la noche sin ninguna interacción
   real, **When** se evalúa si la sesión sigue viva, **Then** el handshake del SSE **no** corre el
   TTL — solo las llamadas REST reales (pedir, cancelar, editar) cuentan como actividad; la sesión
   expira igual a las 4 horas de la última acción real (`RN-CART-22`).
5. **Given** una mesa que cobró y cerró su sesión, abriendo una `table_session` nueva después,
   **When** un comensal reingresa con el token de la sesión vieja (JWT aún sin expirar), **Then**
   el sistema responde `401` — nunca se reutiliza un `table_session_id` que ya no coincide con la
   sesión `active` actual de la mesa (`RN-CART-21`; verificado en `test_table_sessions.py`, pasos
   4-5: "token de una sesión de mesa cerrada → 401" y "token con otra mesa/sesión → 401").

**[Anomalía A-36, porción `RN-CART-18`, `DUDOSA/PENDIENTE`, documentada sin especificar]**: la
comparación de expiración es `expires_at <= now` (no `<`); en el instante exacto de igualdad la
sesión ya se considera expirada. Es una resolución de microsegundos, casi inobservable en la
práctica, sin impacto económico demostrado y sin test que la fije como límite contractual. Esta
spec la documenta como comportamiento observado, sin fijarla como contrato a preservar
exactamente en ese límite.

**[Anomalía A-28, `ACCIDENTAL`, documentada sin especificar como corrección obligatoria]**: el
invariante que hace segura la optimización de User Story 10, escenario 2 —
`SESSION_TTL_REFRESH_SLACK_MINUTES` (10 min) **debe ser menor** que `EMPTY_SESSION_TTL_MINUTES`
(30 min, el umbral que usa el barrido de sesiones abandonadas de la spec 010) — no se valida en el
arranque del sistema (`app/core/config.py:36-44` documenta el invariante en comentario;
`app/core/qr_context.py:121` lo usa). Si alguien cambia uno de esos dos valores en `.env` sin
conocer la relación entre ambos, el barrido periódico de sesiones (fuera de alcance de esta spec,
ver spec 010) podría cerrar mesas **activas** que todavía no tienen ningún pedido, creyéndolas
abandonadas. Tratamiento acordado: corregir en fase de modernización añadiendo un validador de
arranque (Pydantic `model_validator` o equivalente) que rechace una configuración que viole el
invariante.

---

### User Story 11 - [Anomalía A-21 confirmada intencional] El token de sesión del comensal vive en localStorage (Priority: P3)

Un comensal abre el menú por QR y recibe un `session_token`. El frontend lo guarda en
`localStorage` del navegador — no en una cookie `httpOnly`, aunque el diseño documentado
originalmente planeaba esa alternativa más segura contra scripts maliciosos en la página.

**Why this priority**: no es parte del flujo funcional del comensal (agregar/editar/enviar), sino
una decisión de arquitectura de almacenamiento de sesión ya cerrada por el negocio — se documenta
por completitud del ciclo de vida de la sesión (User Story 10), con prioridad baja porque no cambia
ningún comportamiento observable del carrito en sí.

**Independent Test**: se puede probar inspeccionando `diner-token.store.ts` y confirmando que el
token se lee/escribe con `localStorage.getItem`/`setItem`, y que el backend nunca emite un header
`Set-Cookie` para ese token.

**Acceptance Scenarios**:

1. **Given** un comensal que abre sesión vía QR, **When** el frontend recibe el `session_token`,
   **Then** lo guarda en `localStorage` bajo la clave `pos.diner.session_token` — el backend
   devuelve el token en el cuerpo JSON de la respuesta, nunca con `Set-Cookie` (`diner-token.store.ts:15-18`).
2. **Given** un enlace compartido con el token como query param (`?s=...`), **When** el comensal lo
   abre en otro dispositivo, **Then** el query param **gana** sobre cualquier token ya presente en
   `localStorage` de ese dispositivo, reingresando a la sesión del enlace (`diner-token.store.ts:26-33`).
3. **Given** un navegador en modo privado con `localStorage` bloqueado, **When** el comensal usa el
   flujo QR, **Then** el sistema sigue funcionando en memoria durante esa carga de página, pero la
   sesión no sobrevive a un recargo (`diner-token.store.ts:34-38`).

**Anomalía A-21 (porción comensal) — clasificación `INTENCIONAL` (confirmada en entrevista P15,
2/3 testigos: CÓDIGO + NEGOCIO)**: preguntado si seguía siendo prioridad terminar la cookie
`httpOnly` documentada originalmente, o si `localStorage` era aceptable, TI/soporte técnico
respondió **"la forma actual es aceptable, no es prioridad"**. `localStorage` queda como el diseño
definitivo del token de sesión del comensal — esta spec lo especifica tal cual, sin fijar la cookie
`httpOnly` como objetivo de modernización. La actualización de `@angular/core` (6 vulnerabilidades
"high" de XSS reportadas por `npm audit`) sigue siendo una corrección recomendada de inmediato,
independiente de esta decisión de almacenamiento — porque un XSS explotable comprometería el token
sin importar dónde viva.

---

### Edge Cases

- **Un comensal edita una línea de carrito enviando `option_ids=[]` explícito** (lista vacía, no
  ausente): sí se revalida contra el catálogo actual — la excepción de `RN-CART-10` aplica solo
  cuando el campo se omite del todo, no cuando se envía vacío a propósito.
- **Dos comensales de la misma mesa agregan al carrito el último ítem disponible de un sabor casi
  simultáneamente**: ambos pueden pasar el chequeo preventivo de `add_item` (ninguno reserva
  todavía); el conflicto real se resuelve recién al confirmar cada pedido en cocina (fuera de
  alcance, ver User Story 7 y specs 008/009).
- **Un comensal cuyo token de sesión expira mientras tiene un carrito con líneas sin enviar**: al
  expirar, su carrito pasa a `abandonado` (`close_participant`) — las líneas no enviadas se pierden;
  no hay mecanismo de recuperación de un carrito abandonado en esta spec.
- **Reingreso con el `session_token` desde un dispositivo distinto al que lo generó, vía enlace
  compartido**: el sistema no distingue dispositivos — cualquiera que tenga el token válido (por
  `localStorage` o por el query param, ver User Story 11) reingresa a la misma sesión de comensal;
  no hay verificación adicional de dispositivo u origen.
- **El QR físico de la mesa nunca expira** (`RN-CART-24`, sin claim `exp`): la única forma de
  invalidarlo es rotar el secreto de firma del servidor, lo que invalidaría **todos** los QR de
  **todas** las mesas del tenant a la vez — no existe rotación por mesa individual en el código
  documentado por esta spec.

## Requirements *(mandatory)*

### Functional Requirements

**Menú público**

- **FR-001**: El precio mostrado para cada variante en el menú DEBE incluir el descuento de la
  mejor promoción porcentual/fija vigente, calculado sobre cantidad 1 y redondeado con "mitad hacia
  arriba" a 2 decimales; si no hay promoción aplicable a cantidad 1, se muestra el precio de
  catálogo sin descuento (`RN-MENU-01`, `RN-MENU-02`).
- **FR-002**: La disponibilidad de una opción de grupo DEBE evaluarse comparando el stock del
  insumo contra la mayor cantidad que cualquier presentación que la use requiera (peor caso), nunca
  por combinación variante×opción individual (`RN-MENU-03`).
- **FR-003**: Una opción ligada a un insumo que no descuenta cantidad positiva en ninguna
  presentación NUNCA debe marcarse agotada por stock, pero SÍ debe dejar de listarse si su insumo se
  desactiva (`RN-MENU-04`).
- **FR-004**: Una opción cuyo insumo está desactivado DEBE marcarse siempre no disponible, sin
  importar el stock numérico restante (`RN-MENU-05`).
- **FR-005**: Una presentación (variante) con un grupo de opciones obligatorio totalmente agotado
  DEBE marcarse no pedible, sin que eso bloquee las demás variantes activas del mismo producto
  (`RN-MENU-06`).
- **FR-006**: El menú DEBE listar únicamente categorías activas, productos activos y disponibles, y
  variantes activas; un producto sin ninguna variante activa restante DEBE desaparecer del listado
  por completo (`RN-MENU-07`).
- **FR-007**: El acceso al menú vía token QR firmado DEBE aplicar un límite de tasa por IP antes de
  verificar la firma del token, y un segundo límite por mesa una vez el token resulta válido
  (`RN-MENU-08`).
- **FR-008**: El endpoint de menú por token QR DEBE responder `404` si la mesa referenciada por el
  token ya no está activa (`RN-MENU-09`).

**Sesión y seguridad del flujo QR**

- **FR-009**: Escanear el QR de una mesa con una `table_session` activa DEBE unir al comensal a esa
  sesión existente; solo se crea una `table_session` nueva si no hay ninguna activa para la mesa
  (`RN-CART-01`).
- **FR-010**: El `display_name` de un comensal NO es único dentro de la sesión; si coincide con uno
  ya presente, el sistema DEBE asignar una etiqueta desambiguada (`"(2)"`, `"(3)"`, ...) sin alterar
  el `display_name` original que el comensal escribió (`RN-CART-02`).
- **FR-011**: Unirse a una mesa DEBE marcarla `ocupada` únicamente la primera vez (si no lo estaba
  ya); el evento correspondiente NO debe repetirse en cada comensal adicional que se una a la misma
  sesión (`RN-CART-03`).
- **FR-012**: El token de sesión del comensal NO DEBE codificar su `display_name`; solo
  `tenant_id`, `table_id`, `participant_id` y `table_session_id` (`RN-CART-04`).
- **FR-013 [Regla protegida A-24]**: `tenant_id`, `table_id`, `participant_id` y
  `table_session_id` DEBEN derivarse exclusivamente de los claims verificados del token firmado en
  cada endpoint público del flujo QR; ningún endpoint DEBE aceptar esos identificadores como
  parámetro que el cliente controle (`RN-CART-26`).
- **FR-014**: Un token de personal (claims `user`/`refresh`) y un token de mesa (QR o sesión) DEBEN
  ser mutuamente excluyentes — cada uno rechazado explícitamente si se presenta en el dominio del
  otro, aunque ambos compartan el mismo secreto de firma (`RN-CART-25`).
- **FR-015**: El token de QR físico de la mesa NO DEBE llevar claim `exp` — no expira nunca; su
  única forma de invalidación es rotar el secreto de firma del servidor (`RN-CART-24`).
- **FR-016**: La apertura de sesión vía QR DEBE aplicar el límite de tasa primero por IP y luego
  por mesa, en ese orden (`RN-CART-27`).
- **FR-017**: La ventana de inactividad del comensal DEBE ser de 240 minutos (`SESSION_TTL_MINUTES`);
  al superarse sin actividad, la siguiente operación con su token DEBE responder `401` (`RN-CART-17`).
- **FR-018**: `expires_at` del comensal DEBE reescribirse únicamente cuando falten
  `SESSION_TTL_MINUTES − SESSION_TTL_REFRESH_SLACK_MINUTES` minutos o menos para expirar (230 de
  240 por defecto); sondeos anteriores a ese umbral NO deben generar escritura (`RN-CART-19`).
- **FR-019**: El `exp` del JWT de sesión del comensal DEBE fijarse a `SESSION_ABS_MAX_MINUTES`
  (1440 = 24 h) desde la apertura de la `table_session`, como tope absoluto independiente de la
  ventana deslizante de 4 h; el JWT NO se reemite al refrescar esa ventana (`RN-CART-20`).
- **FR-020**: El stream SSE del menú NO DEBE refrescar el TTL del comensal en su handshake; solo
  las llamadas REST reales cuentan como actividad (`RN-CART-22`).
- **FR-021**: Reingresar con un token cuyo `table_session_id` ya no coincide con la sesión `active`
  actual de esa mesa DEBE responder `401`, aunque el JWT en sí no haya expirado (`RN-CART-21`).

**Carrito del comensal**

- **FR-022**: Cada comensal DEBE tener como máximo un carrito en estado `abierto` a la vez; al
  enviar un pedido, el carrito pasa a `confirmado` y la siguiente operación DEBE crearle uno nuevo
  (`RN-CART-05`).
- **FR-023**: Enviar un carrito sin líneas DEBE rechazarse con `409` (`RN-CART-06`).
- **FR-024**: La cantidad de una línea de carrito DEBE ser como mínimo 1, tanto al crearla como al
  editarla; no existe una representación válida de "cantidad 0" (`RN-CART-07`).
- **FR-025**: Un comensal DEBE poder enviar más de un pedido en la misma sesión, sin límite de
  "rondas" mientras la sesión siga activa (`RN-CART-08`).
- **FR-026**: Editar una línea de carrito sin enviar `option_ids` DEBE conservar las opciones ya
  guardadas sin revalidarlas contra las reglas de selección vigentes del catálogo actual
  (`RN-CART-10`).
- **FR-027**: Al agregar un combo al carrito, cada componente DEBE guardarse como línea normal a
  precio de catálogo (sin descuento de combo aplicado), marcada con el `combo_id` correspondiente;
  el ahorro del combo se calcula solo al construir la venta, fuera de alcance de esta spec
  (`RN-CART-11`).
- **FR-028**: Las líneas de carrito marcadas con `combo_id` NO DEBEN recibir además un descuento
  por promoción porcentual/fija en la vista previa (`RN-CART-12`).
- **FR-029**: El descuento por promoción de cada línea en la vista previa del carrito DEBE
  redondearse a 2 decimales con "mitad hacia arriba" (`RN-CART-23`).
- **FR-030 [Anomalía A-08, `ACCIDENTAL`, documentada sin especificar como corrección obligatoria]**:
  hoy la vigencia de promociones en el menú y el carrito se evalúa con un `datetime` naive derivado
  de UTC (`cart/service.py:52-53,205-206`; `menu/router.py:82-83`), interpretado incorrectamente
  como hora local por `promotions.local_now()`. Esta spec documenta el comportamiento observado tal
  cual; no lo fija como contrato deseado. El tratamiento acordado es corregirlo en modernización
  aplicando el patrón ya usado en los caminos de cobro real.
- **FR-031 [Anomalía A-47, `INTENCIONAL` confirmada]**: enviar el carrito NO DEBE descontar
  inventario; solo la confirmación del pedido por cocina lo hace (fuera de alcance, specs 008/009).
  El chequeo de disponibilidad que corre al agregar/editar/enviar una línea DEBE ser preventivo —
  sin lock ni reserva de stock — de forma deliberada, aceptando el costo ocasional de que un ítem
  visto disponible falle al confirmarse en hora pico (`RN-CART-09`).
- **FR-032**: La cancelación de un pedido por el propio comensal DEBE permitirse únicamente si el
  pedido sigue `recibida`, o si está `abierta` con **todos** sus ítems todavía `pendiente` en
  cocina; en cualquier otro caso DEBE rechazarse con `409` (`RN-CART-13`).
- **FR-033**: Si tras una cancelación del comensal no queda ningún pedido facturable y no queda
  ningún comensal activo en la mesa, el sistema DEBE liberar la mesa de inmediato, sin esperar al
  barrido periódico (`RN-CART-14`).
- **FR-034**: La mesa DEBE volver a `libre` automáticamente solo cuando se cumplan **ambas**
  condiciones a la vez: ningún comensal sigue `open` Y no hay ningún pedido en estado distinto de
  `pagada`/`cancelada`; si falta cualquiera de las dos, la mesa DEBE permanecer `ocupada`
  (`RN-CART-15`).
- **FR-035**: `POST /cart/leave` (salir de la mesa) DEBE ser idempotente y NO DEBE devolver un
  error de autenticación aunque el token del comensal ya haya expirado (`RN-CART-16`).
- **FR-036 [Anomalía A-28, `ACCIDENTAL`, documentada sin especificar como corrección obligatoria]**:
  el invariante `SESSION_TTL_REFRESH_SLACK_MINUTES < EMPTY_SESSION_TTL_MINUTES`, del que depende la
  seguridad de FR-018 frente al barrido de sesiones de la spec 010, hoy NO se valida en el arranque
  del sistema. Esta spec documenta la ausencia de esa validación como comportamiento actual, sin
  especificarla como aceptable — el tratamiento acordado es añadir un validador de arranque en
  modernización.
- **FR-037 [Anomalía A-36 (porción `RN-CART-18`), `DUDOSA/PENDIENTE`, documentada sin especificar]**:
  la comparación de expiración de sesión usa `expires_at <= now` (no `<`); en el instante exacto de
  igualdad la sesión ya se considera expirada. Esta spec documenta el comportamiento observado, sin
  fijar ese límite exacto de microsegundos como contrato a preservar.
- **FR-038 [Anomalía A-21 (porción comensal), `INTENCIONAL` confirmada]**: el token de sesión del
  comensal DEBE almacenarse en `localStorage` del navegador (clave `pos.diner.session_token`),
  nunca en una cookie `httpOnly` — el backend nunca emite `Set-Cookie` para este token. Esta spec
  especifica `localStorage` como el diseño definitivo, confirmado por decisión de negocio.

### Key Entities *(include if feature involves data)*

- **TableSession**: sesión activa de una mesa física, contenedor de todos los comensales que se
  unen a ella vía QR. Como máximo una `active` por mesa a la vez (`RN-CART-01`). Su cierre y el
  barrido de sesiones abandonadas están fuera de alcance de esta spec (ver spec 010).
- **SessionParticipant**: un comensal dentro de una `TableSession`. Atributos relevantes:
  `display_name` (tal cual lo escribió), `display_label` (desambiguado, `RN-CART-02`), `status`
  (`open`/`closed`), `expires_at` (ventana deslizante de inactividad, `RN-CART-17`/`RN-CART-19`).
- **Cart**: carrito borrador de un `SessionParticipant`. Como máximo uno `abierto` por comensal a
  la vez (`RN-CART-05`); transiciona a `confirmado` al enviarse, o `abandonado` si el comensal se
  va o expira sin enviarlo.
- **CartItem / CartItemOption**: línea de carrito (variante + cantidad ≥1 + opciones elegidas +
  precio unitario calculado al agregarla). Puede llevar `combo_id` si vino de un combo
  (`RN-CART-11`).
- **CustomerOrder**: pedido que resulta de enviar un carrito (`channel="qr"`). Nace en estado
  `recibida`, sin ningún movimiento de inventario asociado todavía (`RN-CART-09`); su avance por
  cocina y su confirmación (único punto real de descuento) están fuera de alcance de esta spec (ver
  specs 008/009).
- **Token de QR** (`typ="qr"`): identifica tenant + mesa, impreso físicamente, sin `exp`
  (`RN-CART-24`).
- **Token de sesión** (`typ="session"`, o equivalente): identifica tenant + mesa + comensal +
  `table_session_id`; `exp` fijo a 24 h desde la apertura de la `table_session`, independiente de
  la ventana deslizante de 4 h que se controla aparte en base de datos (`RN-CART-20`).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-MENU-01` a `RN-MENU-09` y `RN-CART-01` a `RN-CART-27` puede
  verificarse ejecutando los pasos descritos en esta spec contra un `pos-backend` en ejecución, sin
  necesitar leer `cart/service.py`, `menu/router.py` ni `qr_context.py` para entender el
  comportamiento esperado.
- **SC-002**: Las reglas de seguridad de identidad del flujo QR (`RN-CART-24` a `RN-CART-27`,
  regla protegida A-24) permanecen verificables contra el characterization test existente,
  `test_qr_token.py` — roundtrip de ambos tipos de token, detección de manipulación, expiración del
  token de sesión, y aislamiento de tipo — sin que la modernización requiera reescribirlo desde
  cero para conservar cobertura.
- **SC-003**: Las reglas de unión a sesión y desambiguación (`RN-CART-01`, `RN-CART-02`) y de
  reingreso con sesión cerrada (`RN-CART-21`) permanecen verificables contra
  `test_table_sessions.py` — que, pese a su nombre, prueba directamente
  `cart.service.open_session` — sin reescritura desde cero.
- **SC-004**: Las tres anomalías documentadas sin especificar como corrección obligatoria (A-08,
  A-28, A-36) quedan registradas con su comportamiento observado, su evidencia de código y su
  tratamiento acordado, de forma que el equipo de modernización no las reintroduzca por accidente
  ni las trate como si ya tuvieran una decisión de negocio distinta a la citada aquí.
- **SC-005**: Las dos reglas cuya clasificación pasó de `PENDIENTE` a `INTENCIONAL` confirmado
  durante las rondas de entrevista de negocio (A-21 porción comensal, vía P15; A-47, vía P27-bis)
  quedan especificadas como contrato — no como comportamiento a corregir — citando la pregunta y
  respuesta exactas que cerraron cada una.
- **SC-006 [Gap de caracterización]**: no existe hoy ningún characterization test dedicado
  aislado para el menú público (`_option_availability`, `_build_menu`) ni para el ciclo completo de
  `add_item`/`update_item`/`submit_cart`/`cancel_my_order` del carrito — `test_qr_token.py` y
  `test_table_sessions.py` cubren la capa de sesión/token, no estas reglas de menú y CRUD de
  carrito. Cerrar este gap es prioritario antes de la modernización de esta spec, dado que ninguna
  de las 15 reglas de menú/carrito propiamente dichas (fuera de sesión/token) tiene hoy un
  characterization test que las proteja de una regresión silenciosa.

## Out of Scope

- **Qué pasa una vez que el pedido se envía**: el ciclo de vida en cocina (`estado_cocina`), la
  confirmación que sí descuenta inventario, y la cancelación asimétrica según avance de cocina
  desde el lado del staff — cubierto por las specs 008 y 009.
- **El cobro de la mesa**: cómo se calcula la cuenta, el reparto entre comensales, y el cierre de
  sesión de mesa (más allá de la liberación automática por vaciamiento, `RN-CART-14`/`RN-CART-15`,
  que sí es parte de esta spec) — cubierto por la spec 010.
- **El motor de cálculo de promociones que esta spec consume**: cómo se determina qué promoción es
  la "mejor" para una línea, la vigencia por horario/día, la expansión de combos y su ahorro real —
  cubierto por la spec 012 (`best_line_discount`, `active_discount_promotions`, `expand_combo`,
  `local_now`, dueña de A-07). Esta spec solo documenta *dónde* y *cómo* se invocan esas funciones
  desde el menú y el carrito, y la anomalía puntual (A-08) de que dos de esos puntos de invocación
  no aplican correctamente la corrección de zona horaria que la spec 012 ya hizo en el resto del
  sistema.
- **La administración de mesas físicas, sabores/opciones y receta** — cubiertas por las specs 002
  (catálogo), 003 (consumo de inventario) y 004 (grupos de opciones); esta spec solo consume esas
  reglas ya definidas para calcular disponibilidad y precio.

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia del resto de
  las guías de este template ("evitar detalles de implementación"), aquí los nombres de campo,
  valores por defecto, mensajes de error, códigos de estado HTTP y funciones citadas **son** el
  contrato observable que se está documentando — se citan explícitamente porque los criterios de
  aceptación deben ser verificables directamente contra `pos-backend` en ejecución.
- **A-24 se especifica como regla protegida, tal cual, sin tocar el diseño**: tiene evidencia
  directa de decisión de seguridad (historia real de dos endpoints legacy retirados a propósito,
  `memoria-historica.md` #2) y su reversión reintroduciría una vulnerabilidad ya corregida.
  Cualquier cambio de este invariante requiere una nueva decisión de negocio/seguridad explícita,
  no una decisión técnica unilateral.
- **A-21 (porción comensal) y A-47 se especifican como contrato, no como defectos a corregir**: a
  diferencia de la mayoría de anomalías `PENDIENTE` de este proyecto, ambas tienen evidencia directa
  de decisión de negocio (entrevistas P15 y P27-bis respectivamente) que las promovió a
  `INTENCIONAL` confirmado. Cualquier cambio de este comportamiento requiere una nueva decisión de
  negocio explícita.
- **A-08, A-28 y A-36 se documentan pero NO se especifican como comportamiento deseado**: siguiendo
  el tratamiento acordado en `registro-de-anomalias.md` ("documentar sin especificar" /
  "corregir en modernización"), esta spec describe el comportamiento observado hoy porque documenta
  "lo que el sistema YA hace", pero no lo fija como contrato obligatorio para la modernización.
- **RN-CART-18 (porción de A-36, cierre por microsegundos)**: dado el bajo impacto y la
  ausencia de test que lo fije, esta spec no convierte la comparación exacta (`<=` vs `<`) en un
  criterio de aceptación verificable — se documenta como nota de comportamiento, no como
  requisito.
- **El gap de caracterización (SC-006) es una brecha documentada, no una tarea de esta spec**: esta
  spec no crea los characterization tests faltantes para menú y CRUD de carrito — señala su
  ausencia como prioridad para poder proteger estas 15 reglas con evidencia de test explícita antes
  de tocarlas en la modernización.
- **Los valores numéricos y de ejemplo citados en los escenarios** (`SESSION_TTL_MINUTES=240`,
  `SESSION_ABS_MAX_MINUTES=1440`, `SESSION_TTL_REFRESH_SLACK_MINUTES=10`, precios y nombres de
  producto ilustrativos como «Copa grande» o «Ensalada pequeña») corresponden a los valores por
  defecto documentados en `app/core/config.py` y a ejemplos de `reglas-de-negocio.md` — los
  nombres de producto no representan necesariamente el catálogo real vigente hoy en producción.
