# Phase 1 Contracts: Carga diferida y tarjetas de pedido de Domicilio/Para Llevar

Este feature no expone ni consume ningún endpoint HTTP nuevo (Out of Scope del spec) — no hay
contratos REST que documentar. Los "contratos" aquí son las interfaces observables dentro de la
SPA Angular que otros componentes/tests dependerán: el componente de tarjeta reutilizable y la
superficie pública de `PosTerminalStore` que cambia de comportamiento. `tasks.md` (siguiente
comando) debe generar tareas verificables contra cada contrato de esta página.

## Contrato 1: `OrderSummaryCardComponent` (nuevo)

Componente presentacional standalone, sin dependencias de `PosTerminalStore` ni de ningún
servicio HTTP — recibe todo por `@Input()`, nunca decide de dónde viene el dato (mesa o pedido sin
mesa es indistinguible para este componente, ver research.md §6).

```typescript
@Component({ selector: 'app-order-summary-card', standalone: true, ... })
export class OrderSummaryCardComponent {
  @Input({ required: true }) title!: string;          // "Mesa 3" | "Domicilio" | "Para llevar"
  @Input({ required: true }) statusLabel!: string;     // reutiliza STATUS_META[...].label
  @Input({ required: true }) statusClass!: string;     // reutiliza STATUS_META[...].chip
  @Input({ required: true }) secondaryLabel!: string;  // "N productos" | nombre de cliente
  @Input({ required: true }) elapsedLabel!: string;     // "🕐 12 min"
  @Input({ required: true }) totalLabel!: string;       // ya formateado ($ colombianos)
  @Input() ordersCount?: number;                        // solo mesas con >1 pedido activo
  @Input() selected = false;

  @Output() select = new EventEmitter<void>();          // click en la tarjeta completa
}
```

**Garantías del contrato**:
- No dispara ningún efecto secundario ni petición HTTP por sí mismo — es puramente presentacional
  (testeable sin `TestBed` de servicios/HTTP, solo con inputs/outputs).
- `select` se emite exactamente una vez por click, independientemente del tipo de origen del dato.
- El markup/CSS de la tarjeta es una única fuente (no hay una versión "para mesas" y otra "para
  pedidos") — cualquier cambio visual futuro se hace en un solo lugar (cumple FR-005 del spec:
  "mismo formato visual").

**Consumidores**: `pos-tables-panel.component.ts`, una vez para cada tarjeta de `tablesView()` y
una vez para cada tarjeta de `store.ordersByType(store.orderTypeTab())` (cuando la pestaña activa
sea `'domicilios'` o `'para-llevar'`).

## Contrato 2: Superficie pública de `PosTerminalStore` (extendida)

Signals/computed/métodos nuevos o con contrato de comportamiento cambiado, consumidos por
componentes fuera del store (contrato interno del módulo `tables/`, no expuesto fuera de él):

```typescript
class PosTerminalStore {
  // ── Nuevo: selección de pedido sin mesa ──────────────────────────────────
  /** Selecciona un pedido de Domicilio/Para llevar (sin mesa asociada).
   *  Postcondición: selectedTableId() === null, selectedOrderId() === orderId. */
  selectStandaloneOrder(orderId: string): void;

  /** true si hay una mesa O un pedido sin mesa seleccionado — reemplaza a
   *  `hasActiveOrder` como condición de la que depende `pos-order-panel`. */
  readonly hasActiveSelection: Signal<boolean>;

  // ── Nuevo: pedidos de Domicilio/Para llevar pendientes de cobro ──────────
  /** Pedidos `order_type === 'DELIVERY' | 'TAKEAWAY'`, no pagados, no
   *  cancelados, mapeados a OrderSummaryCardView (ver data-model.md). */
  ordersByType(tab: 'domicilios' | 'para-llevar'): Signal<OrderSummaryCardView[]>;

  // ── Cambiado: disparo de datos de cobro ──────────────────────────────────
  /** Precondición NUEVA: solo hace la petición si paymentMethodService.methods()
   *  y checkoutOptions() siguen vacíos — invocado ahora desde selectTable()
   *  (cuando la mesa tiene pedido) y selectStandaloneOrder(), NO desde init(). */
  private ensureCheckoutDataLoaded(): Promise<void>;
}
```

**Garantías del contrato**:
- `hasActiveSelection() === true` si y solo si `pos-order-panel.component.ts` debe mostrar el
  detalle de un pedido (mesa o sin mesa) en vez de su placeholder — ningún componente debe volver a
  leer `selectedTableId()` directamente para decidir "¿hay algo que mostrar?" (ese es exactamente
  el bug que este feature corrige, spec Historia 3).
- `selectStandaloneOrder(orderId)` es idempotente respecto a la carga diferida: si ya se llamó una
  vez en la sesión de la app, seleccionar otro pedido (con o sin mesa) no repite la petición
  (`FR-003`) — mismo criterio de caché que ya usa hoy `init()` para la carga inicial
  (`methods().length === 0 ? load() : null`).
- Seleccionar una mesa **libre** (`selectTable(id)` sin pedidos) nunca invoca
  `ensureCheckoutDataLoaded()` (`FR-001`).

**Consumidores**: `pos-tables-panel.component.ts` (dispara `selectStandaloneOrder` desde el
`select` de una tarjeta de pedido), `pos-order-panel.component.ts` (lee `hasActiveSelection` en vez
de `hasActiveOrder`), `pos-checkout-panel.component.ts` (sin cambios de contrato — ya funciona
sobre `selectedOrder()`, que ahora también puede resolver a un pedido sin mesa).

## Contrato 3: Efecto de carga diferida (comportamiento observable, no una API)

Documentado como contrato porque `tasks.md` debe poder generar un test verificable sobre él:

| Evento | Efecto esperado |
|---|---|
| `store.init()` completa | `paymentMethodService.methods()` y `checkoutOptions()` siguen vacíos; `cashService.shift()` sigue `null` (a menos que ya estuviera cargado desde otra pantalla, servicios `providedIn: 'root'`). |
| `store.selectTable(tableLibreId)` | Sin cambios en los tres signals anteriores. |
| `store.selectTable(tableConPedidoId)` **o** `store.selectStandaloneOrder(orderId)`, primera vez en la sesión de la app | `paymentMethodService.load()` + `.loadAvailableForCheckout()` + `cashService.restoreShift()` (si aplica) se invocan exactamente una vez cada una. |
| Segunda selección de pedido en la misma sesión de app | Cero peticiones adicionales a esos tres. |

Este contrato es el que hace pasar/fallar `pos-terminal.store.spec.ts` (tarea de tests, siguiente
fase) — reemplaza la aserción implícita de hoy ("todo se carga en `init()`") por una explícita
sobre el momento del disparo.
