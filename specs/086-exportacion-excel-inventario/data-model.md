# Data Model: Exportación de Inventario a Excel

**Feature**: `086-exportacion-excel-inventario` | **Fecha**: 2026-09-22

Esta funcionalidad no crea entidades de datos nuevas ni modifica el modelo existente
(`InventoryItem`, `UnitMeasure` en `pos-backend/app/models/`). Solo lee datos ya
persistidos para proyectarlos en un archivo `.xlsx` generado bajo demanda, que no se
almacena en ningún lado (spec, Key Entities: "no se persiste ni se guarda un historial de
exportaciones"). Este documento describe esa proyección, no un cambio de esquema.

## Entidad fuente: `InventoryItem` (sin cambios)

Definida en `pos-backend/app/models/inventory_item.py`, schema `tenant` (schema-per-tenant).

| Campo (modelo)     | Tipo             | Usado en el export como |
|---------------------|------------------|--------------------------|
| `name`               | `str`            | Columna "Nombre" (tal cual) |
| `type`               | `str` (enum: `raw_material`, `packaged`) | Columna "Tipo", traducido (ver `data-model.md` §Proyección) |
| `unit_measure_id`     | `UUID` (FK → `unit_measures.id`) | Resuelto por join a `UnitMeasure.abbreviation` para la columna "Unidad" |
| `current_stock`       | `Decimal(12,3)`  | Columna "Stock" |
| `min_stock`           | `Decimal(12,3)`  | Columna "Mínimo" |
| `unit_cost`           | `Decimal(12,2)`  | Columna "Costo" |
| `active`              | `bool`           | Entra en el cálculo de la columna "Estado" (ver abajo); NO se exporta como columna propia (Clarification 2 del spec: "sin agregar una columna nueva que los distinga") |

No hay campo persistido de "Estado" — se calcula al vuelo (ver Entidad derivada abajo).

## Entidad fuente: `UnitMeasure` (sin cambios)

Definida en `pos-backend/app/models/unit_measure.py`, schema `tenant`.

| Campo (modelo)   | Tipo   | Usado en el export como |
|-------------------|--------|--------------------------|
| `abbreviation`      | `str`  | Valor exportado en la columna "Unidad" (mismo valor que `unitAbbr()` muestra hoy en pantalla) |

## Entidad derivada (solo en tiempo de exportación, no persistida): `InventoryExportRow`

Fila proyectada para cada `InventoryItem` al momento de construir el `.xlsx`. Vive
únicamente en memoria durante la generación del archivo (helper nuevo en el backend, ver
`research.md` §5); no es un modelo SQLAlchemy ni una entidad de base de datos.

| Columna (encabezado exacto, FR-004) | Origen                                                                 | Tipo de celda en el `.xlsx` |
|---------------------------------------|-------------------------------------------------------------------------|-------------------------------|
| `Nombre`                                | `InventoryItem.name`                                                     | Texto |
| `Tipo`                                   | `InventoryItem.type` → `"Materia prima"` si `raw_material`, `"Empacado"` si `packaged` | Texto |
| `Unidad`                                 | `UnitMeasure.abbreviation` (join por `unit_measure_id`)                  | Texto |
| `Stock`                                  | `InventoryItem.current_stock`                                            | Numérico (sin truncar decimales) |
| `Mínimo`                                 | `InventoryItem.min_stock`                                                | Numérico (sin truncar decimales) |
| `Costo`                                  | `InventoryItem.unit_cost`                                                | Numérico (sin truncar decimales) |
| `Estado`                                 | `"Bajo mínimo"` si `active AND current_stock <= min_stock`, si no `"OK"` (idéntico a `isLow()` del frontend, `research.md` §4) | Texto |

## Reglas de validación / invariantes

- **Alcance de filas**: el conjunto de filas es siempre *todos* los `InventoryItem` del
  tenant autenticado (activos e inactivos), sin aplicar ningún filtro de búsqueda, tipo,
  estado activo/inactivo ni orden que el usuario tenga activo en pantalla (FR-003,
  Clarification 1). Orden de filas: por `name` ascendente (mismo orden que usa hoy
  `list_items_query()` por defecto, `service.py:34`), para que el archivo sea determinista
  y reproducible entre exportaciones consecutivas sin cambios de inventario.
- **Fila de encabezado**: siempre presente, incluso con cero insumos (fila 1, FR-008).
- **Sin filas de datos cuando el inventario está vacío**: 0 `InventoryItem` → 0 filas de
  datos, únicamente el encabezado (Edge case, SC-002 con `N=0`).
- **Consistencia de conteo**: número de filas de datos == número total de `InventoryItem`
  del tenant (activos + inactivos) en el momento de la exportación (SC-002).
- **Tipos de celda numérica**: `current_stock`, `min_stock`, `unit_cost` se escriben como
  valores numéricos nativos de la hoja de cálculo (`float`/`Decimal` soportado por
  `openpyxl`), nunca como texto — condición para que Excel permita autosuma (FR-006,
  Acceptance Scenario 2 y 4 de User Story 2).
- **Sin pérdida de decimales**: los valores numéricos se escriben con la misma precisión
  que tienen en base de datos (`Numeric(12,3)` para stock/mínimo, `Numeric(12,2)` para
  costo) — no se redondean ni truncan al proyectarlos a la celda.
- **Codificación**: nombres con tildes, eñes, comillas y otros caracteres del español se
  preservan sin corrupción — garantizado por el formato `.xlsx` (XML/UTF-8 por
  especificación OOXML), sin transformación adicional necesaria (FR-007).

## Relación con el modelo de datos existente

```text
InventoryItem (N) ──── unit_measure_id ────> (1) UnitMeasure
      │
      └── (proyección en memoria, no persistida) ──> InventoryExportRow ──> fila del .xlsx
```

No hay migraciones, ni nuevos campos, ni nuevas tablas. Principio VIII (Evolución del
Modelo de Datos) no aplica: N/A.
