# Data Model: Mejora de Visibilidad del Buscador y Filtros en Inventario

**N/A — esta spec no introduce, modifica ni elimina ninguna entidad, campo, relación ni
regla de negocio.**

Es un ajuste exclusivamente de presentación (clases CSS y un ícono decorativo) sobre
controles de UI que ya existen y cuyo comportamiento no cambia (FR-005, FR-007):

- El campo de búsqueda por nombre sigue enlazado al mismo estado
  (`InventoryPageComponent.searchSignal` → `InventoryService` vía `onSearchInput`).
- El selector de tipo de insumo sigue enlazado a `InventoryService.itemsType()` /
  `onTypeFilterChange` (valores `''`, `raw_material`, `packaged` — sin cambios, ver
  `InventoryItemType` en `inventory.interface.ts`).
- El selector de estado sigue enlazado a `InventoryService.itemsActive()` /
  `onActiveFilterChange` (valores `''`, `active`, `inactive` — sin cambios).
- No se toca `InventoryItem`, `InventoryMovement`, `Purchase` ni ninguna otra interfaz de
  `inventory.interface.ts`.
- No hay migración de datos, ni en `pos-backend` ni en `pos-heladeria` (Assumptions del
  spec: "No se requiere ningún cambio ... en el modelo de datos").

Ver [research.md](./research.md) para las decisiones de tratamiento visual y
[quickstart.md](./quickstart.md) para la validación de que el comportamiento de estos
mismos bindings permanece idéntico tras el cambio.
