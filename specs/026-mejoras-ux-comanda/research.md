# Research: Rediseño UX de Confirmación de Pago y Comanda en Terminal de Mesas (Skeilopos)

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — las 5 clarificaciones de
negocio ya se resolvieron en `spec.md` (sesiones `/speckit-specify` y `/speckit-clarify`,
2026-08-18) y el resto de incógnitas era puramente técnico, resuelto leyendo directamente
`pos-backend`/`pos-heladeria`. Este documento registra las decisiones de diseño y las alternativas
descartadas.

## Decisión 1 — Fusionar pago→cocina extrayendo `_confirm_order_impl` sin `commit` propio

- **Decisión**: extraer el cuerpo de `confirm_order` (`app/api/v1/orders/checkout.py:311-364`) —
  desde la carga con `with_for_update` hasta antes del `db.commit()` — a una función interna
  `_confirm_order_impl(db: Session, order_id: UUID, user: User) -> CustomerOrder` que hace todo lo
  mismo (precondición de intento confirmado, lock de la orden, `deduct_order_items`, `order.status
  = "abierta"`) pero **no** commitea ni rollea back — deja eso a quien la invoque. `confirm_order`
  queda como un wrapper delgado: llama `_confirm_order_impl` dentro de su propio
  `try/commit/except/rollback`, exactamente como hoy. `confirm_cash_payment_attempt` y
  `approve_payment_attempt` (`checkout.py:697-728`, `640-663`) llaman `_confirm_order_impl(db,
  attempt.order_id, user)` justo después de fijar `attempt.status = "confirmado"` y antes de su
  propio `db.commit()` — así, un solo `commit`/`rollback` cubre la actualización del intento de pago
  **y** el descuento de inventario + cambio de estado de la orden.
- **Rationale**: FR-002 exige que, si el descuento de inventario falla, "ninguna de las dos cosas
  ocurra" (ni pago confirmado, ni pedido en cocina) — la única forma de garantizar eso sin lógica de
  compensación manual es que ambas escrituras vivan en la misma transacción de base de datos.
  Orquestar esto desde el frontend (dos llamadas HTTP seguidas: confirmar pago, luego confirmar
  orden) se descartó explícitamente porque reintroduce la misma ventana de fallo parcial que originó
  esta spec (si la segunda llamada nunca llega — red cortada, pestaña cerrada — el pago queda
  confirmado sin que el pedido llegue a cocina, el defecto exacto reportado en la captura original).
  Extraer la lógica en vez de duplicarla evita repetir la validación de stock/lock/`deduct_order_items`
  en tres lugares.
- **Alternatives considered**: (a) que `confirm_cash_payment_attempt`/`approve_payment_attempt`
  llamen directamente a `confirm_order(db, order_id, user)` (la función pública, con su propio
  commit) justo antes de su propio commit — descartado: quedarían dos transacciones separadas
  dentro de la misma request; si la segunda falla después de que la primera ya comiteó, se vuelve a
  producir el estado ambiguo que FR-002 prohíbe. (b) Un endpoint nuevo que combine ambos pasos en un
  solo `POST` — descartado: cambia el contrato de los dos endpoints ya existentes sin necesidad,
  cuando extender su efecto secundario interno logra lo mismo con un diff menor (Principio VI).

## Decisión 2 — `POST /orders/{id}/confirm` se mantiene expuesto, sin cambios de contrato

- **Decisión**: el endpoint público de `confirm_order` no se elimina ni se oculta — sigue
  aceptando la misma request, devolviendo la misma respuesta, con la misma precondición (intento de
  pago confirmado). Lo único que cambia es que, en el flujo normal de la Terminal de Mesas, ya no
  hay ningún botón que lo invoque manualmente (ver Decisión 3 y FR-001) porque para cuando un
  intento queda "confirmado", el pedido ya fue enviado a cocina en la misma transacción del paso
  anterior — llamarlo de nuevo simplemente no tendría ningún intento pendiente que procesar de forma
  distinta (la orden ya no está en `recibida`).
- **Rationale**: mantenerlo intacto es el cambio de menor riesgo (Principio VI) — sigue disponible
  como vía de recuperación técnica (por ejemplo, si algún dato queda en un estado inconsistente por
  una causa ajena a esta spec) sin inventar un mecanismo de recuperación nuevo que el spec no pidió.
- **Alternatives considered**: retirar el endpoint o marcarlo deprecado — descartado, no lo pide el
  spec y elimina una vía de recuperación sin necesidad, además de obligar a tocar su propio
  characterization test por una razón distinta a la que esta spec autoriza.

## Decisión 3 — El frontend deja de llamar `confirmOrder()` manualmente; el botón "Confirmar" desaparece

- **Decisión**: en `pending-orders-panel.component.ts`, se elimina el botón "Confirmar" (líneas
  ~106-115), el método `confirm()` (línea ~203) y `isPaymentConfirmed()` (línea ~191) — ya no hay
  nada que ese botón necesite decidir, porque `confirmCashPaymentAttempt()`/`approvePaymentAttempt()`
  (invocados desde `payment-attempt-review-panel.component.ts`) ahora completan todo el ciclo en una
  sola llamada. El botón "Rechazar" (línea ~99, `reject()` → `cancelOrder(...)`) se conserva sin
  cambios — sigue siendo la forma en que el cajero cancela un pedido, incluida la salida clara que
  pide FR-002 si, tras un fallo de inventario, decide no reintentar sino cancelar.
- **Rationale**: es la traducción directa de FR-001 a la interfaz — si confirmar el pago ya hace
  todo el trabajo, dejar un botón "Confirmar" separado y ahora redundante reintroduciría la misma
  confusión que esta spec busca eliminar (un paso manual sin función clara).
- **Alternatives considered**: dejar el botón pero auto-deshabilitado permanentemente tras la fusión
  — descartado, es peor UX que quitarlo: un botón inerte es más confuso que uno ausente, y viola
  directamente el objetivo de la Historia 1.

## Decisión 4 — "De inmediato" (FR-004) se resuelve con la respuesta síncrona existente, sin SLA numérico nuevo

- **Decisión**: no se introduce un presupuesto de latencia explícito en la spec ni en este plan —
  `confirm-cash`/`approve` ya responden de forma síncrona dentro del mismo ciclo request/response
  HTTP que hoy dispara el toast de confirmación; mostrar el cambio "de inmediato" se resuelve
  actualizando la misma vista que ya se re-renderiza al recibir esa respuesta (`await this.load()` en
  `confirmCash()`), sin ninguna llamada adicional ni sondeo.
- **Rationale**: no había ninguna categoría de rendimiento pendiente que bloqueara el diseño — el
  único gap real era de presentación (Decisión 6), no de tiempo de respuesta del backend.
- **Alternatives considered**: fijar un SLA explícito (p. ej. "en menos de 2 segundos") — se dejó
  fuera por no aportar nada verificable adicional sobre lo que la arquitectura síncrona ya garantiza;
  quedó registrado como ítem de bajo impacto en el reporte de `/speckit-clarify`, no bloqueante.

## Decisión 5 — El fallo de inventario (FR-002) no necesita un flujo de recuperación nuevo

- **Decisión**: cuando `_confirm_order_impl` lanza una excepción por falta de stock (dentro de
  `confirm_cash_payment_attempt`/`approve_payment_attempt`), el `rollback` de la Decisión 1 revierte
  también el cambio de estado del intento de pago — el intento **no** queda "confirmado". Desde la
  interfaz, el cajero ve exactamente el mismo camino de error que ya existe hoy para cualquier fallo
  de este panel: `toast.error(this.api.extractError(err, ...))` (ya implementado en `confirmCash()`/
  `approvePaymentAttempt()` de `payment-attempt-review-panel.component.ts`), y el intento sigue
  visible como "pendiente de revisión" — el cajero puede volver a intentar el mismo clic tras
  resolver el problema de stock, o usar el botón "Rechazar" ya existente para cancelar el pedido.
- **Rationale**: la atomicidad de la Decisión 1 convierte automáticamente "ninguna de las dos cosas
  ocurre" (texto exacto de FR-002) en el comportamiento por defecto de un `rollback` — no hace falta
  diseñar un estado "error" nuevo ni un botón "reintentar" separado, porque la interfaz ya vuelve al
  mismo estado pendiente de antes del clic.
- **Alternatives considered**: un modal de error dedicado con un botón "Reintentar" explícito —
  descartado por ahora como sobre-construcción; el toast de error + el mismo formulario listo para
  reenviar ya cumple "una vía clara para resolver el problema" (texto de FR-002) sin agregar un
  componente nuevo. Si en pruebas de usuario (fase de tareas) el toast resulta insuficiente para una
  falla poco frecuente, es una mejora aislada y de bajo riesgo agregar después.

## Decisión 6 — El cambio en efectivo (FR-004/FR-005) es un defecto de presentación, no de datos

- **Decisión**: no se modifica ningún modelo, columna ni schema — `OrderPaymentAttempt.
  amount_received`/`change_amount` (`app/models/order_payment_attempt.py:41-42`) ya se calculan en
  `confirm_cash_payment_attempt` (`checkout.py:713-714`) y ya se serializan en
  `PaymentAttemptResponse` (`schemas.py:195-196`); el frontend (`PaymentAttempt`,
  `dining.interface.ts:144-158`) ya los tipa. El único cambio es de plantilla: el bloque `@if
  (last.status === 'confirmado')` de `payment-attempt-review-panel.component.ts` (línea ~111-112),
  que hoy solo imprime `"✓ Pago confirmado ({{ last.payment_method_name }})"`, se extiende para
  mostrar también `last.amount_received` y `last.change_amount` cuando el método es efectivo — de
  forma permanente, no solo en el toast de `confirmCash()` (línea ~200-202), que hoy es el único
  lugar donde el cambio aparece y desaparece en unos segundos.
- **Rationale**: es exactamente la causa raíz del defecto reportado por el usuario ("no se muestra
  el cambio cuando se confirma en efectivo") — el dato siempre existió, pero solo se mostraba de
  forma efímera.
- **Alternatives considered**: agregar `amount_received`/`change_amount` también a
  `CurrentPaymentAttemptSummary`/`OrderResponse.current_payment_attempt` (el DTO más liviano que usa
  la pestaña "Pedido de la mesa") para mostrar el cambio ahí también — se deja fuera de esta primera
  pasada: `payment-attempt-review-panel` (la vista de cajero, dentro de "Por confirmar") ya es,
  según su propio docstring, el paso obligado antes de que un pago quede confirmado, y sigue
  siendo accesible después — cumple FR-005 con el cambio de menor riesgo. Si la fase de tareas
  decide que el cambio también debe verse en "Pedido de la mesa", es una extensión aislada (dos
  campos nullable que ya existen en el modelo, solo faltaría serializarlos en un DTO distinto).

## Decisión 7 — División de cuenta y multi-pago (US3): auditoría + endurecimiento, no reconstrucción

- **Decisión**: `SessionBillPanelComponent`/`SplitBillPanelComponent`
  (`components/session-bill-panel.component.ts`, `split-bill-panel.component.ts`) ya implementan la
  mayoría de FR-006 a FR-009: detalle de cuenta, alternancia "Cuenta única"/"Dividir por comensal"
  (nunca porcentual — asignación por unidad vía `AssignRow.units[]`), y cobro combinando métodos de
  pago (`PaymentInputComponent` + `payment-draft.util.ts`), sobre `compute_bill`/`close_session`
  (spec 010) y `build_sale` (spec 011, que ya emite la factura en la misma transacción). La fase de
  tareas debe recorrer los 4 acceptance scenarios de la Historia 3 contra el comportamiento actual de
  estos componentes y construir solo lo que realmente falte — no reescribirlos.
- **Rationale**: el spec exige "que ya existen hoy a nivel de datos y reglas de negocio" (Key
  Entities) — reescribir un flujo que ya cumple la mayoría de los criterios de aceptación violaría
  Principio V (no refactorizar sin necesidad) y arriesgaría regresiones sobre
  `session-bill-panel.component.spec.ts`, que ya protege contra que el placeholder "Selecciona una
  mesa con consumo" vuelva a aparecer cuando no debería.
- **Alternatives considered**: reconstruir el panel de cuenta desde cero para "unificarlo" con el
  rediseño visual — descartado; el ajuste de tamaño de texto/controles (FR-010/FR-011, Decisión 8)
  no requiere tocar la lógica de reparto o cobro, solo sus clases Tailwind.

## Decisión 8 — Tamaños mínimos (FR-010/FR-011) se expresan en clases Tailwind ya usadas en el proyecto

- **Decisión**: el texto esencial (nombre de mesa, productos, total, estado) sube de `text-xs`/
  `text-sm` (12–14px, uso pervasivo hoy en los tres paneles) a un mínimo `text-base` (16px), con
  `text-lg`/`text-xl`/`font-bold` para total y estado de cada pedido; los controles de acción
  (confirmar pago, aprobar/rechazar comprobante, cobrar) suben a un objetivo táctil mínimo de
  `min-h-11 min-w-11` (44px — la escala nativa de Tailwind ya expresa 44px como `11`), ajustando el
  padding donde haga falta.
- **Rationale**: mantiene la convención de diseño ya establecida (Tailwind puro, sin librería de
  componentes) en vez de introducir un sistema nuevo — coherente con el supuesto ya documentado en
  `spec.md` (Assumptions) de quedarse dentro de esa convención.
- **Alternatives considered**: valores en CSS/px sueltos fuera de la escala Tailwind — descartado,
  rompe la consistencia con el resto del proyecto sin ninguna ventaja sobre usar la escala nativa.

## Decisión 9 — Distinción de estado sin depender del color (FR-003)

- **Decisión**: cada estado de un pedido (pago pendiente, pago rechazado, pago confirmado y en
  cocina) se presenta siempre con una etiqueta de texto corta o un ícono/emoji reconocible además de
  su color — reutilizando el vocabulario ya existente en el proyecto (✓, 🔔, 💳, badges de texto como
  "Pendiente de revisión" que el panel de revisión de pagos ya usa) en los tres paneles de la
  Terminal de Mesas, incluida la lista de mesas del panel izquierdo, donde la fase de tareas debe
  verificar puntualmente si hoy existe algún indicador que dependa solo del color.
- **Rationale**: es la respuesta directa a la clarificación resuelta en `/speckit-clarify` — bajo
  luz de cocina, en pantallas de baja calidad, o para personal con dificultad para distinguir
  colores, un estado que solo se diferencia por color puede seguir generando la misma confusión que
  motivó esta spec.
- **Alternatives considered**: depender de una paleta de colores con más contraste entre sí, sin
  etiqueta — descartado explícitamente en la sesión de clarificación por ser menos robusto ante
  condiciones de luz o percepción del color.
