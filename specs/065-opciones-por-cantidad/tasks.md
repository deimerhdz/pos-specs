---

description: "Lista de tareas para Selección por cantidad en grupos de opciones"
---

# Tasks: Selección por cantidad en grupos de opciones

**Input**: Documentos de diseño de `/specs/065-opciones-por-cantidad/` (plan.md, spec.md,
research.md, data-model.md, contracts/, quickstart.md)

**Tests**: incluidos donde hay lógica nueva que verificar (parseo de cantidad, rechazo de
duplicados, rechazo de `quantity>1` en modo "conteo", multiplicación de precio/consumo, topes,
formateo "2x Nombre"). El wiring puramente mecánico (pasar `options` en vez de `option_ids` a una
función que ya existe) no lleva test propio — se verifica junto con el test de la función que ese
wiring alimenta.

**Organization**: tareas agrupadas por historia de usuario (US1-US6, prioridades de `spec.md`). No
hay fase Foundational formal: la columna `selection_mode`/los topes y la migración completa
(`option_groups` + `cart_item_options` + `order_item_options`, research.md Decisión 5) viven dentro
de la fase de US1 (la primera P1) porque es una sola migración atómica; US2 es la que introduce el
tipo compartido `ChosenOption` y el cambio de contrato `option_ids→options`, del que dependen
US3-US6 explícitamente (ver Dependencies).

**Corrección post-`/speckit-analyze`**: T033-T036 se agregaron tras un análisis cruzado
spec/plan/tasks que encontró que la generalización de `plan_line_consumption` (T031) no bastaba —
sus dos consumidores directos en el mismo módulo (`required_consumption`,
`ensure_lines_consume_inventory`) y la cadena real de deducción de inventario
(`orders/consumption.py`, `sales/consumption.py`, y tres llamadas adicionales en
`cart/service.py`) quedaban con un tipo desalineado, lo que habría roto en tiempo de ejecución la
confirmación de una orden o el cobro de una venta de mostrador con un grupo "cantidad". `plan.md`
(Scale/Scope, Constitution Check, Project Structure) ya refleja esta corrección.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (ficheros distintos, sin dependencia de una tarea sin
  terminar)
- **[Story]**: historia de usuario a la que pertenece (US1-US6)
- Cada tarea incluye la ruta de fichero exacta, relativa a la raíz del repo sibling que
  corresponda (`pos-backend`, `pos-heladeria`)

## Path Conventions

Dos repositorios sibling de `pos-specs` (Constitución §Alcance, plan.md §Project Structure):

- Backend: `pos-backend/app/...` (rutas ya incluyen el prefijo `pos-backend/`)
- Frontend: `pos-heladeria/src/app/...` (rutas ya incluyen el prefijo `pos-heladeria/`)

---

## Phase 1: Setup

**Purpose**: confirmar que el entorno está listo — esta spec no agrega ninguna dependencia nueva.

- [X] T001 Confirmar entorno: `pos-backend` con el venv activado (Python 3.14.4) y `pos-heladeria`
  con `npm install` ya corrido (Node 24.16.0). Crear la rama
  `065-opciones-por-cantidad` en ambos repos sibling, partiendo de
  `064-grupos-opciones-precio-inventario` (research.md, "Nota sobre numeración de ramas") — no de
  `develop`, salvo que esa rama ya esté integrada para entonces. Confirmar que
  `OptionGroup.pricing_type` (spec 063/064) ya existe en el schema antes de continuar.

**Checkpoint**: entornos listos, rama base correcta confirmada.

---

## Phase 2: User Story 1 - Declarar que un grupo se elige "por cantidad" (Priority: P1) 🎯 MVP

**Goal**: `OptionGroup` gana un `selection_mode` explícito ("conteo"/"cantidad", default "conteo")
y dos topes opcionales de cantidad; un grupo "cantidad" deja de exigir `min_select`/`max_select`.

**Independent Test**: crear un grupo sin especificar modo → queda en "conteo"; crear uno con
`selection_mode="cantidad"` → el formulario dejar de pedir rango de opciones distintas y en su
lugar ofrece los dos topes.

### Implementación de User Story 1

- [X] T002 [P] [US1] En `pos-backend/app/models/option_group.py`: agregar
  `selection_mode: Mapped[str] = mapped_column(String(20), nullable=False, server_default="conteo")`
  con `CheckConstraint("selection_mode IN ('conteo', 'cantidad')")`;
  `max_quantity_per_option: Mapped[Optional[int]] = mapped_column(Integer, nullable=True)` con
  `CheckConstraint("max_quantity_per_option IS NULL OR max_quantity_per_option > 0")`;
  `max_total_quantity` análogo (data-model.md).
- [X] T003 [P] [US1] Crear
  `pos-backend/alembic/versions/<hex>_option_group_selection_mode_and_item_option_quantity.py` con
  `down_revision = '68326ed66ebf'` (head real confirmado, research.md Decisión 5): agrega las tres
  columnas de T002 a `option_groups`, y `quantity INTEGER NOT NULL DEFAULT 1` con
  `CHECK (quantity > 0)` a `cart_item_options` y `order_item_options` — una sola migración atómica,
  mismo patrón `_has_table` + `@for_each_tenant_schema` que las migraciones de spec 027/063. Sin
  backfill por consulta (data-model.md: todo dato histórico ya es `conteo`/`quantity=1` por
  construcción). `downgrade()` reversible.
- [X] T004 [US1] En `pos-backend/app/api/v1/catalog/schemas.py`: agregar
  `selection_mode: Literal["conteo", "cantidad"] = "conteo"` (con default, a diferencia de
  `pricing_type`) a `OptionGroupCreate`; `selection_mode: Literal[...] | None`,
  `max_quantity_per_option: int | None`, `max_total_quantity: int | None` a `OptionGroupUpdate`; los
  tres campos a `OptionGroupResponse` (depende de T002;
  [contracts/option-group-selection-mode.md](./contracts/option-group-selection-mode.md)).
- [X] T005 [US1] En `pos-backend/app/api/v1/catalog/router.py::create_option_group` y
  `::update_option_group`: persistir/actualizar `selection_mode`, `max_quantity_per_option`,
  `max_total_quantity` (mismo patrón ya usado para `pricing_type`, spec 063/064) (depende de T004).
- [X] T006 [US1] Crear
  `pos-backend/app/characterization_tests/test_catalog_option_groups_selection_mode.py`: casos para
  T004-T005 — crear/actualizar un grupo con `selection_mode="cantidad"` y ambos topes, confirmar
  que se guardan; confirmar que un grupo sin especificar modo queda en `"conteo"`.
- [X] T007 [P] [US1] En `pos-heladeria/src/app/modules/products/interfaces/product.interface.ts`:
  agregar `selection_mode: 'conteo' | 'cantidad'`, `max_quantity_per_option: number | null`,
  `max_total_quantity: number | null` a `OptionGroup`, `OptionGroupForm`,
  `OptionGroupCreatePayload`, `OptionGroupUpdatePayload`.
- [X] T008 [US1] En
  `pos-heladeria/src/app/modules/option-groups/services/option-group.service.ts`: incluir los tres
  campos nuevos en el payload de `createGroup`/`updateGroup` y en los mapeadores de respuesta
  (depende de T007).
- [X] T009 [US1] En
  `pos-heladeria/src/app/modules/option-groups/components/option-group-form.component.ts`: agregar
  control de UI "Conteo"/"Cantidad" (mismo patrón radio que `pricing_type`); cuando es "cantidad",
  ocultar `min_select`/`max_select` y mostrar los dos topes opcionales (depende de T007).
- [X] T010 [P] [US1] Extender
  `pos-heladeria/src/app/modules/option-groups/components/option-group-form.component.spec.ts`:
  casos para el nuevo control de modo y los topes (depende de T009).

**Checkpoint**: un grupo puede declararse "cantidad", con sus topes, de forma independiente de
cualquier otra historia.

---

## Phase 3: User Story 2 - El cliente elige libremente cuántas unidades de cada topping agregar (Priority: P1)

**Goal**: el carrito, la comanda de mesero y la venta de mostrador aceptan una cantidad por opción
elegida (no solo presencia/ausencia); un grupo "conteo" sigue exigiendo exactamente 1 unidad por
opción, y un grupo "cantidad" nunca bloquea por no elegir nada.

**Independent Test**: en un grupo "cantidad", incrementar "Bobombún" a 2 y "Gomitas" a 1 en el
carrito; confirmar que ambas cantidades llegan y se persisten tal cual, y que el pedido avanza
igual si ambas quedan en 0.

### Implementación de User Story 2

- [X] T011 [P] [US2] En `pos-backend/app/catalog_engine/core.py`: agregar
  `class ChosenOption(NamedTuple): option: Option; quantity: int` (research.md Decisión 2).
- [X] T012 [P] [US2] En `pos-backend/app/api/v1/catalog/schemas.py`: agregar
  `class OptionSelectionIn(BaseModel): option_id: UUID; quantity: int = Field(1, ge=1)` (contracts/
  cart-order-option-quantity.md).
- [X] T013 [US2] En `pos-backend/app/catalog_engine/pricing.py::load_valid_options`
  (líneas 36-59): cambiar la firma a recibir `selections: list[OptionSelectionIn]` y devolver
  `list[ChosenOption]`; rechazar con `422` si el mismo `option_id` aparece dos veces en el payload
  (research.md Decisión 1/2; depende de T011, T012).
- [X] T014 [US2] En `pos-backend/app/catalog_engine/pricing.py::validate_option_selection`
  (líneas 87-167): antes de la lógica de conteo existente, separar por `selection_mode` del grupo
  de cada `ChosenOption` — grupos "conteo" con `quantity != 1` → `422` ("esta opción no admite más
  de una unidad"); grupos "cantidad" nunca entran en la lista de "obligatorio sin elegir nada"
  (líneas 135-138) — no hay mínimo posible (research.md Decisión 3; depende de T013).
- [X] T015 [US2] En `pos-backend/app/api/v1/cart/schemas.py`: reemplazar
  `option_ids: list[UUID]` por `options: list[OptionSelectionIn]` en `CartItemIn` (líneas 40-44) y
  `CartItemUpdate` (líneas 46-49); agregar `quantity: int` a `CartItemOptionResponse` (líneas
  51-55) (depende de T012).
- [X] T016 [US2] En `pos-backend/app/api/v1/cart/service.py`: los dos call sites de
  `load_valid_options(db, data.option_ids, ...)` (líneas 318, 359) pasan a
  `load_valid_options(db, data.options, ...)`; las dos construcciones de `CartItemOption(...)`
  (líneas 337, 384) agregan `quantity=chosen.quantity` (depende de T013, T015).
- [X] T017 [US2] En `pos-backend/app/api/v1/orders/schemas.py` (línea 117): mismo reemplazo
  `option_ids`→`options: list[OptionSelectionIn]` en `OrderItemIn` — reutilizada tal cual tanto por
  `POST /orders` como por `POST /tables/{table_id}/items` (`add_item_to_table`, T019), así que este
  único cambio de schema cubre ambos endpoints (depende de T012).
- [X] T018 [US2] En `pos-backend/app/api/v1/orders/service.py::create_order` (línea 242): pasar
  `line.options`; la construcción de `OrderItemOption(...)` (línea 255) agrega
  `quantity=chosen.quantity` (depende de T013, T017).
- [X] T019 [US2] En `pos-backend/app/api/v1/orders/consolidation.py::add_item_to_table`
  (líneas 187-214, el camino real de `load_valid_options`/`compute_line_price` — no confundir con
  T020): pasar `data.options`; la construcción de `OrderItemOption(...)` (línea 214) agrega
  `quantity=chosen.quantity` (depende de T013, T017).
- [X] T020 [US2] En `pos-backend/app/api/v1/orders/consolidation.py::consolidate_table`
  (líneas 107-154, el camino de COPIA desde el carrito, sin recalcular — distinto de T019): la
  construcción de `OrderItemOption(...)` (línea 154) agrega `quantity=ci_option.quantity` leído
  directamente de cada `CartItemOption` de `ci.options`, sin pasar por `load_valid_options`
  (depende de T016, mismo criterio de "no recomputar" ya documentado en el código).
- [X] T021 [US2] En `pos-backend/app/api/v1/sales/service.py` (líneas 233-247): mismo reemplazo
  `line.option_ids`→`line.options`; `options_snapshot` (líneas 235-241) agrega
  `"quantity": chosen.quantity` a cada dict (depende de T013;
  [contracts/cart-order-option-quantity.md](./contracts/cart-order-option-quantity.md)).
- [X] T022 [US2] En `pos-backend/app/api/v1/cart/service.py` (función de `POST /cart/submit`,
  líneas ~591-610, copia desde el carrito sin recalcular): la construcción de
  `OrderItemOption(...)` (línea 610) agrega `quantity=o.quantity` leído del `CartItemOption` `o`
  (depende de T016).
- [X] T023 [P] [US2] Crear
  `pos-backend/app/characterization_tests/test_catalog_option_groups_quantity.py`: casos para
  `load_valid_options` (cantidad parseada correctamente, `422` por `option_id` repetido),
  `validate_option_selection` (`422` por `quantity>1` en grupo "conteo"; grupo "cantidad" con todas
  las cantidades en 0 no bloquea) (depende de T013, T014).
- [X] T024 [US2] Buscar y actualizar todo characterization test existente que ya no compile o ya
  no represente el contrato real tras T013/T015/T017/T021: (a) `grep -rn "option_ids"
  pos-backend/app/characterization_tests/` — cualquier test que construya `CartItemIn`/línea de
  orden/línea de venta con `option_ids` migra a `options: [OptionSelectionIn(...)]`; (b) `grep -rn
  "plan_line_consumption\|compute_line_price\|validate_option_selection\|required_consumption\|
  ensure_lines_consume_inventory" pos-backend/app/characterization_tests/` — cualquier test que
  llame a estas funciones pasando una lista cruda de `Option` (ej.
  `plan_line_consumption(db, variant_id, qty, [opt])`) migra a envolver cada una en
  `ChosenOption(opt, 1)`; citar esta spec como autorización del cambio de forma (Principio III;
  depende de T015, T017, T021, y de T033-T035 más abajo para las funciones de consumo).
- [X] T025 [P] [US2] En
  `pos-heladeria/src/app/modules/tables/interfaces/diner.interface.ts`: reemplazar
  `option_ids?: string[]` por `options?: OptionSelectionPayload[]` en `CartItemPayload` (líneas
  44-49) y `CartItemUpdatePayload` (líneas 52-56); agregar `quantity: number` a `CartItemOption`
  (líneas 58-62).
- [X] T026 [US2] En
  `pos-heladeria/src/app/modules/tables/services/dining-cart.service.ts` (función `add()`, líneas
  85-100): construir `options: OptionSelectionPayload[]` con cantidades en vez de
  `option_ids: string[]` (depende de T025).
- [X] T027 [P] [US2] En
  `pos-heladeria/src/app/modules/products/interfaces/product.interface.ts`: agregar
  `selection_mode: 'conteo' | 'cantidad'` a `MenuOptionGroup`; agregar `quantity: number` a la
  entrada de opción elegida dentro de `ProductSelection`.
- [X] T028 [US2] En
  `pos-heladeria/src/app/modules/tables/components/product-select.component.ts`: generalizar
  `selected: Record<groupId, string[]>` (línea 247) a
  `Record<groupId, Record<optionId, number>>`; agregar rama de plantilla con control `+`/`-` por
  opción (mismo patrón visual que `dec()`/`inc()` de la cantidad de línea, líneas 206-219), visible
  cuando `group.selection_mode === 'cantidad'`, reemplazando el botón toggle (líneas 133-155) solo
  para esos grupos; ramificar `chosenCount`/`isComplete`/`groupHint`/`requiredCount`
  (líneas 297-471) para que un grupo "cantidad" nunca aparezca como incompleto (research.md
  Decisión 8; depende de T027).
- [X] T029 [P] [US2] Extender
  `pos-heladeria/src/app/modules/tables/components/product-select.component.spec.ts`: casos para
  un grupo "cantidad" — incrementar dos opciones distintas, bajar una a 0, confirmar que
  `canConfirm()` nunca se bloquea por ese grupo (depende de T028).

**Checkpoint**: el cliente puede elegir cantidades libres por topping de punta a punta (carrito →
persistencia), con el modo "conteo" protegido por un rechazo explícito ante `quantity>1`.

---

## Phase 4: User Story 3 - El precio y el consumo de inventario del pedido reflejan la cantidad elegida (Priority: P1)

**Goal**: `compute_line_price` y `plan_line_consumption` multiplican por la cantidad elegida de
cada opción — sin necesitar saber el modo del grupo (research.md Decisión 2) — y esa cantidad
llega intacta hasta la deducción real de inventario en los tres caminos de venta (checkout de
pedido, venta de mostrador, chequeo preventivo del carrito).

**Independent Test**: línea con "Bobombún" (recargo $1.000, cantidad 2) y "Gomitas" (recargo $800,
cantidad 1) sobre una presentación de $15.000 → precio de línea $17.800; si el producto maneja
inventario, el insumo de "Bobombún" se descuenta 2 veces su cantidad configurada — verificado
confirmando una orden real y cobrando una venta de mostrador real, no solo llamando
`plan_line_consumption` de forma aislada.

### Implementación de User Story 3

- [X] T030 [US3] En `pos-backend/app/catalog_engine/core.py::compute_line_price` (líneas 30-35):
  generalizar el bucle a `for chosen in options: price += chosen.option.extra_price *
  chosen.quantity` (depende de T011).
- [X] T031 [US3] En `pos-backend/app/catalog_engine/consumption.py::plan_line_consumption`
  (líneas 89-138): generalizar el bucle de opciones a multiplicar también por `chosen.quantity`
  (`per_unit * qty * chosen.quantity`) (depende de T011).
- [X] T032 [US3] En `pos-backend/app/api/v1/orders/checkout.py`: `_item_options` (líneas 59-63)
  cambia de devolver `list[Option]` a `list[ChosenOption]` (join contra `OrderItemOption.quantity`
  de `item.options`); el snapshot de `options` en `order_sale_lines` (líneas 224-232) agrega
  `"quantity": chosen.quantity` a cada dict (depende de T030, T031).
- [X] T033 [US3] **(hallazgo de `/speckit-analyze`)** En
  `pos-backend/app/catalog_engine/consumption.py`: `required_consumption` (líneas 141-150) cambia
  su firma de `options: Sequence[Option]` a `Sequence[ChosenOption]` — sigue sumando
  `line.quantity` de cada `ConsumptionLine` que ya produce `plan_line_consumption`, sin lógica
  propia que tocar más allá del tipo; `ensure_lines_consume_inventory` (líneas 174-239) cambia su
  firma de `Sequence[tuple[UUID, int, Sequence[Option]]]` a
  `Sequence[tuple[UUID, int, Sequence[ChosenOption]]]` — ambas funciones viven en el mismo módulo
  que `plan_line_consumption` (T031) y son sus dos consumidores directos, pero quedaron fuera del
  rango de líneas de T031 (depende de T030, T031).
- [X] T034 [US3] **(hallazgo de `/speckit-analyze`)** En
  `pos-backend/app/api/v1/orders/consumption.py`: las seis funciones
  (`ensure_consumes_inventory`, `lock_consumption`, `deduct_order_items`, `reverse_order_items`,
  `deduct_order_item`, `reverse_order_item`) cambian su tipado de `list[Option]` a
  `list[ChosenOption]` en la anotación `entries`/`options` — es la vía real de deducción de
  inventario al confirmar una orden o agregar un ítem de mesero; sin este cambio, `deduct_order_items`
  (que llama a `ensure_consumes_inventory`/`lock_consumption`/`required_consumption` internamente)
  queda con un tipo desalineado respecto a T032/T033 (depende de T032, T033).
- [X] T035 [US3] **(hallazgo de `/speckit-analyze`)** En
  `pos-backend/app/api/v1/sales/consumption.py`: `_sale_item_options` (líneas 22-31) cambia de
  devolver `list[Option]` a `list[ChosenOption]`, leyendo la clave `"quantity"` de cada dict del
  snapshot JSONB `SaleItem.options` (ya agregada por T021) — `.get("quantity", 1)` para tolerar
  snapshots anteriores a esta spec sin esa clave; `deduct_sale` (que llama a
  `plan_line_consumption`/`ensure_lines_consume_inventory`/`required_consumption` directamente,
  líneas 35-69) ajusta su tipado en consecuencia. Es el camino de deducción de inventario de la
  venta de mostrador, paralelo a T034 para órdenes (depende de T021, T031, T033).
- [X] T036 [US3] **(hallazgo de `/speckit-analyze`)** En `pos-backend/app/api/v1/cart/service.py`:
  las tres llamadas a `required_consumption` (líneas 216, 322, 370 — chequeo preventivo de stock al
  agregar/actualizar una línea de carrito, distintas de las líneas que T016 ya cubre) reciben
  `list[ChosenOption]` en vez de `list[Option]` (depende de T016, T033).
- [X] T037 [P] [US3] Extender
  `pos-backend/app/characterization_tests/test_catalog_option_groups_quantity.py` (T023): casos de
  multiplicación de precio y de consumo con distintas cantidades por opción; caso de un grupo
  "cantidad" marcado además "Incluido" (spec 063/064) — el precio no suma recargo sin importar la
  cantidad; confirmar que `check_availability`/`RN-CAT-24` recibe el total ya multiplicado; **y**
  (hallazgo de `/speckit-analyze`) al menos un caso de punta a punta por cada camino real de venta
  — confirmar una orden completa (`deduct_order_items`) y cobrar una venta de mostrador completa
  (`deduct_sale`) con un grupo "cantidad", no solo invocar `plan_line_consumption` de forma
  aislada — para probar que T032-T036 quedan correctamente encadenados (depende de T030-T036).

**Checkpoint**: el precio y el consumo de una línea con toppings "por cantidad" son correctos de
punta a punta — incluida la deducción real de inventario al confirmar una orden o cobrar en
mostrador, no solo el cálculo aislado — y su interacción con `pricing_type` de spec 063/064.

---

## Phase 5: User Story 4 - El administrador limita cuántas unidades se pueden pedir (Priority: P2)

**Goal**: los dos topes opcionales (`max_quantity_per_option`, `max_total_quantity`) se validan al
confirmar una selección, y el control `+` del carrito se deshabilita al alcanzarlos.

**Independent Test**: grupo con tope de 3 por opción y 5 en total — subir una opción a 4 falla;
sumar 5 entre varias e intentar una más falla, aunque ninguna individual llegue a 3.

### Implementación de User Story 4

- [X] T038 [US4] En `pos-backend/app/catalog_engine/pricing.py::validate_option_selection`: en la
  rama "cantidad" de T014, sumar la cantidad por opción individual y el total del grupo; rechazar
  con `422` si se supera `max_quantity_per_option` o `max_total_quantity` cuando están configurados
  (`None` = sin tope) (depende de T014).
- [X] T039 [P] [US4] Extender
  `pos-backend/app/characterization_tests/test_catalog_option_groups_quantity.py`: casos de
  violación de cada tope por separado, y de un grupo sin ningún tope configurado (sin límite
  propio) (depende de T038).
- [X] T040 [US4] En `pos-heladeria/.../tables/components/product-select.component.ts`: el botón
  `+` de una opción se deshabilita al alcanzar `max_quantity_per_option`, y todos los `+` del grupo
  se deshabilitan al alcanzar `max_total_quantity` (depende de T028).
- [X] T041 [P] [US4] Extender `product-select.component.spec.ts` (T029): casos de tope alcanzado
  (depende de T040).

**Checkpoint**: los topes configurados en US1 tienen efecto real, tanto en backend como en la UI
del carrito.

---

## Phase 6: User Story 5 - La comanda, el recibo y "Mis pedidos" muestran cuántas unidades de cada topping se pidieron (Priority: P2)

**Goal**: las seis superficies que hoy listan opciones elegidas por nombre muestran "2x Nombre"
cuando la cantidad es mayor a 1, usando un único formateador compartido.

**Independent Test**: confirmar un pedido con cantidades distintas de dos toppings; verificar que
comanda, detalle de orden, "Mis pedidos", panel de validación de pago, recibo y carrito en vivo
muestran la misma cantidad por topping.

### Implementación de User Story 5

- [X] T042 [US5] En `pos-heladeria/src/app/modules/tables/services/menu-lookup.ts`: agregar
  `optionLabelWithQuantity(optionId: string, quantity: number): string` a `MenuLookup` — devuelve
  `"2x Nombre"` si `quantity > 1`, o `optionLabel(optionId)` sin cambios si `quantity === 1`
  (research.md Decisión 9).
- [X] T043 [US5] En
  `pos-heladeria/src/app/modules/tables/components/pos-terminal.store.ts` (línea ~794): cambiar
  `.map(o => lk.optionLabel(o.option_id))` por
  `.map(o => lk.optionLabelWithQuantity(o.option_id, o.quantity ?? 1))` (depende de T042).
- [X] T044 [P] [US5] Mismo cambio en
  `pos-heladeria/src/app/modules/tables/components/order-detail.component.ts` (línea ~144)
  (depende de T042).
- [X] T045 [P] [US5] Mismo cambio en
  `pos-heladeria/src/app/modules/products/pages/public-menu.component.ts` (línea ~1095, "Mis
  pedidos") (depende de T042).
- [X] T046 [P] [US5] Mismo cambio en
  `pos-heladeria/src/app/modules/tables/components/payment-validation-block.component.ts`
  (línea ~131) (depende de T042).
- [X] T047 [P] [US5] Mismo cambio en
  `pos-heladeria/src/app/shared/receipt/receipt.util.ts` (líneas ~88 y ~216-218) (depende de T042).
- [X] T048 [P] [US5] Mismo cambio en
  `pos-heladeria/src/app/modules/tables/components/cart.component.ts` (líneas ~30-32) y su fuente
  `dining-cart.service.ts` (líneas ~146-148) — el carrito en vivo del comensal (depende de T042).
- [X] T049 [P] [US5] Crear/extender el spec de `menu-lookup.ts` con casos para
  `optionLabelWithQuantity` (`quantity=1` → sin prefijo; `quantity>1` → "2x Nombre"; `quantity`
  ausente → se trata como 1) (depende de T042).

**Checkpoint**: las seis superficies muestran la cantidad de forma idéntica entre sí, sin lógica de
formato duplicada.

---

## Phase 7: User Story 6 - El catálogo existente no cambia de comportamiento (Priority: P2)

**Goal**: verificar que la migración de T003 no altera ningún precio, consumo ni pedido/venta ya
confirmados, que todo grupo existente queda en modo "conteo", y que cambiar el modo de un grupo
nunca afecta selecciones ya confirmadas (FR-013).

**Independent Test**: tras `alembic upgrade head`, todo grupo existente tiene `selection_mode=
'conteo'`; vender una presentación que use un grupo migrado produce el mismo precio y el mismo
consumo que antes de la migración; cambiar el modo de un grupo con pedidos ya confirmados no altera
esos pedidos.

### Implementación de User Story 6

- [X] T050 [US6] Ejecutar `alembic upgrade head` sobre el entorno de desarrollo y confirmar, por
  consulta directa, que todo `option_groups` existente quedó con `selection_mode='conteo'`,
  `max_quantity_per_option=NULL`, `max_total_quantity=NULL`, y que todo
  `cart_item_options`/`order_item_options` existente quedó con `quantity=1` (depende de T003).
- [X] T051 [US6] Sobre un tenant de prueba con un grupo "conteo" ya vendido antes de la migración,
  repetir la misma venta después de aplicar T030/T031/T033-T035 y confirmar que el precio de línea
  y el consumo de inventario resultantes son idénticos a los de antes (SC-006; depende de T050,
  T030, T031, T033, T034, T035).
- [X] T052 [US6] **(hallazgo de `/speckit-analyze`, FR-013)** Sobre un grupo ya usado por un pedido
  confirmado, cambiar su `selection_mode` (o sus topes) vía `PATCH /option-groups/{id}` y confirmar
  que el precio, las opciones y el consumo de inventario ya registrados en ese pedido no cambian —
  el cambio de modo solo aplica a selecciones nuevas (spec.md FR-013, Principio VII; depende de
  T005, T050).

**Checkpoint**: la migración y la generalización de precio/consumo quedan verificadas contra datos
reales, sin regresiones, y el cambio de modo de un grupo queda confirmado como no retroactivo.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de que ninguna historia rompió a otra.

- [X] T053 [P] Ejecutar
  `cd pos-backend && python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`
  — confirmar 100% verde, incluyendo los tests nuevos/actualizados de T006, T023, T024, T037,
  T039.
- [X] T054 [P] Ejecutar `cd pos-heladeria && npx ng test --watch=false` — confirmar 100% verde,
  incluyendo los specs nuevos/actualizados de T010, T029, T041, T049.
- [ ] T055 Ejecutar la sección "Verificación manual end-to-end" de
  [quickstart.md](./quickstart.md) (crear grupo "cantidad" con topes, elegir cantidades en el menú
  QR, confirmar comanda/"Mis pedidos"/recibo, revisar el detalle de la venta) y registrar el
  resultado.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede iniciar de inmediato.
- **US1 (Phase 2)**: depende de Setup. Incluye la migración completa (T003, las 5 columnas de las
  3 tablas) porque es una sola operación atómica (research.md Decisión 5) — US2/US3/US6 la dan por
  hecha, no la repiten.
- **US2 (Phase 3)**: depende de US1 (T003, la columna `quantity` de `cart_item_options`/
  `order_item_options` debe existir). Introduce `ChosenOption` y el cambio de contrato
  `option_ids→options`, del que US3, US4 y parcialmente US5 dependen.
- **US3 (Phase 4)**: depende de US2 (T011, `ChosenOption`; T013, `load_valid_options` ya produce
  `ChosenOption`). Internamente, T033 (consumption.py) depende de T031; T034
  (orders/consumption.py) depende de T032 y T033; T035 (sales/consumption.py) depende de T021 y
  T033; T036 (cart/service.py) depende de T016 y T033 — ver también T024 (US2), que depende de
  T033-T035 para poder localizar y corregir todos los tests afectados.
- **US4 (Phase 5)**: depende de US1 (T004, los topes existen en el schema) y de US2 (T014, la rama
  "cantidad" de `validate_option_selection` ya existe para extenderla) y de T028 (el stepper del
  carrito).
- **US5 (Phase 6)**: depende de US2 (T025-T029, la cantidad ya viaja hasta el frontend) para tener
  algo que mostrar — no depende de US3 ni US4.
- **US6 (Phase 7)**: depende de T003 (US1) y de T030, T031, T033, T034, T035 (US3) — es puramente
  verificación.
- **Polish (Phase 8)**: depende de que todas las historias que se vayan a entregar estén completas.

### Parallel Opportunities

- T002 y T003 (US1) son paralelos entre sí (ficheros distintos).
- T007 (US1, frontend interfaces) es paralelo a todo el trabajo backend de US1.
- T011 y T012 (US2) son paralelos entre sí — ambos son la base de T013.
- T025 y T027 (US2, frontend interfaces) son paralelos entre sí y al trabajo backend de US2.
- T044-T048 (US5) son paralelos entre sí una vez completado T042-T043 (el formateador y su primer
  uso) — son el mismo cambio mecánico repetido en cinco ficheros distintos.
- T053 y T054 (Polish) son paralelos entre sí.

---

## Parallel Example: User Story 2

```bash
# Backend: los dos tipos base, en paralelo
Task: "Agregar ChosenOption en pos-backend/app/catalog_engine/core.py"
Task: "Agregar OptionSelectionIn en pos-backend/app/api/v1/catalog/schemas.py"

# Frontend: interfaces en paralelo con el trabajo backend
Task: "Actualizar CartItemPayload/CartItemOption en diner.interface.ts"
Task: "Agregar selection_mode a MenuOptionGroup en product.interface.ts"
```

---

## Implementation Strategy

### MVP (las tres historias P1 juntas)

Igual que spec 063/064: las tres historias P1 (US1, US2, US3) componen el problema central
reportado por el usuario — sin las tres, "elegir cantidad" no llega a ser una funcionalidad
completa (declarar el modo sin poder elegir cantidad, o elegir cantidad sin que el precio y el
consumo real de inventario la reflejen de punta a punta, son ambas a medias). Orden recomendado:

1. Completar Fase 1: Setup.
2. Completar Fase 2: US1 (modo de selección + migración completa).
3. Completar Fase 3: US2 (cantidad libre por opción, de punta a punta).
4. Completar Fase 4: US3 (precio y consumo correctos, incluida la cadena real de deducción de
   inventario — T033-T036).
5. **DETENER y VALIDAR**: correr `quickstart.md` secciones US1-US3 completas.

### Entrega incremental

1. Setup + US1 + US2 + US3 → MVP funcional, demostrable.
2. Agregar US4 (topes) → protección contra pedidos desproporcionados.
3. Agregar US5 (visualización) → cierra la calidad operativa (cocina, recibo, "Mis pedidos").
4. Agregar US6 (verificación de migración y de no-retroactividad) → cierra la trazabilidad de que
   nada se rompió.
5. Polish → confirmación final de las dos suites de test completas y del recorrido manual.

### Estrategia de equipo en paralelo

Con varios desarrolladores, tras completar Setup + US1:

- Desarrollador A: núcleo de US2 (T011-T024, backend) → US3 (T030-T037, comparten `ChosenOption`
  y la cadena de consumo).
- Desarrollador B: frontend de US2 (T025-T029) en cuanto T013/T015/T017 definan el contrato →
  US5 (T042-T049, una vez que la cantidad ya viaja hasta el frontend).
- Desarrollador C: US4 backend (T038-T039) en cuanto T014 (US2) exista.
- US6 la ejecuta quien terminó US1 y US3 (mismo contexto de migración y de las fórmulas).

---

## Notes

- [P] = ficheros distintos, sin dependencia de una tarea sin terminar.
- `ChosenOption` (US2) es el único punto de integración real entre historias — todas las
  dependencias cruzadas están documentadas explícitamente arriba, no son acoplamientos implícitos.
- T033-T036 (US3) no estaban en la primera versión de este documento — `/speckit-analyze` encontró
  que generalizar solo `plan_line_consumption` (T031) dejaba desalineada toda la cadena real de
  deducción de inventario (`required_consumption`, `ensure_lines_consume_inventory`,
  `orders/consumption.py`, `sales/consumption.py`, y tres llamadas más en `cart/service.py`). Sin
  ellas, confirmar una orden o cobrar una venta de mostrador con un grupo "cantidad" habría fallado
  en tiempo de ejecución.
- T024 (actualizar characterization tests existentes que usan `option_ids` o llaman a las
  funciones del motor con listas crudas de `Option`) es de cumplimiento obligatorio antes de dar
  por completa US2/US3 — Principio III exige que ese ajuste sea deliberado, no un arreglo
  silencioso.
- T052 (US6, FR-013) tampoco estaba en la primera versión — `/speckit-analyze` encontró que ningún
  task verificaba explícitamente que cambiar el modo de un grupo no afecta pedidos ya confirmados.
- Verificar que los tests fallan antes de implementar donde corresponda (T006, T023, T037, T039,
  T049 antes de sus tareas de implementación asociadas, cuando el equipo siga TDD).
- Hacer commit tras cada tarea o grupo lógico; detenerse en cada Checkpoint para validar la
  historia de forma independiente antes de continuar.
