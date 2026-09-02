# Quickstart: validar la selección por cantidad en grupos de opciones

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite lo ya detallado
en [data-model.md](./data-model.md) y `contracts/` — solo enlaza a ellas.

**Prerequisitos**: `pos-backend` con el venv activado (`source env/bin/activate`), `pos-heladeria`
con `npm install` ya corrido. La rama de trabajo parte de `064-grupos-opciones-precio-inventario`
en ambos repos (research.md, "Nota sobre numeración de ramas") — confirmar que
`OptionGroup.pricing_type` ya existe antes de empezar.

```bash
cd ../pos-backend
source env/bin/activate
```

## Paso 0 — Confirmar la línea base antes de tocar código

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: todo en verde (553 tests al momento de escribir esta spec, spec 063/064 ya
mergeada). Ninguno asume `option_ids` como único formato de selección — si alguno construye el
payload de `CartItemIn`/`OrderItemCreate` a mano con esa forma, será el primero en fallar tras
cambiar el schema (Fase de implementación, no de este quickstart).

## US1 — Declarar el modo de selección de un grupo

1. `POST /option-groups` sin `selection_mode` → `pricing_type` aparte, confirmar
   `GET` devuelve `"selection_mode": "conteo"` (Acceptance Scenario 1,
   [contracts/option-group-selection-mode.md](./contracts/option-group-selection-mode.md)).
2. `POST /option-groups` con `"selection_mode": "cantidad"` → confirmar que se crea así, y que
   `min_select`/`max_select` enviados no producen ningún efecto de validación al vender (verificar
   en US2/US3, no aquí).
3. `PATCH /option-groups/{id}` cambiando el modo de un grupo existente → confirmar que aplica de
   inmediato a la siguiente venta, sin tocar ventas ya confirmadas (Acceptance Scenario 3).

## US2/US3 — Cantidad libre por opción, precio y consumo multiplicados

Backend — nuevo archivo de characterization tests (mismo patrón de invocación directa que
`test_catalog_option_groups_pricing_type.py`, spec 063):

1. Grupo "cantidad" con "Bobombún" (`extra_price=1000`) y "Gomitas" (`extra_price=800`). Enviar
   `options=[{option_id: bobombun, quantity: 2}, {option_id: gomitas, quantity: 1}]` a
   `load_valid_options` → confirmar `compute_line_price` da `variant.price + 2000 + 800`
   (Acceptance Scenario 1 de US3).
2. Mismo caso, con el producto manejando inventario y ambas opciones con insumo enlazado →
   `plan_line_consumption` genera una línea de consumo por opción, cada una multiplicada por su
   `quantity` (Acceptance Scenario 2).
3. Grupo "cantidad" marcado además "Incluido" (spec 063/064) → `compute_line_price` no suma ningún
   recargo sin importar `quantity` (Acceptance Scenario 3, FR-005).
4. Grupo "conteo" (sin cambios) con una opción enviada con `quantity: 2` → `422` (research.md
   Decisión 3) — confirma que el modo "conteo" no admite más de una unidad por opción.
5. Grupo "cantidad" con todas las cantidades en `0` (arreglo `options` vacío o sin ninguna entrada
   de ese grupo) → la venta se permite, sin ningún bloqueo (Acceptance Scenario 2 de US2, FR-003).

Frontend — `product-select.component.spec.ts` (ajuste): con un grupo `selection_mode: 'cantidad'`,
verificar que aparecen controles `+`/`-` por opción (no botones toggle), que incrementar dos
opciones distintas queda reflejado en `ProductSelection.options` con sus cantidades, y que
`canConfirm()` nunca se bloquea por ese grupo estando en 0 (research.md Decisión 8).

```bash
cd ../pos-heladeria
npx ng test --watch=false
```

## US4 — Topes de cantidad

1. Grupo "cantidad" con `max_quantity_per_option=3`: intentar `quantity=4` en una opción → rechazo
   (backend: `422`; frontend: el botón `+` se deshabilita al llegar a 3 — Acceptance Scenario 1).
2. Mismo grupo con `max_total_quantity=5`: sumar opciones hasta 5 e intentar agregar una más →
   rechazo, aunque ninguna opción individual llegue a su propio tope (Acceptance Scenario 2).
3. Grupo "cantidad" sin ningún tope configurado → sin límite propio; si el producto maneja
   inventario, el chequeo de stock (`RN-CAT-24`, sin cambio de forma) es el único límite real
   (Acceptance Scenario 3).

## US5 — Comanda, "Mis pedidos" y recibo muestran la cantidad

1. Confirmar un pedido con "Bobombún" cantidad 2 y "Gomitas" cantidad 1.
2. Verificar, en la comanda de cocina (`pos-terminal.store.ts`), el detalle de orden
   (`order-detail.component.ts`), "Mis pedidos" (`public-menu.component.ts`) y el recibo
   (`receipt.util.ts`), que las cuatro superficies muestran "2x Bobombún" (o equivalente) de forma
   consistente — todas usando `MenuLookupIndex.optionLabelWithQuantity` (research.md Decisión 9).
3. Repetir con un grupo "conteo" sin cambios → se ve exactamente igual que antes de esta spec (un
   nombre, sin prefijo de cantidad).

## US6 — Migración no destructiva

```bash
alembic upgrade head
```

1. Confirmar que todo grupo de opciones existente queda con `selection_mode='conteo'`,
   `max_quantity_per_option=NULL`, `max_total_quantity=NULL`.
2. Confirmar que toda fila existente de `cart_item_options`/`order_item_options` queda con
   `quantity=1`.
3. Vender una presentación que use un grupo migrado → mismo precio y mismo consumo que antes de la
   migración (SC-006).

## Verificación manual end-to-end (no automatizable sin navegador real)

1. Como Tenant Admin, crear un grupo "Toppings" en modo "cantidad" con tope de 3 por topping y 5 en
   total, con "Bobombún" y "Gomitas" con recargo.
2. Como comensal en el menú QR, abrir "Banana Split Especial", subir "Bobombún" a 2 y "Gomitas" a
   1, confirmar que el precio del pie de página refleja el recargo correcto, y agregar al carrito.
3. Confirmar el pedido y revisar la comanda de cocina y "Mis pedidos" — ambos deben decir "2x
   Bobombún, Gomitas" (o formato equivalente).
4. Cobrar el pedido y revisar el recibo impreso — misma cantidad reflejada; revisar el reporte de
   ventas y confirmar que el detalle de la línea conserva la cantidad por topping.
