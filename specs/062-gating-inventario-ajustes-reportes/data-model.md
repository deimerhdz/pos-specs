# Fase 1 — Data Model: Ocultamiento de Unidades de Medida y Reportes de Inventario sin el Módulo Habilitado

Ver decisiones técnicas en [research.md](./research.md). Esta spec **no agrega, modifica ni
elimina ninguna entidad, columna o migración**. Se documenta aquí, en vez de omitir el archivo,
para dejar explícito qué entidades existentes gobiernan el comportamiento nuevo y por qué ninguna
necesita cambiar de forma (Constitución, Principio VIII: toda spec que toque el modelo de datos
lo especifica explícitamente — esta spec deja constancia de que no lo hace).

## Entidades reutilizadas (sin cambios), definidas en spec 033

### `Plan` (`shared.plans`)

Columna relevante para esta spec: `inventario_access` (`Boolean`). Las tres superficies nuevas
(Unidades de Medida, insumos con stock bajo, Margen) leen esta misma columna, a través del mismo
mecanismo (`require_module_access("inventario")` / `PlanSummaryService.summary().modules.inventario`)
que ya usan Inventario y Promociones. Ver `data-model.md` de spec 033 para su forma completa.

### Asignación de Plan por Tenant (`shared.tenants.plan_id` / `plan_vence_en`)

Columna relevante: `plan_vence_en` (vía `ensure_plan_not_expired`, ya invocada internamente por
`require_module_access`). Un tenant vencido pierde acceso a las tres superficies nuevas por el
mismo camino que ya pierde acceso a Inventario/Promociones — no hay una segunda comprobación de
vencimiento específica de esta spec.

### `PlanSummary` / `ModuleAccess` (frontend, `plan-summary.interface.ts`)

Interfaz ya existente, sin cambios: `ModuleAccess.inventario` (`boolean`) y `PlanSummary.vencido`
(`boolean`) son los dos campos que toda la lógica de gating nueva de esta spec consulta —
`SettingsPageComponent`, `ReportsService` y el guard ya existente (`planModuleGuard`) los leen de
la misma forma que `SidebarComponent` (spec anterior, misma sesión de trabajo).

## Entidades de dominio referenciadas, no modificadas

Estas entidades ya existen y ya son 100% propiedad del módulo Inventario (research.md, Decisión
1) — esta spec no cambia ninguna de sus columnas, solo confirma que ninguna otra pantalla debería
poder leerlas sin el módulo habilitado:

- **`UnitMeasure`** (`unit_measure.py`): `name`, `abbreviation`, `active`. Referenciada
  exclusivamente por `InventoryItem`.
- **`InventoryItem`**: incluye `unit_cost`, usado por `_variant_unit_cost_map` para calcular
  `cogs`/`margin` en `profitability_report`.
- **`RecipeItem`**: `product_variant_id`, `inventory_item_id`, `quantity` — el vínculo entre una
  variante de producto y el costo de sus insumos.

## Sin migraciones

No hay ninguna migración Alembic en esta spec. No hay backfill, no hay `NOT NULL` nuevo, no hay
estrategia de rollback que documentar — el único cambio de código es agregar comprobaciones de
acceso (`Depends(require_module_access("inventario"))`) delante de operaciones que ya existían
tal cual.
