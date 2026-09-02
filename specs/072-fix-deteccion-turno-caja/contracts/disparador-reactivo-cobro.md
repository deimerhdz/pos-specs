# Contrato: disparador del chequeo de cobro dentro de `reloadOrders()` (FR-005 a FR-007)

**Normativo.** Extiende `PosTerminalStore.reloadOrders()`
(`pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts:1074-1081`), el único punto
por el que pasan la carga inicial, el sondeo y los eventos en tiempo real de pedidos.

---

## 1. Dónde y cuándo se dispara

```typescript
private async reloadOrders(): Promise<void> {
  const orders = await this.api.listOrders(undefined, true);
  this.orders.set(orders);
  this.announcePending(orders);
  if (this.hasChargeableOrderNow()) {          // NUEVO
    void this.ensureCheckoutDataLoaded();
  }
}
```

`hasChargeableOrderNow()` (nuevo método privado, extraído para no duplicar la condición) es
verdadero cuando:
- `pendingOrders().length > 0` (algún pedido `QR_MENU`/`recibida` en cualquier mesa — el universo
  del panel "Pagos por confirmar"), **o**
- `selectedTableId()` no es null y `ordersOfTable(selectedTableId())` no está vacío, **o**
- `selectedOrderId()` no es null (un pedido de Domicilio/Para llevar seleccionado vía
  `selectStandaloneOrder()`).

Es la **misma** condición, extraída a un solo lugar, que ya usan `selectTable()`
(línea ~1153, `if (list.length > 0)`) y `selectStandaloneOrder()` (línea ~1182) para decidir si
llaman `ensureCheckoutDataLoaded()` — no se inventa un umbral nuevo, se reutiliza el que ya definió
la spec 059 FR-002.

## 2. Por qué NO es un `effect()`

Ver research.md D1. Enganchar dentro de `reloadOrders()` (llamada a método, no reactividad de
signal) significa que **no** se dispara cuando un test hace `store.orders.set([...])` directamente
— solo cuando el store mismo decide recargar pedidos desde el backend. Los 35 usos directos de
`orders.set(...)` en `pos-terminal.store.spec.ts` quedan intactos.

## 3. Tabla de casos

| Estado antes de `reloadOrders()` | Qué trae la recarga | `hasChargeableOrderNow()` | ¿Dispara `ensureCheckoutDataLoaded()`? |
|---|---|---|---|
| Mesa vacía seleccionada, sin pedidos en ningún lado | Nada nuevo | `false` | No — preserva spec 059 FR-001. |
| Mesa vacía seleccionada | Llega el primer pedido QR de esa mesa | `true` (`pendingOrders`) | **Sí — corrige US1.** |
| Nada seleccionado | Llega un pedido de Domicilio/Para llevar | `true` (`pendingOrders`, si es QR) o depende del flujo de creación (si es manual, no es `pendingOrders` pero si el cajero lo selecciona luego, `selectStandaloneOrder()` ya lo cubre igual que hoy) | Ver nota abajo. |
| Ya se resolvió el turno en esta sesión de pantalla (`cash.shift()` no nulo) | Cualquier cosa | (no importa) | El guard interno de `ensureCheckoutDataLoaded()` ya evita repetir la petición (spec 059 FR-003) — llamarla de más no genera tráfico extra. |
| Turno cerrado o ambiguo (`cash.shift()` sigue en `null`) y sigue habiendo algo pendiente | Cualquier recarga posterior | `true` | Se reintenta el descubrimiento en cada ciclo — ver research.md D4 (satisface FR-005 sin caché especial). |

**Nota sobre Domicilio/Para llevar creado manualmente**: un pedido de mostrador
(`hold_for_payment`) no entra en `pendingOrders()` (esa lista es solo `QR_MENU`); su cobro directo
ya pasa por `selectStandaloneOrder()`, que ya llama `ensureCheckoutDataLoaded()` de forma
imperativa desde antes de esta corrección — sin cambio ahí, y sin necesitar el disparador reactivo
porque ese flujo siempre involucra una selección activa.

## 4. Margen de tiempo (Clarifications, sesión 2026-09-02)

`ensureCheckoutDataLoaded()` puede resolverse hasta ~2 segundos después de que
`reloadOrders()` termina de actualizar `orders()` y el panel de cobro ya se pintó con el estado
previo — no hace falta que el turno de caja ya esté cargado **antes** de que el panel aparezca.
