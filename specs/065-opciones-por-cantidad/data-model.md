# Data Model: Selección por cantidad en grupos de opciones

Las decisiones de diseño detrás de cada elección están en [research.md](./research.md); este
documento se limita a columnas, restricciones, transiciones y la consulta de migración.

## OptionGroup (`option_groups`, schema `tenant`) — MODIFICADA

Columnas nuevas:

| Columna | Tipo | Nulable | Default (ORM/BD) | Notas |
|---|---|---|---|---|
| `selection_mode` | `String(20)` | No | `server_default='conteo'` (research.md Decisión 5) | `CheckConstraint("selection_mode IN ('conteo', 'cantidad')")`. `'conteo'` = comportamiento actual (`min_select`/`max_select` cuentan opciones distintas). `'cantidad'` = el cliente elige unidades libres por opción, sin mínimo posible. |
| `max_quantity_per_option` | `Integer`, nulable | Sí | `NULL` = sin tope | `CheckConstraint("max_quantity_per_option IS NULL OR max_quantity_per_option > 0")`. Solo tiene efecto cuando `selection_mode='cantidad'`. |
| `max_total_quantity` | `Integer`, nulable | Sí | `NULL` = sin tope | `CheckConstraint("max_total_quantity IS NULL OR max_total_quantity > 0")`. Suma de cantidades de todas las opciones del grupo. Solo tiene efecto cuando `selection_mode='cantidad'`. |

Columnas sin cambio: `id`, `name`, `min_select`, `max_select`, `active`, `pricing_type` (spec 063).
`min_select`/`max_select` **se ignoran** cuando `selection_mode='cantidad'` (research.md Decisión 7)
— no se leen ni se exigen; el formulario de administración deja de mostrarlos para ese modo.

**Default a nivel de schema de creación** (a diferencia de `pricing_type`, spec 063, que no tenía
uno): `OptionGroupCreate.selection_mode: Literal['conteo', 'cantidad'] = 'conteo'` — sí existe un
valor de negocio razonable por defecto (research.md Decisión 5).

## Option — SIN CAMBIO DE ESQUEMA

`extra_price`, `inventory_item_id`, `item_quantity` se interpretan igual; su multiplicación por la
cantidad elegida por el cliente ocurre en el motor de precio/consumo (`ChosenOption`, ver abajo),
no en `Option` misma.

## `ChosenOption` (estructura interna, no persistida)

Reemplaza el uso de `list[Option]` como parámetro compartido de `compute_line_price`,
`validate_option_selection` y `plan_line_consumption` (`app/catalog_engine/`):

```python
class ChosenOption(NamedTuple):
    option: Option
    quantity: int  # siempre 1 para opciones de un grupo "conteo" (validado, no asumido)
```

Producida por `load_valid_options` a partir de `list[OptionSelectionIn]` (ver contrato). Rechaza con
`422` si el mismo `option_id` aparece más de una vez en el payload (research.md Decisión 1/2).

## CartItemOption (`cart_item_options`) — MODIFICADA

| Columna | Tipo | Nulable | Default | Notas |
|---|---|---|---|---|
| `quantity` | `Integer` | No | `server_default='1'` | `CheckConstraint("quantity > 0")`. Nueva. |

Sin cambio: `id`, `cart_item_id`, `option_id`, `UniqueConstraint(cart_item_id, option_id)` — **no se
retira** (research.md Decisión 4): sigue habiendo como máximo una fila por (línea, opción); la
cantidad vive en esa fila.

## OrderItemOption (`order_item_options`) — MODIFICADA

Mismo cambio que `CartItemOption`: columna `quantity` nueva (mismo tipo/constraint/default),
`UniqueConstraint(order_item_id, option_id)` intacto.

## SaleItem (`sale_items`) — SIN CAMBIO DE ESQUEMA, cambia el contenido del JSON

`options` (JSONB) gana la clave `"quantity"` en cada dict del snapshot:
`{"option_id": "...", "name": "...", "extra_price": "...", "quantity": 2}`. Filas anteriores a esta
spec no tienen esa clave — todo consumidor debe tratar su ausencia como `1` (`?? 1` en frontend,
`.get("quantity", 1)` en backend si algo relee snapshots antiguos).

## Migración de datos existentes

```text
option_groups.selection_mode        = 'conteo'  para toda fila existente (server_default)
option_groups.max_quantity_per_option = NULL      para toda fila existente
option_groups.max_total_quantity      = NULL      para toda fila existente
cart_item_options.quantity           = 1          para toda fila existente (server_default)
order_item_options.quantity          = 1          para toda fila existente (server_default)
```

Sin consulta `EXISTS`/`CASE` (a diferencia de `pricing_type`, spec 063): no hay ambigüedad que
resolver a partir de datos — toda fila histórica representa exactamente "una unidad, modo conteo"
porque era la única posibilidad antes de esta spec (spec.md, US6, FR-012).

## Transición del modo de selección de un grupo

```text
                    ┌─────────────┐
   (crear grupo)    │   conteo    │  ← default (FR-001)
                    └──────┬──────┘
                           │
              cambiar a "cantidad"
                           │
                           ▼
                    ┌─────────────┐
                    │  cantidad   │  min_select/max_select dejan de aplicarse;
                    │             │  topes por opción/total pasan a aplicarse
                    └──────┬──────┘
                           │
              cambiar a "conteo"
                           │
                           ▼
                    ┌─────────────┐
                    │   conteo    │  topes por opción/total dejan de aplicarse;
                    │             │  min_select/max_select vuelven a aplicarse
                    └─────────────┘
```

- El cambio de modo aplica de inmediato a selecciones nuevas; nunca reescribe pedidos o ventas ya
  confirmados (`spec.md` FR-013, Principio VII).
- No hay estado intermedio ni versión distinta del grupo por presentación — el modo es una única
  columna en `OptionGroup` (research.md Decisión 6).

## Reglas de validación (resumen por historia de usuario)

| Regla | Dónde se aplica | Historia |
|---|---|---|
| `selection_mode` explícito, `'conteo'` por defecto | `OptionGroupCreate.selection_mode` (Pydantic, con default) | US1 |
| Cantidad libre por opción (`+`/`-`) en modo "cantidad" | `product-select.component.ts`, `selected: Record<groupId, Record<optionId, number>>` (research.md Decisión 8) | US2 |
| Nunca bloquea por no elegir nada en modo "cantidad" | `validate_option_selection` no agrega grupos "cantidad" a la lista de obligatorios sin elegir (research.md Decisión 3) | US2, FR-003 |
| Precio multiplica `extra_price × quantity` por opción | `compute_line_price` sobre `ChosenOption` (research.md Decisión 2) | US3, FR-004 |
| "Incluido" (spec 063) sigue en $0 sin importar cantidad | Sin cambio — `extra_price` de una opción "incluida" ya es `0`, `0 × quantity = 0` | US3, FR-005 |
| Consumo multiplica `per_unit × quantity × cantidad_vendida` | `plan_line_consumption` sobre `ChosenOption` (research.md Decisión 2) | US3, FR-006 |
| Chequeo de stock considera el total ya multiplicado | `check_availability` recibe el `required` ya agregado desde `plan_line_consumption` — sin cambio propio | US3, FR-007 |
| Topes opcionales por opción y por grupo | `validate_option_selection`, rama "cantidad" (research.md Decisión 3) | US4, FR-008/FR-009 |
| Cantidad conservada en snapshot de venta | `SaleItem.options[].quantity` (este documento) | US5, FR-011 |
| "2x Nombre" en comanda/pedidos/recibo | `MenuLookupIndex.optionLabelWithQuantity` (research.md Decisión 9) | US5, FR-010 |
| Reclasificación automática de grupos existentes | Migración (research.md Decisión 5) | US6, FR-001/FR-012 |
