---

description: "Tareas de la spec 087 — correcciones de Caja, Terminal de Mesas, Menú QR y Ventas (US1–US11)"
---

# Tasks: Correcciones de Caja, Terminal de Mesas, Menú QR y Ventas

**Input**: Documentos de diseño de `/specs/087-fix-caja-mesas-menu-ventas/`
**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md) (D1–D14 + decisión previa confirmada), [data-model.md](./data-model.md), [contracts/api-changes.md](./contracts/api-changes.md), [quickstart.md](./quickstart.md)

> **Actualización 2026-09-29 (adenda US8–US11)**: las Fases 1–10 (T001–T062, US1–US7) ya estaban
> implementadas y **se conservan intactas**, con sus marcas `[X]`. La adenda de la spec (US8–US11,
> FR-014 a FR-017, D11–D14) se agrega como Fases 11–15 (T063–T092) al final, sin renumerar nada
> anterior para no romper las referencias cruzadas de `registro-de-anomalias.md` ni del historial.

**Tests**: Este proyecto usa *characterization tests* (`app/characterization_tests/`, `python -m unittest`) en `pos-backend` como árbitro de comportamiento (Principio III), no TDD clásico — se generan tareas de test junto a cada endpoint/lógica nueva o corregida, no antes, salvo cuando la propia investigación de esta spec ya identificó un test protegido que congeló el comportamiento incorrecto (US1), donde sí se actualiza primero el test para que falle contra el código actual y luego se aplica el fix. `pos-heladeria` usa `ng test` sobre los `*.spec.ts` ya existentes.

**Organización**: Las tareas se agrupan por historia de usuario (US1 → US7, mismo orden de prioridad que `spec.md`: P1×2, P2×3, P3×2) sobre una única rama de implementación `fix/087-cash-tables-menu-sales-fixes` en `../pos-backend` y `../pos-heladeria` (plan.md, Principio XIV). Todas las rutas de archivo citadas fueron verificadas contra el código real de ambos repos (no solo contra `research.md`) antes de escribir este documento — varios números de línea y una causa raíz difieren de lo que `research.md` conjeturaba; donde eso ocurre se indica explícitamente.

## Formato: `[ID] [P?] [Story] Descripción`

- **[P]**: Puede ejecutarse en paralelo (archivo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: A qué historia de usuario pertenece (US1–US7)
- Cada tarea incluye la ruta exacta del archivo, relativa a `../pos-backend`, `../pos-heladeria` o `pos-specs` (esta carpeta) según corresponda

## Convenciones de ruta

- `../pos-backend/...` — API FastAPI + SQLAlchemy + Alembic (Python 3.12)
- `../pos-heladeria/...` — SPA Angular 21 standalone (TypeScript)
- Sin prefijo — archivo dentro de este repo (`pos-specs`), p. ej. `specs/000-reconocimiento/registro-de-anomalias.md`

## Nota sobre gobernanza (Principio II/XI) y numeración de anomalías

Cuatro historias de esta spec cambian comportamiento existente y requieren una entrada nueva en
`specs/000-reconocimiento/registro-de-anomalias.md` (formato `A-NN — [DECISIÓN DE NEGOCIO — spec
087] ...`) antes de darse por completadas (plan.md, Constitution Check, Principio II). Al momento
de escribir este documento el último número usado es **A-83**, así que las tareas de gobernanza
de abajo asignan **A-84 a A-88** en el orden en que aparecen las historias — **confirmar con
`grep -oE "^### A-[0-9]+" specs/000-reconocimiento/registro-de-anomalias.md | sort -t- -k2 -n |
tail -1`** en el momento real de implementar, por si otra spec en curso ya tomó alguno de estos
números entre tanto.

---

## Phase 1: Setup

**Propósito**: preparar la rama de trabajo en ambos repositorios de código.

- [X] T001 Crear la rama `fix/087-cash-tables-menu-sales-fixes` desde `develop` en `../pos-backend` y en `../pos-heladeria` (ambos repos confirmados en `develop`, árbol de trabajo limpio; plan.md, Principio XIV)

**Checkpoint**: ambos repos listos sobre la rama de esta funcionalidad.

---

## Phase 2: Foundational (Blocking Prerequisites)

**No aplica a esta spec.** A diferencia de specs anteriores (p. ej. 083), las 7 historias de
usuario no comparten ninguna tabla, endpoint ni componente cuya construcción deba resolverse
antes que todas las demás — cada una es autocontenida en su propia capa (ver plan.md, "Structure
Decision": *"unidades por historia de usuario... para poder revisar y revertir cada una de forma
independiente"*). Las únicas dos migraciones de base de datos de esta spec (T017 en US3, T029 en
US4) sirven cada una a una sola historia y viven en la fase de esa historia, no aquí. Se pasa
directo a la Fase 3.

---

## Phase 3: User Story 1 - Cobro correcto de adicionales sobre promociones (Priority: P1) 🎯 MVP

**Goal**: el total mostrado en el armado del pedido, el checkout, "Pagos por confirmar" y el
detalle de venta refleja siempre Precio Promocional + Σ Precio Adicionales.

**Hallazgo de esta investigación (corrige research.md D6)**: el bug NO está en el frontend
(`product-select.component.ts::lineTotal()`, líneas 557-569, ya suma `base + extra`
correctamente) ni en `compute_line_price` (`catalog_engine/core.py:39-46`, ya suma con `+=`).
Vive exclusivamente en el **backend**, en `evaluate_variant_sets()`
(`app/api/v1/promotions/service.py`), rama `type == "package_price"` (líneas 269-272): resta el
descuento de la promoción sobre `unit_price` **completo** (incluye adicionales) en vez de sobre
`base_unit_price` (sin adicionales) — al contrario que la rama `percent`, que sí usa
`base_unit_price` desde spec 083/FR-027 (líneas 274-278). Con el ejemplo de la spec (Granizado de
Mora, variante $6.000, promo `package_price` value=$5.000 min_qty=1, + Leche condensada $2.000 +
Choco-chips $1.500 → `unit_price=$9.500`, `base_unit_price=$6.000`): hoy descuenta
`9500-5000=4500`, dando un total de `9500-4500=$5.000` en vez de `$8.500` — los $3.500 de
adicionales quedan absorbidos por el "descuento". Como `evaluate_variant_sets` es el único motor
reusado por `cart/service.py` (Menú QR), `orders/checkout.py` (`compute_checkout_preview`,
`compute_draft_preview`, `pay_order`, `checkout_and_send`, `confirm_cash_payment_attempt`,
`approve_payment_attempt` — este último respalda "Pagos por confirmar") y `sales/builder.py`, un
único fix en `service.py:269-272` corrige las 4 superficies a la vez. **No se necesita ningún
cambio de frontend para esta historia.**

**Independent Test**: armar un pedido con un producto en promoción y dos adicionales, verificar
el mismo total correcto en el resumen del pedido, en el checkout, en "Pagos por confirmar" y en
el detalle de venta (quickstart.md, Historia 1).

### Implementation for User Story 1

- [X] T002 [US1] **Gate de gobernanza (Principio VII, GATE CRÍTICO — plan.md Constitution Check y research.md D6)**: confirmar con el usuario/negocio, antes de tocar código, que el fix de FR-011 aplica **solo a checkouts nuevos a partir del despliegue** — ninguna `Sale.total`/`Sale.change_given` de una venta ya finalizada se recalcula retroactivamente. Registrar la decisión en `specs/000-reconocimiento/registro-de-anomalias.md` como **A-84** (o el siguiente número real disponible, ver nota de gobernanza arriba), formato `A-NN — [DECISIÓN DE NEGOCIO — spec 087]`, citando research.md D6 y este alcance. Si el usuario confirma una lectura distinta (p. ej. recalcular ventas históricas), esa es una decisión de negocio separada, fuera del alcance de T003-T005 tal como están escritas — ajustar antes de continuar.
- [X] T003 [US1] Backend: en `../pos-backend/app/characterization_tests/test_promotions_service.py`, reemplazar `test_17_package_price_no_excluye_toppings_sin_cambio` (líneas 305-316) por `test_17_package_price_excluye_toppings_de_la_base_min_qty_1` — nuevo docstring citando spec 087 FR-011 (D6 de research.md) como autorización del cambio de comportamiento (Principio III; el test anterior documentaba explícitamente el bug ahora corregido). Nuevo cuerpo, con el ejemplo exacto de la spec:
  ```python
  def test_17_package_price_excluye_toppings_de_la_base_min_qty_1(self):
      """spec 087, FR-011: el fix corrige `package_price` para que también
      descuente sobre `base_unit_price` (sin adicionales), igual que `percent`
      desde FR-027 (spec 083) -- reemplaza el comportamiento congelado en la
      versión anterior de este test, que documentaba el bug ahora corregido
      (A-84, registro-de-anomalias.md)."""
      v = self._variant(6000, "Granizado de Mora")
      self._promo("package_price", 5000, 1, [v])
      linea = _line(v, 1, base_unit_price=Decimal("6000"))
      linea["unit_price"] = Decimal("9500")  # 6000 base + 2000 leche cond. + 1500 choco-chips
      r = promotions.evaluate_variant_sets(self.db, [linea], NOW)
      # Descuento = base(6000) - promo(5000) = 1000 -- nunca sobre 9500 (total final 8500).
      self.assertEqual(r.total, Decimal("1000.00"))
  ```
  Este test debe **FALLAR** contra el código actual (hoy da `r.total == Decimal("4000.00")`... en realidad con estos valores nuevos daría `9500-5000=4500`, confirmar el valor real al ejecutar) antes de aplicar T004, y pasar después.
- [X] T004 [US1] Backend: aplicar el fix en `../pos-backend/app/api/v1/promotions/service.py`, dentro de `evaluate_variant_sets`, rama `if r.type == "package_price":` (líneas 269-272) — cambiar de:
  ```python
  normal_g = sum((u[1] for u in block), Decimal(0))
  descuento_g = max(Decimal(0), normal_g - Decimal(r.value))
  dist_block = block
  ```
  a usar `base_price_by_line` (ya calculado más arriba en la misma función para todas las líneas, línea ~260), mismo patrón que la rama `percent` (líneas 274-278):
  ```python
  base_g = sum((base_price_by_line[u[0]] for u in block), Decimal(0))
  descuento_g = max(Decimal(0), base_g - Decimal(r.value))
  dist_block = [
      (idx, base_price_by_line[idx], pv_id, line_id)
      for idx, _price, pv_id, line_id in block
  ]
  ```
  Actualizar el docstring de la función (líneas ~236-241), que hoy afirma *"`package_price` no cambia: sigue descontando sobre `unit_price` completo"* — ya no es cierto, citar spec 087 FR-011 y A-84. Confirmar que T003 pasa.
- [X] T005 [P] [US1] Backend: ejecutar `python -m unittest app.characterization_tests.test_promotions_service app.characterization_tests.test_cart_service app.characterization_tests.test_table_sessions_service app.characterization_tests.test_orders_checkout app.characterization_tests.test_promotions_toppings_base_price -v` desde `../pos-backend` y confirmar que ningún otro test queda en rojo — estos módulos referencian `package_price` pero sin combinarlo con adicionales (`base_unit_price == unit_price` en sus escenarios), así que no deberían cambiar de resultado; si alguno lo hace, investigar antes de continuar.

**Checkpoint**: US1 completamente funcional — el total correcto se ve igual en las 4 superficies de cobro sin tocar ningún archivo de frontend.

---

## Phase 4: User Story 2 - Consolidación de pedidos abiertos sin duplicados (Priority: P1)

**Goal**: "Agregar producto" sobre un pedido abierto (mesa/para llevar/domicilio) anexa al mismo
`order_id`, nunca crea un pedido duplicado; un pedido pagado/cerrado rechaza la operación.

**Hallazgo de esta investigación (precisa research.md D4)**: el único consumidor de
`POST /orders/tables/{table_id}/items` en todo el frontend es
`pos-terminal.store.ts::saveOrder()` (línea 1730) — el Menú QR del cliente nunca lo usa (agrega
al carrito vía `POST /cart/items`, no a una orden ya enviada). Es seguro retirarlo por completo,
igual que se hizo con Arqueo Parcial en spec 087 US4. La función `get_or_create_open_order`
(`consolidation.py:69-106`) **no se retira** — la sigue usando `consolidate_table` (línea 130),
una función distinta (consolidación de carritos QR) fuera del alcance de esta spec.

**Independent Test**: crear un pedido de mesa, usar "Agregar producto" sobre ese pedido (sin
pagar) y verificar que el total aumenta manteniendo el mismo `order_id`, sin un segundo pedido
nuevo; repetir con para llevar/domicilio; confirmar que un pedido pagado rechaza la operación
(quickstart.md, Historia 2).

### Implementation for User Story 2

- [X] T006 [US2] **Gobernanza (Principio II, research.md D4)**: registrar **A-85** en `specs/000-reconocimiento/registro-de-anomalias.md` documentando el retiro de `POST /orders/tables/{table_id}/items` en favor de `POST /orders/{order_id}/items` — quién/cuándo/qué cambia (el contrato de "Agregar producto" pasa de resolverse por mesa a operar siempre sobre un pedido específico)/por qué (FR-007 permite pedidos paralelos por mesa; resolver "la orden abierta de la mesa" de forma implícita ya no es válido)/funcionalidades afectadas (Terminal de Mesas, `pos-terminal.store.ts::saveOrder()`). **Debe mergearse antes que T008.**
- [X] T007 [US2] Backend: crear `add_item_to_order(db: Session, order_id: UUID, data, user: User) -> CustomerOrder` en `../pos-backend/app/api/v1/orders/consolidation.py` (junto a `add_item_to_table`, líneas 189-238 — mismo cuerpo salvo la resolución del pedido): resolver con `get_or_404(db, CustomerOrder, order_id, "Order not found")`; si `order.status in ("pagada", "cancelada")` → `raise HTTPException(status.HTTP_409_CONFLICT, "Este pedido ya está pagado o cancelado; crea un pedido nuevo.")` (FR-009); si no, insertar `OrderItem`/`OrderItemOption` igual que `add_item_to_table` (mismo `load_valid_options`, `compute_line_price`, `deduct_order_items`), commit y devolver `_reload_order(db, order.id)`.
- [X] T008 [US2] Backend: retirar `add_item_to_table` completo (`consolidation.py:189-238`) — no tocar `get_or_create_open_order` (líneas 69-106), sigue usándola `consolidate_table` (línea 130, fuera de alcance).
- [X] T009 [US2] Backend: en `../pos-backend/app/api/v1/orders/router.py` — quitar el endpoint `add_table_item` (líneas 302-315, `POST /tables/{table_id}/items`) y el import de `add_item_to_table` (línea 24); agregar `POST /{order_id}/items` (`response_model=OrderResponse`, `status_code=201`) que llama `add_item_to_order(db, order_id, body, user)` (contracts/api-changes.md §4), ubicado junto a los demás endpoints `/{order_id}/...` del router.
- [X] T010 [US2] Backend characterization tests: en `../pos-backend/app/characterization_tests/test_orders_consolidation.py` (archivo protegido, prefijo `"CONGELA comportamiento actual:"`), migrar los escenarios vigentes que llaman `add_item_to_table(db, table.id, ...)` (líneas 85-347: validación de opciones tras A-04, selección completa, exceso de máximo del grupo, apertura de sesión/orden sobre la marcha, rechazo sin receta) para que llamen `add_item_to_order(db, order.id, ...)` sobre un pedido ya creado, citando spec 087 D4/A-85 como autorización del cambio de contrato (Principio III); agregar un test nuevo para el 409 de FR-009 (pedido `pagada`/`cancelada` rechaza el anexo) y uno para 404 con `order_id` inexistente. No tocar los tests de `get_or_create_open_order` (líneas 230-253, 356-383) — siguen validando `consolidate_table`, sin cambios.
- [X] T011 [P] [US2] Backend: ejecutar `python -m unittest app.characterization_tests.test_orders_consolidation app.characterization_tests.test_orders_service app.characterization_tests.golden_master_core -v` desde `../pos-backend` y confirmar verde (estos tres módulos referencian `add_item_to_table` en comentarios/imports).
- [X] T012 [US2] Frontend: en `../pos-heladeria/src/app/modules/tables/services/dining-session.service.ts`, reemplazar `addTableItem(tableId, item)` (líneas 222-225, `POST orders/tables/${tableId}/items`) por `addOrderItems(orderId: string, item: OrderItemPayload): Promise<DiningOrder>` que pega a `POST orders/${orderId}/items`.
- [X] T013 [US2] Frontend: en `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts::saveOrder()` (líneas 1719-1751), cambiar de resolver por `this.selectedTableId()` a exigir `this.selectedOrderId()` (ya existente, línea 372, `signal<string|null>(null)`); si no hay orden seleccionada, no llamar al API y mostrar un error/toast guiando a usar "Crear pedido manual" (FR-008/FR-009: "Agregar producto" siempre opera sobre el pedido específico ya identificado); reemplazar las llamadas a `this.api.addTableItem(tableId, ...)` por `this.api.addOrderItems(this.selectedOrderId()!, ...)`; actualizar el docstring de la línea 1721 (*"Persiste el draft en la orden de la mesa (crea la orden si no existe)"*) — ya no crea la orden, solo anexa.
- [X] T014 [US2] Frontend: `hasActiveSelection()` (`pos-terminal.store.ts:680`, `!!selectedTableId() || !!selectedOrderId()`) permite hoy que el panel central (`pos-order-panel.component.ts`) muestre el catálogo/"Guardar pedido" con una mesa libre seleccionada pero sin ningún pedido (`selectedOrderId()` nulo) — tras T013, ese flujo dejaría de funcionar en silencio. Ajustar la UI de ese panel para que, cuando la mesa esté libre (sin pedido), guíe explícitamente a usar "Crear pedido manual" en vez de ofrecer un catálogo que ya no puede guardar nada.
- [X] T015 [P] [US2] Frontend: actualizar `pos-terminal.store.spec.ts`, `dining-session.service.spec.ts` y `pos-order-panel.component.spec.ts` (los tres existen) donde referencien `addTableItem`/el comportamiento anterior de `saveOrder()`; ejecutar `ng test` en `../pos-heladeria` y confirmar verde.

**Checkpoint**: US1 + US2 funcionales de forma independiente — el total es correcto y "Agregar producto" nunca duplica pedidos.

---

## Phase 5: User Story 3 - Numeración estable y cronológica de pedidos de mesa (Priority: P2)

**Goal**: cada pedido de mesa recibe un número secuencial ascendente asignado una sola vez al
crearse, estable para siempre dentro del turno de caja en que nació.

**Hallazgo de esta investigación (precisa research.md D3)**: hay **dos** puntos de creación de
`CustomerOrder` con `order_type="DINE_IN"` que necesitan la numeración, no solo uno:
`orders/service.py::create_order` (mesero/staff, línea 237) **y**
`cart/service.py::submit_cart` (comensal vía Menú QR, línea 615) — ambos construyen su propio
`CustomerOrder(...)` de forma independiente (el segundo ya cita el primero como precedente en un
comentario de spec 073: *"segundo punto de creación de CustomerOrder"*). La causa raíz confirmada
del bug de reordenamiento es `orderTabs()` (`pos-terminal.store.ts:712-723`): rotula
`Pedido ${i + 1}` por **posición en el arreglo** que devuelve `tableOrders()`
(línea 579-583), el cual no reordena y hereda el orden de `GET /orders` — **descendente por
`created_at`** (`list_orders`, `orders/service.py:115-148`, `.order_by(created_at.desc())`) — por
eso el pedido más nuevo aparece como "Pedido 1".

**Independent Test**: crear dos pedidos de mesa consecutivos, verificar que sus números no
cambian al crear un tercero; cerrar el turno, abrir uno nuevo y confirmar que el contador
reinicia en 1 (quickstart.md, Historia 3).

### Implementation for User Story 3

- [X] T016 [US3] **Gobernanza (Principio II, research.md D3)**: registrar **A-86** en `specs/000-reconocimiento/registro-de-anomalias.md` — la numeración de pedidos de mesa pasa de recalcularse por posición de arreglo en el frontend a persistirse una sola vez en el backend, reiniciada en cada apertura de turno de caja (decisión ya confirmada con el usuario en research.md: "un solo turno abierto a la vez", el pedido toma el turno abierto más reciente del tenant).
- [X] T017 [P] [US3] Backend: migración Alembic aditiva en `../pos-backend/alembic/versions/` — generar con `alembic revision -m "087 numeracion pedidos mesa"` (confirmar el head real con `alembic heads`; `c7e2b91a4d35` al momento de escribir este documento). `@for_each_tenant_schema` (mismo patrón que `f3a9c1b7e2d4_congela_instante_vigencia_promociones.py`, 58 líneas, ya citado en research.md/data-model.md como plantilla): agregar a `customer_orders` `cash_shift_id UUID NULL` (FK a `cash_shifts.id`, `ondelete="SET NULL"`, indexada) y `table_order_number INTEGER NULL`, más el índice único parcial `uq_customer_orders_shift_table_order_number` sobre `(cash_shift_id, table_order_number)` `WHERE table_order_number IS NOT NULL` — código completo ya provisto en `data-model.md` §1 (copiar tal cual, con el guard `_has_table(schema, "customer_orders")`). `downgrade()` simétrico, también ya provisto.
- [X] T018 [US3] Backend: agregar `cash_shift_id: Mapped[UUID | None]` y `table_order_number: Mapped[int | None]` a `../pos-backend/app/models/customer_order.py` (junto a los demás campos; reflejar el índice único de T017 en `__table_args__` si el patrón del archivo lo exige).
- [X] T019 [US3] Backend: crear `resolve_dine_in_table_order(db: Session) -> tuple[UUID | None, int | None]` en `../pos-backend/app/api/v1/cash/service.py` (junto a `get_open_shift`, línea 22): `select(CashShift).where(CashShift.status == "open").order_by(CashShift.opened_at.desc()).limit(1).with_for_update()` (el `with_for_update()` bloquea la fila del turno para serializar creaciones concurrentes de pedidos del mismo turno, data-model.md §1); si no hay ninguno, devolver `(None, None)`; si hay uno, contar `SELECT COUNT(*) FROM customer_orders WHERE cash_shift_id = :id` y devolver `(shift.id, count + 1)`.
- [X] T020 [US3] Backend: usar `resolve_dine_in_table_order` en `../pos-backend/app/api/v1/orders/service.py::create_order` (línea 237) — cuando `data.order_type is OrderType.DINE_IN`, resolver `cash_shift_id`/`table_order_number` antes de construir el `CustomerOrder` (línea ~318) y pasarlos al constructor.
- [X] T021 [US3] Backend: el mismo cambio en `../pos-backend/app/api/v1/cart/service.py::submit_cart` (línea 615, `order_type="DINE_IN"` siempre en este flujo) — llamar el helper de T019 y pasar `cash_shift_id`/`table_order_number` al constructor de `CustomerOrder`, con un comentario que cite el mismo criterio de "segundo punto de creación" que ya usa spec 073 ahí (líneas 623-626).
- [X] T022 [US3] Backend: exponer `table_order_number: int | None = None` en `OrderResponse` (`../pos-backend/app/api/v1/orders/schemas.py:197-234`, contracts/api-changes.md §3).
- [X] T023 [P] [US3] Backend: crear `../pos-backend/app/characterization_tests/test_orders_table_numbering.py` — con un turno abierto: crear 3 pedidos DINE_IN consecutivos vía `create_order` y verificar `table_order_number` 1, 2, 3 estables tras crear un cuarto; un pedido TAKEAWAY/DELIVERY no recibe número (`None`); cerrar el turno y abrir uno nuevo reinicia el contador en 1; el mismo comportamiento vía `submit_cart` (QR).
- [X] T024 [US3] Frontend: agregar `table_order_number?: number | null;` a `DiningOrder` en `../pos-heladeria/src/app/modules/tables/interfaces/dining.interface.ts` (línea ~205-230).
- [X] T025 [US3] Frontend: en `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts::orderTabs()` (líneas 712-723), dejar de rotular `Pedido ${i + 1}` por posición y usar `o.table_order_number` (con fallback sin número para pedidos históricos, `Pedido` a secas o similar); ordenar `list` ascendente por `table_order_number` (o por `created_at` si es `null`) antes de mapear, en vez de heredar el orden descendente que hoy entrega `tableOrders()`.
- [X] T026 [P] [US3] Frontend: `ng test` en `../pos-heladeria`, confirmar verde `pos-terminal.store.spec.ts`.

**Checkpoint**: US1 + US2 + US3 funcionales de forma independiente.

---

## Phase 6: User Story 4 - Cierre de caja limpio, sin Arqueo Parcial (Priority: P2)

**Goal**: la interfaz de caja ya no ofrece Arqueo Parcial (ni su historial); el PDF de cierre de
turno muestra solo el resumen financiero, con nombre de archivo `<tenant-slug>-<DD-MM-YYYY>.pdf`.

**Hallazgo de esta investigación (precisa research.md D1)**: el patrón `@media print` de
`table-qr-sheet.component.ts:99-110` **no basta por sí solo** para ocultar el sidebar/header —
ese componente solo aplica `print:hidden` a sus propios controles (líneas 30, 70), nunca a
`<app-sidebar />`/`<app-header />` del shell (`dashboard-layout.component.ts`), que no tiene hoy
ningún tratamiento de impresión. Replicar el patrón tal cual en `cash-report.component.ts` no
cumpliría FR-002 — hace falta además ocultar el shell explícitamente (T036).

**Independent Test**: confirmar que no hay ningún botón/acceso de Arqueo Parcial; cerrar un turno
y descargar el PDF de cierre, revisar nombre y contenido (quickstart.md, Historia 4).

### Implementation for User Story 4

- [X] T027 [US4] **Gobernanza (Principio II, research.md D2)**: registrar **A-87** en `specs/000-reconocimiento/registro-de-anomalias.md` — eliminación completa de Arqueo Parcial (UI + endpoint + tabla + historial, con borrado físico permanente). **Debe mergearse antes que T029** (la migración destructiva).
- [X] T028 [P] [US4] Backend: retirar el endpoint `partial_count` (`../pos-backend/app/api/v1/cash/router.py:140-158`, más el import de `CashPartialCount` si lo hay en ese archivo), los schemas `PartialCountIn`/`PartialCountResponse` (`cash/schemas.py:118-133`), el modelo `CashPartialCount` (eliminar `app/models/cash_partial_count.py` completo) y su import en `app/models/__init__.py:25`.
- [X] T029 [US4] Backend: migración Alembic destructiva (data-model.md §2, código ya provisto) — `op.drop_index` + `op.drop_table("cash_partial_counts", schema=schema)` en `upgrade()`, `@for_each_tenant_schema`, encadenada tras la revisión de T017 (mismo down_revision chain). Confirmado sin FKs entrantes hacia esta tabla. **No ejecutar en ningún entorno con datos reales sin confirmar backup antes** (quickstart.md, "Verificación de la migración destructiva").
- [X] T030 [P] [US4] Backend: confirmar (`grep -rn "CashPartialCount\|partial-count\|partial_count" app/characterization_tests/`) que ningún characterization test referencia el código retirado — la investigación previa no encontró ninguno, pero re-verificar en el momento de implementar antes de dar la tarea por completa.
- [X] T031 [US4] Frontend: en `../pos-heladeria/src/app/modules/cash-register/components/cash-dashboard.component.ts`, retirar el botón que abre el modal (línea 37), el modal completo (líneas 160-197) y los signals/métodos `showPartial`, `partialCounted`, `partialNote`, `partialSubmitting`, `partialDiff`, `diffClass`, `openPartial`, `submitPartial` (líneas 205-238). **No tocar** `cash-arqueo-modal.component.ts` — es el arqueo de *cierre de turno* (conteo por denominación), un componente distinto que esta spec no toca.
- [X] T032 [P] [US4] Frontend: retirar `partialCount()` de `../pos-heladeria/src/app/modules/cash-register/services/cash.service.ts:168-175` y la interfaz `PartialCount` de `cash.interface.ts:101-111` (nombre real en frontend; no `PartialCountResponse`).
- [X] T033 [P] [US4] Confirmar compilación/tests limpios tras T028-T032: `ng test` en `../pos-heladeria` y la suite de caja en `../pos-backend`.
- [X] T034 [US4] Frontend: crear `slugify(text: string): string` en `../pos-heladeria/src/app/shared/slug.util.ts` (mismo estilo que `shared/date-format.util.ts`): minúsculas, sin tildes/eñes (`.normalize('NFD').replace(/[̀-ͯ]/g, '')`), espacios → guiones, cualquier otro carácter no alfanumérico eliminado, guiones repetidos colapsados, sin guion al inicio/fin (FR-003).
- [X] T035 [US4] Frontend: en `../pos-heladeria/src/app/modules/cash-register/services/cash-session.store.ts::imprimirReporte()` (líneas 536-538), antes de `window.print()`: construir `${slugify(tenantInfo.businessName())}-${fecha}` donde `fecha` es `shift.closedAt` (fecha de **cierre del turno**, no la fecha del momento de impresión/reimpresión) formateada `dd-MM-yyyy` en timezone del tenant, mismo mecanismo que `TenantDatePipe`/`formatDate` ya usado por el resto del store, fijar `document.title` a ese valor, y restaurarlo al título original al terminar (evento `afterprint` de `window`). Inyectar `TenantInfoService` en el store si no lo está ya.
- [X] T035b [P] [US4] Frontend: crear `../pos-heladeria/src/app/shared/slug.util.spec.ts` con casos de `slugify`: "Tenant de Prueba" → `tenant-de-prueba`; "Café Ñandú & Hijos" → `cafe-nandu-hijos`; espacios múltiples y guiones repetidos colapsados; sin guion inicial ni final; solo símbolos (`"***"`) → cadena vacía. Definir el fallback del nombre del PDF para el slug vacío (p. ej. `cierre-turno`) y reflejarlo en `imprimirReporte()` (T035) para no producir un archivo llamado `-28-09-2026.pdf`. Ejecutar `ng test` y confirmar verde (cubre SC-004 más allá de la prueba manual T037).
- [X] T036 [US4] Frontend: en `../pos-heladeria/src/app/modules/cash-register/components/cash-report.component.ts`, agregar el bloque `styles: [\`@media print { :host { display: block; } @page { margin: 10mm; } }\`]` (mismo patrón de `table-qr-sheet.component.ts:99-110`). **Además** — ver hallazgo arriba — en `../pos-heladeria/src/app/modules/cash-register/services/cash-session.store.ts::imprimirReporte()`, agregar `document.body.classList.add('printing-cash-report')` inmediatamente antes de `window.print()` y removerla en el evento `afterprint` de `window`; en `../pos-heladeria/src/app/modules/dashboard/layout/dashboard-layout.component.ts` (líneas ~30-40), condicionar el `print:hidden` de `<app-sidebar />`/`<app-header />` a la presencia de `body.printing-cash-report`, para que el shell se oculte únicamente al imprimir el cierre de turno, sin alterar el comportamiento de impresión de otras pantallas que ya usan `window.print()` (p. ej. `table-qr-sheet.component.ts`, que queda fuera del alcance de esta spec — Principios II/V de la constitución).
- [ ] T037 [P] [US4] Manual: recorrer quickstart.md Historia 4 completa (incluye tenant con eñes/tildes/símbolos), probando explícitamente en Chrome, Edge y Firefox de escritorio (los navegadores soportados por el negocio). Si en **cualquiera** de los tres el nombre sugerido por el diálogo de impresión no coincide exactamente con el patrón de FR-003, la Historia 4 no se da por completa hasta implementar la alternativa de PDF vía Blob documentada en research.md D1 — no queda como una reevaluación abierta.

**Checkpoint**: US1-US4 funcionales de forma independiente.

---

## Phase 7: User Story 5 - Nombre de cliente visible y obligatorio, pedidos en paralelo (Priority: P2)

**Goal**: el nombre del cliente es obligatorio y destacado en todo pedido nuevo; "Crear pedido
manual" queda disponible incluso si la mesa ya tiene un pedido abierto.

**Hallazgo de esta investigación (precisa research.md D7)**: `manual-order-page.component.ts` no
usa Angular Reactive Forms — hoy el tab "Mesa"/"Para llevar" **autocompletan** `customer_name`
con `"Consumidor final"` si el campo queda vacío (`applyDefaultCustomerName()`,
líneas 936-950), lo cual hay que retirar para que el campo sea realmente obligatorio en esos dos
tabs (el tab "Domicilio" ya lo exige hoy). No existe ningún patrón de error inline reusable en
este formulario — toda validación hoy es solo `[disabled]` en el botón + toast de error del
backend.

**Independent Test**: bloquear el guardado sin nombre; confirmar que se muestra destacado;
"Crear pedido manual" sobre una mesa con pedido abierto crea un segundo pedido independiente
(quickstart.md, Historia 5).

### Implementation for User Story 5

- [X] T038 [US5] **Gobernanza (Principio II, research.md D7)**: registrar **A-88** en `specs/000-reconocimiento/registro-de-anomalias.md` — `customer_name` pasa de opcional a obligatorio al crear cualquier pedido nuevo (mesa/para llevar/domicilio); los pedidos históricos sin nombre no se migran (FR-005).
- [X] T039 [US5] Backend: en `../pos-backend/app/api/v1/orders/service.py::create_order` (línea 237), extender la validación existente (líneas 281-290, hoy solo exige `customer_name` para `DELIVERY`) para exigirlo también en `DINE_IN`/`TAKEAWAY`: `if not (data.customer_name or "").strip(): raise HTTPException(422, "El nombre del cliente es obligatorio.")`. **Ojo de implementación**: correr esta validación **después** de resolver `participant` (líneas 267-270), donde `customer_name` se autocompleta desde `participant.display_label`/`display_name` para pedidos QR — no antes, o rechazaría pedidos que sí traen nombre del comensal.
- [X] T040 [P] [US5] Backend characterization test (en `test_orders_service.py` o archivo nuevo): `create_order` sin `customer_name` para `DINE_IN`/`TAKEAWAY` responde 422; con `participant_id` y sin `customer_name` explícito sigue funcionando (se autocompleta); `DELIVERY` conserva su validación ya existente sin duplicarla.
- [X] T041 [US5] Frontend: en `../pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`, quitar el auto-relleno `"Consumidor final"` de los tabs Mesa/Para llevar (`applyDefaultCustomerName()`/`onClienteBlur()`, líneas 936-950) y el toggle `editandoCliente()` que lo vuelve de solo-lectura; los 3 tabs (Mesa ~355-398, Para llevar ~400-431, Domicilio ~433-490) deben tratar el campo igual: siempre editable, sin valor por defecto.
- [X] T042 [US5] Frontend: mismo archivo — extender la condición `[disabled]` del botón "Crear pedido" (líneas 770-784, hoy solo exige `customer_name`/`deliveryAddress`/`deliveryFee` para el tab `domicilios`) para que también bloquee cuando `!store.customerName().trim()` en los tabs `mesas`/`para-llevar`; agregar un mensaje de error visible junto al campo (texto rojo bajo el input, no solo el toast del backend) cuando falte el nombre al intentar guardar.
- [X] T043 [US5] Frontend: verificar si existe algún otro punto de creación de pedido de mesa/para-llevar/domicilio además de `manual-order-page.component.ts` (research.md D7 lo deja pendiente) y aplicarle la misma validación si lo hay.
- [X] T044 [P] [US5] Frontend: mostrar `customer_name` de forma destacada (tipografía/posición prominente, placeholder "Cliente sin nombre" cuando sea `null`/vacío) en la tarjeta de pedido de la Terminal (`pos-order-panel.component.ts`) y en `order-detail.component.ts` (detalle/historial admin) — verificar primero si ya se muestra en algún lugar secundario y solo falta resaltarlo (FR-004).
- [X] T045 [US5] Frontend: en `../pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts`, la sub-barra con "Crear pedido nuevo" (líneas 82, 123-130) se oculta hoy por completo cuando `showingDetail()` es `true` (mesa con algo que mostrar, líneas 382-384) — exponer el botón también cuando la mesa seleccionada ya tiene un pedido abierto, para permitir el segundo pedido en paralelo de FR-007.
- [X] T046 [US5] Frontend: en `../pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.ts` (líneas 209-229), el botón "+ Crear pedido nuevo" se oculta cuando `store.selectedOrder()` ya tiene valor (línea 216) — evaluar si T045 ya cubre el caso desde la sub-barra o si conviene mantenerlo también aquí; de mantenerlo, quitar el `!store.selectedOrder()` que lo esconde.
- [X] T047 [P] [US5] Backend: agregar un test explícito (en `test_orders_service.py`) que documente como comportamiento intencional de FR-007 que `POST /orders` ya permite dos `CustomerOrder` con el mismo `dining_table_id` sin ningún rechazo (confirmado en investigación: no hay constraint que lo impida hoy) — hoy es un comportamiento "silencioso" sin ningún test que lo declare.
- [ ] T048 [P] [US5] Manual: recorrer quickstart.md Historia 5 completa.

**Checkpoint**: US1-US5 funcionales de forma independiente.

---

## Phase 8: User Story 6 - Adicionales, notas y presentación explícitos en el resumen del pedido (Priority: P3)

**Goal**: presentación, adicionales con multiplicador exacto ("Nombre xN", incluso x1) y notas
visibles de forma consistente en todas las pantallas de resumen/historial.

**Hallazgo de esta investigación (precisa research.md D10)**: hay **3** implementaciones
independientes del mismo bug de multiplicador (formato `"Nx Nombre"` en vez de `"Nombre xN"`, y
**se omite por completo cuando N=1**, contradiciendo FR-012 directamente):
`menu-lookup.ts::optionLabelWithQuantity` (líneas 106-109, usado por `order-detail.component.ts`,
`pos-terminal.store.ts`/panel de mesa, `public-menu.component.ts`),
`product-select.component.ts::chosenNames()` (líneas 733-744, resumen de cabecera del grupo
plegado) y `dining-cart.service.ts::apply()` (líneas 180-184, alimenta `cart.component.ts` y
`review-step.component.ts`). Además, `SaleItem.description`/`OrderItem` description en backend
(`sales/service.py:228-230`, `orders/checkout.py:293-295` y `447-449`) **siempre** anteponen la
presentación aunque sea el placeholder genérico `"Presentación única"` (constante
`DEFAULT_PRESENTATION_NAME` en `catalog/service.py:53`) — mostrando literalmente "Malteada -
Presentación única" en vez de omitir la etiqueta cuando el producto no maneja tamaños reales,
justo el caso que FR-010 pide omitir.

**Independent Test**: armar un pedido con presentación, dos adicionales distintos y una nota;
verificar que el resumen muestra los tres de forma explícita; un pedido sin adicionales/notas no
muestra contenedores vacíos (quickstart.md, Historia 6).

### Implementation for User Story 6

- [X] T049 [US6] Backend: crear `format_item_description(product, variant) -> str` en `../pos-backend/app/catalog_engine/core.py` (junto a `compute_line_price`): `f"{product.name} - {variant.presentation_name}"` salvo que `variant.presentation_name == DEFAULT_PRESENTATION_NAME` (importar la constante de `app/api/v1/catalog/service.py:53`) — en ese caso devolver solo `product.name` (o `variant.presentation_name` si `product` es `None`, igual que hoy). Reemplazar los 3 duplicados idénticos: `../pos-backend/app/api/v1/sales/service.py:228-230`, `../pos-backend/app/api/v1/orders/checkout.py:293-295` y `../pos-backend/app/api/v1/orders/checkout.py:447-449` (FR-010).
- [X] T050 [P] [US6] Backend: `grep -rn "Presentación única" app/characterization_tests/` y actualizar cualquier test que asserte sobre `description`/`SaleItem.description` incluyendo literalmente ese texto, citando spec 087 FR-010 (Principio III si el test está protegido).
- [X] T051 [US6] Frontend: extraer el formato "Nombre xN" (multiplicador SIEMPRE visible, incluso x1, spec 087 FR-012) a una función compartida — p. ej. `formatQuantifiedLabel(name: string, quantity: number): string` retornando `` `${name} x${quantity}` `` siempre, en `../pos-heladeria/src/app/modules/tables/services/menu-lookup.ts` (junto a `optionLabelWithQuantity`, líneas 106-109) — usarla ahí (reemplazando `quantity > 1 ? ... : label`), en `../pos-heladeria/src/app/modules/tables/components/product-select.component.ts::chosenNames()` (líneas 733-744, mismo patrón `entry[id] > 1 ? ... : name`) y en `../pos-heladeria/src/app/modules/tables/services/dining-cart.service.ts::apply()` (líneas 180-184, mismo patrón `o.quantity > 1 ? ... : name`).
- [X] T052 [P] [US6] Frontend: confirmar tras T051 que ningún contenedor de adicionales/notas queda vacío cuando no hay ninguno — los `@if` ya existentes en `order-detail.component.ts:87`, el panel de mesa, `public-menu.component.ts` y `cart.component.ts`/`review-step.component.ts` siguen funcionando con el nuevo formato.
- [X] T053 [P] [US6] Frontend: `ng test` en `../pos-heladeria`, confirmar verde los specs de los componentes/servicios tocados en T051.
- [ ] T054 [P] [US6] Manual: recorrer quickstart.md Historia 6 completa.

**Checkpoint**: US1-US6 funcionales de forma independiente.

---

## Phase 9: User Story 7 - Desglose de cambio en el detalle de venta (Priority: P3)

**Goal**: el detalle de venta muestra "Cambio: $X" cuando el pago incluyó componente en efectivo,
y lo omite por completo cuando no.

**Hallazgo de esta investigación (precisa research.md D9)**: `SaleResponse` ya serializa
`paid_amount`/`change_given` (`sales/schemas.py:182-183`) y la interfaz frontend `Sale` ya los
declara (`sales.interface.ts:166-168`) — **sin cambios de backend**. Solo falta la línea en el
modal de `sales-page.component.ts`.

**Independent Test**: venta 100% efectivo con vuelto muestra "Cambio: $X"; venta 100% con otro
método no la muestra (quickstart.md, Historia 7).

### Implementation for User Story 7

- [X] T055 [US7] Confirmar (sin cambios de código, solo verificación) que `SaleResponse` (`../pos-backend/app/api/v1/sales/schemas.py:171-198`) ya serializa `paid_amount`/`change_given` — confirmado en esta investigación, líneas 182-183.
- [X] T056 [US7] Frontend: en `../pos-heladeria/src/app/modules/sales/pages/sales-page.component.ts`, agregar `hasCashComponent(sale: Sale): boolean` que recorra `sale.payments` buscando algún `payment_method_id` cuyo `PaymentMethod.is_cash` (vía `this.methods.methods()`, ya inyectado como `methods`, línea 199) sea `true`; agregar la línea `Cambio: $X` en el modal de detalle (dentro del bloque de totales, líneas 165-179, o inmediatamente después del bloque de Pagos, líneas 180-190), visible solo cuando `hasCashComponent(r)` es verdadero, usando `r.change_given` (FR-013).
- [X] T057 [P] [US7] Frontend: si `sales-page.component.spec.ts` existe, agregar un caso con pago 100% efectivo (muestra Cambio) y uno con pago 100% no-efectivo (no lo muestra); recorrer quickstart.md Historia 7.

**Checkpoint**: las 7 historias de usuario son funcionales de forma independiente.

---

## Phase 10: Polish & Cross-Cutting Concerns (US1–US7)

**Propósito**: verificación final exigida por Principio X y por el checklist de `quickstart.md`, para las Historias 1–7. La verificación final de la adenda (US8–US11) está en la Fase 15.

- [X] T058 Ejecutar `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` en `../pos-backend` y confirmar cero regresiones no autorizadas fuera de las citadas explícitamente en T003/T010/T050 (Principio III/X).
- [X] T059 [P] Ejecutar `ng test` completo en `../pos-heladeria` y confirmar verde.
- [X] T060 [P] Probar ambas migraciones (T017, T029) en los dos sentidos (`alembic upgrade head` / `alembic downgrade -1`) contra una base con datos reales de al menos un tenant con pedidos y turnos existentes, confirmando el backup previo a T029 (quickstart.md, "Verificación de la migración destructiva").
- [ ] T061 Recorrer manualmente `quickstart.md` completo (las 7 historias, de punta a punta) y marcar cada Acceptance Scenario de `spec.md` junto con las 7 Success Criteria (SC-001 a SC-007).
- [X] T062 Confirmar que las 5 entradas de gobernanza (A-84 a A-88, o los números reales asignados en T002/T006/T016/T027/T038) quedaron efectivamente creadas y mergeadas en `registro-de-anomalias.md` antes de dar esta spec por completa (Principio II/XII).

---

# Adenda 2026-09-29 — US8 a US11 (FR-014 a FR-017)

**Contexto**: tras implementar US1–US7 se reportaron 4 defectos más, agregados a esta misma spec
(clarificación 2026-09-29) y planificados en `research.md` D11–D14. Se trabaja sobre las mismas ramas
`fix/087-cash-tables-menu-sales-fixes` de `../pos-backend` y `../pos-heladeria` (ya existen; no hay
tarea de Setup ni Foundational nueva). **Sin migraciones, sin dependencias nuevas, sin recalcular
ventas emitidas** (Principio VII, `data-model.md` §6). Solo US9 toca el backend (2 campos de respuesta
derivados); US8, US10 y US11 son 100% frontend. Las rutas de abajo fueron verificadas contra el código
real de ambos repos el 2026-09-29.

**Numeración de anomalías**: el último número usado en `registro-de-anomalias.md` es **A-88**. Esta
adenda asigna **A-89** (US9), **A-90** (US10) y **A-91** (US8 + US11 agrupadas, ver T082) — confirmar
con `grep -oE "^### A-[0-9]+" specs/000-reconocimiento/registro-de-anomalias.md | sort -t- -k2 -n | tail -1`
al momento de implementar, por si otra spec tomó alguno.

---

## Phase 11: User Story 9 - Adicionales visibles en el detalle y pedidos de la mesa (Priority: P1)

**Goal**: cada producto del panel del pedido de la Terminal de Mesas muestra sus adicionales como
"Nombre xN" (incluso x1), tanto en ítems guardados como en borradores, y un adicional guardado sigue
viéndose aunque su producto ya no esté en el menú vigente.

**Causa raíz [hipótesis, D12] — verificada parcialmente contra el código**: el panel ya renderiza
`<app-cart-item-options [options]="it.options">`; el defecto está en cómo se arma `it.options` en
`pos-terminal.store.ts`. (1) `persistedItemsView()` (línea ~950) resuelve el nombre con `lookup()`
(menú *vigente*): si la opción no está, `optionLabelWithQuantity()` devuelve `" x1"` (nombre vacío
formateado) y la línea se pinta como fragmento sin nombre (o el `.filter((o) => o.text)`, línea 975,
la descarta según el caso). `OrderItemOption` solo guarda `option_id` + `quantity`. (2) `cartView()`
(línea ~1079) arma el texto de los borradores con `c.quantity > 1 ? "${qty}x Nombre" : "Nombre"`,
contradiciendo FR-012/FR-015. Solución: el backend resuelve `name`/`group_name` en lectura (aditivo,
sin migración, `contracts/api-changes.md` §7) y el frontend los usa de respaldo.

**Independent Test**: crear un pedido con un producto con "Queso extra" x1 y "Choco-chips" x2,
verificar borrador y guardado, desactivar el producto y reabrir el pedido (quickstart.md, Historia 9).

### Implementation for User Story 9

- [X] T063 [US9] **Gobernanza (Principio II, plan.md Constitution Check adenda)**: registrar **A-89** en `specs/000-reconocimiento/registro-de-anomalias.md`, formato `A-NN — [DECISIÓN DE NEGOCIO — spec 087]` — `OrderItemOptionResponse` gana `name`/`group_name` derivados (aditivo, sin migración, sin snapshot: se resuelven por JOIN en lectura porque `order_item_options.option_id` es FK a `options.id` sin `ON DELETE`, una opción referenciada no puede borrarse); los adicionales dejan de depender del menú vigente para mostrarse en el panel de la terminal. Citar research.md D12 y contracts/api-changes.md §7.
- [X] T064 [P] [US9] Backend (test primero, confirma la hipótesis D12): crear `../pos-backend/app/characterization_tests/test_orders_item_option_names.py` (patrón de `orders_fixtures.py`). Escenarios: (a) un pedido con un ítem y dos opciones (`quantity` 1 y 2) serializado con `OrderResponse` incluye `name` y `group_name` correctos por opción; (b) la opción pertenece a un producto con `active=False`/no disponible y aun así `name`/`group_name` vienen resueltos (Acceptance Scenario 3); (c) conteo de queries acotado al listar/obtener un pedido con varias opciones — engancharse a `sqlalchemy.event` `before_cursor_execute` y afirmar que agregar más opciones al pedido **no** aumenta el número de sentencias (sin N+1, contracts §7). Debe **fallar** contra el código actual (los campos no existen) antes de T065-T066.
- [X] T065 [US9] Backend: en `../pos-backend/app/api/v1/orders/schemas.py::OrderItemOptionResponse` (línea 150), agregar `name: str | None = None` y `group_name: str | None = None` (solo lectura; `model_config` ya usa `from_attributes=True`).
- [X] T066 [US9] Backend: en `../pos-backend/app/models/order_item.py::OrderItemOption` (línea 88) exponer `name` y `group_name` como atributos derivados **sin tocar los ~13 sitios `selectinload(OrderItem.options)`** ya existentes (`orders/service.py:135,187`, `orders/checkout.py:143,284,580,675,818`, `orders/kitchen.py:74,99,187`, `orders/consolidation.py:184`, `orders/router.py:51`, `cart/service.py:434,742`): preferido, `column_property` con subconsulta escalar correlacionada sobre `options.name` y `option_groups.name` (vía `Option.option_group_id`), que viaja en el mismo SELECT (sin N+1). Alternativa solo si la anterior no resuelve limpio con `Base`/imports circulares: `relationship(Option, lazy="joined", innerjoin=True)` + `Option.option_group` cargado con `joinedload` — en ese caso el cambio sí debe aplicarse en los sitios de carga listados. **Ojo**: un `OrderItemOption` recién creado en la sesión y devuelto sin recargar tendrá estos atributos en `None` hasta refrescarse — confirmar que las rutas de respuesta pasan por `_reload_order`/recarga (p. ej. `add_item_to_order`, `create_order`); ambos campos son opcionales, así que un `None` ocasional no rompe al frontend (usa su `lookup()` como primera fuente). Confirmar que T064 pasa.
- [X] T067 [P] [US9] Frontend: en `../pos-heladeria/src/app/modules/tables/interfaces/dining.interface.ts::DiningOrderItemOption` (línea 81), agregar `name?: string | null;` y `group_name?: string | null;`.
- [X] T067b [US9] Verificación previa de alcance (FR-015): `grep -rn "\.options" ../pos-heladeria/src/app/modules/tables --include=*.ts` (excluyendo `*.spec.ts`) y listar todos los componentes que pintan las opciones de un ítem de pedido (`pos-order-panel`, `table-sessions`, `pos-checkout-panel`, otros). Confirmar que todos consumen `persistedItemsView()`/`cartView()`. Si alguno arma las opciones por su cuenta, agregar una tarea que lo alinee al formato "Nombre xN", o dejar constancia de que queda fuera de FR-015. **Resultado (2026-09-29)**: los únicos sitios que pintan las opciones en el panel de la Terminal (`pos-order-panel`, `manual-order-page`) consumen `cartView()`/`ordersView()` → `persistedItemsView()`; `public-menu.itemOptionLines()` (historial del Menú QR) y `dining-cart.service` (carrito del comensal) arman las suyas y quedan fuera de FR-015 (panel de la Terminal); `receipt.util` usa las opciones de `SaleItem`, no de `OrderItem`.
- [X] T068 [US9] Frontend: en `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts::persistedItemsView()` (líneas ~966-976), resolver cada opción guardada como: `const name = lk.optionLabel(o.option_id) || o.name || ''` (verificar que `optionLabel` esté en la interfaz de `menu-lookup.ts`; si no, exponerla, la implementación ya la define en la línea ~117) y `groupLabel: lk.optionGroupLabel(o.option_id) ?? o.group_name ?? null`; `text: formatQuantifiedLabel(name, o.quantity ?? 1)` (import desde `./menu-lookup`, ya existe desde US6). Cambiar el `.filter((o) => o.text)` (línea 975) a `.filter((o) => !!name)` **solo** sobre el nombre ya resuelto con respaldo — ya no se descarta un adicional por no estar en el menú vigente, solo si ni el lookup ni el backend traen nombre (caso que no debe ocurrir con T066). Actualizar el docstring de `persistedItemsView` (líneas ~945-949) citando spec 087 FR-015/D12.
- [X] T069 [US9] Frontend: en `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts::cartView()` (líneas ~1074-1082, rama `draft` de productos), reemplazar `c.quantity > 1 ? \`${c.quantity}x ${c.option.name}\` : c.option.name` por `formatQuantifiedLabel(c.option.name, c.quantity)` (FR-012/FR-015: multiplicador siempre visible, formato "Nombre xN", igual que los guardados). Mismo archivo que T068 — hacerlas en secuencia.
- [X] T070 [P] [US9] Frontend tests: en `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` agregar casos (a) ítem guardado cuya opción **no** está en el menú vigente pero trae `name`/`group_name` → aparece "Nombre x1" bajo su grupo (no `" x1"`, no omitida); (b) borrador con "Queso extra" x1 y "Choco-chips" x2 → `"Queso extra x1"` y `"Choco-chips x2"`; (c) ítem sin opciones → `options` vacío (sin contenedor). Actualizar cualquier caso existente que espere el formato viejo `"2x Nombre"` de borrador. Ejecutar `ng test` y confirmar verde.
- [X] T071 [P] [US9] Backend: ejecutar `python -m unittest app.characterization_tests.test_orders_item_option_names app.characterization_tests.test_orders_service app.characterization_tests.test_orders_checkout app.characterization_tests.test_orders_consolidation app.characterization_tests.test_orders_kitchen app.characterization_tests.test_cart_service -v` desde `../pos-backend` y confirmar verde (ningún `"CONGELA comportamiento actual:"` debería tocar `OrderItemOptionResponse`; si alguno cambia, investigar y citar spec 087 D12/A-89 antes de tocarlo — Principio III).
- [ ] T072 [P] [US9] Manual: recorrer quickstart.md Historia 9 completa (borrador, guardado + recarga, producto desactivado, producto sin adicionales).

**Checkpoint**: US9 funcional de forma independiente — los adicionales se ven igual en borrador y guardado, sin depender del menú vigente.

---

## Phase 12: User Story 10 - Total del pedido actualizado al editar un pedido manual (Priority: P1)

**Goal**: el "TOTAL ORDEN" del pedido manual coincide siempre con el conjunto vigente completo (ítems
guardados no anulados + borradores, con promociones reevaluadas), sin recargar, y muestra $0 cuando
no queda ningún ítem.

**Causa raíz confirmada contra el código (D13)**: (1) el `effect` de
`manual-order-page.component.ts:822-827` depende solo de `draftLines()`, `orderTypeTab()` y
`deliveryFee()`, no de los ítems **guardados**; (2) `draftPreviewPayload()`
(`pos-terminal.store.ts:2085-2098`) manda solo los borradores, y al quedar el borrador vacío tras
guardar, `loadDraftPreview()` (línea 2100) pone `draftPreview = null` y el panel cae al subtotal local
(`totals()`, `discount = 0`, sin promociones); (3) no hay guarda contra respuestas fuera de orden.
Solución sin cambio de contrato (`contracts/api-changes.md` §8): el endpoint ya acepta `OrderItemIn[]`;
el frontend manda el conjunto vigente completo.

**Independent Test**: abrir un pedido manual con total conocido, agregar/quitar productos y guardar;
el "TOTAL ORDEN" se actualiza sin recargar, incluyendo promociones y el caso de cero ítems
(quickstart.md, Historia 10).

### Implementation for User Story 10

- [X] T073 [US10] **Gobernanza (Principio II)**: registrar **A-90** en `specs/000-reconocimiento/registro-de-anomalias.md`, formato `A-NN — [DECISIÓN DE NEGOCIO — spec 087]` — el desglose "TOTAL ORDEN" del pedido manual pasa de calcularse solo sobre borradores a calcularse sobre el conjunto vigente completo (guardados no anulados + borradores), con promociones reevaluadas por el backend (`POST /orders/draft-preview` sin cambio de contrato) y con $0 cuando no queda ningún ítem; ninguna venta emitida se recalcula (Principio VII). Citar research.md D13.
- [X] T074 [P] [US10] Backend: agregar `../pos-backend/app/characterization_tests/test_orders_draft_preview_parity.py` verificando FR-016 ("el total mostrado tras guardar MUST coincidir con el que calcule el backend al cobrar"): con un pedido guardado con un producto en promoción + un ítem nuevo, `compute_draft_preview` (`orders/checkout.py`) sobre el conjunto completo `[ítem guardado + ítem nuevo]` da el mismo `total` que `compute_checkout_preview` del pedido equivalente ya guardado con ambos ítems; y que con solo el ítem nuevo el resultado difiere si la promoción depende de la cantidad conjunta (`min_qty`) — evidencia de por qué el frontend debe mandar el conjunto completo. Si pasa desde el primer intento (el backend ya es correcto), es la línea base del fix de frontend, no un test rojo. Agregar además un caso con **combo** en el pedido: el total mostrado (`total` de `compute_draft_preview` sobre los productos + precio de combo sumado como lo hace el frontend, T079) debe coincidir con `total` de `compute_checkout_preview` del pedido equivalente guardado (FR-016, combos sumados a su precio de combo sin descuento promocional adicional).
- [X] T075 [US10] Frontend (test primero): en `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts` agregar casos para `loadDraftPreview()` con un `api.draftPreview` espiado: (a) con un pedido seleccionado con ítems guardados y un borrador, el payload enviado incluye **ambos** (`product_variant_id`, `quantity`, `options: [{option_id, quantity}]`), sin incluir ítems `anulado` ni ítems con `combo_id`; (b) con ítems guardados y borrador vacío, sigue llamando al endpoint (antes salía con `draftPreview = null`); (c) dos llamadas solapadas donde la primera responde después → prevalece la segunda (guarda de respuesta obsoleta); (d) sin ningún ítem vigente → `draftPreview` queda `null`. (a)-(c) deben **fallar** contra el código actual antes de T076-T077.
- [X] T076 [US10] Frontend: en `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts::draftPreviewPayload()` (líneas 2085-2098) incluir, además de los borradores de tipo `product`, los ítems del pedido seleccionado (`this.selectedOrder()?.items`) con `estado_cocina !== 'anulado'` y sin `combo_id`, mapeados a `{ product_variant_id, quantity, options: (i.options ?? []).map((o) => ({ option_id: o.option_id, quantity: o.quantity ?? 1 })) }` (misma forma `OrderItemIn` que ya usan los borradores). Actualizar el docstring del bloque (líneas ~2069-2078, hoy dice "Desglose del **borrador**") para reflejar que ahora cubre el conjunto vigente (spec 087 FR-016/D13).
- [X] T077 [US10] Frontend: en el mismo `loadDraftPreview()` (líneas 2100-2117) agregar un contador de secuencia (`private draftPreviewSeq = 0`): incrementar al iniciar, capturar el valor local, y tras el `await` (y en el `catch`/`finally`) ignorar la respuesta/error si `seq !== this.draftPreviewSeq` — un preview viejo no debe pisar al nuevo ni apagar `draftPreviewLoading` de la llamada más reciente. Con `payload.items.length === 0` (ningún ítem vigente) conservar el comportamiento actual (`draftPreview = null`, sin error) pero también invalidar la secuencia en curso, para que una respuesta pendiente no reaparezca después (Edge Case: total = $0, no el anterior).
- [X] T078 [US10] Frontend: en `../pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts::_draftPreview` (líneas 822-827), agregar la dependencia `this.store.selectedOrder();` (al mismo nivel de `this.store.draftLines()`) para que el `effect` se recalcule cada vez que cambia la lista de ítems guardados (guardar, anular, refresco por sondeo/tiempo real). Actualizar el docstring (líneas 818-821) — hoy dice "recalcula el desglose del borrador".
- [X] T079 [US10] Frontend: verificar en `manual-order-page.component.ts` (bloque "TOTAL ORDEN", líneas ~716-745, y el refresco previo a confirmar en líneas ~926-930) y en `pos-terminal.store.ts::totals()` que (a) con cero ítems vigentes el panel muestra $0 (no un total anterior); (b) el total mostrado **suma los combos** — los combos (`kind: 'combo'`, líneas con `combo_id`) siguen fuera del `draft-preview` (D13); si hoy no se suman al total mostrado, cubrirlos con el subtotal local ya existente `comboDisplaySubtotal` sumándolo al `total` del preview (a su precio de combo, sin descuento promocional ni impuestos adicionales, según FR-016; la paridad con el cobro la prueba T074), **sin** ampliar el alcance a otro rediseño (Principio V). Documentar el resultado de la verificación en el commit/PR. **Resultado (2026-09-29)**: (a) con cero ítems vigentes `draftPreview` queda `null` y `totals()` da $0 (cubierto por T075(d) y T080(c)); (b) los combos NO estaban sumados al total cuando había preview → se añadió `PosTerminalStore.combosSubtotal` (líneas con `comboId`, guardadas + borrador) y la página lo suma a `subtotal`/`total` del preview; paridad con el cobro probada en `test_orders_draft_preview_parity.py`.
- [X] T080 [P] [US10] Frontend tests: en `../pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts` agregar casos: (a) tras guardar un ítem nuevo, el "TOTAL ORDEN" refleja el `total` del preview del conjunto completo (no el subtotal local); (b) al anular/quitar un ítem, el total desciende; (c) con cero ítems el total mostrado es $0. Ejecutar `ng test` en `../pos-heladeria` y confirmar verde, sin errores de consola durante el recálculo (SC-011).
- [ ] T081 [P] [US10] Manual: recorrer quickstart.md Historia 10 completa (agregar, quitar, promoción, todos los ítems fuera, consola del navegador abierta sin errores).

**Checkpoint**: US9 + US10 funcionales de forma independiente.

---

## Phase 13: User Story 8 - Presentación siempre visible junto a la promoción en el modal del producto (Priority: P2)

**Goal**: en "Elige tu presentación" (modal del producto del Menú QR) cada fila muestra el nombre de la
presentación completo como etiqueta principal y la promoción como etiqueta secundaria en su propia
línea; sin promoción, solo el nombre.

**Causa raíz confirmada contra el código (D11)**: en `product-select.component.ts:132` el nombre va en
`<span class="block text-sm font-semibold text-gray-900 truncate">` dentro de un contenedor `min-w-0`,
mientras que la etiqueta de promoción (línea 141, `inline-flex`) y la columna de precio (línea ~148,
`shrink-0`) no se encogen: en ancho de celular el nombre queda reducido a "…". Cambio solo de
clases/plantilla; `rowPackagePromo`, `discountFor` y `variantPrice` no se tocan.

**Independent Test**: en el Menú QR a ~360px de ancho, abrir un producto con varias presentaciones y
promociones; cada fila muestra el nombre completo y la promoción debajo (quickstart.md, Historia 8).

### Implementation for User Story 8

- [X] T082 [US8] **Gobernanza (Principio II, agrupada US8 + US11)**: registrar **A-91** en `specs/000-reconocimiento/registro-de-anomalias.md`, formato `A-NN — [DECISIÓN DE NEGOCIO — spec 087]` — dos cambios solo de presentación, sin cambio de datos ni de cobro: (US8) la fila de presentación del modal del Menú QR deja de truncar el nombre y muestra la promoción como etiqueta secundaria; (US11) la nota por producto de `app-cart-item-options` pasa a un único estilo reforzado (≥16px, semibold, alto contraste, `rounded-lg`, multilínea). Citar research.md D11 y D14; `order.notes` (nota general) fuera de alcance.
- [X] T083 [US8] Frontend: en `../pos-heladeria/src/app/modules/tables/components/product-select.component.ts`, fila de presentación (líneas ~131-146): cambiar el `<span>` del nombre (línea 132) de `truncate` a `break-words` (se ajusta en varias líneas, nunca "…"); dejar la etiqueta de promoción (línea 141) en su propia línea (`block`/`flex-wrap`) con `max-w-full whitespace-normal break-words` para que un `display_text` largo baje de línea en vez de desplazar al nombre; mantener el `@if (v.promotion)` (fila sin promoción = solo el nombre, sin etiqueta vacía). No tocar la columna de precio ni `rowPackagePromo`/`discountFor`/`variantPrice`. Considerar `items-start` en el contenedor de la fila (línea ~120, hoy `items-center`) para que el radio y el precio no queden centrados respecto a un nombre de varias líneas.
- [X] T084 [P] [US8] Frontend: en `../pos-heladeria/src/app/modules/tables/components/product-select.component.spec.ts` agregar casos: fila con promoción muestra el nombre de la presentación **y** el texto de la promoción por separado (nunca solo la promoción); fila sin promoción no renderiza el elemento de etiqueta; el elemento del nombre no tiene la clase `truncate`. Ejecutar `ng test` y confirmar verde.
- [ ] T085 [P] [US8] Manual: recorrer quickstart.md Historia 8 en un ancho de celular (~360px) con las tres filas del setup (8 oz + 2x1, Mediano + 15%, sin promoción) y el caso de nombre largo + promoción. **Repetir el recorrido en las superficies de la Terminal que reutilizan `app-product-select`** (`manual-order-page.component.ts` y `pos-catalog-drawer.component.ts`): FR-014 aplica a todos los sitios donde aparece "Elige tu presentación" (componente compartido).

**Checkpoint**: US8 funcional de forma independiente.

---

## Phase 14: User Story 11 - Notas por producto legibles y destacadas (Priority: P2)

**Goal**: la nota de cada producto se ve con texto ≥16px, semibold/bold y fondo de alto contraste, con
un único estilo, en las tres pantallas que usan `app-cart-item-options` (panel de la terminal, página de
pedido manual, historial del Menú QR).

**Punto de cambio confirmado contra el código (D14)**: `cart-item-options.component.ts` pinta la nota
con `text-[11px] … rounded-full italic` (`bg-[#fffbeb] text-[#b45309]`) y el bloque está **duplicado**
en las dos ramas de la plantilla (`@if (grupos.length > 0)`, líneas ~33-39, y `@else if (notes)`,
líneas ~42-48). El componente lo consumen `pos-order-panel.component.ts`, `manual-order-page.component.ts`
y `public-menu.component.ts` — exactamente las tres pantallas de FR-017. Colapsar el duplicado es
**exigido** por FR-017 ("un único estilo"), no un refactor oportunista (Principio V).

**Independent Test**: crear un pedido con la nota "sin azúcar" y revisarla en las tres pantallas; con
una nota larga y con un producto sin nota (quickstart.md, Historia 11).

### Implementation for User Story 11

- [X] T086 [US11] **Requiere T082 (A-91) mergeada.** Frontend: en `../pos-heladeria/src/app/modules/tables/components/cart-item-options.component.ts`, definir la nota **una sola vez** en la plantilla (eliminando la copia duplicada de las dos ramas — p. ej. un `<ng-template #notaTpl>` o reordenando el `@if` para que la nota se pinte fuera del bloque de grupos) con: `text-base` (16px), `font-semibold`, sin `italic`, fondo de alto contraste con texto oscuro (paleta ámbar ya presente, p. ej. `bg-amber-100 text-amber-900 border border-amber-300`), `whitespace-pre-wrap break-words`, `rounded-lg` en vez de `rounded-full` (una pill de 16px se deforma con varias líneas) y `w-full`/`block` para no desbordar la tarjeta. Mantener el `@if (notes)` (sin contenedor vacío cuando no hay nota) y el prefijo "Nota:". Verificar contraste ≥ 4.5:1 (WCAG AA) del par texto/fondo elegido. No tocar `order.notes` (nota general) ni la lógica de agrupado de opciones (`optionGroups`).
- [X] T087 [P] [US11] Frontend: actualizar `../pos-heladeria/src/app/modules/tables/components/cart-item-options.component.spec.ts` — la nota renderiza una sola vez (con y sin opciones), con las clases de tamaño/peso reforzadas y sin `italic`; sin `notes` no hay contenedor de nota; una nota larga conserva `break-words`/`whitespace-pre-wrap`. Ejecutar `ng test` y confirmar verde, y revisar que los specs de `pos-order-panel`, `manual-order-page` y `public-menu` que asertan sobre el texto de la nota sigan verdes (el texto no cambia, solo el estilo).
- [ ] T088 [P] [US11] Manual: recorrer quickstart.md Historia 11 completa en las tres pantallas (panel de la terminal, página de pedido manual, historial del Menú QR), con nota corta, larga y producto sin nota; confirmar que `order.notes` (nota general) **no** cambió de aspecto.

**Checkpoint**: las 4 historias de la adenda (US8–US11) son funcionales de forma independiente.

---

## Phase 14b: Corrección US8-b — el nombre de la presentación no llegaba al Menú QR del comensal (2026-09-29)

**Contexto**: tras implementar US8 el usuario reportó (captura) que en el modal del Menú QR seguía
viéndose solo la promoción. **Causa raíz confirmada** (research.md D11, corrección): `diner.service.ts::mapCategory`
leía `v['name']`, pero desde spec 084 el backend manda `presentation_name` → `name: undefined`. Sin cambio
de backend ni de contrato (`contracts/api-changes.md` §9). Gobernanza: amplía A-91 (sin entrada nueva).

- [X] T093 [US8] Frontend (test primero, visto en rojo — `expected [undefined, undefined] to deeply equal ['Grande','Mediano']`): en `../pos-heladeria/src/app/modules/tables/services/diner.service.spec.ts` agregar el caso "el nombre de la variante sale de `presentation_name`" y actualizar la respuesta de ejemplo antigua (que usaba `name`) al contrato real (`presentation_id` + `presentation_name`).
- [X] T094 [US8] Frontend: en `../pos-heladeria/src/app/modules/tables/services/diner.service.ts::mapCategory` mapear `name: (v['presentation_name'] ?? v['name'])` y `presentation_id`. `product-select.component.ts` no se toca.
- [X] T095 [P] [US8] Verificar consumidores del nombre de variante en el Menú QR (`public-menu.component.ts`, `dining-cart.service.ts::indexMenu`): sin cambios de lógica; specs de `public-menu`, `dining-cart`, `product-select` y `diner.service` en verde (91/91).
- [ ] T096 [P] [US8] Manual: abrir el modal de un producto con varias presentaciones **desde el Menú QR del comensal** (enlace del QR de una mesa) y confirmar nombre + promoción por fila, también a ~360px (quickstart.md Historia 8, paso 4).

**Checkpoint**: el Menú QR muestra la presentación igual que la Terminal.

---

## Phase 15: Polish & Cross-Cutting Concerns (adenda US8–US11)

**Propósito**: verificación final de la adenda (Principio X), equivalente a la Fase 10 para US1–US7.

- [X] T089 Ejecutar `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` en `../pos-backend` y confirmar cero regresiones fuera de las citadas explícitamente en esta adenda (Principio III/X); los 940 tests de la línea base previa deben seguir en verde más los nuevos de T064 y T074.
- [X] T090 [P] Ejecutar `ng test` completo en `../pos-heladeria` y confirmar verde (incluye los casos nuevos de T070, T075, T080, T084, T087), y `ng build` sin errores de tipo (SC-011: "sin errores de tipo ni de consola durante el renderizado o el recálculo de montos").
- [ ] T091 Recorrer manualmente `quickstart.md` Historias 8–11 (incluido el paso 4 de la Historia 8 / T096) de punta a punta y marcar los Acceptance Scenarios de US8–US11 de `spec.md` junto con SC-008 a SC-011.
- [X] T092 Confirmar que las 3 entradas de gobernanza de la adenda (A-89, A-90, A-91, o los números reales asignados en T063/T073/T082) quedaron creadas y mergeadas en `registro-de-anomalias.md` antes de dar la adenda por completa (Principio II/XII).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato.
- **Foundational (Phase 2)**: no aplica a esta spec (ver nota arriba) — las historias pasan directo de Setup a su propia fase.
- **US1 (Phase 3)**: depende solo de Setup. Sin dependencia de ninguna otra historia — 100% backend, un único archivo de lógica (`promotions/service.py`). **Recomendado como primer incremento (MVP)** por ser P1 y el de mayor severidad financiera.
- **US2 (Phase 4)**: depende solo de Setup. Independiente de US1 en código (capas distintas). T014 dejará el flujo de creación implícita de pedidos reemplazado por "Crear pedido manual" — relevante para US5, pero no bloqueante (esa vía ya existe y funciona hoy, US5 solo la hace más estricta).
- **US3 (Phase 5)**: depende solo de Setup en código. Operativamente necesita un turno de caja abierto para poder probarse (no es una dependencia de código, es un prerrequisito de datos de prueba, igual que en quickstart.md).
- **US4 (Phase 6)**: depende solo de Setup. Completamente independiente de las demás historias (caja, no toca pedidos).
- **US5 (Phase 7)**: depende solo de Setup en código. Comparte archivos de frontend con US2 (`pos-terminal.store.ts`, `manual-order-page.component.ts`) — **implementar después de US2** para evitar reescribir dos veces el mismo flujo de creación de pedidos, aunque no hay un bloqueo estricto de código.
- **US6 (Phase 8)**: depende solo de Setup. Independiente — cambios de formato/presentación puros.
- **US7 (Phase 9)**: depende solo de Setup. La más pequeña e independiente de todas — sin cambios de backend.
- **Polish (Phase 10)**: depende de que US1–US7 estén completas.
- **Adenda US8–US11 (Fases 11–14)**: dependen solo de que US1–US7 ya estén implementadas (lo están, T001–T062); sin dependencia de código entre sí, con **una restricción práctica**: **US9 (T068-T069) y US10 (T076-T077) editan `pos-terminal.store.ts`** (zonas distintas: `persistedItemsView`/`cartView` vs. `draftPreviewPayload`/`loadDraftPreview`), así que hacerlas en secuencia (US9 → US10) evita conflictos de merge. US8 (`product-select.component.ts`) y US11 (`cart-item-options.component.ts`) son totalmente independientes de todo lo demás.
- **Polish adenda (Phase 15)**: depende de que US8–US11 estén completas.

### Historias con gate de gobernanza antes de dar la historia por completa

US1 (T002 → A-84), US2 (T006 → A-85, antes de T008), US3 (T016 → A-86), US4 (T027 → A-87, antes
de T029), US5 (T038 → A-88) — cada una debe mergear su entrada en `registro-de-anomalias.md`
antes de que esa historia se considere terminada (Principio II/XII), verificado en T062.

Adenda: US9 (T063 → A-89), US10 (T073 → A-90), US8 + US11 (T082 → A-91, una sola entrada agrupada
— ambas son solo presentación) — verificado en T092.

### Dentro de cada historia

- El fix/endpoint/migración va antes que el test que lo verifica solo cuando el test no estaba
  ya protegiendo el comportamiento previo (Principio III) — en US1 y US2, donde sí hay tests
  protegidos que documentan el comportamiento a cambiar, el test se actualiza primero (debe
  fallar contra el código actual) y el fix después.
- Backend antes que frontend dentro de cada historia, salvo US4 (Arqueo Parcial puede retirarse
  en paralelo en ambos repos) y US6 (backend y frontend son bugs independientes del mismo tipo,
  sin dependencia entre sí).

### Parallel Opportunities

- US1, US2, US3 y US4 pueden trabajarse en paralelo por personas distintas — tocan capas/archivos
  disjuntos (promociones vs. consolidación de pedidos vs. numeración vs. caja).
- Dentro de cada historia, las tareas marcadas `[P]` (tests de verificación, archivos de
  interfaz/schemas independientes) pueden ejecutarse en paralelo entre sí.
- US5 debería esperar a que US2 esté mergeada para minimizar conflictos de merge en
  `pos-terminal.store.ts`/`manual-order-page.component.ts`, aunque no es un bloqueo de código.
- Adenda: US8, US11 y (US9 → US10) pueden trabajarse en paralelo entre sí (cuatro archivos de
  frontend distintos, salvo el `pos-terminal.store.ts` compartido por US9/US10). Dentro de US9,
  T064 (test backend) y T067 (interfaz frontend) son `[P]`; dentro de US10, T074 (paridad backend)
  es `[P]` respecto al trabajo de frontend.

---

## Parallel Example: User Story 1

```bash
# T003 (actualizar el test protegido) y T004 (aplicar el fix) son secuenciales -- el test
# debe fallar primero contra el código actual. T005 sí puede correr en paralelo a cualquier
# otra tarea de otra historia una vez T004 está aplicado:
Task: "Ejecutar la suite de promociones/carrito/checkout y confirmar cero regresiones"
```

## Parallel Example: User Story 4

```bash
# Backend y frontend de la eliminación de Arqueo Parcial son independientes entre sí:
Task: "Retirar endpoint, schemas y modelo de Arqueo Parcial en pos-backend (T028)"
Task: "Retirar modal, estado y servicio de Arqueo Parcial en pos-heladeria (T031, T032)"
```

## Parallel Example: Adenda (US8, US9, US11)

```bash
# Tres historias de la adenda que no comparten archivos:
Task: "US8 — fila de presentación sin truncar en product-select.component.ts (T083)"
Task: "US11 — estilo único de nota en cart-item-options.component.ts (T086)"
Task: "US9 — test backend de name/group_name en order_item_options (T064) + interfaz frontend (T067)"
# US10 se hace después de US9 en pos-terminal.store.ts (mismo archivo).
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup.
2. Completar Phase 3: User Story 1 (sin Foundational, no aplica).
3. **STOP and VALIDATE**: probar US1 de forma independiente (quickstart.md, Historia 1).
4. Deploy/demo si está listo — corrige el error de mayor severidad financiera del conjunto.

### Incremental Delivery

1. Setup listo (sin Foundational).
2. US1 (P1) → validar independientemente → deploy/demo (MVP).
3. US2 (P1) → validar → deploy/demo.
4. US3, US4, US5 (P2, en el orden que convenga al equipo — sin dependencias entre sí en código) →
   validar cada una → deploy/demo.
5. US6, US7 (P3) → validar → deploy/demo.
6. Cada historia agrega valor sin romper las anteriores — todas fueron diseñadas para ser
   independientemente reversibles (plan.md, "Structure Decision").
7. **Adenda (US8–US11)**, tras US1–US7: US9 y US10 primero (ambas P1: un adicional que no se ve
   provoca un pedido incompleto y un total desactualizado es un error de dinero visible al
   cliente), luego US8 y US11 (P2, solo presentación). Cada una se valida con su Historia de
   `quickstart.md` antes de pasar a la siguiente. **MVP de la adenda: US9 (T063-T072)**.

### Parallel Team Strategy

Con varios desarrolladores, tras completar Setup: Desarrollador A → US1; Desarrollador B → US2;
Desarrollador C → US3 + US4 (ambas P2, independientes entre sí); US5 se asigna a quien termine
US2 primero (comparte archivos de frontend); US6/US7 se reparten al final, son las más pequeñas.

---

## Notes

- `[P]` tareas = archivos distintos, sin dependencias.
- `[Story]` mapea cada tarea a su historia de usuario para trazabilidad (Principio XII).
- Varias rutas/líneas y una causa raíz (US1) difieren de lo que `research.md`/`data-model.md`
  conjeturaban antes de esta investigación — donde eso ocurre, la fase correspondiente lo indica
  explícitamente ("Hallazgo de esta investigación").
- Los 8 gates de gobernanza (T002, T006, T016, T027, T038 y, en la adenda, T063, T073, T082) no son opcionales: sin la entrada
  correspondiente en `registro-de-anomalias.md`, ninguna de esas 5 historias puede darse por
  completa según el Principio II de la constitución.
- Verificar tests fallando antes de implementar donde aplica (US1 T003, US2 T010 parcialmente).
- Commits pequeños por tarea o grupo lógico de tareas, en inglés, Conventional Commits, sin marca
  de autoría de IA (Principio XV) — solo cuando el usuario lo pida explícitamente.
