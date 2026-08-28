# Contrato de UI/Store: Eliminación de Dividir Cuenta y de Combinar Método de Pago

Esta spec no expone ni consume ninguna API HTTP nueva ni modificada (research.md §4). El "contrato"
relevante aquí es el contrato interno entre `pos-checkout-panel.component.ts`,
`session-bill-panel.component.ts`, `payment-input.component.ts` y `pos-terminal.store.ts`, para que
`/speckit-tasks` pueda descomponer el trabajo sin ambigüedad sobre quién expone qué.

## Contrato: `pos-terminal.store.ts` → `pos-checkout-panel.component.ts` ("Liberar Mesa")

| Miembro | Dirección | Firma | Contrato |
|---|---|---|---|
| `centralState` (ya existente, sin cambios) | store → componente | `computed<'validar-pago' \| 'mesa-libre' \| 'pedido'>` | El componente lee este valor para decidir si renderiza el bloque de "Liberar Mesa"; `'validar-pago'` = hay al menos un pago pendiente en la mesa seleccionada (FR-001) |
| `sessionBill` (ya existente, sin cambios) | store → componente | `signal<SessionBill \| null>` | Condición ya existente para mostrar el bloque de acciones post-cobro; se combina con `centralState()`, no la reemplaza |
| `releaseTable()` (ya existente, sin cambios internos) | componente → store | `() => Promise<void>` | Solo se invoca cuando el botón está renderizado (`centralState() !== 'validar-pago'`); su lógica interna no cambia (FR-002) |

**Nuevo contrato del template**: el bloque `@if (store.sessionBill(); as bill) { … 🔓 Liberar Mesa
… }` (`pos-checkout-panel.component.ts:208-235`) envuelve el botón en una condición adicional
`store.centralState() !== 'validar-pago'`. Ningún método ni signal nuevo se agrega al store.

## Contrato: `pos-checkout-panel.component.ts` — "Dividir la cuenta" (se retira)

| Miembro (hoy) | Estado tras esta spec |
|---|---|
| `import { SplitBillPanelComponent } from './split-bill-panel.component'` | Eliminado |
| Botón "Dividir la cuenta entre varias personas" (dos ocurrencias) | Eliminado |
| `<app-split-bill-panel …>` | Eliminado del template |
| `splitOpen = signal(false)` | Eliminado |
| `sessionOrders(bill)`, `tableLabel()`, `onSplitSaved(bill)` | Eliminados (sin llamador tras retirar `<app-split-bill-panel>`) |

No hay contrato de reemplazo — la spec no introduce ninguna funcionalidad nueva en este punto
(FR-005/FR-006).

## Contrato: `session-bill-panel.component.ts` — "Dividir por comensal" (se retira)

| Miembro (hoy) | Estado tras esta spec |
|---|---|
| `mode = signal<BillingMode>('unified')` | Eliminado — el componente se comporta siempre como `'unified'` |
| Toggle "Cuenta única" / "Dividir por comensal" (template) | Eliminado |
| `splits = signal<SplitDraft[]>([])`, `SplitDraft` (interfaz local) | Eliminados |
| `canSplit()` | Eliminado |
| `setSplitPayment()` | Eliminado |
| `buildPayload()` | Simplificado — retira la rama `billing_mode: 'split'`; siempre construye `{ cash_shift_id, billing_mode: 'unified', payments, customer_name? }` |
| `ready()` | Simplificado — solo evalúa `paymentIssue(this.unifiedPayment(), …)`, sin la rama `splits().every(...)` |
| Desglose de solo lectura por comensal (`bill.split`, líneas 70-100) | **Sin cambios** — no es la acción de "dividir", es información (spec 026 FR-006) |

`@Input bill: SessionBill | null`, `@Input methods`, `@Input cashShiftId`, `@Input customerName`,
`@Input orphan`, `@Input readOnly`, `@Input beforeCharge`, `@Output charged` — **sin cambios de
firma**; el componente sigue siendo consumido igual desde `pos-checkout-panel.component.ts` (modo
`resumen` y `showSessionCharge()`).

## Contrato: `payment-input.component.ts` / `payment-draft.util.ts` — "Combinar método de pago" (se retira)

| Miembro (hoy) | Estado tras esta spec |
|---|---|
| `PaymentDraft.combined` / `.secondMethodId` / `.secondAmount` | Campos eliminados de la interfaz |
| Checkbox "Combinar con otro método" (template) | Eliminado |
| Bloque de segundo método (`<select>` + `<app-money-input>`) | Eliminado |
| `toggleCombined()`, `setSecondMethod()`, `setSecondAmount()`, `remainder()` | Eliminados |
| `paymentLines(draft)` | Simplificado — siempre devuelve `[{ payment_method_id: draft.methodId, amount: draft.amount }]` |
| `paidAmount(draft)` | Simplificado — siempre `draft.amount` |
| `nonCashAmount(draft, methods)` | Simplificado — evalúa solo `draft.methodId`/`draft.amount` |
| `changeDue(draft, total)`, `missingAmount(draft, total)` | **Sin cambios de fórmula** — ya operan sobre `paidAmount()`, que se simplifica por debajo sin cambiar su forma |
| `paymentIssue(draft, total, methods)` | Se retira la validación de segundo método (líneas 77-80); conserva "falta cubrir el total" y "no-efectivo no puede superar el total" |

`@Input total`, `@Input methods`, `@Output changed = EventEmitter<PaymentDraft>()` — **sin cambios
de firma**; ambos consumidores (`pos-checkout-panel.component.ts`, `session-bill-panel.component.ts`
en su único modo `unified`) siguen usando `<app-payment-input>` igual que hoy.

## Contrato: `payment-attempt-review-panel.component.ts` — sin cambios (FR-003 ya cumplido)

| Miembro | Contrato |
|---|---|
| `@Input order`, `@Input cashShiftId`, `@Output resolved` | Sin cambios |
| `confirmCash(attempt)`, `approve(attempt)`, `reject(attempt)`, `rejectOrder(order)` | Sin cambios — ninguno ofrece ni ofreció combinar método de pago (research.md §3) |

No requiere ningún cambio de contrato; se documenta aquí solo para dejar explícito que FR-003 no
demanda trabajo en este componente.
