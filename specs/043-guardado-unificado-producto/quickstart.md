# Quickstart: Guardado Unificado de Producto (Crear y Actualizar)

Validación ejecutable por historia de usuario. Los comandos asumen que se ejecutan desde la raíz de
cada repo (`../pos-backend`, `../pos-heladeria`, siblings de `pos-specs`).

## Prerrequisitos

- `pos-backend`: entorno virtual activado. **Sin migración que aplicar** — esta spec no cambia el
  esquema de base de datos (data-model.md).
- `pos-heladeria`: `npm install` ya ejecutado. Ninguna dependencia nueva.
- Spec, plan, research, data-model y contracts de esta funcionalidad ya revisados — en particular
  research.md Decisión 3 (reconciliación por lista completa de activas) y Decisión 4 (reactivar deja
  de ser una llamada de red) antes de tocar `product-form.component.ts` o `product.service.ts`.

## Historia 1 — Crear un producto completo en un solo guardado

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_products_service -v
python -m unittest app.characterization_tests.test_catalog_service_sku -v
```

- **Esperado (caso nuevo)**: `POST /products` con `variants` trayendo dos presentaciones, cada una
  con `recipe` y `option_groups`, crea el producto y sus dos presentaciones en una sola llamada —
  verificable inspeccionando las filas de `product_variants`, `recipe_items` y
  `variant_option_groups` después de la llamada (contracts/product-save-endpoints.md).
- **Esperado (compatibilidad)**: `POST /products` sin `variants` (u omitiéndolo) sigue creando la
  presentación `"Single"` a precio 0 automática — mismo resultado que antes de esta spec
  (`RN-CAT-05`, `test_products_service.py`/`test_catalog_service_sku.py` sin modificar).
- **Esperado (respuesta completa)**: el `201` trae `variants` con `recipe`/`option_groups` ya
  resueltos — ninguna llamada `GET` adicional es necesaria para reflejar lo guardado (FR-006).

```bash
cd ../pos-heladeria
npx ng test --watch=false --include='**/product-form.component.spec.ts'
```

- **Esperado**: crear un producto con dos presentaciones desde el formulario dispara exactamente
  **una** llamada de red (`POST /products`) — verificable con un espía sobre `HttpClient`/
  `product.service.ts`, en vez de las `1 + 1 + 3·2` llamadas que orquestaba `saveNewProduct` antes
  de esta spec.

## Historia 2 — Editar un producto existente combinando varios cambios en un solo guardado

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_products_service -v
python -m unittest app.characterization_tests.test_product_variant_reorder -v
```

- **Esperado**: `PATCH /products/{id}` con `variants` trayendo una mezcla de entradas con `id`
  (editadas) y sin `id` (nuevas), más una presentación activa existente **no** listada, en una sola
  llamada: actualiza las editadas, crea las nuevas, desactiva la no listada, y asigna
  `display_order` según la posición de cada entrada en la lista — verificable leyendo las filas
  resultantes (data-model.md, tabla de reconciliación).
- **Esperado (reactivar dentro del mismo guardado)**: una entrada con el `id` de una presentación
  previamente desactivada (sin `active: false`) la reactiva y conserva su receta y grupos anteriores
  si no se envían campos nuevos, o los reemplaza si sí (research.md Decisión 4).
- **Esperado (compatibilidad)**: `PATCH /products/{id}` sin `variants` en el body no toca ninguna
  presentación — mismo comportamiento que antes de esta spec.

```bash
cd ../pos-heladeria
npx ng test --watch=false --include='**/product-form.component.spec.ts'
```

- **Esperado**: editar nombre del producto + precio de una presentación + receta de otra + arrastrar
  para reordenar, y guardar, dispara exactamente **una** llamada de red (`PATCH /products/{id}`) —
  en vez de las llamadas por presentación + `reorderVariants` que orquestaba `saveExistingProduct`
  antes de esta spec.
- **Esperado (restaurar sin red)**: hacer clic en "Restaurar" sobre una presentación desactivada ya
  no dispara ninguna llamada HTTP — solo mueve esa presentación de `draft().deactivated` a
  `draft().variants` en memoria (research.md Decisión 4); la petición ocurre recién al presionar
  "Guardar".

## Historia 3 — Un error en cualquier parte del guardado no deja el producto a medias

```bash
cd ../pos-backend
python -m unittest app.characterization_tests.test_catalog_service_sku -v
```

- **Esperado**: `PATCH /products/{id}` con `variants` trayendo tres entradas válidas y una con
  `name` duplicado responde `409` identificando `variant_index` de la entrada en conflicto
  (contracts/product-save-endpoints.md), y **ninguna** de las cuatro presentaciones queda
  modificada — verificable comparando el estado de `product_variants` antes y después de la
  llamada fallida.
- **Esperado (recuperación)**: reenviar el mismo body con el nombre corregido persiste las cuatro
  presentaciones en esa segunda llamada.
- **Esperado (fallo en receta/opciones)**: un `price` negativo, un insumo repetido en `recipe`, o un
  grupo de opciones inactivo/repetido en `option_groups` de cualquier presentación del payload
  producen el mismo resultado: rechazo del guardado completo, sin cambios parciales.

## Historia 4 — Retiro de los endpoints separados una vez migrado el formulario

```bash
cd ../pos-backend
grep -rn "products/{product_id}/variants\"\|/variants/{variant_id}\"\|variants/{variant_id}/recipe\|variants/{variant_id}/option-groups\|variants/reorder" app/api/v1/catalog/router.py
```

- **Esperado (antes del retiro)**: la búsqueda encuentra las cinco rutas listadas en
  contracts/product-save-endpoints.md.
- **Esperado (auditoría previa al retiro, FR-007)**: revisar `pos-heladeria` en busca de cualquier
  llamada a esas rutas fuera de `product.service.ts` (p. ej. `grep -rn "variants/reorder\|/recipe\"\|/option-groups\"" ../pos-heladeria/src`)
  y cualquier otro consumidor conocido del sistema (Menú QR, venta de mostrador) — ninguno debería
  aparecer, porque esas rutas requieren `require_tenant_admin` y son exclusivas del formulario de
  administración. También revisar `pos-backend` en busca de llamadas directas en proceso (no solo
  HTTP) a las funciones detrás de esas rutas (`grep -rn "create_variant\|update_variant\|set_recipe\|set_variant_option_groups\|reorder_product_variants" app --include="*.py"`).
- **Resultado real de esta auditoría** (documentado en tasks.md T030 y
  [`registro-de-anomalias.md` A-55](../000-reconocimiento/registro-de-anomalias.md)): tres de las
  cinco rutas (`PUT recipe`, `PUT option-groups`, `PATCH reorder`) no tenían ningún otro consumidor
  y se retiraron. Las otras dos (`POST /products/{id}/variants`, `PATCH /variants/{id}`) sí tenían
  uno — `app/scripts/test_variantes_duplicadas.py` las llama directamente en proceso — así que
  quedan excluidas del retiro como excepción documentada; siguen existiendo, solo que el formulario
  ya no las usa.
- **Esperado (después del retiro)**: la búsqueda ya no encuentra las tres rutas retiradas; las dos
  restantes (excepción) siguen registradas; el formulario de administración de productos sigue
  funcionando (Historias 1-3) usando solo `POST`/`PATCH /products`.

## Verificación de no-regresión

```bash
cd ../pos-backend
python -m unittest discover app/characterization_tests -v
```

- **Esperado**: toda la suite de characterization tests existente sigue en verde. En particular,
  `test_products_service.py` (`TestUpdateProductA44`, orden borrado-R2/commit) y
  `test_product_variant_reorder.py` (asignación de `display_order`) verifican que la lógica de
  negocio que reutilizan (no la que retiran) no cambió — solo se movió a estar dentro de la
  transacción consolidada.
