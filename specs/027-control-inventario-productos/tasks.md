---

description: "Task list for Control de Inventario por Producto (Switch de Insumos)"
---

# Tasks: Control de Inventario por Producto (Switch de Insumos)

**Input**: Design documents from `/specs/027-control-inventario-productos/` (plan.md, spec.md,
research.md, data-model.md, contracts/, quickstart.md)

**Tests**: incluidos — `plan.md` (Technical Context) y `quickstart.md` fijan de antemano qué
ficheros de test crea o extiende cada historia (Constitución, Principio X: Verificación
Obligatoria; Principio III: el CONGELA `test_catalog_consumption_plan.py` se extiende citando esta
spec), así que no son opcionales para esta spec.

**Organization**: tareas agrupadas por historia de usuario (US1-US4, prioridades de `spec.md`) para
que cada una sea implementable y verificable de forma independiente, per `quickstart.md`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (ficheros distintos, sin dependencia de una tarea sin
  terminar)
- **[Story]**: historia de usuario a la que pertenece (US1..US4)
- Cada tarea incluye la ruta de fichero exacta, relativa a la raíz del repo sibling que corresponda
  (`pos-backend` o `pos-heladeria`)

## Path Conventions

Dos repositorios sibling de `pos-specs` (Constitución §Alcance, plan.md §Project Structure):

- Backend: `pos-backend/app/...` y `pos-backend/alembic/...` (rutas ya incluyen el prefijo
  `pos-backend/`)
- Frontend: `pos-heladeria/src/app/...` (rutas ya incluyen el prefijo `pos-heladeria/`)

---

## Phase 1: Setup

**Purpose**: confirmar que el entorno está listo y que hay una línea base verde antes de tocar
nada.

- [X] T001 Confirmar entorno: `pos-backend` con el venv activado (Python 3.14) y `pos-heladeria`
  con `npm install` ya corrido; correr `python -m unittest discover app/characterization_tests -v`
  en `pos-backend` y `npx vitest run` en `pos-heladeria` como línea base, confirmando que ambas
  suites pasan antes de empezar

**Checkpoint**: entornos listos, línea base verde confirmada.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: hacer existir el atributo `tracks_inventory` de punta a punta (BD → API → tipos de
frontend) sin cambiar ningún comportamiento todavía — prerrequisito real y compartido por las 4
historias (ninguna puede construirse sin que el campo exista primero).

**⚠️ CRITICAL**: ninguna historia puede empezar hasta que esta fase esté completa.

- [X] T002 Crear la migración `pos-backend/alembic/versions/e3f4a5b6c7d8_products_tracks_inventory.py`,
  `down_revision = 'd2e3f4a5b6c7'` (head actual), siguiendo el esqueleto de
  `f5a6b7c8d9e0_availability_change_partial_count.py`: decorador `@for_each_tenant_schema`, guard
  `_has_table(schema, "products")`, y en `upgrade()` únicamente
  `op.add_column("products", sa.Column("tracks_inventory", sa.Boolean(), nullable=False,
  server_default="true"), schema=schema)` (sin backfill dirigido todavía — eso es T020, de la
  Historia 4); en `downgrade()`, `op.drop_column("products", "tracks_inventory", schema=schema)`
  (data-model.md, research.md Decisión 4)
- [X] T003 [P] En `pos-backend/app/models/product.py`, agregar
  `tracks_inventory: Mapped[bool] = mapped_column(Boolean, default=True, server_default="true")`
  junto a `available` (línea ~37-39) — el default `True` es intencional y NO debe cambiarse a
  `False` (research.md Decisión 3: protege toda la suite de characterization tests existente, que
  construye `Product(...)` sin conocer este campo)
- [X] T004 [P] En `pos-backend/app/api/v1/products/schemas.py`, agregar
  `tracks_inventory: bool = False` a `ProductCreate` (líneas 14-23, este es el único lugar donde el
  default es `False`, por FR-001), `tracks_inventory: bool | None = None` a `ProductUpdate` (líneas
  26-33), y `tracks_inventory: bool` a `ProductResponse` (líneas 36-48)
- [X] T005 En `pos-backend/app/api/v1/products/service.py`, pasar `tracks_inventory=data.tracks_inventory`
  al constructor de `Product` dentro de `create_product` (líneas 41-65, junto a `available`), y
  agregar `if data.tracks_inventory is not None: product.tracks_inventory = data.tracks_inventory`
  dentro de `update_product` (líneas 67-92, junto al de `available`) — depende de T003, T004
- [X] T006 [P] En `pos-heladeria/src/app/modules/products/interfaces/product.interface.ts`, agregar
  `tracks_inventory: boolean` a `Product` (líneas 15-27), `ProductCreatePayload` (39-45),
  `ProductUpdatePayload` (48-56) y `ProductDraft` (268-281)
- [X] T007 En `pos-heladeria/src/app/modules/products/services/product.service.ts`, propagar
  `tracks_inventory` en `toProductPayload` (líneas 519-527) y en `toProduct` (644-657) — depende de
  T006

**Checkpoint**: el campo existe de punta a punta y viaja correctamente por creación/edición/lectura
de producto, sin que ningún comportamiento de venta haya cambiado todavía (todo producto, nuevo o
existente, sigue exigiendo receta exactamente como antes de esta spec).

---

## Phase 3: User Story 1 - Crear un producto que no maneja inventario, sin ningún insumo (Priority: P1) 🎯 MVP

**Goal**: que un producto con el switch apagado (su estado por defecto) se pueda crear sin insumos
y venderse sin ningún rechazo relacionado con inventario (spec FR-001/FR-004/FR-005, research.md
Decisión 1).

**Independent Test**: crear un producto dejando el switch en su estado por defecto (apagado),
guardarlo sin insumos, y verificar que puede agregarse a una venta y cobrarse por completo sin
ningún rechazo relacionado con inventario y sin que se genere ningún movimiento de stock.

### Implementación backend para User Story 1

- [X] T008 [US1] En `pos-backend/app/catalog_engine/consumption.py`, agregar la función auxiliar
  `_tracks_inventory(db: Session, variant_id: UUID) -> bool` junto a `variant_label` (línea ~146):
  `SELECT Product.tracks_inventory JOIN ProductVariant ON ProductVariant.product_id == Product.id
  WHERE ProductVariant.id == variant_id`, devolviendo `True` si no encuentra la fila (nunca exime
  en silencio una variante inexistente)
- [X] T009 [US1] En `ensure_lines_consume_inventory` (líneas 156-217), agregar
  `if not _tracks_inventory(db, variant_id): continue` como primera línea del bucle `for variant_id,
  quantity, options in entries:` (línea ~179), antes de `required_consumption(...)` — depende de
  T008

### Tests para User Story 1

- [X] T010 [P] [US1] En `pos-backend/app/characterization_tests/test_catalog_consumption_plan.py`,
  agregar un caso a `EnsureLinesConsumeInventoryTests` (línea ~234+): `Product(tracks_inventory=False)`
  con una variante sin receta ni grupos configurados → `ensure_lines_consume_inventory` NO lanza
  `HTTPException`, y `plan_line_consumption` sobre esa misma variante devuelve `[]` — citar spec 027
  FR-005 como autorización; los 4 tests ya existentes de esta clase NO se modifican (research.md
  Decisión 5) — depende de T009

### Implementación frontend para User Story 1

- [X] T011 [US1] En `pos-heladeria/src/app/modules/products/pages/product-form.component.ts`: (a)
  asegurar que el `draft()` de un producto nuevo inicializa `tracks_inventory: false` (FR-001); (b)
  agregar el switch nuevo copiando el patrón de `toggleHasSizes()`/su markup (líneas 145-150,
  `role="switch"`, `[attr.aria-checked]`, círculo `bg-indigo-600`/`bg-gray-300`) con un método
  `toggleTracksInventory()` (junto a `toggleHasSizes()`, línea 734) que por ahora solo hace
  `setField('tracks_inventory', !draft().tracks_inventory)` (la confirmación de FR-014 se agrega en
  US3, T018); (c) envolver los bloques "Insumos fijos" (líneas 218-242) y "Sabores a elegir"
  (244-350) en `@if (draft().tracks_inventory) { ... } @else { <p>mensaje: activa "Maneja
  inventario" para configurar insumos</p> }` (research.md Decisión 6/7) — depende de T007

### Tests para User Story 1

- [X] T012 [P] [US1] Crear `pos-heladeria/src/app/modules/products/pages/product-form.component.spec.ts`
  cubriendo: un producto nuevo abre con el switch apagado y la sección de insumos deshabilitada
  (mensaje visible, no editable); guardar sin ningún insumo no produce ningún error de validación
  del formulario — depende de T011

**Checkpoint**: la Historia 1 es funcional y verificable de forma independiente — un producto sin
inventario se crea y se vende sin fricción.

---

## Phase 4: User Story 2 - Activar el switch para asociar los insumos que descuenta el producto (Priority: P1)

**Goal**: que activar el switch habilite la sección de insumos sin relajar ninguna regla de spec
003, y que guardar sin insumos configurados avise de inmediato en el formulario (spec FR-002/FR-006/
FR-013, research.md Decisión 8).

**Independent Test**: activar el switch, verificar que la sección de insumos se habilita, asociar
al menos un insumo o grupo, guardar, y confirmar que una venta posterior descuenta el inventario
exactamente igual que hoy; y que guardar sin insumos con el switch activado muestra una advertencia
visible de inmediato.

### Tests para User Story 2 (regresión primero)

- [X] T013 [P] [US2] En `test_catalog_consumption_plan.py`, agregar un caso a
  `EnsureLinesConsumeInventoryTests`: `Product(tracks_inventory=True)` con una variante sin receta
  ni grupos configurados → `ensure_lines_consume_inventory` SIGUE lanzando `409` "no tiene receta
  configurada", exactamente igual que antes de esta spec — guarda de regresión de que el default
  `True` de T003 no relaja `RN-CAT-34` para nadie (research.md Decisión 3/6 del plan) — depende de
  T009 (mismo fichero, después de T010)

### Implementación frontend para User Story 2

- [X] T014 [US2] En `product-form.component.ts`, agregar un banner ámbar nuevo (mismo estilo que el
  ya existente en `groupBreakdown`, líneas 254/286/296: `border-amber-200 bg-amber-50/40`, ícono
  `⚠` como texto plano) justo debajo del switch, visible cuando, al intentar guardar,
  `draft().tracks_inventory === true` y ninguna presentación tiene `recipe.length > 0` ni ningún
  grupo de opciones con consumo real configurado — texto: "Este producto no podrá venderse hasta que
  se le configure al menos un insumo" (research.md Decisión 8, FR-013) — depende de T011

### Tests para User Story 2

- [X] T015 [P] [US2] En `product-form.component.spec.ts`, agregar casos: activar el switch pasa la
  sección de insumos de deshabilitada a editable; intentar guardar con el switch activado y sin
  ningún insumo muestra el banner de advertencia de T014 — depende de T014

**Checkpoint**: las Historias 1 y 2 funcionan juntas de forma independiente — crear sin inventario y
configurar con inventario conviven sin que ninguna relaje a la otra.

---

## Phase 5: User Story 3 - Cambiar el switch de un producto existente sin perder los insumos ya guardados (Priority: P2)

**Goal**: que apagar el switch de un producto con insumos ya guardados pida confirmación, no borre
esos insumos, y no genere ningún movimiento de inventario al vender — el caso que un arreglo parcial
(solo en el guard) dejaría roto (spec FR-008/FR-009/FR-014, research.md Decisión 1/2, el hallazgo
más importante de este plan).

**Independent Test**: asociar insumos a un producto con el switch activado, apagar el switch
(con su confirmación) y guardar, verificar que el producto ya no exige ni aplica esos insumos al
venderse (ni genera movimiento de inventario), y reactivar el switch para confirmar que los insumos
siguen exactamente donde estaban.

### Implementación backend para User Story 3

- [X] T016 [US3] En `plan_line_consumption` (`consumption.py`, líneas 89-129), agregar
  `if not _tracks_inventory(db, variant_id): return []` como primera línea del cuerpo de la función,
  reutilizando el helper de T008 — es el punto que consumen directamente `deduct_order_item`,
  `reverse_order_item` (`app/api/v1/orders/consumption.py`) y `deduct_sale`
  (`app/api/v1/sales/consumption.py`) para el descuento/reversa real (research.md Decisión 1) —
  depende de T008

### Tests para User Story 3

- [X] T017 [P] [US3] En `test_catalog_consumption_plan.py`, agregar el caso más importante de esta
  spec: `Product(tracks_inventory=False)` con una variante que **sí** tiene `RecipeItem` guardados
  (simulando "tenía insumos, se apagó el switch después") → `ensure_lines_consume_inventory` no
  rechaza la venta (ya cubierto por T010, se reafirma aquí) Y `plan_line_consumption` sigue
  devolviendo `[]` a pesar de que `load_recipe` encontraría filas — sin este caso, un arreglo que
  solo tocara el guard pasaría todos los tests existentes y aun así generaría movimientos de
  inventario reales (research.md Decisión 2) — depende de T016

### Implementación frontend para User Story 3

- [X] T018 [US3] En `product-form.component.ts`, extender `toggleTracksInventory()` (T011): al pasar
  de `true` a `false`, revisar si alguna variante del `draft()` tiene `recipe.length > 0` o algún
  grupo con consumo configurado; si es así, llamar `await this.confirm.ask({ tone: 'danger', ... })`
  (`pos-heladeria/src/app/shared/feedback/confirm.service.ts:21-38`, mismo patrón que
  `option-groups-page.component.ts:229`) antes de aplicar el cambio — si se cancela, no se llama
  `setField('tracks_inventory', ...)`; si se acepta, se aplica sin borrar `av.recipe`/
  `av.optionGroups` de ninguna variante (research.md Decisión 9) — depende de T011

### Tests para User Story 3

- [X] T019 [P] [US3] En `product-form.component.spec.ts`, agregar casos: apagar el switch de un
  producto con insumos dispara la confirmación; cancelarla deja `tracks_inventory` en `true` sin
  enviar ningún cambio; aceptarla deshabilita la sección sin vaciar los insumos en memoria;
  reactivar el switch después no dispara ninguna confirmación y muestra los mismos insumos sin
  pedir que se recapturen — depende de T018

**Checkpoint**: las Historias 1-3 funcionan juntas de forma independiente — este es el checkpoint
que valida el hallazgo central de research.md: un producto con insumos "abandonados" nunca vuelve a
descontar inventario mientras su switch esté apagado.

---

## Phase 6: User Story 4 - Los productos que ya existían en el sistema no pierden ni ganan comportamiento por accidente (Priority: P2)

**Goal**: que la migración clasifique automáticamente cada producto existente según si ya superaba
`RN-CAT-34` antes de esta spec, sin intervención manual (spec FR-010/FR-011, research.md Decisión
4).

**Independent Test**: revisar, sobre un catálogo existente, que todo producto con al menos un
insumo o grupo configurado quedó con el switch activado y que una venta suya se comporta
exactamente igual que antes de esta funcionalidad, y que todo producto sin ningún insumo
configurado quedó con el switch apagado y ahora puede venderse sin ser rechazado.

### Implementación para User Story 4

- [X] T020 [US4] En la migración de T002
  (`pos-backend/alembic/versions/e3f4a5b6c7d8_products_tracks_inventory.py`), agregar, justo después
  de `op.add_column`, el backfill dirigido:
  ```python
  op.execute(f"""
      UPDATE {schema}.products p
      SET tracks_inventory = EXISTS (
          SELECT 1 FROM {schema}.product_variants pv
          WHERE pv.product_id = p.id AND pv.active = true AND (
              EXISTS (SELECT 1 FROM {schema}.recipe_items ri
                      WHERE ri.product_variant_id = pv.id)
              OR EXISTS (SELECT 1 FROM {schema}.variant_option_groups vog
                         WHERE vog.product_variant_id = pv.id AND vog.quantity_per_option > 0)
              OR EXISTS (SELECT 1 FROM {schema}.variant_option_groups vog
                         JOIN {schema}.options o ON o.option_group_id = vog.option_group_id
                         WHERE vog.product_variant_id = pv.id AND o.active = true
                           AND o.inventory_item_id IS NOT NULL AND o.item_quantity > 0)
          )
      )
  """)
  ```
  (data-model.md, research.md Decisión 4) — depende de T002
- [X] T021 [US4] Escribir el script de verificación de un solo uso recomendado en research.md
  Decisión 4, `pos-backend/app/scripts/verify_tracks_inventory_backfill.py`: para cada
  `ProductVariant` activa de un tenant, comparar el resultado de la subconsulta `EXISTS` de T020
  contra el resultado real de invocar `load_recipe`/`group_discounts`
  (`app/catalog_engine/consumption.py:50-88`) y reportar cualquier discrepancia — no se ejecuta como
  parte de la migración, es un diagnóstico previo a aplicar en producción — depende de T020

### Validación para User Story 4

- [X] T022 [P] [US4] Correr `alembic upgrade head` en `pos-backend` sobre una base de prueba con al
  menos un producto con receta ya configurada y uno sin nada configurado, y verificar por SQL
  (`SELECT p.name, p.tracks_inventory FROM tenant.products p`) que el primero queda `true` y el
  segundo `false` (quickstart.md, Historia 4) — depende de T020

**Checkpoint**: las 4 historias funcionan juntas — el catálogo existente migra sin romperse y sin
requerir revisión manual.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de no-regresión sobre todo lo tocado.

- [X] T023 [P] Correr `python -m unittest discover app/characterization_tests -v` en `pos-backend`
  completo y confirmar que ningún módulo distinto de `test_catalog_consumption_plan.py` (T010, T013,
  T017) cambia de resultado — en particular `golden_master_core.py` debe seguir exactamente igual,
  porque sus fixtures heredan `tracks_inventory=True` del modelo (research.md Decisión 3/5)
- [X] T024 Ejecutar `quickstart.md` completo (las 4 historias + la sección de Regresión general) de
  principio a fin como validación final antes de dar la funcionalidad por completa (Constitución,
  Principio X)
- [X] T025 [P] Revisión cruzada de FR-012: confirmar que ninguna pantalla de venta/carrito de
  `pos-heladeria` (`modules/tables/`, `modules/cart/` o equivalente) necesitó ni recibió ningún
  cambio para tratar un producto con `tracks_inventory=false` como caso especial — debe verse y
  comportarse idéntico a cualquier otro producto desde la perspectiva del cajero/comensal

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede arrancar de inmediato.
- **Foundational (Phase 2)**: depende de Setup. BLOQUEA las 4 historias — ninguna puede construirse
  sin que `tracks_inventory` exista en BD/API/tipos de frontend.
- **User Stories (Phase 3-6)**: todas dependen de Foundational. US1 y US2 son razonablemente
  paralelas entre sí (comparten `product-form.component.ts` pero en bloques distintos); US3 depende
  de que US1 ya haya creado el switch/`toggleTracksInventory()` base (T011) y del helper de backend
  de US1 (T008); US4 depende solo de Foundational (T002).
- **Polish (Phase 7)**: depende de que las historias que se vayan a entregar ya estén completas.

### Dependencias entre historias

- **US1 (P1)**: depende de Foundational. Es el MVP — crea el switch, el helper de backend
  (`_tracks_inventory`) y el arreglo del guard.
- **US2 (P1)**: depende de Foundational y de T011 (US1, el switch/las secciones ya deben existir
  para poder agregarles el banner de advertencia) — sin dependencia de comportamiento nuevo de US1,
  solo de fichero compartido.
- **US3 (P2)**: depende de T008 (helper de US1) y T011 (switch de US1) — es la historia que
  completa el arreglo de backend (`plan_line_consumption`) que US1 dejó pendiente a propósito
  (research.md Decisión 1: el helper se crea en US1, el segundo punto de inyección se agrega aquí).
- **US4 (P2)**: depende solo de T002 (Foundational) — completamente independiente de US1-US3 a
  nivel de comportamiento (toca la migración, no `consumption.py` ni el frontend).

### Dentro de cada historia

- Tests se escriben junto con o inmediatamente después de la implementación que verifican — este
  proyecto usa characterization tests que documentan comportamiento real, no TDD estricto (mismo
  patrón que specs 024/026).
- Backend antes que frontend dentro de US1 (T008-T010 antes de T011-T012), porque el frontend
  guarda productos que el backend ya debe poder vender sin bloquear.
- Dentro de US3, el backend (T016-T017) y el frontend (T018-T019) son independientes entre sí en
  términos de fichero, pero ambos dependen de piezas de US1 (T008, T011 respectivamente).

### Parallel Opportunities

- T003, T004 y T006 (Foundational) en paralelo entre sí — tocan ficheros distintos sin depender uno
  del otro.
- T010 (US1) es la única tarea de test de esa historia — no hay paralelismo interno ahí, pero T010
  y T012 (frontend) pueden avanzar en paralelo entre sí una vez sus respectivas implementaciones
  (T009, T011) estén listas.
- T013 (US2) y T017 (US3) tocan el mismo fichero de test que T010 (US1) — son secuenciales por
  fichero, no paralelas entre sí, aunque pertenezcan a historias "independientes" en términos de
  comportamiento.
- T021 y T022 (US4) en paralelo entre sí tras T020.
- T023 y T025 (Polish) en paralelo entre sí.

---

## Parallel Example: Foundational

```bash
# Tras T002 (migración base), en paralelo:
Task: "Agregar columna tracks_inventory a pos-backend/app/models/product.py"
Task: "Agregar tracks_inventory a los schemas de pos-backend/app/api/v1/products/schemas.py"
Task: "Agregar tracks_inventory a las interfaces de pos-heladeria/.../product.interface.ts"
```

## Parallel Example: User Story 4

```bash
# Tras T020 (backfill agregado a la migración), en paralelo:
Task: "Escribir script de verificación app/scripts/verify_tracks_inventory_backfill.py"
Task: "Correr alembic upgrade head y verificar por SQL sobre datos de prueba"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup (T001).
2. Completar Phase 2: Foundational (T002-T007) — CRÍTICO, bloquea todo lo demás.
3. Completar Phase 3: User Story 1 (T008-T012).
4. **DETENER y VALIDAR**: correr el bloque de Historia 1 de `quickstart.md` de forma independiente.
5. Esto ya resuelve el problema central: un producto sin inventario se puede crear y vender.

### Incremental Delivery

1. Setup + Foundational → base lista, cero cambios de comportamiento todavía.
2. Agregar Historia 1 (T008-T012) → validar → MVP: crear y vender sin inventario.
3. Agregar Historia 2 (T013-T015) → validar → activar el switch sigue las reglas de siempre, con
   aviso temprano si falta configurar.
4. Agregar Historia 3 (T016-T019) → validar → **el hallazgo más importante de este plan**: apagar
   el switch con insumos guardados no genera movimientos de inventario, con confirmación explícita.
5. Agregar Historia 4 (T020-T022) → validar → el catálogo existente migra solo, sin romperse.
6. Phase 7 (T023-T025) → verificación final de no-regresión antes de dar la spec por completa.

### Parallel Team Strategy

Con más de una persona: una vez completado Foundational, US1 y US4 pueden repartirse de inmediato
entre personas distintas (ficheros completamente disjuntos: `consumption.py`+frontend vs. la
migración). US2 y US3 deben esperar a que US1 termine T008/T011 respectivamente, ya que ambas
construyen sobre esas piezas.

---

## Notes

- [P] tasks = ficheros distintos, sin dependencias entre sí.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Constitución, Principio XII).
- 1 tabla modificada (`products`, +1 columna), 1 migración nueva, 0 endpoints nuevos, 0 dependencias
  nuevas en ningún repositorio (plan.md Technical Context).
- El único characterization test CONGELA que se toca es `test_catalog_consumption_plan.py` (T010,
  T013, T017) — los 4 casos ya existentes de esa clase NO se modifican, citando esta spec como
  autorización de los casos nuevos (Constitución, Principio III).
- Verificar `quickstart.md` tras cada historia, no solo al final.
- Evitar: poner el default de `tracks_inventory` en `False` a nivel de modelo ORM (research.md
  Decisión 3 — rompería en silencio la suite de tests existente); parchar solo el guard
  (`ensure_lines_consume_inventory`) sin tocar `plan_line_consumption` (research.md Decisión 1/2 —
  dejaría generando movimientos de inventario reales para productos con insumos "abandonados");
  reescribir código de migraciones en Python importando `app/` en vez de SQL puro (research.md
  Decisión 4).
