# Contrato de UI/Store: Rediseño de Layout de la Terminal de Mesas

Esta spec no expone ni consume ninguna API HTTP nueva (no hay cambios de backend). El "contrato"
relevante aquí es el contrato interno entre los componentes de presentación (grilla de mesas, "Pagos
por confirmar", panel central, botón de sidebar) y `pos-terminal.store.ts` / `LayoutService`, para que
`/speckit-tasks` pueda descomponer el trabajo sin ambigüedad sobre quién expone qué.

## Contrato: `pos-terminal.store.ts` → `pos-tables-panel.component.ts` (grilla de mesas)

| Miembro | Dirección | Firma | Contrato |
|---|---|---|---|
| `filter` (ya existente) | store ⇄ componente | `signal<TableFilter>` (`'todas'\|'libres'\|'ocupadas'\|'pendientes'`) | Sin cambios — sigue siendo el filtro de ocupación (spec 028, FR-014) |
| `orderTypeTab` | store ⇄ componente | `signal<'mesas' \| 'domicilios' \| 'para-llevar'>` | Nuevo; el componente lee el valor activo para resaltar la pestaña y escribe uno nuevo al hacer clic |
| `setOrderTypeTab(tab)` | componente → store | `(tab: 'mesas' \| 'domicilios' \| 'para-llevar') => void` | Único punto de escritura de `orderTypeTab` (FR-001, FR-003) |
| `tablesView` (ya existente) | store → componente | `computed<TableRowViewModel[]>` | Sin cambios de forma; el componente lo renderiza solo cuando `orderTypeTab() === 'mesas'` — en las otras pestañas muestra el estado vacío (FR-003), sin volver a filtrar `tablesView()` |
| `selectTable(id)` (ya existente) | componente → store | `(id: string) => void` | Reutilizado tal cual al hacer clic en una tarjeta de mesa (spec 028); sin comportamiento nuevo |

## Contrato: `pos-terminal.store.ts` → sección "Pagos por confirmar" (componente nuevo)

| Miembro | Dirección | Firma | Contrato |
|---|---|---|---|
| `pendingOrders` (ya existente) | store (interno) | `computed<DiningOrder[]>` | Fuente de datos; no se consume directamente desde el componente nuevo, se consume vía `pendingPaymentsView` |
| `pendingPaymentsView` | store → componente | `computed<PendingPaymentViewModel[]>` | Nuevo; une `pendingOrders()` con `tables()` para exponer número/nombre de mesa. Vacío cuando `orderTypeTab() !== 'mesas'` (FR-004) |
| `confirmCashPayment(orderId, amount)` (ya existente, mismo método que usa `payment-attempt-review-panel.component.ts`) | componente → store | sin cambios de firma | El componente nuevo llama exactamente el mismo método — no se duplica la lógica de confirmación |
| `approveTransfer(orderId)` / `rejectTransfer(orderId)` (ya existentes) | componente → store | sin cambios de firma | Idéntico razonamiento — reutilizados tal cual |
| `selectTable(id)` (ya existente) | componente → store | `(id: string) => void` | Seleccionar una tarjeta de "Pagos por confirmar" abre esa orden en el centro/derecha igual que seleccionarla desde la grilla (FR-005) |

`PendingPaymentViewModel` (forma mínima que el componente puede asumir):

```ts
interface PendingPaymentViewModel {
  orderId: string;
  tableId: string;
  tableLabel: string;       // p. ej. "Mesa 2" — de tables()
  customerLabel: string;    // referencia de cliente si existe, si no el label de mesa
  paymentMethod: 'efectivo' | 'transferencia';
  reviewStatus: 'pendiente_revision' | 'aprobado';
  totalLabel: string;       // reutilizado tal cual del cálculo ya existente
  createdAt: string;
}
```

## Contrato: `pos-terminal.store.ts` → panel central (lista de ítems + "+ Agregar producto" embebido)

| Miembro | Dirección | Firma | Contrato |
|---|---|---|---|
| `centralState` (ya existente) | store → componente raíz | `computed<'validar-pago' \| 'mesa-libre' \| 'pedido'>` | Sin cambios — sigue determinando qué vista central se renderiza (spec 028) |
| `cartView` (ya existente) | store → `pos-order-panel.component.ts` | `computed<CartLineViewModel[]>` | Sin cambios de forma; sigue siendo la lista de ítems del pedido (FR-006) |
| `catalogOpen` (ya existente) | store ⇄ componente | `signal<boolean>` | Sin cambios de semántica; deja de disparar un overlay de pantalla completa y pasa a alternar, dentro del mismo panel central, entre la lista de ítems y la grilla de "+ Agregar producto" (FR-006, FR-007) |
| `openCatalog()` (ya existente) | componente → store | `() => void` | Invocado por el botón "+ Agregar producto"; sin cambios de contrato |
| `catalogCategoryId` (ya existente) | store ⇄ componente | `signal<string \| null>` | Sin cambios de contrato |
| `setCatalogCategory(id)` (ya existente) | componente → store | `(id: string \| null) => void` | Sin cambios de contrato |
| `catalogSearchText` | store ⇄ componente | `signal<string>` | Nuevo; el componente actualiza en cada tecla del input de búsqueda (FR-007) |
| `setCatalogSearchText(text)` | componente → store | `(text: string) => void` | Único punto de escritura |
| `catalogProductsFiltered` | store → componente | `computed<CatalogProductViewModel[]>` | Nuevo; intersección categoría + texto sobre el `catalogProducts` ya existente; lista vacía cuando no hay coincidencias (el componente renderiza el estado vacío, no decide el filtrado) |
| `addToOrder(productId, ...)` (ya existente) | componente → store | sin cambios | Reutilizado tal cual desde la grilla embebida; al agregar, el panel central vuelve a la lista de ítems (`catalogOpen` pasa a `false`) |

## Contrato: panel derecho (`pos-checkout-panel.component.ts`) — sin cambios de comportamiento

| Miembro | Dirección | Firma | Contrato |
|---|---|---|---|
| `billingCustomerName` (ya existente) | store ⇄ componente | `signal<string>` | Sin cambios — alimenta el campo "Facturar a nombre de" (FR-009) |
| `checkoutAndSend` (ya existente) | componente → store | sin cambios | Botón combinado "Cobrar, Facturar y Enviar a Cocina" para pedidos aún no enviados a cocina; sin cambios de cuándo se activa (FR-009) |
| `<app-split-bill-panel>` / `<app-session-bill-panel>` (ya existentes) | — | — | Sin cambios de comportamiento (FR-010); esta spec solo puede ajustar su disposición visual dentro del panel derecho, nunca su lógica interna |

## Contrato: botón de sidebar (Terminal de Mesas) → `LayoutService`

| Miembro | Dirección | Firma | Contrato |
|---|---|---|---|
| `layoutService.sidebarOpen()` (ya existente) | servicio → componente | `Signal<boolean>` | El botón lee este valor para decidir su ícono/estado (abierto/cerrado) |
| `layoutService.toggle()` (ya existente) | componente → servicio | `() => void` | Único método que el nuevo botón invoca; no se agrega ningún método nuevo a `LayoutService` (FR-012) |

**Nota de compatibilidad**: hoy `toggle()` ya existe pero, en escritorio, `sidebar.component.ts` ignora
`sidebarOpen()` (ver research.md §4). El contrato de `LayoutService` no cambia; lo que cambia es que
`sidebar.component.ts`/`dashboard-layout.component.ts` empiezan a *honrar* ese contrato también en
escritorio.
