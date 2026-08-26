# Data Model: Rediseño de Layout de la Terminal de Mesas

Esta spec **no agrega ni modifica entidades de backend** (spec.md, Key Entities: "no se le agregan
atributos nuevos en esta spec"). No hay migración de base de datos, ni columna, ni endpoint nuevo. Este
documento describe únicamente el estado derivado que se agrega en el **store de frontend**
(`pos-terminal.store.ts`, signals) para soportar el layout actualizado — sigue siendo estado de
presentación, no un modelo de datos persistente.

## Entidades de dominio reutilizadas (sin cambios)

| Entidad | Origen | Uso en esta spec |
|---|---|---|
| `DiningOrder` (incluye `created_at`, `status`, `channel`) | `dining.interface.ts` | Fuente de `pendingOrders()` (ya existente) para la sección "Pagos por confirmar" y del `elapsedLabel` ya usado en cada tarjeta de mesa |
| `Table` / `TableDisplayStatus` | `table.interface.ts`, `pos-terminal.store.ts` | Fuente del badge de estado ("Por confirmar"/"En preparación"/"Libre") que se conserva sin cambios; se une con `pendingOrders()` para mostrar número/nombre de mesa en "Pagos por confirmar" |
| Catálogo (`categories()`, productos) | `pos-terminal.store.ts` (vía `menuService`) | Fuente del grid de productos accesible desde "+ Agregar producto"; sin cambios en su forma |
| Asignación de ítems por comensal (`participant_id`) | spec 010, `split-bill-panel.component.ts` / `session-bill-panel.component.ts` | Reutilizada sin cambios — "Dividir la cuenta entre varias personas" ya está implementado; esta spec solo reubica el botón visualmente (ver Clarifications, sesión durante `/speckit-plan`) |

## Estado nuevo de presentación (frontend, `pos-terminal.store.ts`)

| Signal / Computed | Tipo | Descripción | Regla |
|---|---|---|---|
| `orderTypeTab` | `signal<'mesas' \| 'domicilios' \| 'para-llevar'>` | Pestaña de tipo de orden activa, independiente del `filter` de ocupación ya existente (FR-001, FR-003) | Por defecto `'mesas'`; cuando no es `'mesas'`, tanto la grilla de mesas como `pendingPaymentsView` DEBEN resolver a lista vacía (no existe hoy ninguna orden de esos tipos) |
| `filter` (ya existente) | `signal<TableFilter>` | Filtro de ocupación "Todas/Libres/Ocupadas/Pendientes" (spec 028, FR-014) | Sin cambios; independiente de `orderTypeTab` |
| `pendingPaymentsView` | `computed` | Extiende el `pendingOrders()` ya existente uniendo cada orden con su mesa (`tables()`) para exponer número/nombre, método de pago, estado y total — alimenta la sección "Pagos por confirmar" (FR-004) | Vacío cuando `orderTypeTab() !== 'mesas'`; confirmar/rechazar desde esta vista llama exactamente a los mismos métodos del store que ya usa `payment-attempt-review-panel.component.ts`, sin lógica duplicada |
| `catalogSearchText` | `signal<string>` | Texto del buscador por nombre dentro de la grilla de "+ Agregar producto" (FR-007) | Cadena vacía por defecto; no persiste entre sesiones; se limpia al cerrar la grilla y volver a la lista de ítems |
| `catalogProductsFiltered` | `computed` | Extiende el `catalogProducts` existente combinando `catalogCategoryId` (ya existente) + `catalogSearchText` | Coincidencia por nombre insensible a mayúsculas/acentos; intersección con la categoría activa (FR-007, edge case de la spec) |
| `catalogOpen` (ya existente) | `signal<boolean>` | Controla si el panel central muestra la grilla de "+ Agregar producto" en vez de la lista de ítems del pedido | Sin cambios de semántica; cambia su presentación de overlay de pantalla completa a embebido dentro del panel central (FR-006, FR-007) |

## Estado nuevo de presentación (frontend, módulo `dashboard/layout`)

| Elemento | Descripción | Regla |
|---|---|---|
| `layoutService.sidebarOpen()` (ya existente, sin cambios de API) | Controla si el sidebar global está expandido | El nuevo botón de la Terminal de Mesas llama a `layoutService.toggle()`; ver research.md §4 para el cambio necesario en `sidebar.component.ts`/`dashboard-layout.component.ts` para que también tenga efecto en escritorio (hoy solo controla el slide-over móvil) |

## Transiciones de estado relevantes

- **`orderTypeTab`**: `'mesas' ⇄ 'domicilios' ⇄ 'para-llevar'` — solo `'mesas'` tiene datos reales;
  las otras dos permanecen vacías hasta que una spec futura agregue la capacidad de crear órdenes de
  esos tipos (ver spec.md, Clarifications y Out of Scope). No hay transición a un estado "con datos"
  para `'domicilios'`/`'para-llevar'` en esta spec.
- **`pendingPaymentsView`**: se recalcula automáticamente cuando cambia `pendingOrders()` (ya reactivo)
  o `tables()`; confirmar un pago desde esta vista o desde el panel de la mesa seleccionada actualiza
  la misma fuente de verdad (`orders()`), por lo que ambas vistas quedan sincronizadas sin lógica
  adicional de invalidación.
- **`catalogSearchText` / `catalogCategoryId`**: independientes entre sí, se combinan por intersección;
  ninguno tiene efecto sobre el precio, disponibilidad o stock de los productos (FR-008).
- **`catalogOpen`**: `false` (lista de ítems del pedido) ⇄ `true` (grilla de "+ Agregar producto");
  seleccionar un producto desde la grilla vuelve a `false` automáticamente (FR-007).
