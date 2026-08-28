# Research: Eliminación de Dividir Cuenta y de Combinar Método de Pago en Toda la Aplicación

Todos los puntos de la Technical Context del plan quedaron resueltos por investigación directa del
código de `pos-heladeria` y `pos-backend` — no había ambigüedades de negocio pendientes (resueltas
en spec.md, Clarifications, tres rondas el 2026-08-28). Este documento registra las decisiones
técnicas necesarias para pasar de la spec a un diseño concreto.

## 1. Condicionar "Liberar Mesa" a que no haya pago pendiente (FR-001/FR-002)

- **Decisión**: reutilizar el `computed` `centralState()` ya existente en `pos-terminal.store.ts`
  (líneas 432-441), que ya resuelve `'validar-pago'` cuando
  `pendingOfSelectedTable().length > 0` (líneas 411-414, filtra `pendingOrders()` por
  `dining_table_id === selectedTableId()`). El botón "🔓 Liberar Mesa" de
  `pos-checkout-panel.component.ts` (líneas 227-233), hoy gateado únicamente por
  `@if (store.sessionBill(); as bill)` (línea 208) y `[disabled]="store.submitting()"`, gana una
  condición adicional: `@if (store.centralState() !== 'validar-pago')` envolviendo ese botón (o el
  bloque completo de acciones post-cobro, a decidir en `/speckit-tasks` según legibilidad del
  template).
- **Justificación**: no existe hoy ningún signal/computed llamado `hasPendingPayment`,
  `canReleaseTable` o similar (confirmado por grep) — pero `centralState()` ya calcula exactamente
  la condición que la spec pide ("la mesa seleccionada tiene al menos un pago QR pendiente de
  aprobar/confirmar"), y ya se usa para decidir qué muestra el panel central. Reutilizarlo evita
  duplicar lógica o crear un segundo signal que puede desincronizarse del primero.
- **Alternativas consideradas**: agregar un computed nuevo `hasPendingPaymentForSelectedTable =
  computed(() => this.pendingOfSelectedTable().length > 0)` — rechazada por redundante:
  `centralState() !== 'validar-pago'` expresa la misma condición con cero código nuevo, y ya está
  cubierto por los tests existentes de `pos-terminal.store.spec.ts` sobre `pendingOfSelectedTable`/
  `centralState`.
- **Nota de reaparición inmediata (FR-002/SC-003)**: `centralState()` es un `computed` reactivo
  sobre `orders()` — al confirmar/aprobar un pago pendiente, `store.reload()` (ya invocado por
  `payment-validation-block.component.ts` vía su `(refresh)` output) actualiza `orders()`, lo que
  recalcula `pendingOfSelectedTable()` → `centralState()` → el `@if` del botón, sin ningún código
  adicional. Mismo mecanismo reactivo que ya corrigió spec 044 FR-006 (el pedido confirmado se ve
  de inmediato).

## 2. Eliminación completa de "Dividir la cuenta entre varias personas" (FR-005/FR-006)

- **Decisión**: eliminar `split-bill-panel.component.ts` y su `.spec.ts` por completo. En
  `pos-checkout-panel.component.ts`, retirar: el import de `SplitBillPanelComponent` (línea 13),
  los dos botones trigger "Dividir la cuenta entre varias personas" (líneas 98-104 dentro de
  `showSessionCharge()`, y 137-143 dentro de la rama de mostrador editable), el bloque de render
  `@if (splitOpen() && store.sessionBill(); as bill) { <app-split-bill-panel …> }` (líneas 238-247),
  el signal `splitOpen` (línea 253), y los métodos que solo alimentaban ese componente:
  `sessionOrders(bill)` (321-324), `tableLabel()` (327-330) y `onSplitSaved(bill)` (333-336) — los
  tres quedan sin ningún otro llamador tras retirar `<app-split-bill-panel>` del template.
  En `session-bill-panel.component.ts`, retirar el toggle "Cuenta única"/"Dividir por comensal"
  (líneas 116-140), el signal `mode` (línea 222, se fija implícitamente a `'unified'` porque deja de
  existir la alternativa), `canSplit()` (línea 246), la rama `mode() === 'split'` de `ready()`
  (253-261) y de la plantilla de pago (156-173), la actualización de `splits` en `ngOnChanges()`
  (283-290) y `setSplitPayment()` (299-303), y la rama `split` de `buildPayload()` (346-351) — el
  cierre de sesión pasa a construir siempre el payload `billing_mode: 'unified'`.
- **Lo que NO se toca**: el desglose de solo lectura por comensal (líneas 70-100 de
  `session-bill-panel.component.ts`, iterando `bill.split`) — es información (qué pidió cada
  comensal, spec 026 FR-006), no una acción de "dividir"; sigue poblándose igual que hoy, con datos
  que pueden originarse tanto en pedidos QR individuales por comensal (spec 007, sin relación con
  esta spec) como en pedidos sin comensal asignado ("Sin asignar (mesero)"). Tampoco se toca el
  backend: `POST/DELETE /table-sessions/{id}/participants` y `PUT /table-sessions/{id}/assignments`
  (`pos-backend/app/api/v1/table_sessions/router.py:60,78,93`) quedan sin llamador desde este flujo
  manual, pero el modelo `session_participant` (`app/models/session_participant.py`) sigue siendo
  usado extensamente por el flujo de pedidos QR por comensal (`cart/service.py`,
  `orders/consolidation.py`, `qr_context.py`, `customer_order.py` — confirmado por grep) — eliminarlo
  rompería esa funcionalidad, completamente distinta y fuera de alcance (ver §4).
- **Justificación**: cumple FR-006 (eliminar el punto de entrada y el código que ya no tiene
  ningún llamador) sin arrastrar el modelo de datos compartido de "participante", que sostiene una
  funcionalidad viva no relacionada (Principio VII/VI).
- **Alternativas consideradas**: dejar `split-bill-panel.component.ts` en el árbol pero sin ningún
  botón que lo abra — rechazada explícitamente por FR-006 ("el código que la implementaba… DEBE
  eliminarse del código fuente — no permanece como superficie viva alternativa").

## 3. Eliminación completa de "Combinar método de pago" (FR-003/FR-004/FR-007)

- **Decisión — confirmación clave**: `payment-attempt-review-panel.component.ts` (bloque de
  confirmación de pago pendiente, spec 044) **nunca tuvo** una opción de combinar método de pago —
  el efectivo se confirma con un único campo `amountReceived` (líneas 51-90) y la transferencia se
  aprueba/rechaza sin selector de método (líneas 91-148). Además, el backend
  (`confirm_cash_payment_attempt`, `pos-backend/app/api/v1/orders/checkout.py:942-970`) ya rechaza
  con `422` cualquier `amount_received < total` (comentario propio del código: "FR-010a: impide
  confirmar si `amount_received < total_orden`", de una spec previa a esta). **FR-003 queda
  satisfecho sin ningún cambio de código** — se agrega únicamente un test de regresión que fije
  este comportamiento (research §6), ya que hoy no hay ningún test que lo cubra explícitamente en
  el frontend.
- **Decisión — retiro real**: el mecanismo de "combinar" vive en `payment-draft.util.ts`
  (`PaymentDraft.combined`/`secondMethodId`/`secondAmount`, líneas 12-19; `paymentLines()`,
  26-34; `nonCashAmount()`, 42-50; validaciones de segundo método en `paymentIssue()`, 77-80) y en
  su UI, `payment-input.component.ts` (checkbox "Combinar con otro método", líneas 58-66; bloque de
  segundo método, 69-89; métodos `toggleCombined()`/`setSecondMethod()`/`setSecondAmount()`/
  `remainder()`, 146-164). Ambos se simplifican para un único método: `PaymentDraft` pierde los tres
  campos de combinación, `paymentLines()` siempre devuelve una única línea,
  `paymentIssue()`/`missingAmount()` conservan la regla de "falta cubrir el total" (ya exige monto
  ≥ total con un único método, líneas 82-83) y la regla de "lo no-efectivo no puede superar el
  total" (85-87), ambas sin cambios de fondo — solo se retira la rama de segundo método.
- **Dos puntos de invocación, un único cambio**: `payment-input.component.ts` lo usan tanto
  `pos-checkout-panel.component.ts` (cobro de mostrador, línea 173-177) como
  `session-bill-panel.component.ts` (cobro de "Cuenta de la mesa", spec 026 FR-008, líneas 150-155 y
  156-173) — retirar la opción en el componente compartido cubre ambos call sites con un único
  cambio (amend spec 026 FR-008, spec.md FR-004).
- **Justificación**: `setMethod()` (`payment-input.component.ts`, línea 134) ya precarga
  `amount: this.total` al elegir método — el cajero, para efectivo, puede seguir escribiendo un
  monto mayor y ver el "Vuelto" (`changeDue()`, sin cambios) — spec.md FR-007, edge case "Pago en
  efectivo con excedente" queda cubierto sin código adicional.
- **Alternativas consideradas**: deshabilitar el checkbox en vez de eliminarlo — rechazada por
  FR-007 ("el código que la implementaba… DEBE eliminarse del código fuente").

## 4. Alcance de backend: qué NO se toca, y por qué

- **Decisión**: no se modifica ningún endpoint, schema ni modelo de `pos-backend` en esta spec.
  `payments: list[PaymentIn]` (con `min_length=1`, nunca `max_length`) es el esquema compartido de
  **todos** los flujos de cobro del sistema — `PayIn`/`CheckoutAndSendIn`
  (`app/api/v1/orders/schemas.py:262,286`), `SplitPaymentIn`/`CloseSessionIn`
  (`app/api/v1/table_sessions/schemas.py:117,128`) — no es exclusivo de "combinar método de pago";
  restringirlo a `max_length=1` backend-side tocaría la venta de mostrador (spec 011, foundational,
  fuera de alcance de esta spec) y cualquier otro flujo de cobro futuro. De igual forma,
  `session_participant` sostiene el flujo de pedidos QR por comensal (§2), no solo el botón que se
  elimina aquí.
- **Justificación**: Principio VI (evolución incremental — no mezclar un cambio de contrato de API
  compartido dentro de un retiro de UI) y Principio VII (compatibilidad con datos históricos — ventas
  ya cerradas con reparto por comensal o con métodos combinados siguen siendo válidas y legibles;
  nada en esta spec las recalcula ni cambia su representación). El frontend, tras esta spec, nunca
  vuelve a enviar más de un `PaymentLine` ni a llamar a `assignments`/`participants` manualmente —
  eso ya satisface el resultado observable exigido por FR-004/FR-005/SC-002/SC-004, sin tocar un
  contrato compartido.
- **Alternativas consideradas**: agregar validación backend `max_length=1` sobre `payments` en los
  cuatro schemas — rechazada por mezclar dos clases de cambio (retiro de UI + endurecimiento de un
  contrato de API ya en producción y compartido) en un mismo incremento; si el negocio quiere ese
  refuerzo server-side, es una spec futura independiente y más acotada, no parte de esta.

## 5. Cobertura de tests

- **Decisión**: eliminar `split-bill-panel.component.spec.ts` junto con el componente. Ajustar
  `pos-checkout-panel.component.spec.ts`: el test `T035` (línea 176, "Liberar Mesa" pide la
  liberación…) sigue válido para el caso sin pago pendiente; se agrega un caso nuevo con
  `store.centralState()` en `'validar-pago'` que confirme que el botón no se renderiza; se retiran
  o actualizan las aserciones que buscan el texto "Dividir la cuenta entre varias personas" (línea
  394 y las del bloque `showSessionCharge()`). Ajustar `payment-draft.util.spec.ts` retirando los 9
  casos ligados a `combined`/segundo método, dejando solo los de un único método (falta de monto,
  no-efectivo por encima del total). Ajustar `session-bill-panel.component.spec.ts` retirando el
  caso "cobra con dos métodos y reparte el resto en el segundo" (línea 242) y cualquier caso del
  modo `split`; conservar los de modo `unified` y el desglose de solo lectura. Agregar un caso nuevo
  en `payment-attempt-review-panel.component.spec.ts` (o en un characterization test de
  `pos-backend`, a decidir en tasks) que fije explícitamente que un `amountReceived < total` no
  puede confirmarse — el hallazgo del §3 documentado como test, no solo como comentario.
  `payment-validation-block.component.spec.ts` y `pos-terminal.store.spec.ts` no requieren cambios
  (no tocan ninguno de los dos flujos eliminados).
- **Justificación**: Principio X (verificación obligatoria) exige cobertura de la condición nueva
  (FR-001/FR-002) y de la ausencia verificable de ambas opciones eliminadas (FR-004/FR-005/FR-007);
  no tocar la intención de `payment-attempt-review-panel.component.spec.ts` ni de los tests de
  backend preserva la garantía de que la validación de pago QR (specs 024/028/044) no cambió.
