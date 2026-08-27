# Quickstart: Orden de Presentaciones de un Producto

Validación ejecutable por historia de usuario. Los comandos asumen que se ejecutan desde la raíz de
cada repo (`../pos-backend`, `../pos-heladeria`, siblings de `pos-specs`).

## Prerrequisitos

- `pos-backend`: entorno virtual activado, migración de esta funcionalidad ya aplicada
  (`alembic upgrade head`) — agrega `product_variants.display_order` y su constraint diferible
  (data-model.md, research.md Decisión 5). `down_revision='187e491e597a'` (confirmado como `head`
  actual al planear esta spec, 2026-08-27 — reverificar si otras specs agregaron migraciones nuevas
  entre tanto).
- `pos-heladeria`: `npm install` ya ejecutado. `@angular/cdk` ya está en `package.json`
  (`^21.2.14`); esta es la primera funcionalidad que lo importa (research.md Decisión 6).
- Spec, plan, research y data-model de esta funcionalidad ya revisados — en particular research.md
  Decisión 2 (el reordenamiento se guarda dentro de `saveExistingProduct`, no de inmediato al
  soltar) antes de tocar `product-form.component.ts` o `product.service.ts`.

## Historia 1 — Reordenar presentaciones arrastrándolas en el formulario

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_product_variant_reorder -v
```

- **Esperado (caso nuevo)**: `PATCH /products/{id}/variants/reorder` con una lista de IDs en un
  orden distinto al actual asigna `display_order = 1..N` según esa lista, verificable leyendo las
  filas de `product_variants` después de la llamada (contracts/product-variants-reorder.md).
- **Esperado (validación)**: la misma llamada con un ID que no pertenece al producto, o con un ID
  repetido, o con un ID de una presentación desactivada, responde `422` sin modificar ninguna fila.

```bash
cd ../pos-heladeria
npx ng test --watch=false --include='**/product-form.component.spec.ts'
```

- **Esperado**: arrastrar una fila de `draft().variants` en el test (simulando el evento de
  `CdkDropList`) reordena el array local y el número visible de cada fila (índice + 1) se actualiza
  de inmediato, sin ninguna llamada HTTP todavía (el guardado ocurre al llamar `save()`, no al
  soltar — research.md Decisión 2).
- **Esperado (persistencia)**: al invocar `saveExistingProduct()` sobre un producto con el orden ya
  modificado en memoria, se dispara exactamente una llamada a `reorderVariants(productId, ids)`
  además de las llamadas ya existentes para nombre/precio/SKU — verificable con un espía sobre
  `product.service.ts`.

## Historia 2 — El orden definido se refleja en el detalle del producto en el Menú QR

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_product_variant_reorder -v
```

- **Esperado**: tras reordenar vía `PATCH /products/{id}/variants/reorder`, una consulta al mismo
  producto a través de `Product.variants` (la misma relación que usa `menu/router.py:110-114`)
  devuelve las presentaciones en el nuevo orden — sin necesidad de un test HTTP contra el router del
  menú, porque el `order_by` vive en la relación (data-model.md), no en ese endpoint específico.
- **Esperado (regresión)**: para un producto que nunca fue reordenado, el orden que devuelve
  `Product.variants` después de la migración es idéntico al que tenía antes (mismo orden de
  creación) — verificable comparando contra una lista de referencia capturada antes de aplicar la
  migración de esta spec, o construyendo el escenario desde cero con IDs conocidos.

## Historia 3 — Reordenar convive con crear, editar y eliminar presentaciones

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_product_variant_reorder -v
python -m unittest app.characterization_tests.test_variantes_duplicadas -v
```

- **Esperado (crear)**: agregar una presentación nueva a un producto ya reordenado le asigna
  `display_order = MAX(display_order) + 1` — la nueva aparece al final, sin alterar el orden de las
  demás (data-model.md, tabla de asignación).
- **Esperado (eliminar)**: `DELETE /variants/{id}` sobre una presentación intermedia de un producto
  reordenado dejala fila con `active=False` y su `display_order` intacto; las presentaciones
  restantes conservan su orden relativo — sin ningún `422` de la constraint `UNIQUE` (research.md
  Decisión 4).
- **Esperado (reactivar)**: `test_variantes_duplicadas.py` (spec 002, sin modificar) sigue en verde
  — reactivar una presentación (`PATCH {"active": true}`) no toca `display_order`, así que su receta
  y ahora también su posición quedan intactas.
- **Esperado (editar)**: `PATCH /variants/{id}` con solo `name`/`price` no incluye `display_order`
  en el body y no lo modifica.

## Verificación de no-regresión

```bash
cd ../pos-backend
python -m unittest discover app/characterization_tests -v
```

- **Esperado**: toda la suite de characterization tests existente (specs 002, 003, 004, 027, entre
  otras) sigue en verde sin modificación — esta spec no cambia ninguna regla de creación, precio,
  SKU o consumo de inventario, solo agrega una columna y un endpoint nuevos.
