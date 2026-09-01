# Contrato: gating por plan de `tracks_inventory` y de los campos de insumo de una opción

Cubre FR-006 a FR-014 sobre endpoints que ya existen. A diferencia de
`specs/062-gating-inventario-ajustes-reportes/contracts/inventario-module-gating.md` (que gatea
**rutas completas**), este contrato gatea **campos concretos** dentro de endpoints que deben seguir
funcionando sin el módulo Inventario — ver research.md, Decisión 4.

## `POST /products`

- **Request** — `ProductCreate`, sin cambio de forma. Gana una regla de validación:
  ```json
  { "name": "Copa Grande", "category_id": "…", "tracks_inventory": true }
  ```
  Si `tracks_inventory == true` y el plan vigente del tenant no incluye el módulo Inventario (o
  está vencido) → `403 "Tu plan actual no incluye el módulo de inventario."` (FR-011/FR-012). Con
  `tracks_inventory == false` (o ausente, default `false` desde spec 027), sin cambio — el producto
  se crea con normalidad sin importar el plan del tenant.
- **Efecto**: `ProductService.create_product` (`service.py:55-65`) llama
  `ensure_module_access(db, tenant, "inventario")` antes de construir el `Product`, solo cuando
  `data.tracks_inventory is True`. Requiere `tenant` en el router — `create_product` ya lo recibe
  (`products/router.py:79`).

## `PATCH /products/{id}` (y su alias `PUT /products/{id}`)

- **Request** — `ProductUpdate`, sin cambio de forma:
  ```json
  { "tracks_inventory": true }
  ```
  Mismo criterio que `POST /products`: activar (`true`) exige el módulo; desactivar (`false`) o no
  tocar el campo (ausente/`null`) nunca lo exige.
- **Efecto**: `ProductService.update_product` (`service.py:87-109`) llama
  `ensure_module_access(db, tenant, "inventario")` solo cuando `data.tracks_inventory is True` y
  distinto del valor actual del producto — reactivar un switch que ya estaba en `true` no
  reevalúa el plan en cada `PATCH` no relacionado (evita un `403` sorpresivo en una edición que ni
  siquiera toca este campo). El router gana `tenant: Tenant = Depends(get_tenant)` (hoy no lo
  recibe, `products/router.py:104-112`).
- **Sin bloqueo retroactivo**: un producto que ya tenía `tracks_inventory=true` **antes** de que al
  tenant se le retirara el acceso al módulo conserva ese valor en la base de datos sin ningún
  `UPDATE` forzado — el sistema no lo apaga automáticamente (FR-013). Ese producto simplemente se
  comporta, mientras dure la restricción, como si no manejara inventario en el momento de vender
  (`_tracks_inventory` ya solo lee `Product.tracks_inventory`, y esta spec no le agrega ningún
  chequeo de plan — sería redundante: el gating de acceso vive en la escritura, no en la lectura de
  venta, igual que el resto de superficies de spec 062 leen el dato existente sin bloquearlo).

## `POST /option-groups/{group_id}/options`

- **Request** — `OptionCreate`, sin cambio de forma:
  ```json
  { "name": "Fresa", "inventory_item_id": "…", "item_quantity": 80 }
  ```
  Si `inventory_item_id is not None` **o** `item_quantity > 0`, y el plan vigente del tenant no
  incluye el módulo Inventario (o está vencido) → `403 "Tu plan actual no incluye el módulo de
  inventario."` (FR-011/FR-012). Una opción sin insumo y con `item_quantity` en `0` (el caso de
  un topping puramente de precio) se crea con normalidad sin importar el plan.
- **Efecto**: el endpoint gana `tenant: Tenant = Depends(get_tenant)` (hoy no lo recibe,
  `catalog/router.py:347-354`) y llama `ensure_module_access(db, tenant, "inventario")` antes de
  construir el `Option`, condicionado a la regla anterior.

## `PATCH /options/{option_id}`

- **Request** — `OptionUpdate`, sin cambio de forma. Misma regla que `POST .../options`, evaluada
  sobre el **resultado final** de la opción tras aplicar el `PATCH` (no sobre el payload aislado):
  enlazar un insumo nuevo, o subir `item_quantity` por encima de `0` sobre una opción que hoy no
  consume nada, exige el módulo; desvincular el insumo (`inventory_item_id: null`, que ya fuerza
  `item_quantity=0` por `RN-CAT-38`) nunca lo exige, sin importar el plan.
- **Efecto**: el endpoint gana `tenant: Tenant = Depends(get_tenant)` (hoy no lo recibe,
  `catalog/router.py:380-388`).
- **Sin bloqueo retroactivo**: una opción que ya tenía insumo enlazado antes de que se retirara el
  acceso conserva sus datos — el `PATCH` solo se bloquea si el body intenta **agregar o aumentar**
  consumo de inventario mientras el módulo no está disponible, no por el mero hecho de editar otro
  campo (ej. `name`) de una opción que ya tenía insumo de antes.

## Sin cambios

- `quantity_per_option` en el guardado unificado de producto (`POST`/`PATCH /products`, campo
  `variants[].option_groups[].quantity_per_option`, spec 043): **sin gating de plan nuevo**
  (research.md Decisión 4) — ya es inerte para cualquier producto con `tracks_inventory=false`, y
  `tracks_inventory=true` ya está cubierto por el primer contrato de este documento para ese mismo
  tenant.
- `GET /products`, `GET /products/{id}`, `GET /option-groups`, `GET /variants/{id}/option-groups`:
  sin ningún gating nuevo de lectura — mismo criterio que spec 027 y spec 062, que tampoco ocultan
  datos ya guardados al consultar, solo restringen el guardado de valores nuevos que requieran el
  módulo.
- El mensaje de denegación (`"Tu plan actual no incluye el módulo de inventario."` /
  `"Tu plan venció. Debe renovarse para seguir usando el sistema."`) es exactamente el mismo texto
  ya definido en `app/core/plan_limits.py` (spec 033) — esta spec no introduce ningún mensaje
  nuevo ni variante.
