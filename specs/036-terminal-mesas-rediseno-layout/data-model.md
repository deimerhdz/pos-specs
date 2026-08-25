# Data Model: Rediseño de Layout de la Terminal de Mesas

Esta spec **no agrega ni modifica entidades de backend** (spec.md, Key Entities: "no se le agregan
atributos nuevos en esta spec"). No hay migración de base de datos, ni columna, ni endpoint nuevo. Este
documento describe únicamente el estado derivado que se agrega en el **store de frontend**
(`pos-terminal.store.ts`, signals) para soportar el nuevo layout — sigue siendo estado de presentación,
no un modelo de datos persistente.

## Entidades de dominio reutilizadas (sin cambios)

| Entidad | Origen | Uso en esta spec |
|---|---|---|
| `DiningOrder` (incluye `created_at`) | `dining.interface.ts` | Fuente del tiempo transcurrido de cada tarjeta de la franja superior (ya usada hoy por `elapsedLabel`) |
| `Table` / `TableDisplayStatus` | `table.interface.ts`, `pos-terminal.store.ts` | Fuente del badge de estado ("Por confirmar"/"En preparación"/"Libre") que se conserva sin cambios |
| Catálogo (`categories()`, productos) | `pos-terminal.store.ts` (vía `menuService`) | Fuente del grid de productos embebido; sin cambios en su forma |

## Estado nuevo de presentación (frontend, `pos-terminal.store.ts`)

| Signal / Computed | Tipo | Descripción | Regla |
|---|---|---|---|
| `ordersFilter` | `signal<'todas' \| 'domicilios' \| 'mesas'>` | Pestaña activa de la franja de "Órdenes Recientes" (FR-003) | Por defecto `'todas'`; cambiar de valor reinicia el desplazamiento del carrusel (FR-002) |
| `recentOrdersView` | `computed` | Lista de tarjetas a mostrar en la franja superior, derivada de `tablesView()` existente + `ordersFilter` | Cuando `ordersFilter === 'domicilios'`, DEBE resolver siempre a una lista vacía (no existe hoy ninguna orden de ese tipo — spec.md, Out of Scope) |
| `catalogSearchText` | `signal<string>` | Texto del buscador por nombre del menú central (FR-006) | Cadena vacía por defecto; no persiste entre sesiones |
| `catalogProductsFiltered` | `computed` | Extiende el `catalogProducts` existente combinando `catalogCategoryId` (ya existente) + `catalogSearchText` | Coincidencia por nombre insensible a mayúsculas/acentos; intersección con la categoría activa (FR-006, edge case de la spec) |
| `ordersScrollAtStart` / `ordersScrollAtEnd` | `signal<boolean>` (o derivados del `(scroll)` listener) | Habilitan/deshabilitan las flechas del carrusel (FR-002) | Se recalculan en cada evento de scroll del contenedor de tarjetas |

## Estado nuevo de presentación (frontend, módulo `dashboard/layout`)

| Elemento | Descripción | Regla |
|---|---|---|
| `layoutService.sidebarOpen()` (ya existente, sin cambios de API) | Controla si el sidebar global está expandido | El nuevo botón de la Terminal de Mesas llama a `layoutService.toggle()`; ver research.md §3 para el cambio necesario en `sidebar.component.ts` para que también tenga efecto en escritorio |

## Transiciones de estado relevantes

- **`ordersFilter`**: `'todas' ⇄ 'domicilios' ⇄ 'mesas'`, cualquier transición reinicia
  `ordersScrollAtStart = true` (vuelve al inicio de la franja — FR-002, edge case correspondiente).
- **`catalogSearchText` / `catalogCategoryId`**: independientes entre sí, se combinan por intersección;
  ninguno tiene efecto sobre el precio, disponibilidad o stock de los productos (FR-007).
- **`recentOrdersView` para `'domicilios'`**: no tiene transición a un estado "con datos" en esta spec —
  permanece vacío hasta que una spec futura agregue la capacidad de crear órdenes de ese tipo (ver
  spec.md, Clarifications y Out of Scope).
