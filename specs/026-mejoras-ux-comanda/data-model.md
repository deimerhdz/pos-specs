# Data Model: Rediseño UX de Confirmación de Pago y Comanda en Terminal de Mesas (Skeilopos)

Esta spec **no agrega ni modifica ninguna tabla, columna ni migración** — es comportamiento
(cuándo se dispara una operación que ya existe) y presentación (qué datos que ya existen se
muestran y cómo). Este documento describe las entidades ya existentes que la funcionalidad toca,
sin repetir su definición completa (ver spec 024/010/011/016 para el modelo completo).

## Entidades reutilizadas

### Orden (`CustomerOrder`, `app/models/customer_order.py`)

- Sin cambios de esquema. `status` mantiene su `CHECK` ya protegido
  (`recibida|abierta|bloqueada|pagada|cancelada`) — esta spec no agrega ningún valor nuevo.
- **Cambio de comportamiento** (no de esquema): la transición `recibida → abierta` (hoy solo
  disparada por una llamada manual a `confirm_order`) pasa a dispararse automáticamente, dentro de
  la misma transacción, cuando su Intento de Pago vigente queda `confirmado` (ver research.md,
  Decisión 1). No existe ningún estado nuevo "pago confirmado, pendiente de enviar a cocina" que
  persistir — antes de esta spec ese estado tampoco se persistía (se derivaba de "intento
  confirmado + orden todavía en `recibida`"); después de esta spec, esa combinación deja de ser
  observable porque ambas escrituras ocurren juntas.

### Intento de Pago (`OrderPaymentAttempt`, `app/models/order_payment_attempt.py`)

- Sin cambios de esquema. Los campos que esta spec necesita **ya existen**:
  - `amount_received: Numeric(12,2) | None` (línea 41)
  - `change_amount: Numeric(12,2) | None` (línea 42)
  - `status: "pendiente" | "confirmado" | "rechazado"`
- Ambos campos ya se calculan en `confirm_cash_payment_attempt` (`checkout.py:713-714`) y ya se
  serializan en `PaymentAttemptResponse` (`schemas.py:195-196`) y en el tipo frontend `PaymentAttempt`
  (`dining.interface.ts:144-158`). El gap que corrige FR-004/FR-005 es exclusivamente de plantilla
  (`payment-attempt-review-panel.component.ts`), no de datos — ver research.md, Decisión 6.
- **Cambio de comportamiento**: cuando `status` pasa a `"confirmado"` (efectivo o comprobante
  aprobado), esa misma transacción ahora también avanza la Orden asociada (ver arriba) — antes, ese
  cambio de estado del Intento de Pago no tenía ningún efecto sobre la Orden en la misma llamada.

### Cuenta de la Mesa / Sesión de Mesa (`TableSession`, spec 010/016)

- Sin cambios de esquema ni de cálculo. `compute_bill`/`close_session`
  (`app/api/v1/table_sessions/service.py`) se reutilizan sin modificar — la spec solo cambia cómo
  se presentan sus resultados en el panel "Cuenta de la mesa" (`SessionBillPanelComponent`/
  `SplitBillPanelComponent`) y verifica que ya cumplan los acceptance scenarios de la Historia 3
  (research.md, Decisión 7).
- El reparto por ítem/unidad (`AssignRow.units[]`, frontend) sigue siendo la única forma de
  dividir la cuenta — esta spec no introduce ni habilita ninguna división porcentual.

### Venta / Pago / Factura (`Sale`, `Payment`, `SaleInvoice`, spec 011)

- Sin cambios de esquema ni de lógica. `build_sale` (`app/api/v1/sales/builder.py`) ya acepta una
  lista de pagos combinando métodos y ya emite la factura en la misma transacción — esta spec solo
  verifica que el panel "Cuenta de la mesa" lo exponga con claridad (FR-008/FR-009), sin tocar el
  motor de cálculo.

## Sin entidades nuevas

Confirmado por el propio `spec.md` (sección "Key Entities"): esta funcionalidad reutiliza
exclusivamente **Orden**, **Intento de Pago**, **Comprobante**, **Método de Pago** (spec 024) y
**Sesión de Mesa / Cuenta de la Mesa**, **Venta**, **Pago**, **Factura** (spec 010/011/016). No hay
migraciones que planear, ni estrategia de rollback de esquema que definir (Principio VIII no
aplica).
