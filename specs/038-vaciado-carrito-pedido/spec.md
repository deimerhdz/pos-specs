# Feature Specification: Vaciado del Carrito del Participante al Crear el Pedido (Menú QR)

**Feature Branch**: `038-vaciado-carrito-pedido`

**Created**: 2026-08-26

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad nueva (fase de evolución funcional, Principio I de
la [Constitución](../../.specify/memory/constitution.md)), no una corrección de una anomalía ya
registrada en `registro-de-anomalias.md` — no existe ahí ninguna entrada sobre vaciado de
carrito. La decisión de negocio de fondo ("un carrito confirmado debe desaparecer, no
archivarse") la toma directamente quien encarga esta spec, en esta misma conversación, ejerciendo
el rol de negocio (ver Clarifications). Se construye sobre `submit_cart`
(`app/api/v1/cart/service.py`), la función que las specs
[015](../015-caracterizacion-cart/spec.md) (characterization), [024](../024-pagos-ordenes-mesa/spec.md)
y [025](../025-revision-pago-antes-envio/spec.md) fueron dejando en su forma actual. Esta spec
**modifica deliberadamente** un comportamiento hoy protegido por characterization tests: que el
carrito confirmado se conserva como fila con `status='confirmado'` en vez de borrarse. Los dos
tests con prefijo `"CONGELA comportamiento actual:"` que ejercitan exactamente eso quedan
identificados y su modificación autorizada por esta spec (Principio III de la Constitución):

1. `test_submit_cart_confirma_pedido_y_abre_carrito_nuevo`
   (`app/characterization_tests/test_cart_service.py:484-522`) — asume en su cuerpo
   (`self.assertEqual(old_cart.status, "confirmado")`) que la fila del carrito sobrevive al
   envío del pedido. Esta spec vuelve esa aserción falsa a propósito (FR-003/FR-004): la fila
   deja de existir tras confirmar. Se actualiza citando esta spec.
2. `test_submit_cart_endpoint_evento_tras_commit`
   (`app/characterization_tests/test_cart_router.py:216-252`) — consulta, tras el commit,
   `Cart.status == "confirmado"` sobre una fila fresca (línea 241-243) para verificar el orden
   evento/transacción. Esta spec conserva la garantía de fondo que el test protege (el evento se
   publica después del commit, nunca antes ni dentro), pero la fila que consulta ya no existe: la
   aserción se reescribe para verificar la ausencia de la fila en vez de su estado, sin tocar el
   resto del test.

Ningún otro test `"CONGELA comportamiento actual:"` de `pos-backend` se ve afectado —
en particular, `test_remove_item_ultima_linea_deja_carrito_vacio`
(`test_cart_service.py:314-330`, "el carrito sigue 'abierto', no se borra") protege
`remove_item`, una función distinta que esta spec no toca, y sigue vigente tal cual.

**Input**: User description: en el menú público por QR, las líneas del carrito de un
participante siguen existiendo en la base de datos y en el frontend después de que su pedido ya
se creó, lo que permite reenviar el mismo carrito como un pedido duplicado y hace que, al
recargar la página, el comensal vuelva a ver líneas que ya pidió sin saber si su pedido salió.
Se pide que, al crear el pedido, el carrito del participante que lo originó desaparezca de forma
permanente, en el servidor y en la pantalla, sin ninguna vía para reenviarlo.

## Clarifications

### Session 2026-08-26

- Q: RF-03 pide borrar físicamente las filas del carrito. Hoy el diseño protegido por 2 tests
  CONGELA es marcar la fila vieja como `'confirmado'` (nunca se borra) y abrir una fila nueva
  vacía después. ¿Cuál debe ser el comportamiento final? → A: Borrado físico real — `DELETE` de
  las filas del carrito del participante dentro de la misma transacción del pedido, aceptando
  modificar los 2 tests CONGELA identificados arriba y perder el registro histórico
  `'confirmado'` del carrito origen.
- Q: RF-01 exige que el pedido conserve el "descuento aplicado" por línea, y que la operación
  falle si falta ese dato. Hoy no existe ningún campo persistido para eso — el descuento se
  recalcula al vuelo tanto al pintar el carrito (`serialize_cart`) como al cobrar
  (`promotions.evaluate`/`combo_discount_for_lines`, de forma independiente). ¿Cómo tratamos
  esto? → A: Nuevo campo persistido — agregar el snapshot de descuento por línea (precio unitario
  y total ya descontados, calculados con el mismo motor de promociones que usa hoy
  `serialize_cart`) al crear el `OrderItem` en `submit_cart`, y exponerlo en la respuesta del
  pedido que ve el comensal. Este snapshot **no** sustituye ni alimenta el cálculo de descuento
  del cobro/checkout de staff (`orders/checkout.py`), que sigue recalculando de forma
  independiente exactamente como hoy — ver Fuera de Alcance.
- Q: Para los pedidos que ya existen en la base de datos (creados antes de esta spec, sin las
  nuevas columnas de descuento persistido), ¿qué debe devolver el endpoint de "pedidos enviados"
  en el campo de descuento por línea? → A: Null = sin descuento — el campo nuevo es nullable; las
  filas de `OrderItem` anteriores a esta spec devuelven `null` y el frontend lo trata como "sin
  descuento", sin recalcular ni alterar ningún dato histórico (Principio VII de la Constitución).
- Q: RF-06 pide que un segundo intento de confirmar muestre un mensaje de "el pedido ya fue
  enviado". Hoy, según la carrera exacta, el backend responde 404 ("No hay carrito abierto"), 409
  ("El carrito está vacío") o 409 ("Ya tienes una orden activa") — ninguno dice literalmente eso.
  ¿Qué se exige? → A: Mensaje nuevo y específico — cuando el carrito esté vacío o no exista
  *y* el participante ya tenga un pedido no terminal (el mismo criterio que hoy usa el chequeo de
  "una orden activa por comensal"), el sistema responde con un mensaje explícito de que el pedido
  ya fue enviado, en vez del mensaje genérico de carrito vacío/inexistente.

## User Scenarios & Testing *(mandatory)*

<!--
  Nota de trazabilidad (Principio XII de la Constitución): las etiquetas entre paréntesis
  HU-N/RF-N/CA-N que aparecen abajo remiten a la numeración de la solicitud original que dio
  origen a esta spec (sección "Input" arriba), preservada aquí solo para rastrear de dónde salió
  cada escenario/requisito. A diferencia de citas como "A-04" (que resuelven contra
  `registro-de-anomalias.md`, un documento del repo), esa numeración original no vive en ningún
  archivo aparte — el contrato vinculante es el texto de esta spec (User Stories, FR-XXX, SC-XXX),
  no esas etiquetas.
-->

### User Story 1 - El carrito desaparece en cuanto el pedido se confirma (Priority: P1)

Un comensal arma su carrito desde el menú QR y confirma el pago. En cuanto el servidor confirma
que el pedido se creó, la pantalla del comensal deja de mostrar esas líneas: ve su pedido enviado
y un carrito vacío, listo para una segunda ronda si quiere. Si recarga la página o vuelve más
tarde, el carrito sigue vacío y el pedido sigue apareciendo como enviado — nunca vuelve a ver las
líneas que ya pidió.

**Why this priority**: es el problema de negocio central que motiva la spec — hoy el comensal no
tiene forma de saber, con certeza, que su pedido salió, y una recarga reintroduce ambigüedad.

**Independent Test**: agregar ítems al carrito, confirmar el pedido con un método de pago válido,
y verificar que (a) la respuesta del pedido confirmado incluye esas líneas con sus datos y (b)
una consulta posterior del carrito (incluida tras recargar la página) devuelve cero líneas.

**Acceptance Scenarios**:

1. **Given** un carrito con ítems, **When** el comensal confirma el pedido con éxito, **Then** la
   pantalla muestra el estado del pedido y un carrito vacío, sin error y sin pantalla en blanco
   (CA-1; requisito original RF-09).
2. **Given** un pedido ya confirmado, **When** el comensal recarga o reabre la página, **Then**
   el carrito sigue vacío y el pedido sigue mostrándose como enviado (CA-2).
3. **Given** un carrito con una promoción de descuento activa sobre alguna línea, **When** el
   comensal confirma el pedido, **Then** el pedido enviado muestra exactamente los mismos
   productos, cantidades, precios y descuento por línea que mostraba el carrito justo antes de
   confirmar (CA-6).
4. **Given** un carrito con ítems, **When** la creación del pedido falla por cualquier motivo
   (ej. sin stock, método de pago inválido), **Then** el carrito sigue completo, con las mismas
   líneas, y el comensal puede reintentar (CA-8).

---

### User Story 2 - Un segundo intento de confirmar nunca duplica el pedido (Priority: P1)

Con mala conexión, el comensal presiona "confirmar" más de una vez, o tiene dos pestañas abiertas
con el mismo carrito cargado en memoria (una vieja, otra recién actualizada). En cualquiera de
esos casos, solo se crea un pedido; el segundo intento se rechaza con un mensaje claro de que el
pedido ya fue enviado.

**Why this priority**: sin esta garantía, vaciar el carrito por sí solo no resuelve el problema
de negocio que motiva la spec (el duplicado es el riesgo más caro).

**Independent Test**: confirmar un carrito con éxito y, sin agregar nada nuevo, confirmar de
nuevo (o simular una segunda pestaña con el carrito viejo) — verificar que solo existe un pedido
y que la segunda respuesta indica explícitamente que ya fue enviado.

**Acceptance Scenarios**:

1. **Given** un carrito recién confirmado, **When** el mismo comensal pulsa confirmar una segunda
   vez de inmediato, **Then** no se crea ningún pedido nuevo y la respuesta indica que el pedido
   ya fue enviado (CA-3, RF-06).
2. **Given** dos pestañas del mismo comensal, una con el carrito ya confirmado en otra pestaña y
   otra con el carrito viejo todavía en memoria, **When** la pestaña vieja intenta confirmar,
   **Then** no se crea un segundo pedido y la respuesta indica que ya fue enviado (CA-4).
3. **Given** un carrito vacío (sin ítems, sin pedido previo), **When** el comensal intenta
   confirmar, **Then** el sistema muestra un aviso de carrito vacío — un mensaje distinto al de
   "ya fue enviado" — y no crea nada (CA-7).

---

### User Story 3 - Segunda ronda desde un carrito limpio (Priority: P2)

Después de confirmar su primer pedido, el comensal sigue en la mesa y quiere pedir algo más. Sin
recargar la página ni hacer nada especial, puede volver a agregar ítems: empieza una segunda
ronda desde cero, sin arrastrar ninguna línea de la ronda anterior.

**Why this priority**: valor claro para el comensal, pero depende de que User Story 1 ya
funcione (el vaciado); sin ella no hay "carrito limpio" del que partir.

**Independent Test**: confirmar un pedido y, en la misma sesión, agregar un ítem nuevo — verificar
que el carrito resultante contiene solo ese ítem nuevo, no los de la ronda ya confirmada, y que
se puede confirmar como un segundo pedido independiente.

**Acceptance Scenarios**:

1. **Given** un comensal que ya confirmó un pedido, **When** agrega un ítem nuevo, **Then** el
   carrito muestra únicamente ese ítem (RF-08).
2. **Given** ese carrito de la segunda ronda, **When** el comensal lo confirma, **Then** se crea
   un segundo pedido independiente del primero, y ambos pedidos siguen existiendo con sus
   respectivas líneas (User Story 3, Acceptance Scenario 2).

---

### User Story 4 - El vaciado de un comensal no afecta a los demás de su mesa (Priority: P2)

En una mesa con varios comensales pidiendo cada uno desde su propio teléfono, cuando uno de ellos
confirma su pedido, los carritos de los demás —con sus propias líneas, sin confirmar todavía—
siguen intactos y visibles solo para cada uno de ellos.

**Why this priority**: es una garantía de aislamiento necesaria para que el vaciado sea seguro en
el caso de uso real (mesas con varios comensales), pero no bloquea la entrega de la User Story 1
para un comensal solo.

**Independent Test**: con dos comensales en la misma mesa, cada uno con su carrito, confirmar el
pedido de uno y verificar que el carrito del otro no cambió ni de contenido ni de cantidad de
líneas.

**Acceptance Scenarios**:

1. **Given** una mesa con dos comensales, cada uno con su carrito propio, **When** uno de ellos
   confirma su pedido, **Then** el carrito del otro comensal conserva exactamente las mismas
   líneas que tenía antes (CA-5, RF-05).
2. **Given** esa misma mesa, **When** un comensal intenta cualquier operación sobre el carrito de
   otro comensal, de otra sesión de mesa o de otro tenant, **Then** el sistema la rechaza sin
   vaciar ni modificar ese carrito ajeno (RF-10).

---

### Edge Cases

- **Confirmación doble desde dos pestañas del mismo participante**: cubierto por User Story 2,
  Acceptance Scenario 2 (CA-4) — la garantía de última instancia sigue siendo el índice único
  `idx_active_order_per_participant` (spec 025, FR-013), sin cambios; esta spec solo mejora el
  mensaje del camino de aplicación que ya intercepta el caso más común.
- **Respuesta del servidor perdida después de haber creado el pedido** (el pedido se creó y el
  carrito se vació, pero el comensal nunca vio la confirmación): al reabrir o recargar, el
  comensal ve un carrito vacío y su pedido en la lista de pedidos enviados (User Story 1,
  Acceptance Scenario 2) — nunca un error ni un carrito con las líneas "de vuelta". Un reintento
  de confirmar en ese estado cae en User Story 2 (RF-06), no crea un segundo pedido.
- **Carrito vacío al confirmar**: rechazado con un aviso de carrito vacío, distinguible del aviso
  de "ya fue enviado" (User Story 2, Acceptance Scenario 3 / CA-7) — sin crear nada.
- **Un producto del carrito se desactiva o se agota entre añadirlo y confirmar**: sin cambios
  respecto al comportamiento actual de `submit_cart` (el chequeo de disponibilidad de stock ya
  existente se sigue ejecutando antes de crear el pedido y de vaciar el carrito; si falla, ninguno
  de los dos ocurre, ver RF-04) — no es parte del alcance de esta spec redefinir esa validación.
- **Una promoción vence entre añadir la línea y confirmar**: el descuento que se persiste en el
  pedido (RF-01) es el que esté vigente en el momento exacto de confirmar, calculado con el mismo
  motor que ya usa el carrito — no el que se veía cuando se agregó la línea. Este comportamiento
  ya es el actual para el total que ve el comensal en el carrito antes de confirmar; esta spec no
  lo cambia, solo lo persiste.
- **Pedidos creados antes de esta spec, sin snapshot de descuento persistido**: el campo de
  descuento por línea (FR-002/FR-013) es nuevo; las filas de `OrderItem` que ya existían al
  desplegar esta spec no lo tienen. El endpoint de pedidos enviados DEBE devolver `null` en ese
  campo para esas filas — nunca recalculado ni inferido a partir del motor de promociones actual
  — y el frontend DEBE tratarlo como "sin descuento" (FR-015).
- **La sesión de mesa se cierra mientras el participante confirma**: sin cambios respecto al
  comportamiento actual de `submit_cart` — esta spec no agrega ni quita ninguna validación sobre
  el estado de la sesión de mesa; queda gobernado por las specs que ya definen ese ciclo de vida
  (010, 016).
- **Dos participantes confirmando simultáneamente en la misma sesión**: sin conflicto — el
  vaciado es siempre por participante (RF-05), así que dos confirmaciones simultáneas de
  comensales distintos son independientes entre sí (User Story 4).
- **El participante abandona sin confirmar y su carrito queda huérfano**: fuera de alcance (ver
  Fuera de Alcance) — esta spec no purga carritos que nunca se convirtieron en pedido.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Historia 1]**: Al confirmar un pedido, el sistema DEBE construirlo enteramente a
  partir del carrito almacenado en el servidor para ese participante — nunca a partir de líneas,
  precios o cantidades que envíe el cliente en la petición de confirmación. *(Sin cambio: ya es
  el comportamiento actual de `submit_cart`; se deja explícito porque el vaciado que introduce
  esta spec depende de que el pedido y el carrito sigan siendo la misma fuente de datos hasta el
  instante de la confirmación.)*
- **FR-002 [Historia 1]**: Por cada línea del carrito que pase a formar parte del pedido, el
  sistema DEBE conservar en el propio pedido, sin depender del carrito para reconstruirlos:
  producto, variante, selecciones de insumos (opciones), cantidad, precio unitario aplicado,
  descuento aplicado (unitario y de línea, `null` —nunca cero— cuando no aplicó ninguno) y total
  de línea ya con ese descuento reflejado.
- **FR-003 [Historia 1, decisión de esta spec]**: Al crear correctamente el pedido, el sistema
  DEBE eliminar de forma permanente (no marcar, no archivar) las filas del carrito del
  participante que lo originó — la fila del carrito y sus líneas dejan de existir en la base de
  datos. *(Modifica el comportamiento protegido por
  `test_submit_cart_confirma_pedido_y_abre_carrito_nuevo` y
  `test_submit_cart_endpoint_evento_tras_commit`, citados en "Naturaleza de esta spec".)*
- **FR-004 [Historia 1, Principio VIII de la constitución — atomicidad]**: La eliminación del
  carrito (FR-003) DEBE ocurrir dentro de la misma transacción que la creación del pedido (FR-001
  a FR-002). Si la creación del pedido falla por cualquier motivo (validación de pago, stock
  insuficiente, error inesperado), el carrito NO DEBE eliminarse ni modificarse — la transacción
  completa se revierte y el comensal puede reintentar con el mismo carrito intacto (CA-8).
- **FR-005 [Historia 1, condición previa]**: Si algún dato exigido por FR-002 no puede
  determinarse en el momento de confirmar (por ejemplo, el motor de descuento no puede resolver
  el descuento aplicable a una línea), el sistema DEBE rechazar la confirmación sin crear el
  pedido ni eliminar ninguna línea del carrito — igual que cualquier otra falla cubierta por
  FR-004.
- **FR-006 [Historia 4]**: El sistema DEBE eliminar únicamente las filas del carrito del
  participante que confirmó el pedido. Los carritos de los demás participantes de la misma sesión
  de mesa, o de cualquier otra sesión de mesa o tenant, NO DEBEN verse afectados en su contenido
  ni en su cantidad de líneas por esta operación (CA-5, RF-10).
- **FR-007 [Historia 2, decisión de esta spec]**: Cuando el comensal intente confirmar y no
  exista ya un carrito con líneas para él (porque el carrito que originó su último pedido ya fue
  eliminado por FR-003) **y** ese comensal ya tenga un pedido no terminal creado a partir de un
  envío anterior, el sistema DEBE responder con un mensaje que indique explícitamente que el
  pedido ya fue enviado — en lugar del mensaje genérico de carrito vacío o inexistente — y NO
  DEBE crear ningún pedido nuevo.
- **FR-008 [Historia 2, sin cambio]**: El sistema DEBE seguir impidiendo, a nivel de aplicación y
  de base de datos, que un mismo participante tenga más de un pedido no terminal a la vez —
  garantía de última instancia ante confirmaciones casi simultáneas (spec 025, FR-013,
  `idx_active_order_per_participant`), sin modificaciones por esta spec.
- **FR-009 [Historia 2]**: Intentar confirmar un carrito vacío (sin ítems y sin que exista un
  pedido no terminal previo de ese comensal) DEBE mostrar un aviso de carrito vacío, distinguible
  del mensaje de FR-007, y no debe crear nada (CA-7).
- **FR-010 [Historia 1, frontend]**: El frontend DEBE vaciar su copia local del carrito
  únicamente después de recibir la confirmación exitosa del servidor a la petición de envío del
  pedido — nunca antes, nunca de forma optimista. *(Sin cambio: ya es el comportamiento actual de
  `payment-method-step.component.ts`/`transfer-details-step.component.ts`, que llaman
  `cart.clear()` solo después de que `submitCart()` resuelve con éxito; se deja explícito para
  que la nueva relación "vaciado real en servidor ⇒ frontend debe reflejarlo" no se rompa por una
  futura optimización que adelante el vaciado local.)*
- **FR-011 [Historia 1, frontend]**: Si el frontend se recarga o se reabre, DEBE reconstruir el
  estado del carrito consultando al servidor — la copia local nunca es la fuente de verdad.
  *(Sin cambio: ya es el comportamiento actual de `DiningCartService.load()`, que proyecta la
  respuesta de `GET /cart`; con el vaciado por eliminación de FR-003, esa consulta ahora refleja
  un carrito verdaderamente vacío en vez de uno reabierto automáticamente.)*
- **FR-012 [Historia 1, frontend]**: Tras confirmar con éxito, el frontend DEBE mostrar el estado
  del pedido enviado y un carrito vacío listo para una eventual segunda ronda — nunca un mensaje
  de error ni una pantalla en blanco, incluso si la eliminación del carrito en el servidor implica
  que una consulta inmediata al carrito ya no encuentra ninguna fila previa que proyectar.
- **FR-013 [Historia 1, CA-6]**: La respuesta del pedido que ve el comensal (al confirmarlo y al
  consultar sus pedidos enviados) DEBE mostrar, por cada línea, el precio unitario y el total ya
  con el descuento aplicado (FR-002) reflejado — igual a lo que el carrito mostraba justo antes de
  confirmar. *(Hoy el pedido devuelto al comensal expone únicamamente `unit_price` sin descuento
  — `OrderItemResponse`, `app/api/v1/orders/schemas.py:139-157` — mientras que el carrito sí
  distingue precio con/sin descuento en `CartItemResponse`; esta spec cierra esa inconsistencia
  para el pedido recién confirmado.)*
- **FR-014 [Fuera de alcance delimitado]**: El snapshot de descuento persistido por FR-002/FR-013
  NO DEBE sustituir, alimentar ni alterar el cálculo de descuento que ya hace el flujo de cobro de
  staff (`orders/checkout.py`, `promotions.evaluate`/`combo_discount_for_lines`) al confirmar o
  cobrar el pedido — ese cálculo sigue siendo independiente y recalculándose exactamente como hoy.
- **FR-015 [Historia 1, compatibilidad de datos]**: El nuevo campo de descuento por línea
  (FR-002/FR-013) DEBE ser opcional (`null`) para las filas de `OrderItem` creadas antes de esta
  spec. El sistema NO DEBE recalcular ni backfillear ese valor para pedidos históricos; el
  frontend DEBE interpretar `null` en ese campo como "sin descuento" para esas filas.

### Key Entities *(include if feature involves data)*

- **Carrito (Cart)**: borrador de líneas de un participante antes de confirmar. Cambia de ciclo de
  vida con esta spec: hoy pasa de `'abierto'` a `'confirmado'` (se conserva); con esta spec, al
  confirmarse con éxito, la fila y sus líneas se eliminan por completo en vez de marcarse. El
  estado `'abandonado'` (sesión que expira sin confirmar) no cambia — sigue fuera de alcance.
- **Pedido (CustomerOrder) / Línea de pedido (OrderItem)**: a partir de esta spec, cada línea de
  pedido debe bastarse a sí misma para reconstruir lo que el comensal pidió y pagó, incluyendo el
  descuento que tenía aplicado, sin ya depender de que exista una fila de carrito de origen (que,
  de hecho, ya no existirá). El nuevo campo de descuento por línea es nullable: las filas de
  `OrderItem` anteriores a esta spec quedan con `null` (sin backfill retroactivo, FR-015); solo
  los pedidos confirmados a partir de esta spec lo llevan poblado.
- **Participante (SessionParticipant)**: dueño de su propio carrito y de sus propios pedidos; el
  vaciado y el mensaje de "ya fue enviado" operan siempre en su alcance, nunca en el de otro
  participante de la misma o de otra sesión de mesa.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los pedidos confirmados con éxito dejan el carrito del participante que
  los originó en cero líneas, verificable tanto en la respuesta inmediata como en una consulta
  posterior del carrito (incluida tras recargar la página).
- **SC-002**: El 100% de los intentos de confirmar un carrito ya vaciado por un pedido anterior
  (sin agregar nada nuevo) son rechazados sin crear un pedido adicional, con un mensaje que indica
  que el pedido ya fue enviado.
- **SC-003**: El 0% de las confirmaciones dobles (mismo comensal, casi simultáneas, incluyendo el
  caso de dos pestañas) producen más de un pedido — verificable contando pedidos por comensal tras
  el escenario.
- **SC-004**: El 100% de las confirmaciones fallidas (stock insuficiente, pago inválido, error
  inesperado) dejan el carrito exactamente como estaba antes del intento, sin ninguna línea
  perdida.
- **SC-005**: El 100% de los pedidos confirmados con una promoción activa muestran, en la
  respuesta que ve el comensal, el mismo precio con descuento que mostraba el carrito justo antes
  de confirmar.
- **SC-006**: El 0% de las confirmaciones de un comensal modifican el contenido o la cantidad de
  líneas del carrito de cualquier otro comensal, sesión de mesa o tenant.

## Out of Scope

- **La purga de carritos abandonados** que nunca se convirtieron en pedido (`status='abandonado'`
  o carritos `'abierto'` huérfanos de una sesión expirada) — esta spec solo vacía el carrito que
  sí se convirtió en pedido.
- **El carrito del terminal de staff** (mostrador/mesero) — esta spec es exclusiva del carrito del
  comensal en el menú público por QR (`app/api/v1/cart/`).
- **Cualquier cambio en el cálculo o la resolución de descuentos** — el motor de promociones
  (`promotions.evaluate`, `combo_discount_for_lines`, `best_line_discount`) no se modifica; esta
  spec solo persiste, en el pedido, el resultado que ese motor ya produce hoy al confirmar, sin
  tocar cómo se calcula (ver FR-014).
- **Modificar o cancelar un pedido ya enviado** — una vez creado y con el carrito vacío, el ciclo
  de vida del pedido en sí (confirmación de staff, cobro, cancelación) sigue gobernado sin cambios
  por las specs 017, 024, 025, 028 y 029.
- **Añadir una excepción manual al índice único de "una orden activa por participante"** — el
  mecanismo de última instancia contra duplicados (spec 025, FR-013) no se toca; esta spec solo
  mejora el mensaje del camino de aplicación que se ejecuta antes de llegar a ese índice.
- **Validar de nuevo la disponibilidad de un producto o la vigencia de una variante** al momento
  de confirmar más allá de lo que `submit_cart` ya valida hoy — esta spec no amplía ni reduce esa
  validación existente.
- **Registrar formalmente esta decisión en `registro-de-anomalias.md`** — ya no es un ítem de
  alcance abierto: la entrada existe como A-53 en
  `specs/000-reconocimiento/registro-de-anomalias.md`, exigida sin condicional por el Principio II
  de la Constitución antes de implementar. Esta spec ya documentaba la decisión en su propio texto
  ("Naturaleza de esta spec" y "Clarifications"); A-53 solo la traslada al libro de autorizaciones
  vigente.

## Assumptions

- **"Pedido no terminal" es el mismo criterio que ya usa `submit_cart` hoy** para impedir una
  segunda orden activa (`_NON_TERMINAL_ORDER_STATUSES`, spec 024/025) — FR-007/FR-008 lo
  reutilizan tal cual, sin definir un nuevo estado ni una nueva noción de "pedido activo".
  Vinculado a: FR-007, FR-008.
  Riesgo si es incorrecta: el mensaje de "ya fue enviado" podría no dispararse (o dispararse de
  más) en combinaciones de estado no cubiertas por ese criterio existente.
- **El snapshot de descuento por línea (FR-002/FR-013) se calcula con el mismo motor que hoy usa
  `serialize_cart`** para pintar el carrito (`promotions.active_discount_promotions` +
  `best_line_discount`), evaluado en el instante de confirmar, no en el instante en que se agregó
  la línea — igual que ya ocurre hoy para lo que el comensal ve en el carrito antes de confirmar.
- **No existe hoy ningún otro consumidor que dependa de que la fila de carrito confirmada siga
  existiendo** después de convertirse en pedido, más allá de los dos tests CONGELA ya identificados
  y modificados por esta spec — verificado revisando los usos de `Cart`/`CartItem` fuera del
  módulo `cart` (`orders/consolidation.py`, `orders/checkout.py`), que solo consultan carritos con
  `status='abierto'` y no se ven afectados por la eliminación de uno ya confirmado.
- **El estado `'confirmado'` del check constraint de `Cart` queda sin ningún camino que lo
  produzca** tras esta spec (era el único código que lo asignaba) — se documenta como implicación
  de datos a resolver en la fase de planeación (`/speckit-plan`), no como un requisito funcional
  de esta spec.
