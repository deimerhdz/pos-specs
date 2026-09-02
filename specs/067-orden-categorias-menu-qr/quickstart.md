# Quickstart: Orden personalizado de categorías en el filtro del menú QR

Validación ejecutable por historia de usuario. Los comandos asumen que se ejecutan desde la raíz de
cada repo (`../pos-backend`, `../pos-heladeria`, siblings de `pos-specs`).

## Prerrequisitos

- `pos-backend`: entorno virtual activado, migración de esta funcionalidad ya aplicada
  (`alembic upgrade head`) — agrega `categories.display_order` y su `CheckConstraint`
  (data-model.md, research.md Decisión 5). `down_revision='a96852d7be6a'` (confirmado como `head`
  real al planear esta spec, 2026-09-01 — reverificar si otras specs agregaron migraciones nuevas
  entre tanto).
- `pos-heladeria`: `npm install` ya ejecutado. Sin dependencias nuevas.
- Spec, plan, research y data-model de esta funcionalidad ya revisados — en particular research.md
  Decisión 4 (la fórmula exacta del backfill, `ORDER BY name DESC`) antes de tocar la migración.

## Historia 1 — Definir el orden al crear o editar una categoría

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_category_display_order -v
```

- **Esperado (crear con valor explícito)**: `POST /categories` con `"display_order": 10` crea la
  categoría con ese valor exacto (contracts/categorias-orden.md).
- **Esperado (crear sin valor)**: `POST /categories` sin `display_order` (u omitiéndolo) asigna
  `MAX(display_order) + 1` entre todas las categorías existentes — verificable creando dos
  categorías seguidas sin especificar el campo y comprobando que la segunda recibe un valor mayor
  que la primera (FR-004, data-model.md).
- **Esperado (editar)**: `PATCH /categories/{id}` con un nuevo `display_order` reemplaza el valor
  anterior; sin el campo en el body, el valor existente no cambia (FR-002).
- **Esperado (validación)**: `POST`/`PATCH` con `"display_order": -1` o un valor no numérico
  responde `422` sin crear/modificar la categoría (FR-003).

```bash
cd ../pos-heladeria
npx ng test --watch=false --include='**/category-form.component.spec.ts'
```

- **Esperado**: el formulario expone un campo numérico de orden en creación y edición; al editar
  una categoría existente, el campo se pre-llena con su `display_order` actual; guardar sin tocarlo
  no envía un valor distinto al que ya tenía.

## Historia 2 — El filtro del Menú QR se muestra ordenado de mayor a menor

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_category_display_order -v
```

- **Esperado**: con tres categorías activas de `display_order` 10, 5 y 1, `GET /menu` (o
  `GET /menu/qr-token/{token}`) devuelve el arreglo `categories`/`menu` en esa misma secuencia
  (10, 5, 1) — data-model.md, "Menú QR — consulta modificada".
- **Esperado (empate)**: dos categorías activas con el mismo `display_order` aparecen ordenadas
  entre sí alfabéticamente por `name` (FR-006).
- **Esperado (inactiva)**: una categoría inactiva con un `display_order` alto no aparece en la
  respuesta, sin importar su valor (FR-007, comportamiento ya existente, sin cambio).
- **Esperado (listado de administración, sin cambio)**: `GET /categories` sigue devolviendo las
  categorías ordenadas por `name`, no por `display_order` (data-model.md, "SIN CAMBIO DE ORDEN") —
  cubre la posible regresión de tocar la consulta equivocada.

## Historia 3 — Ver el orden configurado en la administración de categorías

```bash
cd ../pos-heladeria
npx ng test --watch=false --include='**/categories-page.component.spec.ts'
```

- **Esperado**: la tabla de administración de categorías muestra una columna con el
  `display_order` de cada fila, sin alterar el orden en que las filas ya se muestran (por nombre).

## Migración — no reordena categorías existentes al desplegar

```bash
cd ../pos-backend
# Antes de aplicar la migración: capturar el orden actual del Menú QR para un tenant con
# varias categorías activas (p. ej. vía GET /menu contra ese tenant).
alembic upgrade head
# Después: repetir la misma consulta GET /menu y comparar.
```

- **Esperado**: la secuencia de categorías que devuelve el Menú QR es idéntica antes y después de
  aplicar la migración — el backfill (`ROW_NUMBER() OVER (ORDER BY name DESC)`, research.md
  Decisión 4) reproduce exactamente el orden alfabético que ya existía (FR-009, SC-003).

## Verificación de no-regresión

```bash
cd ../pos-backend
python -m unittest discover app/characterization_tests -v
```

- **Esperado**: toda la suite de characterization tests existente sigue en verde sin
  modificación — esta spec no cambia ninguna regla de creación, precio, inventario ni promoción,
  solo agrega una columna nueva a `categories` y ordena por ella un único punto de lectura
  (`_build_menu`).
