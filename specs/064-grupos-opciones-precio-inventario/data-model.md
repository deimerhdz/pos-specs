# Data Model: Tipo de precio e inventario condicional en grupos de opciones

Las decisiones de diseño detrás de cada elección están en [research.md](./research.md); este
documento se limita a columnas, restricciones, validaciones cruzadas y la consulta de migración.

## OptionGroup (`option_groups`, schema `tenant`) — MODIFICADA

Columna nueva:

| Columna | Tipo | Nulable | Default (ORM/BD) | Notas |
|---|---|---|---|---|
| `pricing_type` | `String(20)` | No | `server_default="con_recargo"` (research.md Decisión 1) | `CheckConstraint("pricing_type IN ('incluido', 'con_recargo')", name="ck_option_group_pricing_type")`. `'incluido'` bloquea `extra_price != 0` en todas sus opciones (FR-002); `'con_recargo'` permite precio libre (FR-003, sin cambio respecto a hoy). |

Columnas sin cambio: `id`, `name` (único global, sin cambio de alcance), `min_select`,
`max_select`, `active`.

**Asimetría de default** (research.md Decisión 1): `server_default='con_recargo'` protege
cualquier `INSERT` que no mencione la columna. El schema Pydantic `OptionGroupCreate.pricing_type`
es, en cambio, **obligatorio sin default** — a diferencia de `Product.tracks_inventory` (spec 027),
aquí no existe un valor de negocio "razonablemente por defecto" entre los dos casos de uso.

## Option (`options`, schema `tenant`) — SIN CAMBIO DE ESQUEMA, gana validación cruzada

Columnas sin cambio: `id`, `option_group_id`, `name`, `extra_price`, `inventory_item_id`,
`item_quantity`, `active`.

**Validación nueva (servicio, no `CHECK` de una sola tabla)**: al crear o actualizar una opción,
si `Option.option_group.pricing_type == 'incluido'` y el `extra_price` resultante es distinto de
`0`, la operación se rechaza con `422` (research.md Decisión 2). Al cambiar `OptionGroup.pricing_type`
de `'con_recargo'` a `'incluido'`, el sistema fuerza `extra_price = 0` en todas las opciones del
grupo como efecto del propio `PATCH` (no rechaza el cambio — mismo patrón ya usado por `RN-CAT-38`
al desvincular `inventory_item_id`).

**Validación nueva (gating por plan, no relacionada con `pricing_type`)**: guardar una opción con
`inventory_item_id is not None` o `item_quantity > 0` exige que el tenant tenga el módulo
Inventario incluido en su plan vigente (research.md Decisión 4) — independiente de a qué producto
esté o no asignado el grupo de esa opción.

## Product / ProductVariant — SIN CAMBIO DE ESQUEMA

`Product.tracks_inventory` (spec 027) no cambia de forma. Gana una validación nueva: activar
`tracks_inventory=True` (al crear o actualizar) exige que el tenant tenga el módulo Inventario
incluido en su plan vigente (research.md Decisión 4) — antes de esta spec no existía ningún gating
de plan sobre este campo.

`ProductVariant`, `RecipeItem`, `VariantOptionGroup`: sin cambio de esquema ni de validación nueva.
`VariantOptionGroup.quantity_per_option` sigue sin gating de plan propio (research.md Decisión 4,
"sin gating nuevo para `quantity_per_option`") — ya es inerte para cualquier producto con
`tracks_inventory=False`, y `tracks_inventory=True` ya queda cubierto por el punto anterior.

## Migración de datos existentes

Ver research.md, Decisión 6, para la migración completa (`op.add_column` + `op.create_check_constraint`
+ `op.execute(UPDATE ... CASE WHEN EXISTS ...)`). Resumen de la regla de backfill:

```text
pricing_type(grupo) = CASE
    WHEN EXISTS(alguna opción del grupo, activa o no, con extra_price > 0) THEN 'con_recargo'
    ELSE 'incluido'
END
```

Ningún precio, insumo, cantidad de consumo ni resultado de venta cambia por esta migración
(`spec.md` FR-015, SC-002) — es puramente una etiqueta derivada de datos ya existentes.

## Transición de `pricing_type` en el formulario de grupo

```text
                    ┌─────────────┐
   (crear grupo)    │  (elegir)   │  ← sin default; FR-001 exige elección explícita
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
       ┌─────────────┐          ┌──────────────┐
       │  incluido   │          │ con_recargo  │
       └──────┬──────┘          └──────┬───────┘
              │                        │
   cambiar a con_recargo      cambiar a incluido
   (sin confirmación —        (CON confirmación si alguna opción
    no hay nada que perder)    tiene extra_price > 0 — FR-004)
              │                        │
              ▼                        ▼
       ┌─────────────┐          ┌──────────────┐
       │ con_recargo │          │   incluido   │  (todas las opciones
       │ (precios en │          │ (precios      │   quedan en extra_price=0,
       │  $0, editab.)│         │  forzados a $0)│  efecto del PATCH)
       └─────────────┘          └──────────────┘
```

- Ningún dato de opción se borra en ninguna transición — solo `extra_price` se fuerza a `0` al
  entrar a "Incluido" (mismo criterio no destructivo de spec 027 para `tracks_inventory`).
- La confirmación de la transición "Con recargo → Incluido" con precios existentes es enteramente
  responsabilidad del frontend (research.md Decisión 2) — el backend aplica el cambio directamente
  si lo recibe.

## Reglas de validación (resumen por historia de usuario)

| Regla | Dónde se aplica | Historia |
|---|---|---|
| Grupo nuevo exige `pricing_type` explícito, sin default | `OptionGroupCreate.pricing_type` (Pydantic, requerido) | US1/US2 |
| "Incluido" bloquea precio ≠ $0 en sus opciones | `add_option`/`update_option` (`catalog/router.py`) | US1 |
| "Con recargo" permite precio libre ≥ $0 | Sin cambio — mismo `OptionCreate.extra_price`/`OptionUpdate.extra_price` de hoy | US2 |
| Cambiar a "Incluido" con precios existentes fuerza $0 | `update_option_group` (`catalog/router.py`) | US1, FR-004 |
| Confirmación antes de forzar $0 | Frontend (`ConfirmService`), antes del `PATCH` | US2, FR-004 |
| `pricing_type` independiente de consumo de inventario | Sin relación de código entre `pricing_type` y `inventory_item_id`/`quantity_per_option` | US1-US3, FR-005 |
| Selector de grupo, min/max de variante visibles sin inventario | `product-form.component.ts`, fuera del `@if` de inventario (research.md Decisión 5) | US3 |
| Insumo/cantidad de consumo ocultos sin inventario (por variante) | `product-form.component.ts`, `sectionsEnabled()` (research.md Decisión 5) | US3 |
| Sin movimiento de inventario si `tracks_inventory=False` | `_tracks_inventory` (`app/catalog_engine/consumption.py:163-171`) — sin cambio, ya existente (spec 027) | US3 |
| Insumo/cantidad del editor de catálogo compartido ocultos sin módulo Inventario | `option-groups-page.component.ts`/`option-form.component.ts`, gating por plan (research.md Decisión 5) | US5 |
| Un solo criterio para "¿descuenta inventario?" | `grupos_que_descuentan` unificado con `group_discounts` (`app/catalog_engine/pricing.py`) | US4, FR-009 |
| Activar `tracks_inventory` exige módulo Inventario en el plan | `ProductService.create_product`/`update_product` (research.md Decisión 4) | US5 |
| Guardar insumo/cantidad en una opción exige módulo Inventario en el plan | `add_option`/`update_option` (research.md Decisión 4) | US5 |
| Retirar el módulo no borra `tracks_inventory` ni datos de insumo | Sin `UPDATE`/`DELETE` nuevo sobre esos campos al revocar acceso — solo deja de aplicarse/mostrarse | US5, FR-013 |
| Reclasificación automática de grupos existentes | Migración `option_groups_pricing_type` (research.md Decisión 6) | US6 |
