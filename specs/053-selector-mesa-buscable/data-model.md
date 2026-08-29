# Data Model: Selector de mesa buscable en la creación de orden manual

Sin entidades ni campos de backend nuevos ni modificados (spec.md, Key Entities; Principio VIII:
N/A). El único cambio de "forma de datos" es de frontend, y es puramente de UI:

## Tipo de UI extendido

- **`SearchableSelectOption`** (`pos-heladeria/src/app/shared/searchable-select/
  searchable-select.component.ts`) — gana un campo opcional nuevo: `disabled?: boolean`. Sigue
  representando una opción genérica de cualquier select buscable de la app (insumos, grupos de
  opciones, etc.), no algo específico de mesas. Retrocompatible: los 4 consumidores existentes no
  lo pasan, se comporta igual que antes para ellos (research.md D2).

## Mapeo derivado (sin persistencia)

- `manual-order-page.component.ts` agrega un `computed` (`mesaOptions`) que mapea
  `store.tablesView()` (ya existente, sin cambios) a `SearchableSelectOption[]`: `id: t.id`,
  `label` combinando nombre + estado (research.md D3), `disabled` con la misma condición booleana
  que ya usaba el botón de la rejilla (`t.statusLabel !== 'Libre' && t.id !==
  store.selectedTableId()`). No se agrega ningún signal ni estado nuevo al store — es una
  transformación de vista, igual que `catalogProductsFiltered` u `ordersView`.
