---

description: "Lista de tareas para Tipo de precio e inventario condicional en grupos de opciones"
---

# Tasks: Tipo de precio e inventario condicional en grupos de opciones

**Input**: Documentos de diseño de `/specs/064-grupos-opciones-precio-inventario/` (plan.md,
spec.md, research.md, data-model.md, contracts/, quickstart.md)

**Tests**: incluidos donde hay lógica nueva que verificar (validación cruzada de `pricing_type`,
unificación del criterio de A-32, gating por plan a nivel de campo). El wiring puramente mecánico
(agregar `tenant: Tenant = Depends(get_tenant)` a un endpoint) no lleva test propio — se verifica
junto con el test de la validación que ese `tenant` habilita.

**Organization**: tareas agrupadas por historia de usuario (US1-US6, prioridades de `spec.md`).
No hay fase Foundational formal: la columna `pricing_type` y su migración/schema (necesarias para
US1, US2 y US6) viven dentro de la fase de US1 (la primera P1 que las necesita, mismo criterio que
spec 062 aplicó a su señal `inventarioIncluido()`) — US2 y US6 las referencian explícitamente en
vez de repetirlas. US3, US4 y US5 son independientes entre sí y de US1/US2, salvo un punto de
integración señalado explícitamente en US5 (extiende una señal creada en US3).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (ficheros distintos, sin dependencia de una tarea sin
  terminar)
- **[Story]**: historia de usuario a la que pertenece (US1-US6)
- Cada tarea incluye la ruta de fichero exacta, relativa a la raíz del repo sibling que
  corresponda (`pos-backend`, `pos-heladeria`, o `pos-specs` para el registro de anomalías)

## Path Conventions

Tres repositorios: dos sibling de `pos-specs` (Constitución §Alcance, plan.md §Project Structure),
más el propio `pos-specs` para el registro de anomalías vivo:

- Backend: `pos-backend/app/...` (rutas ya incluyen el prefijo `pos-backend/`)
- Frontend: `pos-heladeria/src/app/...` (rutas ya incluyen el prefijo `pos-heladeria/`)
- Registro de anomalías: `specs/000-reconocimiento/registro-de-anomalias.md` (dentro de este mismo
  repo `pos-specs`)

---

## Phase 1: Setup

**Purpose**: confirmar que el entorno está listo — esta spec no agrega ninguna dependencia nueva a
`requirements.txt`/`package.json` (plan.md, Technical Context).

- [X] T001 Confirmar entorno: `pos-backend` con el venv activado (`source env/bin/activate`,
  Python 3.14.4) y `pos-heladeria` con `npm install` ya corrido (Node 24.16.0). Confirmar que
  `alembic heads` en `pos-backend` resuelve a un único head (`94b7e35f5e5e`, research.md Decisión
  6) antes de crear la migración nueva. Crear la rama `064-grupos-opciones-precio-inventario` en
  ambos repos sibling, partiendo de su rama de desarrollo habitual (mismo patrón que spec 062,
  T001 de esa spec).

**Checkpoint**: entornos listos, sin instalar nada nuevo, head de Alembic confirmado.

---

## Phase 2: User Story 1 - Un grupo de sabores incluido nunca puede llevar recargo (Priority: P1) 🎯 MVP

**Goal**: `OptionGroup` gana un `pricing_type` explícito ("incluido"/"con_recargo"); un grupo
"incluido" bloquea cualquier precio distinto de $0 en sus opciones, tanto en el formulario como en
el backend.

**Independent Test**: crear un grupo marcado "Incluido", intentar agregarle una opción con precio
mayor a $0 → rechazado (422 en backend, campo bloqueado en frontend); con precio $0 → aceptado;
vender una presentación con esa opción elegida no suma ningún recargo al precio de línea.

### Implementación de User Story 1

- [X] T002 [P] [US1] En `pos-backend/app/models/option_group.py`: agregar columna
  `pricing_type: Mapped[str] = mapped_column(String(20), nullable=False, server_default="con_recargo")`
  y `CheckConstraint("pricing_type IN ('incluido', 'con_recargo')", name="ck_option_group_pricing_type")`
  a `__table_args__` (data-model.md, research.md Decisión 1).
- [X] T003 [P] [US1] Crear `pos-backend/alembic/versions/<hex>_option_groups_pricing_type.py` con
  `down_revision = '94b7e35f5e5e'`: `op.add_column` + `op.create_check_constraint` + backfill
  `UPDATE option_groups SET pricing_type = CASE WHEN EXISTS (opción con extra_price > 0) THEN
  'con_recargo' ELSE 'incluido' END`, envuelto en `@for_each_tenant_schema` con guarda `_has_table`
  (mismo patrón que `b8c9d0e1f2a3_option_group_active.py` y
  `e3f4a5b6c7d8_products_tracks_inventory.py`; research.md Decisión 6). Incluye `downgrade()`
  reversible (`drop_constraint` + `drop_column`).
- [X] T004 [US1] En `pos-backend/app/api/v1/catalog/schemas.py`: agregar
  `pricing_type: Literal["incluido", "con_recargo"]` **requerido, sin default** a
  `OptionGroupCreate`; `pricing_type: Literal["incluido", "con_recargo"] | None = None` a
  `OptionGroupUpdate`; `pricing_type: str` a `OptionGroupResponse` (cerca de las clases
  `OptionGroupCreate`/`OptionGroupUpdate`/`OptionGroupResponse`, `schemas.py:120-178`; depende de
  T002; [contracts/option-group-pricing-type.md](./contracts/option-group-pricing-type.md)).
- [X] T005 [US1] En `pos-backend/app/api/v1/catalog/router.py::create_option_group` (línea 281):
  pasar `pricing_type=body.pricing_type` al construir `OptionGroup(...)` (depende de T004).
- [X] T006 [US1] En `pos-backend/app/api/v1/catalog/router.py::add_option` (línea 347): tras
  `get_or_404(db, OptionGroup, group_id, ...)`, si `group.pricing_type == "incluido"` y
  `body.extra_price != 0` → `HTTPException(422, "Los grupos «Incluido» no permiten precio
  distinto de $0.")`, antes de construir la `Option` (depende de T004; FR-002).
- [X] T007 [US1] En `pos-backend/app/api/v1/catalog/router.py::update_option` (línea 380): mismo
  chequeo que T006 cuando `body.extra_price is not None`, resolviendo el grupo vía
  `option.option_group_id` (depende de T004, T006 — mismo fichero, aplicar en el mismo cambio o
  inmediatamente después).
- [X] T008 [US1] Crear
  `pos-backend/app/characterization_tests/test_catalog_option_groups_pricing_type.py`: casos para
  T005-T007 — crear grupo "incluido"/"con_recargo", agregar opción con precio > 0 en cada uno
  (422 vs. 201), actualizar `extra_price` de una opción existente de un grupo "incluido" (422),
  y vender una variante con una opción de un grupo "incluido" elegida confirmando que
  `compute_line_price` no suma recargo (quickstart.md, sección US1/US2, pasos 1-3 y 5).
- [X] T009 [P] [US1] En `pos-heladeria/src/app/modules/products/interfaces/product.interface.ts`:
  agregar `pricing_type: 'incluido' | 'con_recargo'` a `OptionGroup` (líneas 157-164),
  `OptionGroupForm` (167-171), `OptionGroupCreatePayload` (174-178) y
  `OptionGroupUpdatePayload` (181-186).
- [X] T010 [US1] En
  `pos-heladeria/src/app/modules/option-groups/services/option-group.service.ts`: incluir
  `pricing_type` en el payload de `createGroup`/`updateGroup` (líneas 66-82) y en los mapeadores
  `toGroup`/DTOs de respuesta (líneas 18-35, 141-162) (depende de T009).
- [X] T011 [US1] En
  `pos-heladeria/src/app/modules/option-groups/components/option-group-form.component.ts`:
  agregar `pricing_type: [null, Validators.required]` al `FormGroup` (líneas 82-86), con un
  control de UI (radio/select "Incluido"/"Con recargo") que no permite enviar sin elegir uno
  (FR-001; depende de T009).
- [X] T012 [US1] En
  `pos-heladeria/src/app/modules/option-groups/components/option-form.component.ts`: agregar
  `@Input() groupPricingType: 'incluido' | 'con_recargo' | null = null`; cuando sea `'incluido'`,
  el campo "Precio extra" (`app-money-input`, líneas 47-51) queda deshabilitado y fijo en `0`
  (depende de T009).
- [X] T013 [US1] En
  `pos-heladeria/src/app/modules/option-groups/pages/option-groups-page.component.ts`: pasar
  `[groupPricingType]="group.pricing_type"` al `option-form.component.ts` que abre para cada
  grupo (depende de T009, T012).
- [X] T014 [P] [US1] Crear
  `pos-heladeria/src/app/modules/option-groups/components/option-group-form.component.spec.ts`
  (hoy no existe): confirmar que el formulario no permite enviar sin `pricing_type` elegido, y que
  ambos valores se envían correctamente al servicio (depende de T011).
- [X] T015 [P] [US1] Crear
  `pos-heladeria/src/app/modules/option-groups/components/option-form.component.spec.ts` (hoy no
  existe): con `groupPricingType="incluido"`, el campo de precio aparece deshabilitado/fijo en
  `0`; con `"con_recargo"`, aparece editable con normalidad (depende de T012).

**Checkpoint**: un grupo "Incluido" no puede llevar precio distinto de $0, verificado en backend y
frontend, de forma independiente de cualquier otra historia.

---

## Phase 3: User Story 2 - Un grupo de toppings con recargo exige y muestra precio por opción (Priority: P1)

**Goal**: un grupo "Con recargo" permite precio libre por opción (sin cambio respecto a hoy); y
reclasificar un grupo de "Con recargo" a "Incluido" fuerza sus precios a $0, con confirmación
explícita del administrador antes de aplicar el cambio.

**Independent Test**: crear un grupo "Con recargo" con una opción con precio > 0 → funciona igual
que hoy; cambiarlo a "Incluido" → el sistema pide confirmación, y al aceptarla todas sus opciones
quedan en $0.

### Implementación de User Story 2

- [X] T016 [US2] En `pos-backend/app/api/v1/catalog/router.py::update_option_group` (línea 301):
  cuando `body.pricing_type == "incluido"` y `group.pricing_type` actual es `"con_recargo"`,
  ejecutar `db.execute(update(Option).where(Option.option_group_id == group_id).values(extra_price=0))`
  antes del `commit()` final (depende de T004/T005 de US1; FR-004,
  [contracts/option-group-pricing-type.md](./contracts/option-group-pricing-type.md)).
- [X] T017 [US2] Extender
  `pos-backend/app/characterization_tests/test_catalog_option_groups_pricing_type.py` (T008): caso
  con un grupo "con_recargo" con opciones en precio > 0, `PATCH` a `pricing_type="incluido"` →
  `200` y las opciones quedan en `extra_price=0`; caso inverso ("incluido"→"con_recargo") sin
  ningún efecto lateral sobre precios existentes.
- [X] T018 [US2] En
  `pos-heladeria/src/app/modules/option-groups/components/option-group-form.component.ts`: antes
  de enviar un cambio de `pricing_type` a `"incluido"` sobre un grupo que tiene opciones con
  `extra_price > 0` (recibidas como `@Input()`), invocar `ConfirmService.ask(...)` (mismo patrón
  que `toggleTracksInventory()` en `product-form.component.ts:854-868`, spec 027); si se cancela,
  el formulario no envía el `PATCH` y el control vuelve a su valor anterior (depende de T011).
- [X] T019 [P] [US2] Extender
  `pos-heladeria/src/app/modules/option-groups/components/option-group-form.component.spec.ts`
  (T014): casos de confirmación — aceptar dispara el `PATCH`, cancelar no lo dispara y restaura el
  valor previo del control (depende de T018).

**Checkpoint**: US1 y US2 juntas cubren el ciclo completo de "tipo de grupo" — creación, precio
libre en "con_recargo", bloqueo y reclasificación segura hacia "incluido".

---

## Phase 4: User Story 3 - Sin "maneja inventario", sus grupos de opciones tampoco descuentan stock (Priority: P1)

**Goal**: el formulario de producto deja de ocultar por completo la posibilidad de ofrecer un
grupo de opciones cuando `tracks_inventory` está apagado — solo oculta los campos específicamente
relacionados con inventario (cantidad de consumo, detalle de insumo).

**Independent Test**: con el switch "Maneja inventario" apagado, el bloque "Sabores a elegir"
sigue permitiendo elegir un grupo y su `min_select`/`max_select`; el input "descuenta [cantidad]"
y el detalle de insumo no aparecen; vender esa presentación con una opción elegida no genera
ningún movimiento de inventario.

### Implementación de User Story 3

- [X] T020 [US3] En
  `pos-heladeria/src/app/modules/products/pages/product-form.component.ts`: restructurar el
  bloque "Sabores a elegir" (líneas 302-408, hoy completo dentro del `@if (draft().tracks_inventory)`
  de la línea 275). Sacar del `@if` de inventario: el selector de grupo (`g.option_group_id`,
  líneas 314-316), el par `min_select`/`max_select` (líneas 322-329), y el botón "Agregar sabores
  a elegir" (línea 405) — deben verse y editarse siempre. Envolver en una señal nueva
  `readonly sectionsEnabled = computed(() => this.draft().tracks_inventory)` (punto de extensión
  para US5, research.md Decisión 5): el input "descuenta [cantidad] por cada uno"
  (`quantity_per_option`, líneas 330-334), el resumen "Descuenta de: …" y su tabla de detalle
  (líneas 347-401). "Insumos fijos" (líneas 276-300) permanece sin fraccionar, detrás de
  `@if (draft().tracks_inventory)` sin cambios — es 100% inventario. El bloque `@else` (línea 409)
  ya no aplica a "Sabores a elegir" — solo al bloque "Insumos fijos" ahora separado (FR-006).
- [X] T021 [US3] Actualizar
  `pos-heladeria/src/app/modules/products/pages/product-form.component.spec.ts`: el test "la
  sección de insumos aparece deshabilitada mientras el switch está apagado" (líneas 104-110) se
  ajusta — con `tracks_inventory=false`, el DOM ya no debe carecer del selector de grupo ni de
  `min_select`/`max_select` (deben seguir presentes y editables), solo debe carecer del input de
  cantidad y del detalle de insumo. Agregar un caso nuevo: elegir un grupo de opciones y guardar
  con `tracks_inventory=false` persiste correctamente esa asignación (depende de T020; cita esta
  spec como autorización del cambio de comportamiento respecto al test anterior, Principio III).

**Checkpoint**: un producto sin inventario puede ofrecer grupos de opciones con precio con
normalidad — ya no es una limitación accidental de la UI. Sin cambios en backend (`_tracks_inventory`
ya cubre correctamente el descuento real, spec 027).

---

## Phase 5: User Story 4 - Un solo criterio decide si una opción descuenta inventario (Priority: P2) — corrige A-32

**Goal**: `grupos_que_descuentan` y `group_discounts` responden siempre lo mismo ante la misma
opción — se unifican hacia el criterio de tres condiciones ya usado por `group_discounts`.

**Independent Test**: una opción con `item_quantity=10` e `inventory_item_id=None` deja de
clasificarse como "descuenta inventario" por `grupos_que_descuentan`, igual que ya no lo hacía
`group_discounts` — ambas funciones coinciden en cualquier combinación de datos.

### Implementación de User Story 4

- [X] T022 [US4] En `pos-backend/app/catalog_engine/pricing.py::grupos_que_descuentan` (líneas
  62-84): agregar `Option.active.is_(True)` y `Option.inventory_item_id.is_not(None)` a la
  cláusula `where` de `por_opcion` (líneas 73-78), igualando exactamente el criterio de
  `group_discounts` (`consumption.py:78-84`) (research.md Decisión 3).
- [X] T023 [US4] Actualizar `pos-backend/app/characterization_tests/test_catalog_line_pricing.py`:
  localizar el/los test(s) que fijan el criterio de una sola condición (`item_quantity > 0` sin
  exigir insumo enlazado) y actualizarlos para reflejar el nuevo criterio de tres condiciones,
  citando en un comentario esta spec (064) y la entrada de `registro-de-anomalias.md` (T025) como
  autorización del cambio (Principio III de la Constitución; depende de T022).
- [X] T024 [P] [US4] Agregar un caso nuevo (en `test_catalog_line_pricing.py` o
  `test_catalog_consumption_plan.py`) que invoque `grupos_que_descuentan` y `group_discounts` sobre
  el mismo `VariantOptionGroup`/`Option` con datos idénticos y confirme que ambas devuelven el
  mismo resultado en al menos tres combinaciones: (a) insumo enlazado + activo +
  cantidad > 0; (b) `item_quantity > 0` sin insumo enlazado; (c) insumo enlazado pero
  `active=False` (depende de T022).
- [X] T025 [US4] Agregar una entrada nueva en
  `specs/000-reconocimiento/registro-de-anomalias.md` marcando `A-32` como **resuelta**: cita spec
  064, fecha, qué cambia (`grupos_que_descuentan` unificado hacia el criterio de
  `group_discounts`), por qué (ese criterio ya era el tratado como canónico en la migración
  `e3f4a5b6c7d8_products_tracks_inventory.py`), y qué funcionalidades se ven afectadas
  (`validate_option_selection`, spec 004 `RN-CAT-39`) — sin editar la entrada original de A-32
  (research.md Decisión 7; Principio II de la Constitución).

**Checkpoint**: A-32 queda resuelta con evidencia de código, tests actualizados deliberadamente, y
trazabilidad en el registro de anomalías — independiente de cualquier otra historia.

---

## Phase 6: User Story 5 - Sin el módulo Inventario, ningún producto puede activar "maneja inventario" (Priority: P2)

**Goal**: activar `tracks_inventory` en un producto, o guardar un insumo/cantidad de consumo en
una opción, exige que el tenant tenga el módulo Inventario incluido en su plan vigente — a nivel
de campo, no de ruta completa (research.md Decisión 4).

**Independent Test**: con un tenant sin el módulo Inventario, `POST /products` con
`tracks_inventory=true` → `403`; con `tracks_inventory=false` → `201` sin importar el plan;
`POST /option-groups/{id}/options` con insumo enlazado → `403`; con solo precio → `201`.

### Implementación de User Story 5

- [X] T026 [P] [US5] En `pos-backend/app/core/plan_limits.py`: extraer
  `ensure_module_access(db: Session, tenant: Tenant, module_key: str) -> None` con la lógica
  interna de `require_module_access` (líneas 126-149) — `require_module_access(module_key)` pasa a
  ser un wrapper delgado que llama a `ensure_module_access` dentro de su `_dependency`, sin cambiar
  su firma ni su comportamiento como dependencia FastAPI para los routers que ya la usan (spec
  033/062) (research.md Decisión 4).
- [X] T027 [US5] En `pos-backend/app/api/v1/products/router.py::update_product` (línea 104):
  agregar `tenant: Tenant = Depends(get_tenant)` (hoy no lo recibe; `create_product` ya lo tiene,
  línea 79).
- [X] T028 [US5] En `pos-backend/app/api/v1/products/service.py`: en `create_product` (línea 55) y
  `update_product` (línea 87), llamar `ensure_module_access(db, tenant, "inventario")` cuando
  `data.tracks_inventory is True` — en `update_product`, solo cuando además sea distinto del valor
  actual del producto (evita reevaluar el plan en cada `PATCH` que no toca este campo). Requiere
  que ambos métodos reciban `tenant: Tenant` como parámetro nuevo, pasado desde el router (depende
  de T026, T027; [contracts/inventory-field-plan-gating.md](./contracts/inventory-field-plan-gating.md)).
- [X] T029 [US5] En `pos-backend/app/api/v1/catalog/router.py::add_option` (línea 347) y
  `::update_option` (línea 380): agregar `tenant: Tenant = Depends(get_tenant)` a ambos, y llamar
  `ensure_module_access(db, tenant, "inventario")` cuando el resultado final de la opción tenga
  `inventory_item_id is not None` o `item_quantity > 0` — evaluado **después** de aplicar las
  reglas ya existentes de `RN-CAT-38` (desvincular resetea `item_quantity` a `0`) (depende de T026;
  mismo fichero que T006/T007 de US1 — aplicar como cambios adicionales sobre las mismas funciones,
  no en conflicto: T006/T007 validan `pricing_type`, esta tarea valida acceso a módulo).
- [X] T030 [P] [US5] Crear `pos-backend/app/characterization_tests/test_plan_gating_inventory_fields.py`:
  mismo patrón de invocación directa que `test_plan_module_access.py` — casos para T028 (crear/
  actualizar producto con `tracks_inventory=true` sin módulo → 403; con `false` → 201 sin importar
  el plan) y T029 (crear/actualizar opción con insumo enlazado sin módulo → 403; opción solo con
  precio → 201); caso de no-regresión: un producto/opción que ya tenía inventario configurado
  **antes** de perder el acceso conserva sus datos sin que ningún `GET` los borre o modifique
  (depende de T028, T029; quickstart.md sección US5).
- [X] T031 [US5] En `pos-heladeria/src/app/modules/products/pages/product-form.component.ts`:
  inyectar `PlanSummaryService`; agregar
  `readonly inventarioIncluido = computed(() => { const s = this.planSummaryService.summary(); return s !== null && s.modules.inventario && !s.vencido; })`
  (fail-closed, mismo estilo que `comprasIncluido` en `inventory-page.component.ts:385-388`);
  deshabilitar (no ocultar) el switch "Maneja inventario" (línea 181) cuando
  `!inventarioIncluido()`; extender la señal `sectionsEnabled()` creada en T020 (US3) a
  `computed(() => this.draft().tracks_inventory && this.inventarioIncluido())` (research.md
  Decisión 5; depende de T020).
- [X] T032 [US5] En
  `pos-heladeria/src/app/modules/option-groups/pages/option-groups-page.component.ts`: inyectar
  `PlanSummaryService`; agregar el mismo computed `inventarioIncluido()` que T031; pasar
  `[inventarioIncluido]="inventarioIncluido()"` al `option-form.component.ts` que abre.
- [X] T033 [US5] En
  `pos-heladeria/src/app/modules/option-groups/components/option-form.component.ts`: agregar
  `@Input() inventarioIncluido = false`; ocultar el bloque "Insumo que consume"/"Cantidad
  consumida" (líneas 53-73) cuando `!inventarioIncluido` — gating por plan, independiente del
  `groupPricingType` de T012 y de cualquier producto específico (research.md Decisión 5; depende
  de T032; mismo fichero que T012 de US1 — cambios adicionales sobre el mismo componente, sin
  conflicto: un `@Input()` gobierna precio, el otro gobierna insumo).
- [X] T034 [P] [US5] Extender `product-form.component.spec.ts` (T021) con casos de plan: switch
  deshabilitado cuando `modules.inventario=false`; un producto existente con `tracks_inventory=true`
  conserva ese valor visible aunque el switch esté deshabilitado (no se fuerza a `false` en el
  formulario). Extender `option-form.component.spec.ts` (T015) con: `inventarioIncluido=false`
  oculta insumo/cantidad sin importar `groupPricingType` (depende de T031, T033).

**Checkpoint**: activar inventario (a nivel de producto u opción) exige el módulo del plan, sin
bloquear la creación de productos ni de toppings con precio para tenants sin ese módulo.

---

## Phase 7: User Story 6 - Los grupos existentes se reclasifican sin cambiar su comportamiento de venta (Priority: P2)

**Goal**: verificar que la migración de T003 (US1) clasifica correctamente el catálogo existente,
sin alterar ningún precio, insumo o resultado de venta ya vigente.

**Independent Test**: sobre un tenant con un grupo con precios > 0 y otro en $0 exclusivamente,
tras `alembic upgrade head` cada uno queda clasificado como corresponde, y vender presentaciones
que los usan produce exactamente el mismo precio y consumo que antes de la migración.

### Implementación de User Story 6

- [X] T035 [US6] Crear `pos-backend/app/scripts/verify_option_groups_pricing_type_backfill.py`
  (mismo patrón que `verify_tracks_inventory_backfill.py`, spec 027): contra un tenant real o de
  prueba, compara la clasificación resultante de la migración de T003 contra el criterio esperado
  (grupo con alguna opción `extra_price > 0` → `con_recargo`; todas en `$0` → `incluido`), y
  reporta cualquier discrepancia sin modificar datos.
- [X] T036 [US6] Ejecutar `alembic upgrade head` sobre un esquema de tenant con datos de prueba
  mixtos (grupos con y sin precio) y correr `verify_option_groups_pricing_type_backfill.py` (T035)
  — confirmar cero discrepancias, y que vender una presentación de cada grupo migrado produce el
  mismo precio de línea y el mismo consumo de inventario que antes de la migración (quickstart.md,
  sección US6; depende de T003, T035).

**Checkpoint**: la migración de T003 queda verificada contra datos reales, sin regresiones de
precio ni de inventario.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de que ninguna historia rompió a otra.

- [X] T037 [P] Ejecutar
  `cd pos-backend && python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`
  — confirmar 100% verde, incluyendo los tests nuevos/actualizados de T008, T017, T023, T024, T030.
- [X] T038 [P] Ejecutar `cd pos-heladeria && npx ng test --watch=false` — confirmar 100% verde,
  incluyendo los specs nuevos/actualizados de T014, T015, T019, T021, T034.
- [ ] T039 Ejecutar la sección "Verificación manual end-to-end" de
  [quickstart.md](./quickstart.md) (3 escenarios: producto sin inventario con toppings con precio;
  reasignación del módulo Inventario; grupo "Incluido" con insumo enlazado que sí descuenta pese a
  no cobrar recargo) y registrar el resultado.
  **Pendiente**: requiere navegador real (Angular dev server + sesión de usuario) — no ejecutable
  desde este entorno de implementación. Verificado por vía equivalente: suite completa de
  characterization tests backend (553/553 verde), suite completa de tests frontend (589/592 verde,
  las 3 fallas son preexistentes y no relacionadas — `app.spec.ts`, `auth.service.spec.ts`,
  `pos-checkout-panel.component.spec.ts`, confirmado sin diff contra `develop`), y migración
  `68326ed66ebf` aplicada y verificada contra los tres tenants reales del entorno de desarrollo
  (`heladeria`, `heladeria2`, `heladeria3` — cero discrepancias, ver T036).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciar de inmediato.
- **US1 (Phase 2)**: depende de Setup. Es la base de datos/schema de `pricing_type` que consumen
  US2 y US6 — no bloquea a US3, US4 o US5, que son enteramente independientes de `pricing_type`.
- **US2 (Phase 3)**: depende de T004/T005 de US1 (el campo `pricing_type` debe existir en el
  schema y persistirse al crear un grupo antes de poder reclasificarlo).
- **US3 (Phase 4)**: depende solo de Setup — independiente de US1/US2. Crea la señal
  `sectionsEnabled()` que US5 extiende (T031 depende de T020).
- **US4 (Phase 5)**: depende solo de Setup — completamente independiente de las demás historias
  (toca `catalog_engine/pricing.py`, ningún otro archivo tocado por US1/US2/US3/US5).
- **US5 (Phase 6)**: depende de Setup y, para el frontend (T031), de T020 (US3). El backend de US5
  (T026-T030) es independiente de todas las demás historias.
- **US6 (Phase 7)**: depende de T003 (la migración de US1) — es puramente verificación, no agrega
  código de producción nuevo.
- **Polish (Phase 8)**: depende de que todas las historias que se vayan a entregar estén completas.

### Parallel Opportunities

- T002 y T003 (US1) son paralelos entre sí (ficheros distintos, sin dependencia de código entre
  ellos).
- T009 (US1, frontend interfaces) es paralelo a todo el trabajo backend de US1 (T002-T008).
- T014 y T015 (US1, tests frontend nuevos) son paralelos entre sí.
- **US3 y US4 pueden implementarse en paralelo con US1/US2** — no comparten ningún archivo.
- El backend de US5 (T026) puede empezar en paralelo con US1/US2/US3/US4 — solo su parte frontend
  (T031) espera a T020 (US3).
- T037 y T038 (Polish) son paralelos entre sí.

---

## Parallel Example: User Story 1

```bash
# Backend: modelo y migración en paralelo
Task: "Agregar columna pricing_type en pos-backend/app/models/option_group.py"
Task: "Crear migración option_groups_pricing_type en pos-backend/alembic/versions/"

# Frontend: interfaces TypeScript en paralelo con el resto del backend de US1
Task: "Agregar pricing_type a OptionGroup/OptionGroupForm en product.interface.ts"

# Tests frontend nuevos, en paralelo entre sí
Task: "Crear option-group-form.component.spec.ts"
Task: "Crear option-form.component.spec.ts"
```

---

## Implementation Strategy

### MVP (las tres historias P1 juntas)

A diferencia de un MVP de una sola historia, aquí las tres historias P1 (US1, US2, US3) componen
el problema central reportado por el usuario: sin las tres, el modelo de "tipo de grupo" queda a
medias (US1 sin US2 no permite reclasificar) o el problema de fondo persiste (US3 es la que
realmente destraba "un producto sin inventario puede tener toppings con precio", el pedido
original). Orden recomendado:

1. Completar Fase 1: Setup.
2. Completar Fase 2: US1 (tipo de grupo, base de datos y bloqueo de precio).
3. Completar Fase 3: US2 (reclasificación segura) — **valida y demuestra** de punta a punta.
4. Completar Fase 4: US3 (cascada del switch) — puede hacerse en paralelo con 2-3 por un segundo
   desarrollador, ya que no comparte archivos.
5. **DETENER y VALIDAR**: correr `quickstart.md` secciones US1-US3 completas.

### Entrega incremental

1. Setup + US1 + US2 + US3 → MVP funcional, demostrable.
2. Agregar US4 (corrige A-32) → mejora de correctitud, sin cambio visible para el usuario final.
3. Agregar US5 (gating por plan) → cierra el pedido explícito de "coincidir con las reglas de
   acceso en el plan".
4. Agregar US6 (verificación de migración) → cierra la trazabilidad de que nada se rompió.
5. Polish → confirmación final de las dos suites de test completas y del recorrido manual.

### Estrategia de equipo en paralelo

Con varios desarrolladores, tras completar Setup:

- Desarrollador A: US1 → US2 (comparten archivos, mejor en secuencia por la misma persona).
- Desarrollador B: US3 (independiente, sin fricción de archivos con A).
- Desarrollador C: US4 (independiente, backend puro).
- Desarrollador D: backend de US5 (T026-T030, independiente); su parte frontend (T031) espera a
  que B termine T020.
- US6 la ejecuta quien terminó US1 (mismo contexto de la migración).

---

## Notes

- [P] = ficheros distintos, sin dependencia de una tarea sin terminar.
- La columna `pricing_type` (US1) y la señal `sectionsEnabled()` (US3) son los dos únicos puntos
  de integración entre historias — ambos están documentados explícitamente en Dependencies, no son
  acoplamientos implícitos.
- T025 (entrada en `registro-de-anomalias.md`) es de cumplimiento obligatorio antes de considerar
  US4 completa — Principio II de la Constitución no la considera opcional.
- Verificar que los tests fallan antes de implementar donde corresponda (T008, T017, T023-T024,
  T030 antes de sus tareas de implementación asociadas, cuando el equipo siga TDD).
- Hacer commit tras cada tarea o grupo lógico; detenerse en cada Checkpoint para validar la
  historia de forma independiente antes de continuar.
