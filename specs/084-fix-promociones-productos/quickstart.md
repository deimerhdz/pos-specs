# Quickstart: Corrección de Bugs en Promociones y Productos

Validación ejecutable por historia de usuario, sobre una rama nueva desde `develop` en ambos
repos. Requiere el entorno local habitual de `pos-backend` (Postgres + venv) y `pos-heladeria`
(`npm install` si aplica).

## Paso 0 — Suite existente en verde (Principio X)

```bash
cd ../pos-backend
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v

cd ../pos-heladeria
ng test
```

Ambas deben pasar **antes** de empezar (línea base limpia) y **después** de cada incremento.

## Paso 1 — Migración de la columna nueva (bug 2, prerrequisito de US1)

```bash
cd ../pos-backend
alembic upgrade head   # aplica <rev>_084_variante_presentacion.py, down_revision da7581f7bb18
```

Verificar en `psql` (contra el schema de un tenant de prueba): `product_variants` tiene
`presentation_id` (nullable) y el `UNIQUE (product_id, presentation_id)`.

## US1 — Asociar variante con presentación (P1)

1. Crear una presentación de prueba "Grande" (`POST /presentations`), o reusar una existente.
2. Crear o editar un producto; en una fila de variante, seleccionar "Grande" en el nuevo `<select>`
   de presentación → el campo nombre se autocompleta con "Grande" y queda de solo lectura.
3. Guardar → `GET /products/{id}` confirma `variants[].presentation_id` apuntando a esa
   presentación.
4. Renombrar la presentación a "Extra Grande" (`PATCH /presentations/{id}`) → `GET /products/{id}`
   confirma que el nombre de esa variante ahora es "Extra Grande".
5. Intentar asociar otra variante del mismo producto a la misma presentación → `409`.
6. En el formulario, volver esa variante a "Sin presentación" → el nombre vuelve a ser editable,
   conservando el texto que tenía.

## US2 — "Configurar" deshabilitado en promociones activas (P1)

1. Activar una promoción de prueba con al menos una regla.
2. En el listado, confirmar que el botón "Configurar" de esa fila está deshabilitado.
3. Pausarla desde el listado → confirmar que "Configurar" queda habilitado sin recargar la página,
   y que al abrirla todos los campos son editables (no solo nombre/fin de vigencia/días/horas).
4. Repetir con una promoción `active` cuya vigencia ya venció (badge "Vencida") → sigue
   deshabilitado (criterio: `status`, no el badge).

## US3 — Precio mínimo en la tarjeta del menú QR (P2)

1. Crear una regla de precio de paquete "2 x $15.000" vigente sobre la única variante de un
   producto de prueba y activar la promoción.
2. Abrir el menú QR público de una mesa de prueba, pestaña "Promociones".
3. Confirmar que la tarjeta muestra "2 x $15.000 · $7.500 c/u" (o el formato equivalente) además
   de la insignia "🎉 Promo", sin necesidad de abrir el producto.
4. Repetir con un producto con dos variantes cubiertas por reglas de precios distintos → confirmar
   que la tarjeta muestra el precio equivalente por unidad más bajo, con "Desde $X".

## US4 — Aplicación masiva de una regla (P2)

1. Crear una promoción nueva, seleccionar 3 productos en el Paso 1 que compartan la variante
   "Presentación única".
2. En la pantalla de configuración, definir una regla "Presentación única x 2 unidades $15.000" →
   confirmar que aparece una lista de los 3 productos con checkbox premarcado.
3. Desmarcar uno, confirmar → verificar que se generan 2 filas de regla (no 3, no 1 combinada).
4. Editar el precio de una de las 2 filas generadas → confirmar que la otra no cambia.

## US5 — Exclusividad de producto entre promociones vigentes (P1)

1. Con la promoción `active` de US3 (cubre la variante única de un producto), intentar
   seleccionar ese mismo producto en el Paso 1 de una promoción nueva distinta.
2. Confirmar que el sistema lo impide y explica que ya hace parte de la promoción activa de US3.
3. Pausar la promoción de US3 → confirmar que el producto vuelve a estar disponible para
   seleccionarse en la promoción nueva.
4. Dentro de una misma promoción en edición, seleccionar dos variantes del mismo producto (p. ej.
   "Pequeña" y "Grande") → confirmar que se permite sin bloqueo.

## US6 — Menú de acciones sin recorte (P3)

1. En el listado de promociones, con suficientes filas para que haya scroll, desplazarse hasta una
   fila cercana al borde inferior del contenedor visible.
2. Abrir su menú ⋮ → confirmar que se muestra completo, sin recortarse, sin generar una barra de
   scroll adicional dentro de la tabla.
3. Hacer scroll de la tabla con el menú abierto → confirmar que se cierra.
4. Hacer clic fuera del menú abierto → confirmar que se cierra y la posición de scroll de la tabla
   no cambió.

## Checklist final (Principio X)

- [x] Suite de characterization tests de backend en verde (incluye los nuevos:
      `test_products_variant_presentation.py`, `test_promotions_product_overlap.py`) — 891/891.
- [x] `ng test` en verde — 934 tests pasan; los 19 que fallan (6 archivos) son deuda preexistente
      de `develop` sin relación con esta spec (auth, tenant/super-admin, checkout/order panel,
      terminal de menú, confirmado idéntico antes y después de los seis incrementos).
- [x] Anomalías **A-76**, **A-77**, **A-78** registradas en `registro-de-anomalias.md` antes del
      merge de los commits correspondientes (Principio II) — commit `9cef914` de este repo.
- [x] `test_promotions_service.py`/`test_promotions_router.py` siguen en verde sin edición (motor
      de cálculo intacto, research.md D0). `test_promotions_rules_admin.py` **sí requirió edición**
      (a diferencia de lo previsto acá): dos tests caracterizaban comportamiento que A-76/A-78
      reemplazan a propósito (editar escalares de una promoción activa; ventanas horarias
      disjuntas sobre el mismo producto) — actualizados explícitamente citando la anomalía
      correspondiente, con evidencia en el mismo archivo de que el resto sigue sin tocarse
      (Principio III) — ver `plan.md` §Constitution Check fila III y `tasks.md` T018/T032.
