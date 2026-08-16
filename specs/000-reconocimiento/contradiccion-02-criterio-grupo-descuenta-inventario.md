# Contradicción 02 — "¿Este grupo de opciones descuenta inventario?" tiene dos respuestas distintas según quién pregunte

**Fecha**: 2026-08-15
**Alcance**: `pos-backend`, `app/api/v1/catalog/line_pricing.py` y
`app/api/v1/catalog/consumption_plan.py`.
**Método**: lectura directa de ambas funciones y de sus llamadores; trazado del flujo
completo desde la validación de selección hasta el mensaje de error que ve el usuario.
Corresponde al hallazgo ya anotado como `RN-CAT-39 [DISCREPANCIA ENTRE VALIDADORES]` en
`reglas-de-negocio.md:383-388`; este documento lo amplía con el efecto observable de punta
a punta y ejemplos concretos.

---

## 1. Las implementaciones implicadas

**`grupos_que_descuentan(db, links)`** — `app/api/v1/catalog/line_pricing.py:69-91`.

```python
por_grupo = {l.option_group_id for l in links if l.quantity_per_option > 0}
por_opcion = set(db.execute(
    select(Option.option_group_id)
    .where(Option.option_group_id.in_(gids), Option.item_quantity > 0)
    .distinct()
).scalars().all())
return por_grupo | por_opcion
```

Único llamador: `validate_option_selection` (mismo fichero, líneas 108-172), invocada desde
`load_valid_options(db, option_ids, variant=...)` (líneas 43-66) cuando se pasa `variant` —
es decir, en el camino de **alta de línea de pedido** (agregar un ítem con sus opciones al
carrito/comanda). Decide dos cosas: (a) si un grupo obligatorio debe exigir el **máximo**
de opciones y no solo el mínimo (`_exige_maximo`, líneas 94-105), y (b) qué problemas de
validación son **siempre bloqueantes** incluso con `STRICT_OPTION_SELECTION=False` (línea
164-172: `blocking = [p for p in problems if p[0] in consumen]`).

**`group_discounts(db, link)`** — `app/api/v1/catalog/consumption_plan.py:79-95`.

```python
if link.quantity_per_option > 0:
    return True
return db.execute(
    select(Option.id)
    .where(
        Option.option_group_id == link.option_group_id,
        Option.active.is_(True),
        Option.inventory_item_id.is_not(None),
        Option.item_quantity > 0,
    )
    .limit(1)
).first() is not None
```

Único llamador: `ensure_lines_consume_inventory` (mismo fichero, líneas 165-227), invocada
en el camino de **confirmación de venta** (el paso común a todos los caminos que
efectivamente descuentan stock, según su propio docstring: "es el paso común de todos los
caminos que descuentan"). Decide si un grupo cuenta como "fuente de consumo configurada"
para distinguir dos mensajes de error distintos cuando una línea no descontaría nada.

## 2. ¿Usan la misma convención o algoritmo?

No. Ambas responden a la misma pregunta de negocio ("¿elegir una opción de este grupo
mueve stock?") con dos vías declaradas como equivalentes en sus propios docstrings
("cantidad del tamaño, o si no la define, la de la opción"), pero difieren en los
**requisitos de la segunda vía** (`por_opción`):

| Requisito | `grupos_que_descuentan` | `group_discounts` |
|---|---|---|
| `item_quantity > 0` | Sí | Sí |
| `Option.active = True` | **No exige** | **Exige** |
| `Option.inventory_item_id IS NOT NULL` | **No exige** | **Exige** |

`grupos_que_descuentan` cuenta una opción como "descuenta" con solo mirar
`item_quantity > 0`, sin importar si esa opción está activa ni si de verdad tiene un
insumo de inventario enlazado. `group_discounts` exige las tres condiciones a la vez. Una
opción con `item_quantity` puesto pero **sin** `inventory_item_id` (una configuración de
catálogo a medio terminar, o un insumo que se desvinculó después) es "sí descuenta" para
una función y "no descuenta" para la otra, para el mismo grupo, en el mismo instante.

## 3. Ejemplo concreto con resultado distinto

Variante "Copa Grande" ofrece un único grupo de opciones, "Extras" (`min_select=0`,
`max_select=1`), con una sola opción activa: "Extra Dulce", configurada con
`item_quantity=10` pero `inventory_item_id=None` (el admin puso la cantidad al crear la
opción, pero nunca llegó a enlazarla a un insumo real del inventario — un estado de
catálogo incompleto, no forzado por ninguna validación de creación). La variante no tiene
receta fija (`RecipeItem`) para ningún insumo.

**Camino de alta** (`validate_option_selection`, vía `create_order` o, cuando se corrija
la Contradicción 05, también `add_item_to_table`): `grupos_que_descuentan` ve
`item_quantity=10 > 0` → el grupo "Extras" **sí** cuenta como "que descuenta". Como
`min_select=0`, `_exige_maximo` no aplica (línea 105: `lo > 0 and ...`, y aquí `lo=0`), así
que no elegir nada sigue siendo válido — sin efecto visible en este caso concreto porque el
grupo es opcional.

**Camino de confirmación** (`ensure_lines_consume_inventory`, común a los tres caminos de
staff que descuentan y a `confirm_order`): el cliente elige "Extra Dulce". Pero
`plan_line_consumption` (`consumption_plan.py:121-123`) descarta explícitamente cualquier
opción con `inventory_item_id is None` ("`continue`" antes de calcular cantidad) — la
elección de "Extra Dulce" **no genera ningún `ConsumptionLine`**. Como no hay receta fija
ni otro grupo, `required_consumption` da `{}` (vacío) y se dispara la rama de error de
`ensure_lines_consume_inventory` (líneas 188-198). Para decidir **qué mensaje** mostrar,
llama a `group_discounts` sobre el grupo "Extras": como "Extra Dulce" no tiene
`inventory_item_id`, `group_discounts` devuelve `False` → `configurada = False` → se
dispara el mensaje **`sin_receta`** (línea 200-212):

> «Copa Grande» no tiene receta configurada, así que venderlo no descontaría inventario.
> Cárgasela en Productos → Recetas para poder venderlo.

Este mensaje es objetivamente engañoso: el catálogo **sí** tiene algo configurado (un
grupo, una opción, una cantidad) — lo que falta es el enlace al insumo, no la receta
entera. Un administrador que reciba este mensaje y vaya a "Productos → Recetas" (como el
mensaje le indica) no encontrará nada que arreglar ahí, porque el problema real está en la
opción "Extra Dulce" de "Productos → Opciones", un lugar distinto del catálogo. Si en vez
de eso `group_discounts` compartiera el criterio de `grupos_que_descuentan` (sin exigir
`inventory_item_id`), el sistema mostraría el mensaje **`sin_eleccion`** — que tampoco
sería correcto aquí porque sí hubo elección — evidenciando que ninguno de los dos mensajes
actuales describe con precisión "hay una opción con cantidad puesta pero sin insumo
enlazado".

## 4. Cuándo se manifiesta y cuándo coinciden

Ambas funciones coinciden siempre que el catálogo esté bien formado: toda opción con
`item_quantity > 0` también tiene `inventory_item_id` asignado y está activa. Dado que el
flujo normal de alta de una opción en el panel de catálogo probablemente pide ambos campos
juntos, este es presumiblemente el caso más común — lo que explica por qué la discrepancia
no se ha notado en operación normal. Se manifiesta específicamente cuando:

- Se desvincula el insumo de una opción sin resetear `item_quantity` a 0 — el propio
  `RN-CAT-38` (`reglas-de-negocio.md:376-381`) documenta que el sistema normalmente sí
  resetea `item_quantity` a 0 al desvincular vía `PATCH {"inventory_item_id": null}`, así
  que este escenario requeriría un camino distinto de escritura a la base de datos que no
  pase por ese endpoint (una migración, una corrección manual, un estado heredado de antes
  de que existiera esa regla), o
- Se desactiva la opción (`active=False`) sin tocar `item_quantity` ni `inventory_item_id`
  — un insumo temporalmente descontinuado. Aquí `grupos_que_descuentan` seguiría contando
  el grupo como "que descuenta" (no filtra por `active`) mientras `group_discounts` ya no
  lo haría, incluso con `inventory_item_id` intacto.

## 5. Historia probable

`consumption_plan.py` se autodescribe como "Definición **única** del inventario que
consume una línea vendida" (líneas 1-8 del módulo) y narra su propio origen: antes la
lógica de consumo "vivía triplicada" entre `deduct_order_item`, `reverse_order_item` y
`deduct_sale`, y se centralizó aquí precisamente para evitar que una reversa divergiera del
descuento. `group_discounts` nació dentro de ese esfuerzo de centralización, con la
condición más estricta (`active` + `inventory_item_id`) porque es la que **de verdad**
mueve stock — coherente con el propósito del módulo. `grupos_que_descuentan`, en cambio,
vive en `line_pricing.py`, un módulo con una responsabilidad distinta (validar que la
*selección* del cliente respeta los grupos de la variante), y responde una pregunta
parecida pero no idéntica: no "¿esto moverá stock ahora mismo?" sino "¿debería tratar este
grupo con el rigor de un grupo que mueve stock?" — una pregunta más permisiva por diseño,
para no dejar pasar por error una opción que *casi* está bien configurada. El problema es
que ambas funciones responden, en la práctica, a la misma pregunta de UX/negocio ("este
grupo importa para el inventario") sin que ningún comentario reconozca que la respuesta
puede diferir para el mismo grupo — tal como ya señala `RN-CAT-39`.

---

**Pregunta abierta al negocio**: ¿existen hoy en el catálogo opciones con `item_quantity`
puesto pero sin insumo de inventario enlazado (o inactivas) que sigan asignadas a algún
grupo de alguna variante vendible? Una consulta directa a la base de datos podría
confirmarlo, pero excede el alcance de lectura de código de este documento.
