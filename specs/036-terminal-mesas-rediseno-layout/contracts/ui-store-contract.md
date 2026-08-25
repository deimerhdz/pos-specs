# Contrato de UI/Store: Rediseño de Layout de la Terminal de Mesas

Esta spec no expone ni consume ninguna API HTTP nueva (no hay cambios de backend). El "contrato"
relevante aquí es el contrato interno entre los componentes de presentación (franja de órdenes, menú
central, botón de sidebar) y `pos-terminal.store.ts` / `LayoutService`, para que `/speckit-tasks` pueda
descomponer el trabajo sin ambigüedad sobre quién expone qué.

## Contrato: `pos-terminal.store.ts` → `pos-tables-panel.component.ts` (franja de "Órdenes Recientes")

| Miembro | Dirección | Firma | Contrato |
|---|---|---|---|
| `ordersFilter` | store → componente (lectura), componente → store (escritura) | `signal<'todas' \| 'domicilios' \| 'mesas'>` | El componente lee el valor activo para resaltar la pestaña; escribe un nuevo valor al hacer clic en una pestaña |
| `setOrdersFilter(filter)` | componente → store | `(filter: 'todas' \| 'domicilios' \| 'mesas') => void` | Único punto de escritura de `ordersFilter`; reinicia el scroll del carrusel como efecto colateral documentado (no responsabilidad del componente) |
| `recentOrdersView` | store → componente | `computed<OrderCardViewModel[]>` | Ya filtrado según `ordersFilter`; el componente NO vuelve a filtrar, solo renderiza |
| `selectOrder(orderOrTableId)` | componente → store | `(id: string) => void` | Reutiliza exactamente la selección ya implementada al hacer clic en una mesa (spec 028); sin comportamiento nuevo |

`OrderCardViewModel` (forma mínima que el componente puede asumir, derivada de lo ya calculado por
`tablesView()`):

```ts
interface OrderCardViewModel {
  id: string;
  titleLabel: string;      // p. ej. "Mesa 4" o el identificador de la orden
  chipClass: string;       // reutilizado tal cual de STATUS_META
  statusLabel: string;     // reutilizado tal cual de STATUS_META
  elapsedLabel: string;    // reutilizado tal cual (texto, p. ej. "12 min")
  elapsedRatio: number;    // 0..1, nuevo — alimenta el ancho de la barra visual
}
```

## Contrato: `pos-terminal.store.ts` → panel central embebido (menú)

| Miembro | Dirección | Firma | Contrato |
|---|---|---|---|
| `catalogCategoryId` (ya existente) | store ⇄ componente | `signal<string \| null>` | Sin cambios de contrato |
| `setCatalogCategory(id)` (ya existente) | componente → store | `(id: string \| null) => void` | Sin cambios de contrato |
| `catalogSearchText` | store ⇄ componente | `signal<string>` | Nuevo; el componente actualiza en cada tecla del input de búsqueda |
| `setCatalogSearchText(text)` | componente → store | `(text: string) => void` | Único punto de escritura |
| `catalogProductsFiltered` | store → componente | `computed<CatalogProductViewModel[]>` | Intersección categoría + texto; lista vacía cuando no hay coincidencias (el componente renderiza el estado vacío, no decide el filtrado) |
| `addToOrder(productId, ...)` (ya existente) | componente → store | sin cambios | Reutilizado tal cual desde el grid embebido |

## Contrato: botón de sidebar (Terminal de Mesas) → `LayoutService`

| Miembro | Dirección | Firma | Contrato |
|---|---|---|---|
| `layoutService.sidebarOpen()` (ya existente) | servicio → componente | `Signal<boolean>` | El botón lee este valor para decidir su ícono/estado (abierto/cerrado) |
| `layoutService.toggle()` (ya existente) | componente → servicio | `() => void` | Único método que el nuevo botón invoca; no se agrega ningún método nuevo a `LayoutService` |

**Nota de compatibilidad**: hoy `toggle()` ya existe pero, en escritorio, `sidebar.component.ts` ignora
`sidebarOpen()` (ver research.md §3). El contrato de `LayoutService` no cambia; lo que cambia es que
`sidebar.component.ts` empieza a *honrar* ese contrato también en escritorio.
