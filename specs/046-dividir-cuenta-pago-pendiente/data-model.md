# Data Model: Eliminación de Dividir Cuenta y de Combinar Método de Pago

Esta spec **no agrega ni modifica entidades de backend** (spec.md, Key Entities). No hay migración
de base de datos, ni columna, ni endpoint nuevo o eliminado (research.md §4). Este documento
describe únicamente el estado de **frontend** que cambia — sigue siendo estado de presentación, no
un modelo de datos persistente — y dónde termina cada capacidad hoy modelada por tipos existentes.

## Entidades de dominio reutilizadas (sin cambios)

| Entidad | Origen | Uso en esta spec |
|---|---|---|
| `DiningOrder` / `pendingOrders()` / `pendingOfSelectedTable()` | `dining.interface.ts`, `pos-terminal.store.ts` | Fuente de `centralState()`, reutilizado tal cual para condicionar "Liberar Mesa" (FR-001/FR-002) — cero campo nuevo |
| `SessionBill` / `SessionBillLine.participant_id` | `dining.interface.ts` | El desglose de solo lectura por comensal (`session-bill-panel.component.ts`, líneas 70-100) sigue poblándose igual; no se toca |
| `PaymentMethodCheckoutOption` | `sales.interface.ts` | Sin cambios — sigue siendo la lista de métodos que ofrece el `<select>` de `payment-input.component.ts` |

## Estado de presentación que se elimina (frontend, sin equivalente de reemplazo)

| Elemento | Ubicación hoy | Regla tras esta spec |
|---|---|---|
| `PaymentDraft.combined` / `.secondMethodId` / `.secondAmount` | `payment-draft.util.ts:12-19` | Campos retirados de la interfaz — un `PaymentDraft` solo puede describir un método (FR-003/FR-004/FR-007) |
| `PosCheckoutPanelComponent.splitOpen` | `pos-checkout-panel.component.ts:253` | Signal retirado — deja de existir el estado "panel de reparto abierto" (FR-005/FR-006) |
| `SessionBillPanelComponent.mode` (`'unified' \| 'split'`) | `session-bill-panel.component.ts:222` | Signal retirado — el cobro de "Cuenta de la mesa" ya no tiene modo, siempre es equivalente a `'unified'` (FR-005) |
| `SessionBillPanelComponent.splits` (`SplitDraft[]`) | `session-bill-panel.component.ts:225` | Signal retirado junto con `mode` |
| `SessionBillPanelComponent.canSplit()` | `session-bill-panel.component.ts:246` | Computed retirado — ya no hay alternativa de cobro que habilitar/deshabilitar |

## Estado nuevo de presentación

Ninguno. La condición de FR-001/FR-002 se resuelve reutilizando el `computed` `centralState()` ya
existente (`pos-terminal.store.ts:432-441`) — ver research.md §1. No se agrega ningún
signal/computed nuevo al store ni a ningún componente.

## Entidades/esquemas de backend que quedan sin llamador desde estos dos flujos (no se eliminan)

| Elemento | Ubicación | Motivo por el que permanece intacto |
|---|---|---|
| `POST/DELETE /table-sessions/{id}/participants` | `pos-backend/app/api/v1/table_sessions/router.py:60,78` | `session_participant` sostiene el flujo de pedidos QR por comensal (spec 007), no solo "Dividir la cuenta" — eliminarlo rompería esa funcionalidad viva (research.md §2/§4) |
| `PUT /table-sessions/{id}/assignments` | `pos-backend/app/api/v1/table_sessions/router.py:93` | Mismo motivo — respalda también asignaciones ya persistidas de sesiones históricas |
| `SplitPaymentIn` / `payments: list[PaymentIn]` (sin `max_length`) | `app/api/v1/table_sessions/schemas.py:107-117`, `app/api/v1/orders/schemas.py:262,286` | Esquema compartido por **todos** los flujos de cobro (mostrador, cierre de sesión); no es exclusivo de "combinar método de pago" (research.md §4) |

## Transiciones de estado relevantes

- **`centralState()`**: sin cambios de transición — ya prioriza `'validar-pago'` sobre cualquier
  otro estado (`pos-terminal.store.ts:432-441`). Lo nuevo es que `pos-checkout-panel.component.ts`
  ahora **lee** esta transición para decidir si el botón "Liberar Mesa" existe en el DOM, algo que
  hoy no hace.
- **`sessionBill()`**: sin cambios — sigue siendo la condición que decide si el bloque de acciones
  post-cobro ("Imprimir Factura", "Liberar Mesa") se muestra en absoluto; FR-001/FR-002 se
  superponen a esa condición existente, no la reemplazan.
- **`PaymentDraft` (simplificado)**: `emptyPaymentDraft() → { methodId: '', amount: 0 }` (sin
  campos de combinación) ⇄ `{ methodId, amount }` tras elegir método — una única transición, sin el
  estado intermedio "combinado" que existía hoy.
