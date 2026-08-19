# Contrato: exención de venta/descuento para productos sin inventario

Cubre FR-005, FR-006, FR-012 sobre el guard de venta y el descuento real de inventario
(`app/catalog_engine/consumption.py`, reexportado como fachada desde
`app/api/v1/catalog/consumption_plan.py`). **El request y el response de los endpoints de venta no
cambian de forma** — solo el efecto interno, igual que el estilo de contrato ya usado en spec 026.

## Cualquier endpoint que arme o cobre un pedido/venta (`orders/*`, `sales/*`)

- **Request/Response**: sin cambios de forma en ningún endpoint de `orders`/`sales` — mismo body,
  mismos campos.
- **Efecto (nuevo) — línea de un producto con `tracks_inventory=false`**:
  - El guard `ensure_lines_consume_inventory` **nunca** rechaza esa línea por "no tiene receta
    configurada" ni por "no se eligió ninguna opción" (los dos `409` de spec 003, RN-CAT-34/35),
    sin importar si la presentación vendida tiene o no insumos guardados de una configuración
    anterior (FR-008).
  - `plan_line_consumption` devuelve una lista vacía para esa línea — **cero** `ConsumptionLine`
    generadas, por lo que `deduct_order_item`/`deduct_sale` no aplican ningún movimiento de
    inventario para ella, y `reverse_order_item` tampoco tiene nada que revertir si esa orden se
    cancela después.
  - Ninguna respuesta HTTP marca esta línea como un caso especial (FR-012): el `200`/`201` de
    confirmar/cobrar es indistinguible, en su forma, del de cualquier otra línea.
- **Efecto (sin cambios) — línea de un producto con `tracks_inventory=true`**:
  - Idéntico a spec 003, sin ninguna modificación: si la presentación vendida no tiene receta fija
    ni grupo configurado que descuente, `409` "no tiene receta configurada"
    (`variantes_sin_receta`); si tiene un grupo configurado pero no se eligió nada de él, `409`
    "consume inventario según la opción que elija el cliente..." (`variantes_sin_opcion`); si sí
    tiene algo configurado y elegido, el descuento ocurre exactamente con las mismas cantidades y
    reglas de siempre (RN-CAT-17 a RN-CAT-26).

## Evaluación por presentación, no por producto (FR-006)

Un producto con `tracks_inventory=true` que tiene dos presentaciones — una con receta configurada y
otra sin nada — permite vender la primera con normalidad y sigue rechazando la segunda con el mismo
`409` de siempre. `tracks_inventory` decide **si** se evalúa la regla de spec 003 para las
presentaciones de ese producto; no decide, por sí sola, que todas pasen.

## Sin cambios

- El chequeo preventivo de disponibilidad de stock (`RN-CAT-24` a `RN-CAT-26`, spec 003) no se toca
  — simplemente nunca se ejecuta para una línea cuyo `required_consumption` ya viene vacío por
  `tracks_inventory=false` (el mismo camino que hoy sigue una línea sin nada que descontar).
- `PUT /variants/{id}/recipe`, los endpoints de grupos de opciones, y el chequeo de disponibilidad
  en sí (`check_availability`) no cambian de forma ni de comportamiento propio — ver
  `product-tracks-inventory-field.md`.
