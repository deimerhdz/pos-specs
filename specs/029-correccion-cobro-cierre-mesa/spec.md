# Feature Specification: Correcciones de Cobro, Anulación y Descuento en la Terminal de Mesas (Skeilopos)

**Feature Branch**: `029-correccion-cobro-cierre-mesa`

**Created**: 2026-08-21

**Status**: Draft

**Naturaleza de esta spec**: correcciones de comportamiento **nuevas**, no de ingeniería inversa
(fase de evolución funcional, Principio I de la
[Constitución](../../.specify/memory/constitution.md)). Se construye directamente sobre
[spec 028](../028-terminal-mesas-modo-hibrido/spec.md) — cuya implementación ya está en producción
(commit "feat: improve table terminal") — y corrige cuatro comportamientos observados en esa
implementación:

1. Spec 028 definió dos acciones de impresión distintas para el mismo documento ya emitido:
   "Reimprimir Factura POS" (FR-007, secundaria del modo "Resumen de Cuenta") y "Reimprimir
   Factura" (FR-012, acceso desde cabecera/barra lateral). En la implementación ambas quedaron
   visibles al mismo tiempo tras confirmar el pago, como dos botones que hacen exactamente lo
   mismo. Esta spec las **consolida en una sola acción**.
2. Para el negocio, "Listo" es un estado compuesto: significa que el pedido **ya se cobró y ya
   terminó de prepararse**, las dos cosas a la vez. Hoy la insignia "Listo" (listado de mesas y
   detalle del pedido) se calcula mirando solo el estado de cocina, sin verificar si el pago ya
   se confirmó — por eso una mesa con un pedido sin cobrar pero con cocina terminada muestra
   "Listo" al mismo tiempo que un aviso de que tiene pedidos sin cobrar, una contradicción visible
   que le da al personal la falsa impresión de que el pedido ya quedó resuelto. Esta spec corrige
   el cálculo de esa insignia para que exija ambas condiciones a la vez.
3. Aplica al ámbito de la Terminal de Mesas una regla ya identificada como comportamiento
   accidental durante el reconocimiento del sistema: `void_item` hoy no valida si la orden
   padre ya está pagada (`RN-ORD-39`, ver
   [`reglas-de-negocio.md`](../000-reconocimiento/reglas-de-negocio.md)), por lo que un ítem o una
   orden ya pagados y entregados todavía se pueden anular. Esta spec convierte esa anomalía
   accidental en una **decisión de negocio explícita**: un pedido pagado se asume entregado y ya
   no es anulable.
4. Aplica a la Terminal de Mesas la decisión de negocio **A-11**, ya registrada en
   [spec 011](../011-venta-mostrador-constructor-factura/spec.md) (FR-015) y referenciada por
   [spec 008](../008-confirmacion-cobro-legado-y-cancelacion-de-pedidos/spec.md) (Historia 5): el
   cajero no debe poder aplicar descuento manual en ningún punto de cobro. El atajo "Aplicar
   descuento (F4)" de la Terminal de Mesas es la última vía de ese tipo que sigue expuesta y
   queda eliminada por esta spec; el único descuento permitido es el que resulte automáticamente
   del motor de promociones/combos (spec 012, spec 013) cuando un producto califique.

**Input**: User description: ajustar varias cosas al confirmar el pago del pedido por parte del
cliente. Primero, hoy aparecen dos botones que hacen lo mismo ("Reimprimir factura" e "Imprimir
factura POS"); debe quedar una sola acción "Imprimir Factura". Segundo, al cerrar la mesa el
pedido sigue apareciendo como "Listo", dando la impresión de que faltan pedidos por cobrar.
Tercero, cuando la orden ya se pagó y quedó marcada como "Listo" no debe poderse anular, porque se
asume que el pedido fue entregado y no puede devolverse. Cuarto, se debe deprecar la opción de
aplicar descuento manual por parte del cajero: el descuento solo debe aplicar cuando un producto
hace parte de una promoción. Se adjuntó una captura de la Terminal de Mesas (Mesa 1, orden con un
"Cono sencillo" marcado "Listo", acción "Anular" visible junto al ítem, atajo "Aplicar descuento
(F4)" visible en la barra superior, y un aviso "No se puede cobrar esta mesa — Tiene pedidos sin
cobrar, pero su sesión está cerrada") como evidencia del estado actual de la pantalla.

## Clarifications

### Session 2026-08-21

- Q: ¿Algún rol (por ejemplo, Administrador) conserva la posibilidad de aplicar un descuento
  manual como excepción, o la prohibición es absoluta para todos los roles? → A: Prohibición
  absoluta para todos los roles, sin excepción — ningún usuario, ni siquiera Administrador, puede
  aplicar descuento manual; el único descuento posible es el automático por promoción.
- Q: ¿Qué significa exactamente la insignia "Listo" a nivel de mesa/pedido? → A: "Listo" significa
  que el pedido ya se cobró y que la orden se terminó de preparar — las dos condiciones a la vez,
  no una sola.
- Q: Dado ese significado, ¿el defecto reportado es que la insignia "Listo" se muestra sin que el
  pago se haya confirmado realmente (calculada solo a partir del estado de cocina)? → A: Sí — ese
  es exactamente el defecto: la insignia hoy ignora el estado de pago de la orden.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Un pedido ya pagado no se puede anular (Priority: P1)

Una vez que una orden (o alguno de sus ítems) ya tiene el pago confirmado y facturado, ni el
cajero ni el mesero pueden anularla desde la Terminal de Mesas, sin importar el estado de
preparación en cocina del ítem (pendiente, en preparación o listo). El sistema asume que un
pedido pagado ya fue entregado al cliente y, por lo tanto, no es retornable por esa vía.

**Why this priority**: es la corrección con mayor riesgo si no se aplica — hoy un pedido ya
cobrado y entregado se puede anular igual, lo que puede generar descuadres de caja y reversas de
inventario sobre mercancía que ya salió físicamente del negocio.

**Independent Test**: sobre una orden ya pagada y facturada, intentar anular uno de sus ítems (o
la orden completa) desde la Terminal de Mesas y verificar que el sistema rechaza la acción con un
mensaje claro, sin alterar el pago, la factura ni el inventario.

**Acceptance Scenarios**:

1. **Given** una orden con el pago confirmado y ya facturada, **When** el cajero o el mesero
   intenta anular uno de sus ítems, **Then** el sistema rechaza la acción e informa que el pedido
   ya fue pagado y no puede anularse.
2. **Given** la misma orden pagada, **When** se observa la interfaz, **Then** la acción "Anular"
   ya no se muestra como disponible para ninguno de sus ítems.
3. **Given** una orden todavía sin pago confirmado, **When** el mesero o cajero anula uno de sus
   ítems, **Then** la anulación se sigue permitiendo exactamente como hoy (esta spec no cambia el
   comportamiento de anulación sobre pedidos sin pagar).
4. **Given** un ítem marcado "Listo" en cocina pero cuya orden todavía no tiene el pago
   confirmado, **When** se intenta anular ese ítem, **Then** el sistema lo permite — el bloqueo
   depende únicamente de si la orden ya está pagada, no del estado de cocina del ítem.

---

### User Story 2 - Ningún rol puede aplicar descuento manual (Priority: P1)

En la Terminal de Mesas ya no existe ninguna vía para que un usuario introduzca un descuento
manual sobre el total de la cuenta (ni el atajo "Aplicar descuento (F4)", ni ningún otro control
equivalente) — la prohibición es absoluta para todos los roles, incluido Administrador, sin
excepción alguna (decisión confirmada en la sesión de clarificación de esta spec). El único
descuento que puede aparecer en el total es el que el motor de promociones/combos calcula
automáticamente cuando alguno de los productos de la orden califica para una promoción activa.

**Why this priority**: aplica una decisión de negocio ya tomada (regla A-11) que hasta ahora no
se había aplicado a esta pantalla; dejarla pendiente expone al negocio a descuentos no
autorizados aplicados por criterio del cajero.

**Independent Test**: abrir la Terminal de Mesas sobre una mesa con una cuenta activa y verificar
que no existe ningún atajo, botón ni campo para ingresar un descuento manual; agregar un producto
que sí califica para una promoción activa y verificar que el descuento aparece calculado
automáticamente, sin intervención del cajero.

**Acceptance Scenarios**:

1. **Given** la Terminal de Mesas con una cuenta activa, **When** cualquier usuario —incluido uno
   con rol Administrador— revisa la barra superior y el panel de la cuenta, **Then** no encuentra
   ningún atajo "Aplicar descuento (F4)" ni ningún otro control para ingresar un descuento
   manual, sin excepción por rol.
2. **Given** una cuenta cuyos productos no califican para ninguna promoción activa, **When** se
   revisa el total, **Then** el descuento mostrado es $0, sin ninguna vía disponible para que el
   cajero lo modifique.
3. **Given** una cuenta con al menos un producto que sí hace parte de una promoción activa,
   **When** se calcula el total, **Then** el descuento correspondiente aparece aplicado
   automáticamente, calculado por el motor de promociones (spec 012), sin que el cajero lo haya
   introducido.
4. **Given** un cajero que intenta invocar el atajo de teclado F4 (por costumbre del flujo
   anterior), **When** lo presiona, **Then** no ocurre ninguna acción — el atajo ya no está
   asociado a ninguna función.

---

### User Story 3 - La insignia "Listo" solo aparece cuando el pedido realmente ya se cobró (Priority: P1)

"Listo" es, para el negocio, un estado compuesto: significa que el pedido ya se cobró **y** que
cocina ya terminó de prepararlo, las dos condiciones a la vez. Hoy el sistema muestra "Listo" —
tanto en el listado de mesas como en el detalle del pedido— mirando solo el estado de cocina, sin
revisar si el pago ya se confirmó. Esta historia corrige esa insignia para que solo aparezca
cuando el pedido esté realmente pagado y preparado; mientras falte el pago, el sistema muestra un
estado distinto que refleja con precisión que todavía hay algo pendiente de cobrar.

**Why this priority**: es la causa raíz de la confusión reportada por el usuario y comparte el
mismo patrón de fondo que la Historia 1 — un punto del sistema que no valida el estado de pago de
la orden antes de mostrar o permitir algo. Mientras la insignia "Listo" pueda aparecer sin que el
pago esté confirmado, el personal no puede confiar en ella para saber si de verdad falta cobrar.

**Independent Test**: sobre una mesa con un pedido cuyo ítem de cocina ya está "Listo" pero cuyo
pago todavía no se ha confirmado, verificar que el listado de mesas y el detalle del pedido NO
muestran la insignia "Listo" — deben mostrar un estado que indique que falta cobrar. Confirmar el
pago y verificar que, solo entonces, la insignia cambia a "Listo".

**Acceptance Scenarios**:

1. **Given** un pedido cuyo ítem ya está terminado en cocina pero cuyo pago todavía no se ha
   confirmado, **When** se revisa el listado de mesas, **Then** la mesa NO muestra la insignia
   "Listo" — muestra un estado que indica que el pedido está pendiente de cobro.
2. **Given** el mismo pedido, **When** se abre el detalle de la mesa, **Then** el encabezado del
   pedido tampoco indica "listo para cobrar" como si ya estuviera resuelto — refleja con
   precisión que el cobro sigue pendiente.
3. **Given** ese mismo pedido, **When** el cajero confirma el pago, **Then** la insignia "Listo"
   aparece de inmediato, tanto en el listado de mesas como en el detalle del pedido, porque ahora
   sí se cumplen las dos condiciones (pagado y preparado).
4. **Given** un pedido ya pagado cuyo ítem todavía está "pendiente" o "en preparación" en cocina
   (no "listo"), **When** se revisa su insignia, **Then** tampoco muestra "Listo" — la insignia
   exige las dos condiciones a la vez, no basta con que ya esté pagado.

---

### User Story 4 - Una sola acción para imprimir la factura ya emitida (Priority: P2)

Después de confirmar el pago de una orden (por QR o manual) y de que su factura ya quedó
generada, el cajero ve una única acción "Imprimir Factura" para reimprimirla — en vez de dos
botones distintos ("Reimprimir Factura POS" e "Imprimir Factura") que reimprimen exactamente el
mismo documento.

**Why this priority**: es una corrección de claridad operativa — no bloquea el cobro ni pone en
riesgo datos, pero genera dudas diarias sobre cuál de los dos botones usar.

**Independent Test**: sobre una mesa con una orden ya pagada y facturada, verificar que en la
barra lateral y en la cabecera aparece una sola acción de impresión de factura, y que al usarla
se reimprime el mismo documento ya emitido sin generar una nueva venta.

**Acceptance Scenarios**:

1. **Given** una orden (QR o manual) ya pagada y con factura emitida, **When** el cajero revisa
   la cabecera y la barra lateral de la mesa, **Then** encuentra una única acción "Imprimir
   Factura" — ya no coexisten "Reimprimir Factura POS" e "Imprimir Factura"/"Reimprimir Factura"
   como accesos separados.
2. **Given** esa acción única, **When** el cajero la usa, **Then** el sistema reimprime la misma
   factura ya emitida, sin crear un nuevo registro de venta ni afectar el pago ya registrado.
3. **Given** una orden que todavía no tiene el pago confirmado, **When** el cajero revisa la
   pantalla, **Then** sigue viendo "Imprimir Pre-cuenta" como una acción distinta e independiente
   — esta spec no elimina ni combina la pre-cuenta con la factura, solo unifica las dos acciones
   de reimpresión posteriores al pago.
4. **Given** una orden que nunca llegó a pagarse (cancelada o con el pago rechazado sin
   reintento), **When** el cajero revisa las acciones disponibles, **Then** "Imprimir Factura" no
   aparece, porque nunca existió un documento de venta que reimprimir.

---

### Edge Cases

- **Se intenta anular un ítem de una orden en el mismo instante en que se confirma su pago**
  (condición de carrera): el sistema garantiza que, si el pago ya quedó registrado, la anulación
  se rechaza — nunca deben coexistir un ítem anulado y un pago confirmado sobre el mismo ítem.
- **Se intenta anular una orden completa (no solo un ítem) que ya está pagada**: se rechaza con
  el mismo motivo que a nivel de ítem — un pedido pagado se asume entregado en su totalidad.
- **Una promoción que aplicaba a un producto de la cuenta se desactiva o expira mientras la
  cuenta sigue abierta**: el descuento automático se recalcula sin ese producto; el cajero sigue
  sin tener ninguna vía para compensarlo manualmente.
- **Un producto de la cuenta califica para más de una promoción activa a la vez**: se aplica el
  mecanismo de prioridad ya definido por el motor de promociones (spec 012); esta spec no cambia
  esa lógica, solo elimina la posibilidad de que el cajero la sustituya por un descuento manual.
- **Un pedido con pago confirmado pero cuyo ítem de cocina todavía está "pendiente" o "en
  preparación"**: la insignia "Listo" tampoco debe mostrarse en ese caso — hacen falta las dos
  condiciones (pagado y preparado), no basta con que ya esté pagado.
- **Una mesa con varios ítems donde algunos ya están "listo" en cocina pero el pago general de la
  orden todavía no se ha confirmado**: la insignia de la mesa no debe mostrar "Listo" mientras el
  pago siga pendiente, sin importar cuántos ítems individuales ya estén listos en cocina.
- **Se intenta reimprimir la factura de una orden cuya mesa ya fue cerrada/liberada**: la acción
  "Imprimir Factura" sigue disponible desde el historial de la venta aunque la mesa ya esté
  libre, porque reimprimir no depende del estado actual de la mesa, solo de que exista un
  documento de venta emitido.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Tras confirmar el pago de una orden (de origen QR o manual) y quedar facturada, el
  sistema DEBE ofrecer una única acción "Imprimir Factura", eliminando la duplicación actual
  entre "Reimprimir Factura POS" (spec 028, FR-007) y "Reimprimir Factura" (spec 028, FR-012).
- **FR-002**: La acción "Imprimir Factura" DEBE reimprimir el mismo documento de venta ya
  emitido, sin generar un nuevo registro de venta ni reprocesar el pago, sin límite de veces.
- **FR-003**: "Imprimir Pre-cuenta" DEBE seguir disponible como acción independiente antes de que
  el pago quede confirmado, sin verse afectada por la consolidación de FR-001.
- **FR-004**: "Imprimir Factura" NO DEBE ofrecerse para ninguna orden que no tenga un documento
  de venta emitido (cancelada, o con el pago rechazado sin reintento).
- **FR-005**: El sistema NO DEBE permitir anular (a nivel de ítem o de orden completa) ningún
  pedido que ya tenga el pago confirmado y facturado, sin importar su estado de preparación en
  cocina (pendiente, en preparación o listo).
- **FR-006**: Cuando se intente anular un pedido ya pagado, el sistema DEBE rechazar la acción e
  informar explícitamente que el pedido ya fue pagado y no puede anularse.
- **FR-007**: La acción "Anular" NO DEBE mostrarse como disponible en la interfaz para ningún
  ítem u orden que ya tenga el pago confirmado.
- **FR-008**: El bloqueo de anulación (FR-005 a FR-007) NO DEBE aplicar a pedidos sin pago
  confirmado — la anulación sobre pedidos sin pagar se sigue comportando exactamente como hoy.
- **FR-009**: El sistema NO DEBE ofrecer, en la Terminal de Mesas, ningún control para ingresar o
  aplicar un descuento manual sobre el total de una cuenta — incluyendo el atajo "Aplicar
  descuento (F4)" y su formulario asociado, que quedan eliminados. La prohibición es absoluta
  para todos los roles, sin excepción — ningún usuario, incluido uno con rol Administrador, tiene
  acceso a un control de descuento manual, ni en la Terminal de Mesas ni mediante ninguna vía
  alterna de esa pantalla.
- **FR-010**: El único descuento que el sistema DEBE reflejar sobre el total de una orden es el
  que resulte automáticamente del motor de promociones/combos vigente (spec 012, spec 013)
  cuando al menos un producto de la orden califique para una promoción activa.
- **FR-011**: Cuando ningún producto de la orden califique para una promoción activa, el sistema
  DEBE mostrar el descuento en $0, sin ofrecer ninguna vía para que el cajero lo modifique.
- **FR-012**: El sistema DEBE calcular la insignia "Listo" —tanto en el listado de mesas como en
  el detalle del pedido— exigiendo dos condiciones a la vez: que el pago de la orden ya esté
  confirmado, y que su(s) ítem(s) ya estén en estado "listo" en cocina. Ninguna de las dos
  condiciones por sí sola DEBE producir la insignia "Listo".
- **FR-013**: Mientras el pago de una orden no esté confirmado, el sistema NO DEBE mostrar la
  insignia "Listo" en el listado de mesas ni la leyenda "listo para cobrar" en el detalle del
  pedido, sin importar el estado de cocina de sus ítems — DEBE mostrar en su lugar un estado que
  refleje con precisión que el pedido está pendiente de cobro.
- **FR-014**: En cuanto se confirma el pago de una orden cuyos ítems ya estaban terminados en
  cocina, el sistema DEBE actualizar de inmediato la insignia de la mesa y el detalle del pedido
  para reflejar "Listo".

### Key Entities *(include if feature involves data)*

Esta spec no agrega entidades nuevas — reutiliza **Orden** (y su estado de pago/facturación),
**Ítem de Pedido** (con su estado de cocina `pendiente`/`en_preparación`/`listo`/`anulado`),
**Venta/Factura** y **Promoción** (spec 012), y la insignia de estado de **Mesa** ya definida en
spec 028 (FR-014). Introduce dos reglas nuevas sobre esas entidades ya existentes: (1) el estado
de pago de la Orden pasa a condicionar si un Ítem de Pedido puede anularse, y (2) la insignia
"Listo" de una Orden/Mesa pasa a depender simultáneamente del estado de pago de la Orden y del
estado de cocina de sus Ítems, en vez de depender solo de cocina como ocurre hoy.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las órdenes ya pagadas y facturadas muestran una sola acción de
  impresión de factura ("Imprimir Factura"), y el 0% de las pantallas post-pago muestran dos
  botones distintos para reimprimir el mismo documento.
- **SC-002**: El 100% de los intentos de anular un ítem u orden ya pagados son rechazados por el
  sistema, con un mensaje que explica por qué.
- **SC-003**: El 0% de las cuentas abiertas en la Terminal de Mesas ofrecen algún control visible
  para que el cajero introduzca un descuento manual.
- **SC-004**: El 100% de los descuentos que aparecen en el total de una cuenta corresponden a una
  promoción activa calculada automáticamente, verificable comparando el valor mostrado contra el
  motor de promociones, sin ninguna entrada manual del cajero.
- **SC-005**: El 0% de los pedidos sin pago confirmado muestran la insignia "Listo" (sin importar
  su estado de cocina), y el 100% de los pedidos con pago confirmado y cocina terminada muestran
  "Listo" de inmediato tras confirmarse el pago.

## Out of Scope

- Cambiar las condiciones bajo las cuales "Cerrar Mesa"/"Liberar Mesa" se acepta o se rechaza
  (spec 010, spec 028 FR-016), ni el mecanismo de transición de estados de cocina en sí mismo
  (spec 009) — esta spec solo agrega la verificación del estado de pago al momento de decidir qué
  insignia mostrar, sin tocar esos otros mecanismos.
- Cambiar el motor de cálculo de promociones/combos (spec 012, spec 013) — esta spec solo elimina
  la vía de descuento manual, no modifica cómo se calculan los descuentos por promoción.
- Cualquier vía de excepción o reversa administrativa para anular un pedido ya pagado (por
  ejemplo, un flujo de "reembolso" o "nota crédito" con permisos elevados) — si el negocio
  necesita una forma de revertir un pago ya confirmado, es una funcionalidad nueva e
  independiente, fuera del alcance de esta spec.
- Cambios al flujo de validación de pagos QR, al cobro manual en caja, o a cualquier otra regla
  de negocio ya definida en spec 024, spec 026 o spec 028 que no esté explícitamente listada en
  los Functional Requirements de esta spec.
- Registrar formalmente en `registro-de-anomalias.md` la corrección de `RN-ORD-39` como decisión
  de negocio — esta spec ya documenta esa decisión en su propio texto (ver "Naturaleza de esta
  spec"); si el proceso del proyecto exige además una entrada separada en ese registro, se
  gestiona como una tarea administrativa aparte, no como parte del comportamiento a implementar.

## Assumptions

- **"Pago confirmado y facturado" es el criterio único para bloquear la anulación** (Historia 1):
  no se distingue entre pago en efectivo, transferencia o datáfono, ni entre origen QR o manual —
  cualquier orden con un documento de venta ya emitido queda protegida contra anulación por igual.
- **El estado de cocina `listo` de un Ítem de Pedido (campo interno) y la insignia "Listo" que ve
  el personal (mesa/pedido) son conceptos distintos**: el campo interno de cocina sigue
  reflejando solo si ese ítem terminó de prepararse, sin cambios; lo que esta spec corrige es la
  insignia visible al personal, que pasa a exigir además el pago confirmado antes de mostrarse
  como "Listo". Un ítem puede seguir estando `listo` internamente en cocina mientras la insignia
  visible muestra un estado distinto de "Listo" porque el pago sigue pendiente.
- **Mientras la insignia "Listo" no aplica (pago pendiente), el estado que se muestra en su lugar
  no necesita un nombre nuevo definido por esta spec** — basta con que distinga con claridad que
  el cobro sigue pendiente (por ejemplo, reutilizando o adaptando el texto "pendiente de cobro" ya
  usado en otras pantallas); el nombre y estilo visual exactos de ese estado se resuelven en fase
  de planeación (`/speckit-plan`).
- **El aviso "No se puede cobrar esta mesa — su sesión está cerrada" (turno/caja cerrado)
  observado en la captura es un caso distinto** al defecto de esta spec: refleja que la sesión de
  caja del cajero está cerrada mientras queda un pedido sin cobrar — un bloqueo legítimo aparte,
  no la causa de que la insignia "Listo" se calculara mal. Esta spec no modifica las reglas de
  apertura/cierre de turno de caja (spec 006); si ese aviso necesita mejorarse, se atiende con una
  spec separada.
- **"Descuento manual" se refiere únicamente al monto o porcentaje que el cajero introduce por
  criterio propio** — no incluye ajustes de precio a nivel de catálogo (precios especiales,
  variantes) ni las promociones/combos configurados por administración, que siguen aplicándose
  con normalidad.
- **La consolidación de acciones de impresión (Historia 4) no elimina la posibilidad de imprimir
  más de una vez** — "Imprimir Factura" se puede usar todas las veces que se necesite; lo que se
  elimina es la duplicación de botones para la misma acción, no la posibilidad de reimprimir.
