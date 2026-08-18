---

description: "Task list for: Corrección del orden de borrado de imagen en R2 (A-44)"
---

# Tasks: Corrección del orden de borrado de imagen en R2 (A-44)

**Input**: Design documents from `/specs/021-correccion-orden-borrado-imagen-r2/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md),
[contracts/update-product-endpoint.md](./contracts/update-product-endpoint.md),
[quickstart.md](./quickstart.md)

**Tests**: FR-006 exige explícitamente al menos un test de characterization dedicado — los tests
están incluidos abajo, no son opcionales en esta spec.

**Alcance**: todo el trabajo de código vive en el repositorio sibling `../pos-backend` (Constitución
§Alcance). Rutas de fichero relativas a la raíz de `pos-backend`.

**Nota sobre paralelismo**: esta spec toca exactamente **dos ficheros** —
`app/api/v1/products/service.py` (dos líneas reordenadas) y el fichero de test nuevo
`app/characterization_tests/test_products_service.py` (no existe hoy ningún
`test_products_*.py`, verificado en research.md/plan.md) — el segundo cambio de producción más
pequeño de `pos-specs` después de la spec 020 (Constitución, Principio III: un módulo, sin
extracción). No hay oportunidades reales de `[P]` entre tareas que editan el mismo fichero de
test; se marca `[P]` solo entre corridas de suites de test independientes en la fase de Polish.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3)

## Phase 1: Setup

**Purpose**: confirmar la línea base antes de tocar código — a diferencia de la spec 020, aquí no
existe ningún test `CONGELA` previo que confirmar, porque `products.service` no tiene hoy ninguna
caracterización.

- [X] T001 Confirmar que `app/characterization_tests/` no contiene ningún fichero
      `test_products_*.py` (`ls app/characterization_tests/ | grep products` sin salida) —
      quickstart.md Paso 0. Línea base: el fix y su test se escriben juntos en esta spec, no se
      modifica un test `CONGELA` preexistente.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: infraestructura compartida que las tres historias necesitan antes de empezar.

- [X] T002 Crear `app/characterization_tests/test_products_service.py` con docstring citando A-44
      (`registro-de-anomalias.md`, `registro-riesgos.md` R23), imports (`unittest`,
      `unittest.mock`, `app.characterization_tests.fixtures as fx`,
      `app.api.v1.products.service.ProductService`,
      `app.api.v1.products.schemas.ProductUpdate`) y la clase `TestUpdateProductA44(unittest.TestCase)`
      vacía — infraestructura compartida por las tres historias, ninguna depende de otra primero.
      `fx.make_product(db, image_url=...)` (`fixtures.py:112`) ya soporta `image_url` por kwarg —
      no se necesita ningún helper nuevo en `fixtures.py` (verificado leyendo su firma).

**Checkpoint**: el fichero de test existe y puede ejecutarse (aunque vacío) antes de escribir
ningún caso — se pasa a Phase 3.

---

## Phase 3: User Story 1 - Un fallo de guardado no deja al producto sin ninguna imagen válida (Priority: P1) 🎯 MVP — anomalía A-44

**Goal**: `update_product` NO borra el objeto de imagen anterior en R2 cuando el `db.commit()`
posterior falla (FR-001/FR-002), dejando el objeto viejo y `product.image_url` consistentes entre
sí.

**Independent Test**: llamar `update_product` con un `image_url` nuevo sobre un producto que ya
tiene imagen, forzando un fallo en el `commit` posterior, y verificar que el objeto de imagen
anterior sigue existiendo (mock de `delete_object` nunca invocado).

### Implementation for User Story 1

- [X] T003 [US1] Escribir `test_a44_fallo_de_commit_no_deja_referencia_rota` en
      `app/characterization_tests/test_products_service.py`: crea un producto con `image_url`
      existente vía `fx.make_product`, mockea `app.api.v1.products.service.delete_object` y fuerza
      un fallo de `db.commit()` (`mock.patch.object(db, "commit", side_effect=RuntimeError(...))`),
      llama `service.update_product` con un `image_url` distinto dentro de
      `self.assertRaises(RuntimeError)`, y afirma `mock_delete.assert_not_called()` — quickstart.md
      Paso 1, data-model.md tabla "Commit fallido" (depende de T002).
- [X] T004 [US1] Ejecutar
      `python3 -m unittest app.characterization_tests.test_products_service -v` y confirmar que
      T003 **falla** contra el código actual — el fallo mismo es la evidencia con datos reales del
      defecto A-44 (`mock_delete.assert_not_called()` no se cumple porque el código actual sí borra
      antes del commit). Ningún cambio de código en este paso (depende de T003).
- [X] T005 [US1] Aplicar la corrección en `update_product`
      (`app/api/v1/products/service.py:78-89`): mover `delete_object(old_key)` de antes a después
      de `db.commit()`, guardando `old_key` en una variable local calculada antes del commit —
      research.md Decisión 1, quickstart.md Paso 2 (depende de T004).
- [X] T006 [US1] Re-ejecutar
      `python3 -m unittest app.characterization_tests.test_products_service -v` y confirmar que
      T003 pasa en verde: `delete_object` nunca se llama cuando el commit falla (FR-002, CA2)
      (depende de T005).

**Checkpoint**: `update_product` ya no deja referencias de imagen rotas ante un fallo de guardado —
verificable de forma aislada con T003/T006, sin tocar todavía la verificación del camino feliz.

---

## Phase 4: User Story 2 - El camino feliz produce el mismo resultado final que hoy (Priority: P2)

**Goal**: confirmar que, en éxito, el resultado final es idéntico al de antes de la corrección
(imagen vieja borrada, nueva persistida — FR-003), y que el borrado sigue siendo best-effort ante
un fallo propio suyo (FR-004).

**Independent Test**: llamar `update_product` con un `image_url` nuevo sobre un producto que ya
tiene imagen, dejando que el guardado tenga éxito, y verificar que `delete_object` se invoca
después de `commit` y que el cambio de imagen no se revierte si `delete_object` falla.

### Implementation for User Story 2

- [X] T007 [US2] Escribir `test_a44_camino_feliz_borra_despues_del_commit` en
      `test_products_service.py`: mockea `delete_object` y `db.commit` para registrar el orden
      relativo de invocación en una lista compartida, llama `update_product` con un `image_url`
      nuevo, y afirma `orden == ["commit", "delete"]` — quickstart.md Paso 3, research.md
      Decisión 2 (depende de T005).
- [X] T008 [US2] Escribir `test_a44_fallo_de_delete_object_no_revierte_el_cambio_de_imagen` en
      `test_products_service.py`: mockea `delete_object` para que lance una excepción (simulando
      R2 no disponible) y confirma que `update_product` no propaga esa excepción y que
      `product.image_url` en la sesión ya refleja el valor nuevo — FR-004, RN2 de `spec.md`
      (depende de T005; nota: `delete_object` ya captura toda excepción internamente en
      `app/core/storage.py`, así que este test documenta el contrato ya existente, no agrega
      manejo de errores nuevo en `update_product`).

**Checkpoint**: US1 + US2 juntas entregan la corrección completa — ante fallo, no hay referencia
rota (US1); ante éxito, el resultado es idéntico a hoy y el best-effort se conserva (US2).

---

## Phase 5: User Story 3 - La corrección no altera ningún producto ya actualizado antes del cambio (Priority: P1)

**Goal**: confirmar que el fix no introduce ningún mecanismo de recálculo retroactivo sobre
`Product`s ya actualizados (FR-005, Principio V de la Constitución). Historia de garantía
estructural, no de comportamiento nuevo a probar en tiempo de ejecución.

**Independent Test**: revisar que el código nuevo de T005 no lee ni escribe sobre `Product`s
distintos al que recibe `id` como parámetro.

### Implementation for User Story 3

- [X] T009 [US3] Revisar `app/api/v1/products/service.py` completo (`create_product`,
      `update_product`, `soft_delete`) y confirmar que ninguno ejecuta un recorrido, `UPDATE` en
      lote ni migración sobre `Product`s existentes distintos al `id` recibido — el cambio de T005
      es local a la actualización de un único producto, sin ningún backfill que alcance filas ya
      escritas antes del fix (FR-005, quickstart.md nota final). **Verificación**:
      `grep -n "db\.execute\|\.query(Product)\|update(Product)" app/api/v1/products/service.py`
      sobre el fichero no debe mostrar ninguna consulta de escritura en lote fuera de los
      `db.commit()`/`db.refresh()` puntuales ya existentes (depende de T005).

**Checkpoint**: las tres historias quedan cubiertas — la corrección es completa (US1), verificada
sin cambio en el camino feliz (US2) y confirmada como no retroactiva (US3).

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T010 Ejecutar
      `python3 -m unittest app.characterization_tests.test_products_service -v` completo y
      confirmar que los cuatro tests (T003, T007, T008, más el propio T003 ya en verde) pasan sin
      ninguna regresión (Principio II) — `app/characterization_tests/test_products_service.py`.
- [X] T011 [P] Ejecutar
      `python3 -m unittest app.characterization_tests.test_catalog_service_sku -v` y confirmar que
      sigue en verde sin cambios — esta corrección no toca `catalog_engine` ni el resto de
      `ProductService` fuera de `update_product` —
      `app/characterization_tests/test_catalog_service_sku.py`.
- [X] T012 [P] Ejecutar
      `python3 -m unittest app.characterization_tests.test_catalog_line_pricing -v` y confirmar que
      sigue en verde sin cambios, por la misma razón que T011 —
      `app/characterization_tests/test_catalog_line_pricing.py`.
- [X] T013 Ejecutar `python3 -m unittest discover -s app/characterization_tests -p "test_*.py"`
      (la suite completa de `pos-backend`, no solo los módulos tocados) y confirmar cero
      regresiones fuera del alcance de esta spec (Principio II) (depende de T010-T012).
- [X] T014 Recorrer [quickstart.md](./quickstart.md) de punta a punta (Pasos 0-4) contra el
      `pos-backend` con el fix aplicado y confirmar SC-001 a SC-004: SC-001 (T006), SC-002 (T007),
      SC-003 (T009), SC-004 (T003+T006, contraste antes/después) (depende de T013).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — arranca de inmediato
- **Foundational (Phase 2)**: depende de Setup; crea el fichero de test que todas las historias
  comparten
- **US1 (Phase 3)**: depende de Foundational; es la única historia que toca producción (T005)
- **US2 (Phase 4)**: depende de T005 (US1) — sus tests de camino feliz y best-effort solo tienen
  sentido una vez aplicado el fix; T007 y T008 son independientes entre sí a nivel de caso, pero
  editan el mismo fichero
- **US3 (Phase 5)**: depende de T005 (US1) — revisa el código ya cambiado, no introduce cambio
  propio
- **Polish (Phase 6)**: depende de que US1+US2+US3 estén completas

### Notas de secuencia

Igual que en la spec 020, aquí **solo US1 modifica producción** (dos líneas reordenadas, T005);
US2 y US3 son verificación (camino feliz + best-effort, y garantía de no-retroactividad) sobre ese
mismo cambio. Sus *tests* (T007, T008) no dependen entre sí, solo de T005 — cada historia sigue
siendo *verificable* de forma aislada, tal como exige `spec.md` en su sección "Independent Test" —
pero la *implementación* real de esta spec es un único cambio de orden (T005) más su batería de
tests, en un fichero de test que se crea desde cero (a diferencia de 020, que modificaba uno ya
existente).

### Parallel Opportunities

Ninguna entre tareas que editan `test_products_service.py` (mismo fichero: T003, T007, T008). En
Polish, T011 y T012 corren suites sobre ficheros distintos entre sí y respecto a T010 — pueden
lanzarse en paralelo.

---

## Parallel Example: Polish

```bash
# Estas dos corridas de suite son independientes entre sí y de T010 (ficheros de test distintos):
python3 -m unittest app.characterization_tests.test_catalog_service_sku -v    # T011
python3 -m unittest app.characterization_tests.test_catalog_line_pricing -v   # T012
```

---

## Implementation Strategy

### Orden recomendado (no hay "MVP parcial" real en esta spec)

Dado que toda la implementación es un reordenamiento de dos líneas de producción (T005), la
entrega real es:

1. Phase 1 (Setup): confirmar que no hay caracterización previa que romper (T001).
2. Phase 2 (Foundational): crear el fichero de test compartido (T002).
3. Phase 3 (US1): T003-T004 (RED, documenta el defecto), T005 (fix), T006 (GREEN) — al terminar
   esta fase, `update_product` ya queda corregido y verificable de forma aislada.
4. Phase 4 (US2): T007-T008 — confirma que el camino feliz no cambió y que el best-effort se
   conserva.
5. Phase 5 (US3): T009 — confirma por revisión que no hay recálculo retroactivo.
6. Phase 6 (Polish): corridas completas + validación de quickstart.md.

### Incremental Delivery

No aplica despliegue por historia (un solo servicio, sin flag de feature, un solo commit de
producción) — el PR natural de esta spec entrega US1+US2+US3 juntas, dado su tamaño (2 líneas de
producción reordenadas + 3 tests nuevos en un fichero también nuevo). El desglose por historia
existe para trazabilidad de requisito → tarea → test (FR-001/FR-002→US1, FR-003/FR-004→US2,
FR-005→US3), no para sugerir despliegues separados.
