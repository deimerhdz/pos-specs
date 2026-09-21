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

## US7 — Variante sin nombre: lo da la presentación (P1, enmienda 2026-09-20)

**Antes de empezar** (sobre una copia de datos, no producción): en un tenant de prueba, tener (a) un
producto con dos tamaños asociados a "Pequeña" y "Grande", (b) otro con una variante de nombre libre
"Familiar" sin presentación y (c) uno sin tamaños ("Presentación única"). Anotar los nombres que
muestran hoy el menú QR, el carrito de mesa y el selector de reglas de promoción.

**Paso A (backend aditivo)** — sin migración:

1. `GET /products/{id}` y el endpoint del menú público devuelven `presentation_id` y `presentation_name` además
   de `name`.
2. `POST /products` con una variante `{"presentation_id": null, "price": 1000}` (sin `name`) ⇒ la
   variante queda asociada a "Presentación única" (créala si el catálogo no la tenía; verificar con
   `GET /presentations`). Dos variantes así en el mismo producto ⇒ `409`.
3. Crear un producto en una categoría con presentaciones asociadas ⇒ sus variantes heredadas traen
   `presentation_id`.

**Paso B (frontend)**:

4. Abrir el producto (a): la tabla muestra `# · Presentación · Precio · Eliminar`; **no** hay
   columna ni campo "Nombre".
5. "+ Agregar tamaño": la fila nueva muestra "Elige una presentación", el botón Guardar queda
   deshabilitado y el `<select>` no ofrece "Pequeña"/"Grande" (ya usadas en otras filas) ni "Sin
   presentación".
6. Elegir una presentación, guardar, recargar: el tamaño conserva su presentación y precio.
7. Apagar y volver a encender el interruptor de tamaños en un producto: se crean tres filas que
   conservan precio y receta, con Grande/Mediana/Pequeña preseleccionadas si están en el catálogo
   (las que no, quedan sin elegir y bloquean Guardar).
8. En la pantalla de configuración de una promoción, aplicar una regla "Presentación única × 2" a varios
   productos ya seleccionados: se generan filas para los que tienen esa presentación (el nombre
   sale de la presentación, que es única en el catálogo).

**Paso C (backend destructivo)** — migración:

9. `alembic upgrade head` sobre la copia. En `psql`: `product_variants` ya **no** tiene `name`;
   `presentation_id` es `NOT NULL`; `SELECT count(*) FROM product_variants WHERE presentation_id IS
   NULL` = 0; la variante "Familiar" ahora existe en `presentations`.
10. Recargar menú QR, carrito de mesa y selector de reglas: los textos son idénticos a los
    anotados al inicio (SC-007).
11. `PATCH /presentations/{id}` renombrando "Grande" a "Extra Grande" ⇒ `200`; menú QR y formulario
    muestran "Extra Grande" sin ningún otro cambio; ningún `409` de colisión de nombres.
12. `DELETE FROM presentations WHERE id = '<una con variantes>'` en `psql` ⇒ error de FK
    (`RESTRICT`).
13. `alembic downgrade -1` y `alembic upgrade head` una vez: sin errores y con los mismos nombres
    visibles.

## US8 — Tabla de tamaños antes de "Maneja inventario" (P3, enmienda 2026-09-20)

1. Abrir un producto con tamaños y "Maneja inventario" **apagado**: justo bajo "Tamaños del
   producto" está la tabla completa (con "+ Agregar tamaño"); debajo, las presentaciones
   desactivadas (si hay); debajo, "Maneja inventario"; debajo, el detalle del tamaño activo con el
   aviso "Activa «Maneja inventario» arriba…".
2. Encender "Maneja inventario": la tabla no cambia de lugar ni de contenido; el detalle del tamaño
   pasa a mostrar "Insumos fijos".
3. Apagar el interruptor de tamaños: desaparece la tabla y "Maneja inventario" queda justo bajo el
   encabezado, con el precio y el detalle de la variante única debajo, como antes.
4. Vista angosta (ancho de teléfono): la tabla sigue siendo usable y el interruptor no queda
   tapado (la columna "Nombre" ya no ocupa ancho).

## US4 enmendada — Presentaciones independientes de los productos (A-80)

1. Crear una promoción nueva (tipo precio de paquete) y entrar a su configuración, **sin**
   seleccionar productos en el Paso 1.
2. Abrir "Presentación / Tamaño" en el Paso 2: aparecen todas las presentaciones activas del
   catálogo (también las que ningún producto usa); una presentación desactivada no aparece.
3. Elegir una presentación: bajo el selector dice "Selecciona productos en el Paso 1…".
4. En el Paso 1 elegir una categoría y marcar dos productos: el selector conserva la presentación y
   el aviso pasa a "Se aplicará a N de 2 productos seleccionados".
5. Elegir una presentación que ningún producto marcado tiene: el aviso lo dice y "Agregar a la
   lista" responde "Ningún producto seleccionado tiene esa presentación".
6. Con unidades y precio válidos, "Agregar a la lista" con una presentación compartida por varios
   productos abre la confirmación con casilla por producto y genera una fila por producto.

## US4 por presentación (A-81) — una regla por presentación, productos intercambiables

1. En una promoción de precio de paquete, seleccionar en el Paso 1 dos productos con presentaciones
   en común (p. ej. Ojo de diablo y Perla negra, de Granizados).
2. En el Paso 2 agregar «8 onzas · 2 unidades · $12.000» y «12 onzas · 2 unidades · $17.000»: la
   lista muestra **dos** filas, cada una con los dos productos en su descripción; no hay panel de
   confirmación.
3. Intentar agregar otra vez «8 onzas»: se rechaza indicando que se quite la regla para cambiarla.
4. En el Paso 1 quitar un producto y marcar otro: la lista de reglas no cambia; las descripciones
   sí (a qué productos se aplica cada una).
5. Guardar, volver a abrir «Configurar»: la lista vuelve a mostrar las dos reglas y el Paso 1 trae
   los productos seleccionados.
6. Agregar una presentación que ningún producto seleccionado tiene: aparece «sin productos
   seleccionados con esta presentación» y el aviso de que no se guardará mientras siga así.

## Configuración de una promoción — "Guardar y sincronizar" (FR-045/FR-046)

1. Abrir "Configurar" de una promoción en Borrador o Pausada: la cabecera solo tiene "Volver", y
   "Guardar y sincronizar" está al final del formulario, deshabilitado, con "No hay cambios por
   guardar".
2. Cambiar el nombre, una fecha, un día o agregar/quitar una regla: el botón se habilita. Devolver
   el valor original lo deshabilita otra vez.
3. Vaciar el nombre: el botón sigue deshabilitado (formulario inválido).
4. Abrir una promoción Finalizada: no hay botón.

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

**Enmienda 2026-09-20 (US7/US8) — pendiente hasta implementar Fase 10 de tasks.md:**

- [ ] Anomalía **A-79** registrada en `registro-de-anomalias.md` antes del merge de T052/T057
      (Principio II) — **registrada 2026-09-20**, falta solo confirmar el orden respecto del merge.
- [ ] Suite de characterization tests de backend en verde; solo cambiaron los tests citados en T057
      (línea base previa 891/891).
- [ ] `ng test` sin regresiones respecto de la línea base (934/954, 19 fallas preexistentes).
- [ ] Migración de T052 ensayada sobre una copia de datos reales: 0 variantes sin presentación,
      nombres visibles idénticos, `downgrade` + `upgrade` sin errores (T064).
- [ ] Pasos A, B y C desplegados en ese orden, con un ciclo de despliegue de B en producción antes
      de C (research.md D10).
