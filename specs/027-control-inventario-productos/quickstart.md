# Quickstart: Control de Inventario por Producto (Switch de Insumos)

Validación ejecutable por historia de usuario. Los comandos asumen que se ejecutan desde la raíz de
cada repo (`../pos-backend`, `../pos-heladeria`, siblings de `pos-specs`).

## Prerrequisitos

- `pos-backend`: entorno virtual activado (`source env/bin/activate`), migración de esta
  funcionalidad ya aplicada (`alembic upgrade head`) sobre la base de pruebas — los
  characterization tests usan SQLite en memoria vía `fixtures.py`, no requieren Postgres real para
  correr, pero la migración sí debe existir para validar Historia 4 contra Postgres real (ver más
  abajo).
- `pos-heladeria`: `npm install` ya ejecutado.
- Spec, plan, research y data-model de esta funcionalidad ya revisados — en particular
  research.md Decisión 3 (el default del modelo ORM es `True`, no `False`) antes de tocar
  `app/models/product.py`, porque invertirlo rompe en silencio la suite de characterization tests.

## Historia 1 — Crear un producto sin inventario, sin ningún insumo

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_catalog_consumption_plan -v
```

- **Esperado (casos existentes, sin modificar)**: los 4 tests ya existentes de
  `EnsureLinesConsumeInventoryTests` siguen en verde sin cambios — sus fixtures no mencionan
  `tracks_inventory`, heredan `True` del modelo (research.md Decisión 3).
- **Esperado (caso nuevo, agregado en esta implementación)**: crear un `Product(tracks_inventory=False)`
  con una variante sin receta ni grupos, invocar `ensure_lines_consume_inventory` sobre esa línea, y
  confirmar que **no** lanza `HTTPException` — a diferencia del caso RN-CAT-34 ya existente sobre un
  producto con `tracks_inventory=True` (o sin especificar, que hereda `True`).
- **Esperado (descuento real)**: invocar `plan_line_consumption` directamente sobre esa misma
  variante y confirmar que devuelve `[]` — cero `ConsumptionLine`, sin importar la cantidad vendida.

```bash
cd ../pos-heladeria
npx ng test --watch=false --include='**/product-form.component.spec.ts'
```

*(este archivo de spec no existe todavía — se crea como parte de la implementación de esta
historia; hoy `product-form.component.ts` no tiene cobertura de test propia. Nota: este proyecto
usa `@angular/build:unit-test` — `npx vitest run` sin pasar por `ng test` no carga el entorno de
Angular y falla con "describe is not defined").

- **Esperado**: al abrir el formulario de un producto nuevo, `draft().tracks_inventory` es `false`
  y los bloques "Insumos fijos"/"Sabores a elegir" se muestran deshabilitados (research.md Decisión
  7); guardar sin ningún insumo no produce ningún error de validación del formulario.

## Historia 2 — Activar el switch y asociar insumos

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_catalog_consumption_plan -v
```

- **Esperado (caso nuevo)**: `Product(tracks_inventory=True)` con una variante sin receta ni
  grupos configurados sigue rechazándose con `409` "no tiene receta configurada" — activar el
  switch, por sí solo, no exime de `RN-CAT-34` (contrato de `sale-consumption-guard.md`).
- **Esperado (caso nuevo, aviso de guardado)**: no es verificable por `unittest` — cubierto por el
  spec de Angular de abajo (FR-013 es responsabilidad exclusiva del frontend, research.md Decisión
  8).

```bash
cd ../pos-heladeria
npx ng test --watch=false --include='**/product-form.component.spec.ts'
```

- **Esperado**: al activar el switch, "Insumos fijos"/"Sabores a elegir" pasan de deshabilitados a
  editables; al intentar guardar sin haber agregado ningún insumo, aparece el banner ámbar de
  advertencia (research.md Decisión 8) antes de que la petición HTTP salga.

## Historia 3 — Cambiar el switch de un producto existente sin perder insumos

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_catalog_consumption_plan -v
```

- **Esperado (caso nuevo, el más importante de esta spec)**: `Product(tracks_inventory=False)` con
  una variante que **sí** tiene `RecipeItem` guardados (simulando "tenía insumos, se apagó el
  switch después") — `ensure_lines_consume_inventory` no rechaza la venta, y `plan_line_consumption`
  sigue devolviendo `[]` a pesar de que `load_recipe` encontraría filas. Este es el caso que
  research.md Decisión 2 identificó como el que un parche incompleto (solo en el guard) dejaría
  roto.

```bash
cd ../pos-heladeria
npx ng test --watch=false --include='**/product-form.component.spec.ts'
```

- **Esperado**: apagar el switch de un producto con insumos ya guardados dispara
  `ConfirmService.ask(...)` antes de guardar; cancelar la confirmación deja `draft().tracks_inventory`
  en `true` sin enviar ningún `PATCH`; aceptar la confirmación envía el `PATCH` y deshabilita la
  sección sin vaciar `av.recipe`/`av.optionGroups` en memoria. Reactivar el switch después no
  dispara ninguna confirmación y muestra los mismos insumos sin pedir que se recapturen.

## Historia 4 — Productos existentes migran sin romperse

```bash
cd ../pos-backend
python -m app.scripts.verify_tracks_inventory_backfill
```

- **Esperado (antes de migrar)**: `✔ ... heurística SQL y Python coinciden` para cada tenant — sin
  discrepancias entre la consulta `EXISTS` del backfill y `load_recipe`/`group_discounts` reales.

```bash
cd ../pos-backend
alembic upgrade head
```

- **Esperado**: la migración corre sin error sobre una base con datos de prueba que incluya al
  menos un producto con receta ya configurada y uno sin nada configurado (ver seed de
  `pos-backend`). Verificación manual con `psql`/cliente SQL:
  ```sql
  SELECT p.name, p.tracks_inventory
  FROM heladeria.products p  -- o el schema real del tenant, no "tenant" literal
  ORDER BY p.name;
  ```
- **Esperado**: todo producto que ya tenía receta fija o grupo con consumo en alguna presentación
  activa queda con `tracks_inventory = true`; todo producto sin nada configurado (hoy bloqueado por
  `RN-CAT-34`) queda con `tracks_inventory = false`.

```bash
cd ../pos-backend
python -m unittest discover app/characterization_tests -v
```

- **Esperado**: toda la suite de characterization tests sigue en verde — en particular
  `test_catalog_consumption_plan.py` (los 4 casos ya existentes) y `golden_master_core.py`
  (líneas 63-304, construyen variantes con receta ya configurada) no cambian de resultado, porque
  sus fixtures heredan `tracks_inventory=True` del modelo (research.md Decisión 3).

## Regresión general

```bash
cd ../pos-backend
python -m unittest discover app/characterization_tests -v
```

- **Esperado**: ningún módulo fuera de `test_catalog_consumption_plan.py` cambia de resultado — el
  cambio vive enteramente dentro de `app/catalog_engine/consumption.py`
  (research.md Decisión 1/2/10), sin tocar `orders/consumption.py` ni `sales/consumption.py`.

```bash
cd ../pos-heladeria
npx ng test --watch=false
```

- **Esperado**: el resto de la suite de Angular no cambia de resultado — solo se agrega
  `product-form.component.spec.ts` (nuevo) y se extienden las interfaces de `Product` sin romper
  ningún consumidor existente de esos tipos.
