# Quickstart: validar el Catálogo de Presentaciones y el rediseño de Promociones

Valida, en orden, las tres historias de usuario de `spec.md` contra una implementación real.
Requiere `../pos-backend` en la rama de esta funcionalidad (`feat/090-...`, spec.md
§Assumptions) con la migración de esta spec aplicada, y `../pos-heladeria` en su rama equivalente
corriendo contra ese backend.

## Paso 0 — Línea base: la suite existente sigue en verde

```bash
cd ../pos-backend
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

Ningún test de `test_promotions_*`, `test_products_service.py` (RN-CAT-05) ni de categorías debe
fallar antes de empezar — confirma que no se rompió nada existente por accidente al aplicar la
migración nueva.

## Paso 1 — Migración y siembra (FR-019)

```bash
cd ../pos-backend
alembic upgrade head
```

Verificar, por cada schema de tenant relevante (`tenant_default` al menos):

```sql
SELECT count(*) FROM tenant_default.presentations;               -- > 0 si ya había variantes
SELECT count(*) FROM tenant_default.category_presentations;      -- = 0 (FR-019, sin asociaciones)
SELECT name, active FROM tenant_default.presentations ORDER BY name;
```

Confirmar que cada nombre distinto que ya existía en `product_variants.name` aparece exactamente
una vez, activo. Repetir `alembic downgrade -1` y verificar que ambas tablas desaparecen sin
tocar `product_variants`/`categories`.

## Paso 2 — Historia 1: administrar el catálogo global (US1)

Backend:

```bash
# Crear
curl -X POST http://localhost:8000/api/v1/presentations -H "..." -d '{"name": "Pequeño"}'
curl -X POST http://localhost:8000/api/v1/presentations -H "..." -d '{"name": "Mediano"}'
curl -X POST http://localhost:8000/api/v1/presentations -H "..." -d '{"name": "Grande"}'

# Duplicado -> 409
curl -X POST http://localhost:8000/api/v1/presentations -H "..." -d '{"name": "Pequeño"}'

# Editar nombre -> también valida unicidad (FR-003)
curl -X PATCH http://localhost:8000/api/v1/presentations/{id} -H "..." -d '{"name": "Chico"}'

# Desactivar y reactivar (US1 Escenario 4 y 5)
curl -X PATCH http://localhost:8000/api/v1/presentations/{id} -H "..." -d '{"active": false}'
curl -X PATCH http://localhost:8000/api/v1/presentations/{id} -H "..." -d '{"active": true}'
```

Frontend: entrar a Catálogo → Presentaciones, repetir el mismo flujo desde la UI y verificar que
la desactivada deja de listarse como opción al editar una categoría (Paso 3), pero sigue visible
en el listado de Presentaciones con su estado.

## Paso 3 — Historia 2: asociación y herencia (US2)

1. Asociar "Pequeña"/"Mediana"/"Grande" a la categoría "Ensaladas" (`PATCH /categories/{id}` con
   `presentation_ids`, o desde el formulario de categoría en la UI).
2. Crear el producto "Ensalada César" dentro de "Ensaladas" **sin** enviar `variants` (`POST
   /products` con `variants` ausente, o desde la UI sin tocar el bloque de variantes).
3. Verificar `GET /products/{id}`: 3 variantes ("Pequeña", "Mediana", "Grande"), `price: 0` cada
   una.
4. Crear una categoría "Bebidas sin tamaño" sin ninguna presentación asociada; crear un producto
   dentro de ella sin `variants`; verificar que nace con una única variante `"Presentación
   única"`.
5. Cambiar las presentaciones asociadas a "Ensaladas" (quitar "Grande"); verificar que el producto
   "Ensalada César" ya creado conserva sus 3 variantes sin cambio (FR-008).
6. Sobre ese mismo producto, agregar manualmente una variante "Familiar" vía `PATCH
   /products/{id}` y confirmar que se permite sin restricción (FR-007).

## Paso 4 — Historia 3: promoción rediseñada (US3)

Prerrequisito: el producto "Granizado de Mora" existe con variante "8oz" (heredada o manual) y
"16oz".

1. Backend en verde: correr los characterization tests de promociones — deben seguir pasando sin
   ninguna edición (confirma research.md D9: cero cambios en el motor):
   ```bash
   python -m unittest app.characterization_tests.test_promotions_service -v
   python -m unittest app.characterization_tests.test_promotions_rules_admin -v
   ```
2. UI: Catálogo → Promociones → "Nueva promoción".
3. Pantalla de creación: nombre "2X Granizados 8oz", tipo "Precio de paquete" → continuar.
4. Pantalla de configuración:
   - Vigencia: lunes a miércoles, 14:00 a 18:00, fecha de inicio hoy.
   - Buscar "Granizado de Mora", elegir presentación "8oz" (debe mostrarse con ese nombre si "8oz"
     es una `Presentation` activa asociada a la categoría del producto — verificar el mapeo de
     etiqueta, FR-013), 2 unidades, $12.000 → "Agregar a la lista".
   - Verificar la fila con precio regular tachado y ahorro calculado.
   - Agregar una segunda fila con "16oz" y su propio precio.
   - Guardar.
5. Verificar en el listado: la promoción aparece con su vigencia en lenguaje llano, resumen de 2
   reglas, estado `Borrador` (o `Activa` si se activó).
6. Intentar guardar una regla cuyo precio de paquete no representa ahorro (p. ej. `value` mayor al
   precio normal más barato del conjunto) → esperar 409 con el mensaje ya existente
   (`_guard_package_is_discount`, FR-016, sin cambio de criterio).
7. Activar la promoción; reabrir su configuración y verificar que el tipo aparece fijo
   ("Fijado en creación") y las reglas ya guardadas no son editables, pero nombre/descripción/fin
   de vigencia/días/horas sí (FR-018).
8. Comparar visualmente las tres pantallas contra los prototipos
   (`~/Escritorio/promociones/*.html`) — SC-004, verificable sin leer código.

## Antes de dar por completada esta spec (Principio X)

- [X] Suite de characterization tests completa en verde (Paso 0 repetido al final) — 857 tests,
      `python -m unittest discover -s app/characterization_tests -p 'test_*.py'`.
- [X] Migración probada en ambos sentidos (`upgrade`/`downgrade`) contra una base con datos reales
      de al menos un tenant con variantes existentes — tenant `heladeria`, 17 `product_variants`,
      siembra de 5 presentaciones verificada, downgrade limpio sin tocar `product_variants`/`categories`.
- [X] Anomalía **A-74** registrada en `registro-de-anomalias.md` (D7: rename "Single" →
      "Presentación única") antes de que el código correspondiente se mergee.
- [X] Los 5 Acceptance Scenarios de US1, 5 de US2 y 7 de US3 verificados manualmente o con test
      nuevo, según corresponda — cobertura exhaustiva vía characterization tests nuevos
      (`test_presentations_service.py`, `test_categories_presentations.py`,
      `test_products_presentations_inheritance.py`) + recorrido manual en navegador real contra
      Postgres (crear/asociar presentación, crear/configurar/activar promoción "2X Granizados 8oz"
      con la Historia 1→2→3 completa, confirmando el bloqueo de edición de reglas en `Activa`, FR-018).
- [X] Las 6 Success Criteria (SC-001 a SC-006) verificadas — SC-004 (fidelidad visual) confirmada
      contra las 3 pantallas reales en el navegador.
- [X] Ningún test `"CONGELA comportamiento actual:"` quedó en rojo sin autorización explícita.
