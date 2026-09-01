# Quickstart: validar tipo de precio e inventario condicional en grupos de opciones

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite lo ya detallado
en [data-model.md](./data-model.md) y `contracts/` — solo enlaza a ellas. Reutiliza los fixtures de
catálogo (`app/characterization_tests/fixtures.py`) y de plan
(`app/characterization_tests/plan_fixtures.py`) ya existentes — esta spec no crea fixtures propios.

**Prerequisitos**: `pos-backend` con el venv activado (`source env/bin/activate`), `pos-heladeria`
con `npm install` ya corrido. No hace falta PostgreSQL real para los tests automatizados (SQLite en
memoria vía fixtures existentes); sí hace falta para probar la migración real.

```bash
cd ../pos-backend
source env/bin/activate
```

## Paso 0 — Confirmar la línea base antes de tocar código

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: todo en verde. Anotar cuáles de `test_catalog_line_pricing.py` y
`test_catalog_consumption_plan.py::EnsureLinesConsumeInventoryTests` hoy fijan el criterio de una
sola condición de `grupos_que_descuentan` — esos son los que la Decisión 3 de research.md va a
tocar deliberadamente (no accidentalmente).

## US1/US2 — Tipo de grupo "Incluido"/"Con recargo"

Backend — extender/crear tests en `app/characterization_tests/` (nuevo archivo o sección dentro de
uno existente de catálogo):

1. Crear un `OptionGroup` con `pricing_type="incluido"`, agregar una opción con `extra_price=500`
   vía `add_option` → `422` (Acceptance Scenario 1 de US1,
   [contracts/option-group-pricing-type.md](./contracts/option-group-pricing-type.md)).
2. La misma opción con `extra_price=0` → `201`, se crea con normalidad.
3. Crear un `OptionGroup` con `pricing_type="con_recargo"`, agregar una opción con
   `extra_price=500` → `201` (Acceptance Scenario 1 de US2).
4. `PATCH /option-groups/{id}` de un grupo "Con recargo" con al menos una opción en `extra_price>0`
   a `pricing_type="incluido"` → `200`, y `GET` posterior de esa opción confirma `extra_price=0`
   (Acceptance Scenario 3 de US2, FR-004).
5. Vender una variante con una opción de un grupo "Incluido" elegida → el precio de línea no suma
   ningún recargo (Acceptance Scenario 2 de US1, reutiliza `compute_line_price` sin cambios).

Frontend — nuevos specs para `option-group-form.component.spec.ts` (hoy no existe):

6. Con `pricing_type` no seleccionado, el formulario no permite enviar (validación de campo
   requerido, FR-001).
7. Con `pricing_type="incluido"` elegido, `option-form.component.spec.ts` (hoy no existe) confirma
   que el campo de precio no acepta edición (o no se renderiza) al agregar una opción a ese grupo.
8. Cambiar un grupo con precios existentes a "Incluido" dispara `ConfirmService.ask` antes de
   guardar (Acceptance Scenario 3 de US2); cancelar la confirmación no envía el `PATCH`.

```bash
cd ../pos-heladeria
npx ng test --watch=false
```

## US3 — Cascada del switch "maneja inventario" en los grupos de opciones de un producto

Backend — sin test nuevo necesario: `_tracks_inventory` (`app/catalog_engine/consumption.py:163-171`)
y `plan_line_consumption`/`ensure_lines_consume_inventory` no cambian de forma (research.md
Decisión 4, "sin gating nuevo para `quantity_per_option`") — el comportamiento de venta con
`tracks_inventory=False` ya está cubierto por `test_catalog_consumption_plan.py` (spec 027,
`spec_027_fr005`/`fr006`/`fr008`).

Frontend — ajustes a `product-form.component.spec.ts` (existente, ya tiene los tres tests de
spec 027 citados en research.md, Decisión 5):

1. Con `draft().tracks_inventory = false`, el selector de grupo (`option_group_id`) y los campos
   `min_select`/`max_select` de "Sabores a elegir" **sí** aparecen y son editables — a diferencia
   del comportamiento anterior a esta spec, donde toda la sección desaparecía (Acceptance Scenario
   1 de US3, la restructuración documentada en research.md Decisión 5).
2. Con el mismo estado, el input "descuenta [cantidad] por cada uno" y el resumen "Descuenta de: …"
   **no** aparecen.
3. Vender esa presentación con una opción de ese grupo elegida no genera ningún movimiento de
   inventario (Acceptance Scenario 2 de US3 — verificado en backend, no en este spec de
   componente).
4. Con `draft().tracks_inventory = true`, ambos bloques (selector de grupo y cantidad de consumo)
   aparecen — comportamiento sin cambio respecto a hoy.

```bash
cd ../pos-heladeria
npx ng test --watch=false
```

## US4 — Un solo criterio decide si una opción descuenta inventario (corrige A-32)

Backend — extender `test_catalog_line_pricing.py` con el caso exacto de la discrepancia:

1. Crear una opción con `item_quantity=10` e `inventory_item_id=None` dentro de un grupo
   obligatorio. Invocar `grupos_que_descuentan` → **antes** de esta spec devolvía ese grupo como
   "descuenta"; **después**, no lo devuelve (Acceptance Scenario 1 de US4, research.md Decisión 3).
2. La misma opción, ahora con `inventory_item_id` enlazado y `active=True` → `grupos_que_descuentan`
   y `group_discounts` coinciden en "sí descuenta" (Acceptance Scenario 2).
3. Vender una variante sin receta fija cuyo único grupo tiene una opción a medio configurar (caso
   1) → el mensaje de rechazo de `ensure_lines_consume_inventory` ya decía correctamente "sin
   insumo enlazado" (esa función no cambia); confirmar que `validate_option_selection` ya no exige
   "elige exactamente el máximo" sobre ese mismo grupo a medio configurar antes de llegar a la venta
   (Acceptance Scenario 3 de US4, FR-010).

**Tarea de trazabilidad obligatoria** (no un test): agregar la entrada de resolución de A-32 en
`specs/000-reconocimiento/registro-de-anomalias.md` (research.md Decisión 7) — verificar que existe
antes de dar por completa esta historia (Principio II de la Constitución).

```bash
python -m unittest discover -s app/characterization_tests -p 'test_catalog_line_pricing.py' -v
python -m unittest discover -s app/characterization_tests -p 'test_catalog_consumption_plan.py' -v
```

## US5 — Gating por plan del switch de inventario y de los campos de insumo

Backend — nuevo archivo o sección en `app/characterization_tests/`, mismo patrón directo de
invocación que `test_plan_module_access.py`:

1. Tenant con `inventario_access=False` (fixture `plan_fixtures.make_plan`): `POST /products` con
   `tracks_inventory=true` → `403` (Acceptance Scenario 1 de US5,
   [contracts/inventory-field-plan-gating.md](./contracts/inventory-field-plan-gating.md)). Con
   `tracks_inventory=false` (o ausente) → `201`, sin importar el plan.
2. Mismo tenant: `POST /option-groups/{id}/options` con `inventory_item_id` enlazado → `403`. Con
   `extra_price=500` y sin `inventory_item_id` (un topping puro) → `201` (Acceptance Scenario 2).
3. Tenant con un producto que ya tenía `tracks_inventory=true` y opciones con insumo **antes** de
   perder el acceso al módulo: tras retirárselo, `GET /products/{id}` sigue devolviendo
   `tracks_inventory: true` y las opciones conservan su `inventory_item_id` — nada se borra
   (Acceptance Scenario 3, FR-013). Vender esa presentación no genera movimiento de inventario ni
   se rechaza por falta de receta mientras dure la restricción (mismo criterio de spec 027 para
   `tracks_inventory=false`, aplicado aquí por ausencia de módulo en vez de por el switch).
4. Al Super Admin reasignarle el módulo a ese tenant, el mismo `POST /option-groups/{id}/options`
   con insumo enlazado del paso 2 ahora responde `201` (Acceptance Scenario 4, FR-014).

Frontend — nuevos specs, mismo patrón que `settings-page.component.spec.ts` (mock de
`PlanSummaryService.summary` como señal):

5. `product-form.component.spec.ts` (ajuste): con `modules.inventario=false`, el switch "Maneja
   inventario" aparece deshabilitado — pero un producto existente con `tracks_inventory=true` sigue
   mostrando ese valor, no se fuerza a `false` en el formulario (Acceptance Scenario 1/3, gating
   fail-closed vía `inventarioIncluido()`, research.md Decisión 5).
6. `option-group-form.component.spec.ts`/`option-form.component.spec.ts` (nuevos): con
   `modules.inventario=false`, los campos "Insumo que consume"/"Cantidad consumida" no aparecen,
   sin importar qué producto use ese grupo (Acceptance Scenario 2).

```bash
cd ../pos-heladeria
npx ng test --watch=false
```

## US6 — Migración no destructiva de grupos existentes

Verificación de la migración contra una base real (no characterization test — mismo criterio que
`verify_tracks_inventory_backfill.py` de spec 027):

```bash
cd ../pos-backend
source env/bin/activate
alembic upgrade head
```

1. Sobre un tenant de prueba con un grupo "Toppings" con al menos una opción en `extra_price>0`:
   tras la migración, `GET /option-groups` devuelve `pricing_type: "con_recargo"` para ese grupo,
   con los mismos precios de antes (Acceptance Scenario 1 de US6).
2. Sobre un grupo "Sabores" con todas sus opciones en `extra_price=0`: tras la migración,
   `pricing_type: "incluido"` (Acceptance Scenario 2).
3. Vender una presentación que use cualquiera de los dos grupos migrados: precio de línea y
   movimientos de inventario resultantes son idénticos a los de antes de la migración (Acceptance
   Scenario 3, SC-002) — comparar contra una captura tomada antes de `alembic upgrade head`.

## Verificación manual end-to-end (no automatizable sin navegador real)

1. Como Tenant Admin con plan sin Inventario: crear un producto "Copa sencilla" con
   `tracks_inventory` deshabilitado en pantalla; asignarle un grupo "Toppings" (Con recargo) con
   precio; guardar y confirmar que el producto se vende con normalidad en el menú QR sin ningún
   movimiento de inventario (SC-005).
2. Al Super Admin reasignarle el módulo Inventario a ese tenant: confirmar que el switch "Maneja
   inventario" del mismo producto vuelve a estar disponible, y que el editor de "Grupos de
   opciones" en Ajustes vuelve a mostrar los campos de insumo (SC-006), sin cerrar sesión.
3. Como Tenant Admin con el módulo Inventario incluido: crear un grupo "Sabores" (Incluido) con
   insumo enlazado por opción, asignarlo a una variante sin marcar "descuenta" alguna cantidad —
   confirmar que el precio de línea nunca suma recargo por ese grupo, y que la venta sí descuenta
   el insumo de la opción elegida (Edge Case de `spec.md`: "Incluido" solo bloquea precio, no
   inventario).
