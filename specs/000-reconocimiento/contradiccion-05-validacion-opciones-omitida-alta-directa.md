# Contradicción 05 — Añadir un ítem a la orden valida la selección de opciones en un camino y no en el otro, aunque ambos comparten la misma función

**Fecha**: 2026-08-15
**Alcance**: `pos-backend`, `app/api/v1/catalog/line_pricing.py`,
`app/api/v1/orders/service.py`, `app/api/v1/orders/consolidation.py`. Incluye
reconstrucción de historia vía `git log`/`git show`.
**Método**: lectura directa de los tres caminos de alta de línea de pedido del staff y de
la función compartida que ambos invocan; comparación de los argumentos exactos con los que
la invoca cada uno; reconstrucción de la cronología real de commits que llevó a este
estado. Amplía con evidencia nueva el punto 1 de la sección "Puntos donde dos caminos hacen
'lo mismo' con código distinto" de `flujo-pedido-qr.md:512-517`, que documenta la
duplicación de código entre estos dos caminos pero no había detectado esta divergencia de
comportamiento concreta.

---

## 1. Las implementaciones implicadas

**La función compartida**: `load_valid_options(db, option_ids, *, variant=None)` —
`app/api/v1/catalog/line_pricing.py:43-66`.

```python
def load_valid_options(
    db: Session, option_ids: list[UUID], *, variant: ProductVariant | None = None
) -> list[Option]:
    """Carga las opciones (dedup) validando existencia y que estén activas.

    Pasar `variant` valida además la selección contra los grupos del producto (ver
    `validate_option_selection`). Es opcional solo por compatibilidad con los pocos
    llamadores que aún no tienen la variante a mano; **pasarla siempre que se pueda**.
    """
    ...
    if variant is not None:
        validate_option_selection(db, variant, options)
    return options
```

El propio docstring es explícito: pasar `variant` es lo que activa
`validate_option_selection` — la función que comprueba `min_select`/`max_select` por
grupo, que un grupo obligatorio y que descuenta inventario exija el máximo (no solo el
mínimo), y que cualquier problema en un grupo que descuenta inventario sea **siempre
bloqueante**, incluso con `STRICT_OPTION_SELECTION=False` (`line_pricing.py:108-172`,
detallado también en la Contradicción 02).

**Camino 2 — `create_order`** (`app/api/v1/orders/service.py:37-135`), llamada en la
línea 102:

```python
options = load_valid_options(db, line.option_ids, variant=variant)
```

Pasa `variant` — la validación de selección **se ejecuta**.

**Camino 3 — `add_item_to_table`** (`app/api/v1/orders/consolidation.py:180-234`), llamada
en la línea 199:

```python
options = load_valid_options(db, data.option_ids)
```

**No** pasa `variant`, pese a que la variable `variant` ya está en el ámbito local dos
líneas arriba (línea 196: `variant = get_or_404(db, ProductVariant, data.product_variant_id,
"Variant not found")`) — no falta el dato, falta el argumento. La validación **no se
ejecuta**: `load_valid_options` solo comprueba que las opciones existan y estén activas,
sin verificar que pertenezcan a un grupo que la variante realmente ofrezca, ni sus
`min_select`/`max_select`.

Un tercer camino, `consolidate_table` (`consolidation.py:106-169`), no llama a
`load_valid_options` en absoluto — copia las opciones directamente de los `CartItem` ya
validados al añadirse al carrito del comensal (líneas 147-152). Esto está justificado y no
forma parte de la contradicción: esas opciones ya pasaron por su propia validación en el
momento en que el comensal las eligió desde el QR.

## 2. ¿Usan la misma convención o algoritmo?

Es literalmente la misma función invocada con distinto argumento — no hay dos algoritmos
distintos, hay una única función de validación que un camino activa y el otro
inadvertidamente desactiva. Por diseño, según `flujo-pedido-qr.md`, ambos caminos
pretenden hacer lo mismo ("expandir combo o validar variante+opciones → crear `OrderItem`
→ `deduct_order_items`") y de hecho ambos usan `load_valid_options` y `compute_line_price`
como piezas compartidas — la intención de mantenerlos equivalentes es clara en el propio
código; lo que falla es un argumento de una llamada.

## 3. Ejemplo concreto con resultado distinto

Variante "Copa Grande — 3 bolas" ofrece el grupo "Sabores" (`min_select=2`,
`max_select=3`, `quantity_per_option=120` gramos por sabor elegido — un grupo que, por
`grupos_que_descuentan`, **sí** descuenta inventario y es obligatorio). Un mesero, desde la
terminal, agrega un ítem "Copa Grande" con un solo sabor elegido (`option_ids=["chocolate"]`)
— por ejemplo por un error de la interfaz, un doble toque accidental que deselecciona,
o simplemente porque nada en el backend se lo impide.

- **Vía `create_order`** (camino 2, sin caller conocido en la UI hoy según
  `flujo-pedido-qr.md` §12/H1, pero funcionalmente activo): `load_valid_options(db,
  ["chocolate"], variant=variant)` → `validate_option_selection` calcula que "Sabores" es
  un grupo obligatorio (`min_select=2`) que descuenta inventario, así que por
  `_exige_maximo` exige exactamente el máximo (`max_select=3`) — con solo 1 sabor
  enviado, dispara: `"exige exactamente 3 opción(es), se enviaron 1"`, marcado como
  bloqueante (está en `consumen`) sin importar `STRICT_OPTION_SELECTION` → **422,
  rechazado**. La comanda no se crea.
- **Vía `add_item_to_table`** (camino 3, el que sí tiene botón real en la terminal del
  mesero según `flujo-pedido-qr.md`): `load_valid_options(db, ["chocolate"])` — sin
  `variant` — `validate_option_selection` nunca se ejecuta. La única opción enviada existe
  y está activa, así que se acepta sin más. El `OrderItem` se crea con un solo sabor.

**Efecto sobre el inventario**: al descontar (`deduct_order_items` →
`plan_line_consumption`), como el grupo "Sabores" tiene `quantity_per_option=120`
(el tamaño manda), se descuentan 120 g **por cada opción efectivamente elegida** — en este
caso, 120 g de un solo sabor, no los 360 g (3 × 120 g) que corresponden a una "Copa Grande
de 3 bolas" servida de verdad en el mostrador. La comanda se cobra al precio completo de la
"Copa Grande" (el precio no depende de cuántas opciones se eligieron, solo de la variante)
mientras el inventario solo registra el consumo de una bola. El kardex de insumos queda
sobrestimado en silencio — exactamente el problema que el propio módulo describe en el
docstring de `ensure_lines_consume_inventory` ("sobrestima el inventario en silencio hasta
el conteo físico... es peor que un error, porque nadie se entera").

## 4. Cuándo se manifiesta y cuándo coinciden

Coinciden siempre que la selección de opciones que llega a `add_item_to_table` ya sea
válida por sí misma (el número correcto de sabores, ninguna opción de un grupo ajeno) —
algo que depende enteramente de que la interfaz del mesero construya bien la petición, sin
ningún respaldo del servidor. Divergen exactamente cuando la petición violaría
`min_select`/`max_select` o incluye una opción de un grupo que la variante no ofrece: en
`create_order` esa petición se rechaza con 422; por el camino 3, que es el que
efectivamente usa el mesero en el día a día según la evidencia de `flujo-pedido-qr.md`, se
acepta sin aviso. Que nadie lo haya detectado es coherente con que la interfaz normalmente
no deje construir una selección inválida — pero el backend, que es la última línea de
defensa contra un bug de UI, un cliente API distinto, o una petición manual, no la tiene en
este camino.

## 5. Historia probable — reconstruida con `git log`

A diferencia de los demás hallazgos de este documento, aquí no hace falta especular: la
secuencia de commits deja rastro exacto.

1. **`a95fe31`** (2026-07-17, autor Deimer Hernandez) — introduce tanto
   `add_item_to_table` como `load_valid_options` (sin el parámetro `variant` todavía; en
   ese momento la función no podía validar selección de grupos, así que no había
   divergencia posible).
2. **`03469ca`** (2026-08-03, mismo autor, "Refactor option selection and inventory
   deduction logic") — añade el parámetro `variant` a `load_valid_options` y
   `validate_option_selection` como pieza central de la corrección. El propio mensaje del
   commit dice: "Updated `checkout` process to validate option selections against
   variant-specific rules, ensuring compliance with `min_select` and `max_select`
   constraints." El diff de este commit **sí** actualiza `add_item_to_table` para pasar
   `variant=variant` (`git show 03469ca -- app/api/v1/orders/consolidation.py`) — en este
   punto de la historia, la Contradicción 05 **no existía**: ambos caminos validaban igual.
3. **`ee94f30`** (2026-08-04, un día después, autor **distinto**: "LeonardoGomezz") —
   "feat(promotions): add promotions and combos module". Esta rama añadió soporte de
   combos a `add_item_to_table`, reescribiendo la función para separar el caso `combo_id`
   del caso variante+opciones normales. El diff de este commit muestra que **reintroduce**
   la llamada sin `variant`: la línea que cambia es
   `-    options = load_valid_options(db, data.option_ids)` →
   `+        options = load_valid_options(db, data.option_ids)` (el mismo texto, solo
   reindentado dentro del nuevo bloque `else`) — es decir, la rama de promociones se
   desarrolló partiendo de una copia de `consolidation.py` **anterior** a la corrección de
   `03469ca` (probablemente creada en paralelo, antes de que esa corrección se integrara a
   la rama compartida), y al fusionarse trajo consigo la versión vieja de esa línea sin que
   git marcara conflicto — el resto del fichero sí incorporó cambios reales (la
   estructuración `if combo_id / else`), pero esa línea específica retrocedió a su estado
   anterior a la corrección de un día antes.

Es, con evidencia directa y no una suposición, una **regresión por fusión entre ramas
paralelas**: una corrección puntual (`03469ca`) y una funcionalidad nueva desarrollada en
paralelo por otra persona (`ee94f30`) tocaron la misma línea partiendo de bases de código
distintas, y el resultado de integrarlas conservó la versión sin la corrección. `git
log --oneline -- app/api/v1/orders/consolidation.py` (que usa simplificación de historial)
ni siquiera muestra `03469ca` como un commit que tocó el fichero, lo que probablemente
contribuyó a que nadie relacionara visualmente ambos cambios al revisar la historia del
fichero superficialmente.

---

**Pregunta abierta al negocio**: ¿se ha detectado alguna vez, en el conteo físico de
inventario, un desfase que apunte a ítems de sabor múltiple (copas de varias bolas,
productos con extras obligatorios) consumiendo menos insumo del que deberían? Esto podría
ser evidencia indirecta de que este camino ya se ha usado con selecciones incompletas en
producción.
