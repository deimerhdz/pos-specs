# Research: Selección por cantidad en grupos de opciones

Fase 0 de `/speckit-plan`. Cada decisión cita el código real inspeccionado en `../pos-backend` y
`../pos-heladeria` (ambos en disco, sibling de `pos-specs`). Sin `NEEDS CLARIFICATION` pendiente:
las dos decisiones de diseño (topes de cantidad, "por cantidad" siempre opcional) ya se resolvieron
en `spec.md` (sección Clarifications) antes de este plan.

## Nota sobre numeración de ramas

Mientras se investigaba esta spec, se detectó que las ramas/commits de la spec 063
(`grupos-opciones-precio-inventario`, ya implementada) fueron renombradas manualmente en ambos
repos código a `064-grupos-opciones-precio-inventario` (commits `89c1689` en `pos-heladeria`,
`3d7909f` en `pos-backend`). `pos-specs` conserva ese trabajo documentado como spec **063** — no se
renombró para no reescribir historia ya cerrada. Esta spec nueva, por decisión explícita del
usuario, se numera **065** en `pos-specs` para no chocar con el "064" que el código ya usa. Toda
cita a "spec 063" en este documento se refiere a ese trabajo ya implementado (pricing_type +
gating por plan), independientemente de cómo esté rotulado en las ramas de código.

**Consecuencia para el Setup de esta spec**: la implementación de spec 065 debe partir de la rama
`064-grupos-opciones-precio-inventario` en ambos repos (donde ya vive `OptionGroup.pricing_type`,
del que esta spec depende — FR-005), no de `develop` — salvo que para entonces esa rama ya se haya
integrado a `develop`, en cuyo caso se parte de `develop` como de costumbre.

## Decisión 1 — Formato de "opciones elegidas" en la API: unificar en pares (opción, cantidad)

**Decisión**: reemplazar `option_ids: list[UUID]` por `options: list[OptionSelectionIn]`, donde
`OptionSelectionIn = {option_id: UUID, quantity: int = 1}`, en los cuatro puntos de entrada reales
que hoy aceptan `option_ids` (research previo de esta spec, sesión de `/speckit-clarify`):

- `CartItemIn`/`CartItemUpdate` (`app/api/v1/cart/schemas.py:40-49`)
- El schema de línea de `POST /orders` (`app/api/v1/orders/schemas.py:117`)
- El equivalente en `orders/consolidation.py` (mesero agrega ítem a mesa, `data.option_ids`)
- El schema de línea de venta de mostrador (`app/api/v1/sales/service.py`, `line.option_ids`)

`quantity` por defecto `1` — **exactamente el significado que "elegir esta opción" ya tiene hoy en
modo "conteo"**: cada entrada del arreglo es una opción elegida una vez. Ningún cliente HTTP
existente que siga enviando el arreglo antiguo se rompe si el frontend migra ambos repos a la vez
(mismo criterio de despliegue coordinado que ya usa este proyecto para todo cambio de contrato,
Constitución §Alcance — dos repos, un mismo cambio).

**Rationale**: la alternativa (mantener `option_ids` para "conteo" y agregar un campo paralelo
`option_quantities` solo para "cantidad") obligaría a cada uno de los ~8 puntos donde
`load_valid_options` se invoca hoy (`cart/service.py:318,359`, `orders/service.py:242`,
`orders/consolidation.py:193`, `sales/service.py:233`, más los tres callers análogos del lado
staff) a fusionar dos arreglos con semántica distinta antes de construir la lista de opciones —
una fuente de bugs sutiles (¿qué pasa si un id aparece en ambos?) sin ningún beneficio: un solo
campo con `quantity` (default 1) expresa ambos modos sin ambigüedad y sin casos especiales.

**Alternativas consideradas**:
- Campo paralelo `option_quantities: dict[UUID, int]` además de `option_ids`: rechazado por la
  razón anterior — dos fuentes de verdad para "qué se eligió".
- Permitir repetir el mismo `option_id` N veces en `option_ids` como forma implícita de cantidad:
  rechazado — contradice frontalmente cómo se investigó que funciona hoy el sistema (deduplicación
  explícita en `load_valid_options:47-50`, más un `UniqueConstraint` en `CartItemOption`/
  `OrderItemOption` que lo impide a nivel de base de datos) — habría que primero desmontar esas tres
  protecciones para luego reinterpretarlas, más riesgoso que agregar un campo explícito.

## Decisión 2 — `ChosenOption`: una sola forma de "opción + cantidad" para todo el motor de precio/consumo/validación

**Decisión**: introducir un tipo pequeño en `app/catalog_engine/core.py` (junto a `ConsumptionLine`,
ya existente ahí):

```python
class ChosenOption(NamedTuple):
    option: Option
    quantity: int
```

`load_valid_options` (`app/catalog_engine/pricing.py:36-59`) pasa de devolver `list[Option]` a
devolver `list[ChosenOption]` — carga cada `Option` una sola vez (rechaza con 422 si el mismo
`option_id` aparece dos veces en el payload, en vez de deduplicar en silencio: mismo criterio ya
usado por `RN-CAT-14`, receta con insumo repetido) y le asocia la `quantity` que trajo su
`OptionSelectionIn`. `compute_line_price`, `validate_option_selection` y `plan_line_consumption`
reciben `Sequence[ChosenOption]` en vez de `Sequence[Option]`.

**Rationale — por qué el precio y el consumo NO necesitan enterarse del modo del grupo**: con
`ChosenOption.quantity` ya resuelto en la carga, las fórmulas existentes se generalizan con un
único cambio (multiplicar por `quantity` en vez de contar 1 por opción):

```python
# compute_line_price (antes: price += option.extra_price)
for chosen in options:
    price += chosen.option.extra_price * chosen.quantity

# plan_line_consumption (antes: per_unit * qty)
for chosen in options:
    ...
    lines.append(ConsumptionLine(chosen.option.inventory_item_id, per_unit * qty * chosen.quantity, ...))
```

Como en modo "conteo" `quantity` siempre vale `1` (garantizado por la Decisión 3, no por estas dos
funciones), multiplicar por `quantity` es una operación neutra para todo el catálogo existente —
**cero cambio de comportamiento para grupos "conteo"**, sin necesitar que `compute_line_price` ni
`plan_line_consumption` sepan qué modo tiene el grupo de cada opción. Esto confirma, con código
real, la intuición de `spec.md` (FR-004/FR-006): el precio y el consumo son "el mismo cálculo,
generalizado", no un camino nuevo paralelo.

**Alternativas consideradas**:
- Ramificar `compute_line_price`/`plan_line_consumption` según `option.option_group.selection_mode`:
  rechazado — añade una consulta/relación extra en el camino más caliente del sistema (toda venta
  pasa por aquí) para un resultado idéntico a "multiplicar por quantity siempre".

## Decisión 3 — `validate_option_selection`: un segundo camino de validación para "cantidad", sin mínimo posible

**Decisión**: dentro de `validate_option_selection` (`app/catalog_engine/pricing.py:87-167`), antes
de aplicar la lógica de conteo ya vigente (`chosen[gid] += 1`, `bounds`, `_exige_maximo`), separar
las opciones elegidas por el modo de su grupo:

- **Grupos "conteo"** (comportamiento sin cambios): cada `ChosenOption` de un grupo "conteo" DEBE
  traer `quantity == 1` — si trae más, `422` inmediato ("esta opción no admite más de una unidad")
  **antes** de correr la lógica de bounds ya existente, para que un cliente que ignore el modo del
  grupo reciba un error claro en vez de que su `quantity` se pierda silenciosamente.
- **Grupos "cantidad"** (nuevo): se suma la `quantity` de cada `ChosenOption` por opción individual
  y el total del grupo; se valida contra `max_quantity_per_option`/`max_total_quantity` del grupo
  si están configurados (`None` = sin tope, spec.md FR-008). **Nunca** se exige un mínimo — no hay
  equivalente de `min_select` ni de `_exige_maximo` (`core.py:38-49`) para este modo (spec.md
  FR-003, Clarifications sesión 2026-09-01): un grupo "cantidad" jamás aparece en la lista de
  "grupos obligatorios sin elegir nada" (`pricing.py:135-138`).
- **`grupos_que_descuentan`** (criterio unificado de spec 063/A-32, `pricing.py:73-89`) no cambia:
  sigue decidiendo "¿este grupo descuenta inventario?" igual para ambos modos — un grupo "cantidad"
  que descuenta inventario simplemente nunca entra en la rama de "exige el máximo" porque esa rama
  ya está condicionada a grupos obligatorios (`min_select>0`), y "cantidad" nunca es obligatorio.

**Rationale**: mantiene el chequeo preventivo de disponibilidad (`RN-CAT-24`, spec 003) y el
criterio "violación en grupo que descuenta siempre bloquea" (`RN-CAT-30`) intactos y aplicables a
ambos modos, sin necesitar tocar esas dos reglas — el único código nuevo es el de los dos topes
opcionales, más estricto en su propio carril, nunca en el de `min_select`/`max_select`.

**Alternativas consideradas**:
- Reinterpretar `min_select`/`max_select` como "mínimo/máximo total de unidades" para grupos
  "cantidad": rechazado explícitamente por Clarifications (sesión 2026-09-01, "siempre opcional, sin
  mínimo posible") — habría reintroducido un mínimo por la puerta trasera.

## Decisión 4 — Persistencia: una columna `quantity` en las tablas de unión, no una fila por unidad

**Decisión**: agregar `quantity: Mapped[int] = mapped_column(Integer, nullable=False,
server_default="1")` a `CartItemOption` (`app/models/cart_item.py:51-72`) y `OrderItemOption`
(`app/models/order_item.py:88-108`), con `CheckConstraint("quantity > 0", ...)` en ambas. **El
`UniqueConstraint` existente (`cart_item_id, option_id` / `order_item_id, option_id`) NO se toca ni
se retira** — sigue habiendo como máximo una fila por (línea, opción); la cantidad vive en esa
misma fila, no en filas repetidas.

`SaleItem.options` (JSONB, `app/models/sale.py:118-155`) gana la clave `"quantity"` en cada dict
del snapshot (`{"option_id", "name", "extra_price", "quantity"}`), construido en los dos puntos
donde hoy se arma ese snapshot (`app/api/v1/orders/checkout.py:228-232`,
`app/api/v1/sales/service.py:235-241`).

**Rationale**: es la lectura más simple de la investigación previa (sesión `/speckit-clarify`): esa
investigación especuló que haría falta "soltar las unique constraints" para permitir repetir un
`option_id`, pero una columna `quantity` en la misma fila logra el mismo resultado de negocio (2
Bobombún) sin duplicar filas, sin tocar ninguna constraint existente, y sin que
`load_valid_options`/`plan_line_consumption` necesiten iterar "N veces la misma opción" — iteran
una vez por opción distinta, cada una con su cantidad ya resuelta (Decisión 2).

**Alternativas consideradas**:
- Quitar el `UniqueConstraint` y permitir filas repetidas (una fila = una unidad): rechazado —
  multiplica el número de filas sin necesidad (2 Bobombún = 2 filas idénticas salvo por `id`), y
  cada consumidor de `OrderItemOption`/`CartItemOption` (incluida la resolución de nombres en el
  frontend, `menu-lookup.ts`) tendría que agrupar por `option_id` para mostrar "2x" en vez de leer
  un campo ya agregado.

## Decisión 5 — Migración de esquema

**Decisión**: una migración nueva, `down_revision` apuntando al head real confirmado
(`68326ed66ebf`, la migración de `pricing_type` de spec 063 — confirmado con `alembic current` y
`alembic upgrade head` reales contra la base de desarrollo, ver Nota sobre numeración de ramas
arriba), que agrega en una sola operación (mismo patrón `_has_table` + `@for_each_tenant_schema` ya
usado en las migraciones de specs 027/063):

```text
option_groups:      + selection_mode VARCHAR(20) NOT NULL DEFAULT 'conteo'
                       CHECK (selection_mode IN ('conteo', 'cantidad'))
                     + max_quantity_per_option INTEGER NULL
                       CHECK (max_quantity_per_option IS NULL OR max_quantity_per_option > 0)
                     + max_total_quantity INTEGER NULL
                       CHECK (max_total_quantity IS NULL OR max_total_quantity > 0)
cart_item_options:   + quantity INTEGER NOT NULL DEFAULT 1, CHECK (quantity > 0)
order_item_options:  + quantity INTEGER NOT NULL DEFAULT 1, CHECK (quantity > 0)
```

Sin backfill de datos más allá de los defaults de columna: todo grupo existente ya debe quedar en
`'conteo'` (spec.md FR-001/US6), y toda fila de `cart_item_options`/`order_item_options` ya
existente representa "una unidad elegida" por construcción (era la única cantidad posible antes de
esta spec), así que `quantity=1` es el valor correcto para el 100% de las filas históricas — no
hace falta ninguna consulta `EXISTS`/`CASE` como sí necesitó spec 063 (research.md Decisión 6 de
esa spec) para clasificar `pricing_type` a partir de datos ambiguos.

**Rationale del default `'conteo'` a nivel de schema (a diferencia de `pricing_type`, que no tenía
default en el schema de creación)**: aquí sí existe un valor de negocio razonable por defecto — es
literalmente el pedido del usuario ("el modo actual (conteo) sigue siendo el comportamiento por
defecto para no romper catálogos existentes", spec.md FR-001) — así que `OptionGroupCreate.
selection_mode` SÍ lleva default `'conteo'` en el schema Pydantic, a diferencia de
`OptionGroupCreate.pricing_type` (spec 063), que no tenía default posible.

## Decisión 6 — Dónde viven los topes de cantidad: en `OptionGroup`, no por presentación

**Decisión**: `max_quantity_per_option`/`max_total_quantity` viven en `OptionGroup` (catálogo,
compartido entre todas las presentaciones que ofrezcan el grupo) — no se agrega una versión
"override por variante" equivalente a como `VariantOptionGroup` ya permite `min_select`/
`max_select` propios por tamaño.

**Rationale**: `spec.md` (Key Entities) describe los topes como atributos del grupo, sin pedir
granularidad por presentación — igual que `pricing_type` (spec 063) vive solo en `OptionGroup` y no
en `VariantOptionGroup`. Agregar el override por variante sin que el spec lo pida sería
complejidad no solicitada (Principio V de la Constitución, "Nuevas Funcionalidades Antes que
Refactorizaciones Oportunistas" aplicado en sentido inverso: no anticipar alcance no pedido).

**Alternativas consideradas**:
- Override por variante (mismo patrón que `min_select`/`max_select` en `VariantOptionGroup`):
  diferido — no lo pidió el spec; si el negocio lo necesita más adelante (ej. "la copa grande
  admite más toppings que la pequeña"), es una funcionalidad independiente sobre esta misma base.

## Decisión 7 — `selection_mode` no aplica `min_select`/`max_select` en modo "cantidad"

**Decisión**: `OptionGroup.min_select`/`max_select` y `VariantOptionGroup.min_select`/`max_select`
permanecen en el modelo sin cambio de tipo, pero **se ignoran por completo** cuando el grupo está
en modo "cantidad" — ni se leen ni se exigen. El formulario de administración (frontend) deja de
mostrarlos para un grupo "cantidad" y en su lugar muestra los dos topes de la Decisión 6.

**Rationale**: evita el estado inconsistente de "un grupo cantidad con min_select=2 que nadie
aplica pero que confunde al admin que lo mira" — más simple que intentar reutilizar esos campos
con un significado distinto (ya descartado en Decisión 3).

## Decisión 8 — Selección del cliente en el menú QR/POS: stepper en vez de toggle, por grupo

**Decisión**: `product-select.component.ts` (`../pos-heladeria/.../tables/components/`) gana una
rama de renderizado nueva condicionada a `group.selection_mode === 'cantidad'` (campo nuevo en
`MenuOptionGroup`, expuesto por `GET /menu`): en vez del botón toggle actual
(`toggleOption(group, opt)`, líneas 133-155) muestra un control `+`/`-` por opción, con el mismo
patrón visual ya usado para la cantidad de la línea completa (`dec()`/`inc()`, líneas 206-219) pero
por opción. `selected: Record<groupId, string[]>` (línea 247) se generaliza a
`Record<groupId, Record<optionId, number>>` — mismo objeto, ahora con cantidades en vez de solo
presencia/ausencia; para grupos "conteo" el valor sigue siendo efectivamente `0` o `1` por opción,
así que ninguna lógica de "conteo" cambia de forma, solo de representación interna.

`chosenCount`/`isComplete`/`groupHint`/`requiredCount` (líneas 297-slice, 425-471) se ramifican: en
modo "cantidad", `groupHint` muestra el total de unidades elegidas (sin comparar contra ningún
mínimo, porque no existe) y `isComplete` es trivialmente `true` siempre (nunca bloquea, Decisión
3/spec FR-003); los topes de la Decisión 6 deshabilitan el botón `+` de una opción (o de todas)
cuando se alcanzan, en vez de aparecer como error de validación al confirmar.

**Rationale**: reutiliza el mismo widget de cantidad que la propia pantalla ya usa para la cantidad
de la línea completa (`dec()`/`inc()`), en vez de inventar un control nuevo — consistencia visual
mínima sin necesitar diseño nuevo (spec.md, Out of Scope: "el diseño visual concreto... se resuelve
en la fase de planeación" — esta decisión resuelve la mecánica, no el pixel).

## Decisión 9 — Mostrar cantidad en comanda/recibo/"Mis pedidos": un solo formateador compartido

**Decisión**: `MenuLookupIndex`/`buildMenuLookup` (`../pos-heladeria/.../tables/services/
menu-lookup.ts:8-47`) gana un método `optionLabelWithQuantity(optionId, quantity)` que devuelve
`"2x Bobombún"` si `quantity > 1`, o `"Bobombún"` (igual que hoy) si `quantity === 1` — los seis
puntos de renderizado ya identificados en la investigación previa
(`pos-terminal.store.ts:794`, `order-detail.component.ts:144`, `public-menu.component.ts:1095`,
`payment-validation-block.component.ts:131`, `receipt.util.ts:88,216-218`,
`cart.component.ts:30-32`/`dining-cart.service.ts:146-148`) cambian su `.map(o => lk.optionLabel(o.
option_id))` por `.map(o => lk.optionLabelWithQuantity(o.option_id, o.quantity ?? 1))` — un cambio
mecánico idéntico en los seis, sin lógica de formato duplicada seis veces.

**Rationale**: un solo lugar decide el formato "2x Nombre" — si el negocio pide cambiarlo (ej.
"Nombre ×2") se cambia una vez, no seis. `?? 1` cubre las filas persistidas antes de esta spec
(sin la clave `quantity` en su JSON, o con `CartItemOption`/`OrderItemOption` migradas al default
`1`).

## Resumen de versiones y entorno (sin cambios respecto a spec 063)

**Backend**: Python 3.14.4, FastAPI + SQLAlchemy + Alembic, PostgreSQL 16 schema-per-tenant. Sin
dependencias nuevas.

**Frontend**: Angular 21.1.x (standalone components + signals), TypeScript 5.9.2, Node 24.16.0. Sin
dependencias nuevas.

**Testing**: backend — `unittest` vía `python -m unittest discover -s app/characterization_tests -p
'test_*.py'`; frontend — Vitest vía `ng test`.
