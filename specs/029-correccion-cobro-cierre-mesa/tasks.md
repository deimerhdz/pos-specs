---

description: "Task list template for feature implementation"
---

# Tasks: Correcciones de Cobro, Anulación y Descuento en la Terminal de Mesas

**Input**: Design documents from `/specs/029-correccion-cobro-cierre-mesa/`

**Prerequisites**: [plan.md](./plan.md) (required), [spec.md](./spec.md) (required for user stories),
[research.md](./research.md), [data-model.md](./data-model.md), [contracts/api-contracts.md](./contracts/api-contracts.md)

**Tests**: Este proyecto ya tiene una convención de tests establecida (characterization tests en
backend, specs Vitest co-ubicados en frontend) y la Constitución exige verificación obligatoria
(Principio X) antes de dar una spec por completa — por eso cada historia incluye tareas de test,
agregadas junto a la implementación, citando el spec/FR que verifican. El test
`test_transition_kitchen_y_void_item_no_validan_status_de_la_orden_a16` requiere además una
actualización explícita citando esta spec y la anomalía **A-16** (Principio III de la
Constitución — ver research.md D3).

**Organization**: Tasks are grouped by user story to enable independent implementation and testing
of each story. Las tres historias P1 (US1, US2, US3) son el MVP: cada una corrige, por sí sola, uno
de los problemas de mayor riesgo reportados (pérdida de mercancía/inventario, descuento no
autorizado, insignia engañosa). US4 (P2) es una corrección de claridad operativa independiente.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4)
- Include exact file paths in descriptions. Rutas relativas a cada repo hermano de `pos-specs`:
  `../pos-backend/` y `../pos-heladeria/` (ver [plan.md](./plan.md), sección Project Structure).

## Path Conventions

- **Backend** (`../pos-backend/`): `app/api/v1/orders/{kitchen,schemas,service,router}.py`,
  `app/characterization_tests/test_*.py`.
- **Frontend** (`../pos-heladeria/`): `src/app/modules/tables/{pages,components,services,interfaces}/`,
  specs co-ubicados `*.component.spec.ts` / `*.store.spec.ts`.

---

## Phase 1: Setup

**Purpose**: Confirmar la línea base antes de tocar código.

- [X] T001 Verificar la rama `029-correccion-cobro-cierre-mesa` en ambos repos hermanos
  (`../pos-backend`, `../pos-heladeria`) y confirmar que la suite base pasa antes de empezar:
  backend `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` (desde
  `../pos-backend`), frontend `ng test` (desde `../pos-heladeria`).

**Checkpoint**: Línea base verde conocida antes de cualquier cambio.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: La señal única de "este pedido ya está pagado" (`Sale.customer_order_id`, D2/D3 de
[research.md](./research.md)) que necesitan tanto US1 (bloquear anulación) como US3 (insignia
"Listo"). US2 y US4 no dependen de esta fase.

**⚠️ CRITICAL**: US1 y US3 no empiezan antes de completar esta fase. US2 y US4 sí pueden avanzar
en paralelo con esta fase si hay más de una persona.

- [X] T002 [P] Agregar `order_has_sale(db: Session, order_id: UUID) -> bool` y
  `paid_order_ids(db: Session, order_ids: list[UUID]) -> set[UUID]` en
  `../pos-backend/app/api/v1/orders/service.py`, reutilizando exactamente el mismo patrón de
  subconsulta ya probado en `has_billable_orders`
  (`../pos-backend/app/api/v1/table_sessions/service.py:65-83`, `Sale.customer_order_id.isnot(None)`)
  — **sin modificar** esa función ni `table_sessions/service.py` (research.md D2, nota sobre
  reutilización sin refactorizar).
- [X] T003 Agregar el campo `paid: bool` a `OrderResponse` en
  `../pos-backend/app/api/v1/orders/schemas.py`; en
  `../pos-backend/app/api/v1/orders/router.py`, hacer que `list_orders` calcule `paid_order_ids`
  **una sola vez** para todos los pedidos de la respuesta (evita N+1) y que `_load_order`/
  `get_order` usen `order_has_sale` para un solo id, asignando el valor a cada `CustomerOrder` antes
  de serializar (contracts/api-contracts.md). Depende de T002.
- [X] T004 [P] Tests backend: `order_has_sale`/`paid_order_ids` — cubrir una orden `status="abierta"`
  con una `Sale` ya asociada (el caso real de QR/mostrador que motivó la spec) y una orden
  `status="pagada"` sin `Sale` en el fixture (compatibilidad con el characterization test existente
  de A-16, ver Phase 3/T007), en `../pos-backend/app/characterization_tests/test_orders_service.py`
  (o el fichero de characterization tests correspondiente a `orders/service.py`).
- [X] T005 [P] Agregar el campo `paid: boolean` a la interfaz `DiningOrder` en
  `../pos-heladeria/src/app/modules/tables/interfaces/dining.interface.ts` (espejo de
  `OrderResponse.paid`).

**Checkpoint**: La señal `paid` existe de punta a punta (backend → frontend); US1 y US3 pueden
empezar.

---

## Phase 3: User Story 1 - Un pedido ya pagado no se puede anular (Priority: P1) 🎯 MVP (parte A)

**Goal**: Ni el cajero ni el mesero pueden anular un ítem u orden que ya tenga un pago registrado,
sin importar su estado de cocina.

**Independent Test**: sobre una orden ya pagada y facturada, intentar anular uno de sus ítems (o
la orden completa) desde la Terminal de Mesas y verificar que el sistema rechaza la acción con un
mensaje claro, sin alterar el pago, la factura ni el inventario.

### Backend for User Story 1

- [X] T006 [US1] En `void_item()` (`../pos-backend/app/api/v1/orders/kitchen.py:93-176`), agregar un
  guard al inicio (antes de validar `estado_cocina == "anulado"`): cargar `order.status` de la
  `CustomerOrder` padre del ítem y, si `order.status == "pagada"` o `order_has_sale(db,
  item.order_id)` (T002) es verdadero, responder `409 "El pedido ya fue pagado y no puede
  anularse"` sin mutar el ítem, sin registrar `OrderItemVoidLog` ni ejecutar reversa de inventario
  o reemplazo (FR-005, FR-006). `transition_kitchen` y `mark_order_ready` en el mismo fichero
  **no cambian** (research.md D3). Depende de T002.
- [X] T007 [US1] Actualizar `test_transition_kitchen_y_void_item_no_validan_status_de_la_orden_a16`
  en `../pos-backend/app/characterization_tests/test_orders_kitchen.py`: dividir en dos métodos —
  uno que conserva sin cambios la aserción sobre `transition_kitchen` (sigue protegida, esta spec no
  la toca) y otro que cambia la aserción sobre `void_item` de "se ejecuta igual" a
  `self.assertRaises(HTTPException)` con `status_code == 409`, citando en el docstring que spec 029
  (Historia 1, FR-005/006/007) autoriza este cambio sobre la anomalía **A-16**. Depende de T006.
- [X] T008 [P] [US1] Test backend nuevo: `void_item` rechazado sobre una orden `status="abierta"`
  con una `Sale` ya asociada (el caso real del camino QR/mostrador, distinto del `status="pagada"`
  que ya cubre T007) — en el mismo fichero de T007. Depende de T006.

### Frontend for User Story 1

- [X] T009 [US1] Ocultar el botón "Anular" en
  `../pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.ts:99-102` cuando
  `store.selectedOrder()?.paid === true` (FR-007). Depende de T005.
- [X] T010 [P] [US1] En `voidPersistedItem`/`voidPersistedCombo`
  (`../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts:1039-1064`), manejar el
  nuevo `409` del backend mostrando el mensaje del servidor vía el mecanismo de error/toast ya
  existente (defensa en profundidad — el botón ya está oculto por T009, pero una llamada directa
  igual debe fallar con claridad). Depende de T006. **Ya satisfecho sin cambios**: ambos métodos ya
  usan `this.api.extractError(err, ...)` en su `catch`, que ya extrae correctamente un `detail`
  string de cualquier `HTTPException` — verificado, no requirió modificación.
- [X] T011 [P] [US1] Tests frontend: `pos-order-panel.component.spec.ts` — botón "Anular" ausente
  cuando el pedido seleccionado está `paid`; `pos-terminal.store.spec.ts` — `voidPersistedItem`
  maneja el `409` sin lanzar una excepción no controlada.

**Checkpoint**: User Story 1 funcional y probable de forma independiente — un pedido ya pagado
queda protegido contra anulación.

---

## Phase 4: User Story 2 - Ningún rol puede aplicar descuento manual (Priority: P1) 🎯 MVP (parte B)

**Goal**: Ninguna vía de la Terminal de Mesas —ni el atajo F4, ni una llamada directa a la API—
permite aplicar un descuento manual; el único descuento posible es el automático por promoción.

**Independent Test**: abrir la Terminal de Mesas sobre una mesa con una cuenta activa (probar con
un usuario Administrador también) y verificar que no existe ningún atajo, botón ni campo para
ingresar un descuento manual; invocar `POST /orders/{id}/checkout-and-send` con `discount > 0`
directamente y verificar el rechazo `422`.

No depende de la Fase 2 (Foundational) — es independiente de la señal `paid`.

### Backend for User Story 2

- [X] T012 [P] [US2] Cambiar `discount: Decimal = Field(0, ge=0, max_digits=12, decimal_places=2)`
  a `Field(0, ge=0, le=0, max_digits=12, decimal_places=2)` en `CheckoutAndSendIn`
  (`../pos-backend/app/api/v1/orders/schemas.py:265`) — cualquier valor distinto de `0` responde
  `422` desde la validación del propio esquema (FR-009, FR-010, FR-011; research.md D4). El campo
  compartido `discount` de `sales/schemas.py` (mostrador/cierre unificado/dividido, alcance de spec
  011) **no se toca**.
- [X] T013 [P] [US2] Test backend: `POST /orders/{id}/checkout-and-send` con `discount: 5000`
  responde `422` sin cobrar el pedido; con `discount: 0` u omitido, se comporta exactamente igual
  que antes — en `../pos-backend/app/characterization_tests/test_orders_checkout.py`.

### Frontend for User Story 2

- [X] T014 [P] [US2] Retirar el atajo de teclado `F4` (y su pista visual en la barra superior) de
  `../pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts:56,206-211`.
- [X] T015 [P] [US2] Retirar el botón "Aplicar descuento (F4)" y su popover completo de
  `../pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.ts:122,151-167`.
- [X] T016 [US2] Retirar las señales/métodos `discountPanelOpen`, `discountType`, `discountValue`,
  `discountReason`, `appliedDiscount`, `toggleDiscountPanel`, `setDiscountType`, `applyDiscount`,
  `cancelDiscount` de `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts:
  226-231,1181-1196`; quitar su uso en `totals()` (`:562-570`, el total deja de restar ningún
  descuento manual) y en el payload de `checkoutAndSend()` (`:1319-1327`, deja de enviar
  `discount` salvo `0`/omitido). Depende de T014, T015 (elimina las últimas referencias de UI antes
  de retirar el estado que las alimentaba).
- [X] T017 [P] [US2] Tests frontend: `pos-order-panel.component.spec.ts`/
  `pos-terminal.store.spec.ts` — confirmar que no existe ningún control de descuento manual, que
  `totals().discount` solo refleja promociones activas, y que presionar F4 no produce ningún
  efecto. Depende de T016.

**Checkpoint**: User Story 2 funcional de forma independiente. Junto con US1 y US3, el MVP completo
de esta spec queda cubierto.

---

## Phase 5: User Story 3 - La insignia "Listo" solo aparece cuando el pedido realmente ya se cobró (Priority: P1) 🎯 MVP (parte C)

**Goal**: La insignia "Listo" (listado de mesas y detalle del pedido) exige que la orden esté
pagada **y** que cocina haya terminado — nunca una condición sola.

**Independent Test**: sobre un pedido con cocina ya en `listo` pero sin pago confirmado, verificar
que el listado de mesas y el detalle muestran "Pago pendiente", no "Listo"; confirmar el pago y
verificar que la insignia cambia a "Listo" de inmediato.

Enteramente frontend — reutiliza la señal `paid` ya disponible desde la Fase 2. No depende de US1
ni US2.

- [X] T018 [US3] En `deriveTableStatus()`
  (`../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts:147-166`), la rama que
  hoy devuelve `'listo'` cuando todos los ítems vivos están `estado_cocina === 'listo'` pasa a
  exigir además que las órdenes relevantes tengan `paid === true`; si cocina ya terminó pero al
  menos una orden relevante no está `paid`, devolver `'pago_pendiente'` en su lugar (FR-012,
  FR-013, FR-014; research.md D2, T2 de data-model.md). Depende de T005.
- [X] T019 [P] [US3] En
  `../pos-heladeria/src/app/modules/tables/components/pos-order-panel.component.ts:28`, cambiar el
  texto de cabecera de dos ramas (`store.kitchenReady() ? 'listo para cobrar' : 'en preparación'`)
  a tres: "listo para cobrar" solo si además `store.selectedOrder()?.paid`; "pago pendiente" si
  cocina ya terminó pero no hay pago; "en preparación" en cualquier otro caso.
  `store.kitchenReady()` no cambia — sigue controlando, sin relación con el pago, cuándo se oculta
  el botón "Marcar pedido listo". Depende de T005.
- [X] T020 [P] [US3] Tests frontend: `pos-terminal.store.spec.ts`
  (`describe('deriveTableStatus', ...)`) — agregar los casos "cocina lista + pagado → listo" y
  "cocina lista + sin pagar → pago_pendiente"; test nuevo/actualizado en
  `pos-order-panel.component.spec.ts` para las tres ramas de texto de cabecera. Depende de T018,
  T019.

**Checkpoint**: User Stories 1, 2 y 3 (el MVP completo) funcionan de forma independiente y en
conjunto — los tres problemas de mayor riesgo reportados quedan corregidos.

---

## Phase 6: User Story 4 - Una sola acción para imprimir la factura ya emitida (Priority: P2)

**Goal**: Tras confirmar el pago, existe una única acción "Imprimir Factura" — ya no dos botones
("Imprimir factura" del diálogo de éxito y "Reimprimir Factura POS" de la barra lateral) para el
mismo documento de un solo comprobante.

**Independent Test**: cobrar un pedido de un solo comensal y verificar que el diálogo de éxito no
ofrece impresión para ese caso; verificar que la barra lateral ofrece exactamente una acción
"Imprimir Factura" que reimprime el mismo documento sin generar una nueva venta.

Enteramente frontend. No depende de las Fases 2-5.

- [X] T021 [US4] En el diálogo de éxito de
  `../pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts:144-176`, retirar el
  botón de impresión para el caso de un solo comprobante (`store.lastReceipts().length === 1`) —
  ese caso deja de imprimir desde el diálogo; los botones "Imprimir todos"/por comensal del caso de
  cuenta dividida (`length > 1`) **no cambian** (research.md D1, alternativa descartada de
  eliminarlos también).
- [X] T022 [P] [US4] Renombrar "🧾 Reimprimir Factura POS" a "🧾 Imprimir Factura" en
  `../pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.ts:159-164` —
  sigue invocando `store.printOrderInvoice(order.id)` sin cambios (FR-001, FR-002).
- [X] T023 [P] [US4] Tests frontend: actualizar el test T033 existente en
  `pos-checkout-panel.component.spec.ts` (o `pos-terminal.store.spec.ts`, según donde viva hoy) para
  reflejar la nueva etiqueta "Imprimir Factura"; test nuevo confirmando que el diálogo de éxito no
  muestra ningún botón de impresión cuando `lastReceipts().length === 1`, y que sí los conserva
  cuando `length > 1`.

**Checkpoint**: Las cuatro historias de usuario funcionan de forma independiente y en conjunto.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final contra el spec completo, no ligada a una sola historia.

- [ ] T024 [P] Ejecutar los 4 escenarios de [quickstart.md](./quickstart.md) de punta a punta contra
  ambos repos corriendo localmente, registrando el resultado de cada uno (incluye repetir los
  escenarios 1 y 2 con un usuario de rol Administrador, per la clarificación de la spec).
- [X] T025 [P] Verificar la suite completa de characterization tests en verde:
  `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` desde
  `../pos-backend` — ningún test `"""CONGELA comportamiento actual"""` distinto del
  explícitamente actualizado en T007 debe cambiar de resultado.
- [X] T026 [P] Verificar la suite completa de tests frontend en verde: `ng test` desde
  `../pos-heladeria`.
- [X] T027 Revisión manual: confirmar por diff que `transition_kitchen` y `mark_order_ready`
  (`../pos-backend/app/api/v1/orders/kitchen.py`) quedan exactamente sin cambios — es la garantía
  de que el guard de T006 no se filtró accidentalmente a esas dos funciones (Principio III de la
  Constitución).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — empieza de inmediato.
- **Foundational (Phase 2)**: depende de Setup — **bloquea** a US1 (Phase 3) y US3 (Phase 5). US2
  (Phase 4) y US4 (Phase 6) no dependen de esta fase y pueden avanzar en paralelo con ella.
- **User Story 1 (Phase 3)**: depende de Foundational (T002, T005). No depende de US2 ni US3.
- **User Story 2 (Phase 4)**: sin dependencias de fase — puede empezar en paralelo con Setup/
  Foundational.
- **User Story 3 (Phase 5)**: depende de Foundational (T005, y de la lectura de `order.paid` que
  T002/T003 exponen). No depende de US1 ni US2, aunque comparte archivo
  (`pos-terminal.store.ts`) con ambas — conviene secuenciar para minimizar conflictos de edición,
  sin que exista una dependencia funcional dura.
- **User Story 4 (Phase 6)**: sin dependencias de fase — puede empezar en paralelo con cualquier
  otra historia.
- **Polish (Phase 7)**: depende de todas las historias que se vayan a entregar en este incremento.

### Dentro de cada historia

- Los tests se agregan junto a su implementación correspondiente, citando el FR que verifican.
- Backend antes que frontend dentro de US1 (el guard de T006 debe existir antes de que el frontend
  dependa de su respuesta `409`) — no aplica a US2 (el frontend solo deja de enviar el campo, no
  depende del rechazo del backend para funcionar) ni a US3/US4 (enteramente frontend).

### Parallel Opportunities

- T002 y T005 (Foundational) son paralelos entre sí (backend vs. frontend, sin dependencia mutua);
  T003 depende de T002; T004 depende de T002.
- US1, US2 y US3 (las tres P1 del MVP) son paralelas entre sí una vez completada la Fase 2 (dos o
  tres personas distintas) — con la salvedad de coordinación de archivo entre US1/US3 en
  `pos-terminal.store.ts` mencionada arriba.
- US4 es paralela a cualquier otra historia en todo momento.
- Dentro de US1: T008 es paralelo a T009/T010 (archivos distintos); T010/T011 son paralelos entre
  sí.
- Dentro de US2: T012/T013 (backend) son paralelos a T014/T015 (frontend, archivos distintos);
  T016 depende de T014+T015; T017 depende de T016.
- Dentro de US3: T019 es paralelo a T018 (archivos distintos); T020 depende de ambos.
- Dentro de US4: T022 es paralelo a T021 (archivos distintos); T023 depende de ambos.

---

## Parallel Example: Foundational + User Story 2 en simultáneo

```bash
# Mientras una persona completa la Fase 2 (T002-T005):
Task: "Retirar atajo F4 de table-sessions.component.ts (US2, T014)"
Task: "Retirar botón/popover de descuento de pos-order-panel.component.ts (US2, T015)"
Task: "Endurecer CheckoutAndSendIn.discount a le=0 en orders/schemas.py (US2, T012)"
```

---

## Implementation Strategy

### MVP First (User Stories 1 + 2 + 3)

Las tres son P1 y juntas forman el MVP real de esta spec — cada una corrige, por sí sola, uno de
los tres problemas de mayor riesgo (pérdida de inventario/mercancía sin control, descuento no
autorizado, insignia que induce a error sobre si falta cobrar):

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (bloquea US1 y US3; US2 puede avanzar en paralelo desde ya).
3. Completar Fase 3 (US1), Fase 4 (US2) y Fase 5 (US3) — en paralelo si hay varias personas.
4. **STOP and VALIDATE**: correr los escenarios 1, 2 y 3 de [quickstart.md](./quickstart.md).
5. Desplegar/demostrar el MVP.

### Incremental Delivery

1. Setup + Foundational → base lista para US1/US3 (US2 ya pudo empezar antes).
2. US1 → probar de forma independiente → corrige el riesgo de anulación indebida.
3. US2 → probar de forma independiente → cierra la vía de descuento no autorizado.
4. US3 → probar de forma independiente → corrige la insignia engañosa (el defecto de la captura
   original).
5. US4 → probar de forma independiente → limpieza de UI post-pago.
6. Polish → verificación final contra el spec completo.

### Parallel Team Strategy

Con más de una persona disponible:

1. Persona A: Fase 2 (Foundational).
2. En paralelo desde el inicio: Persona B toma User Story 2 (no depende de Foundational).
3. Una vez lista la Fase 2: Persona A toma User Story 1, Persona C toma User Story 3 (coordinando
   con Persona A sobre `pos-terminal.store.ts`).
4. Cualquier persona libre toma User Story 4 en cualquier momento (sin dependencias).

---

## Notes

- [P] tasks = distintos archivos, sin dependencias entre sí.
- [Story] label mapea cada tarea a su historia de usuario para trazabilidad (Principio XII de la
  Constitución).
- El campo `paid` (T002-T005) es la única pieza de infraestructura nueva de esta spec — cero
  endpoints nuevos, cero cambios de esquema de base de datos (ver Constitution Check en
  [plan.md](./plan.md)).
- `test_transition_kitchen_y_void_item_no_validan_status_de_la_orden_a16` (T007) es el único
  characterization test que esta spec autoriza a cambiar — cualquier otro test
  `"""CONGELA comportamiento actual"""` en rojo tras estos cambios es una regresión no autorizada,
  no una señal para ajustarlo.
- Commit después de cada tarea o grupo lógico.
- Detenerse en cualquier checkpoint para validar una historia de forma independiente.
- Evitar: tareas vagas, conflictos de archivo simultáneos, dependencias cruzadas entre historias
  que rompan su independencia.

---

## Notas de implementación (2026-08-21)

Ejecutado directamente (sin agentes delegados) sobre las ramas `029-correccion-cobro-cierre-mesa`
creadas desde `develop` en ambos repos hermanos, sin commits (working tree para revisión). T001-T027
completadas; T024 queda **sin marcar** deliberadamente por el mismo motivo que spec 028: este repo
(`pos-specs`) no puede levantar los servidores reales de `pos-backend`/`pos-heladeria` — la
cobertura de esta iteración es por tests automatizados (backend `unittest`, frontend Vitest) y
revisión de código/diff, no por click-through manual en navegador. Se recomienda ejecutar
quickstart.md manualmente antes de mergear.

**Verificación final**:
- **Backend**: 270/270 tests (`python -m unittest discover -s app/characterization_tests`), frente
  a 262/262 de línea base — 8 tests nuevos (T004 ×4, T008 ×1 nuevo + T007 divide 1 en 2, T013 ×2),
  ninguno de los characterization tests preexistentes cambió de resultado salvo el explícitamente
  autorizado (T007). Confirmado por diff (T027) que `transition_kitchen`/`mark_order_ready`
  (`kitchen.py`) quedan exactamente sin cambios — el guard nuevo vive solo dentro de `void_item`.
- **Frontend**: 314/317 tests (`ng test`) frente a 299/302 de línea base — mismas 3 fallas
  preexistentes sin relación (`app.spec.ts`, `auth.service.spec.ts`,
  `payment-method.service.spec.ts`), verificadas también como línea base antes de tocar código.

**Hallazgo durante la implementación, no anticipado en el plan**: `TableSessionsComponent` declara
`providers: [PosTerminalStore]` en su propio `@Component` (instancia de store aislada por
componente, no la del `TestBed`) — los tests que necesitan `fixture.detectChanges()` sobre ese
componente deben tomar el store desde `fixture.componentInstance.store`, no desde
`TestBed.inject(PosTerminalStore)`, o el mock de `store.init()` queda sobre una instancia distinta a
la que usa el componente (el síntoma es la pantalla de carga renderizando "Cargando terminal…" para
siempre, con 0 elementos del contenido real — un falso positivo si el test solo comprueba ausencia
de algo). Documentado en el spec de `table-sessions.component.spec.ts` para la próxima vez que haga
falta testear ese componente con `detectChanges()`.

**Decisión técnica no anticipada en research.md, tomada durante T003**: el campo `paid` no podía
resolverse con una propiedad computada en el modelo ORM (dispararía una consulta N+1 por pedido al
serializar `list_orders`, el endpoint que sondea la terminal). Se optó por dos funciones en
`orders/service.py` (`order_has_sale` para un solo id, `paid_order_ids` en bloque para una lista) y
asignar `.paid` dinámicamente sobre las instancias de `CustomerOrder` ya cargadas, antes de pasarlas
al adaptador de Pydantic (`from_attributes=True` lee el atributo dinámico sin problema). `_load_order`
(compartida por ~10 endpoints de mutación) también quedó actualizada para que **todo** consumidor de
`OrderResponse` reciba `paid` correctamente, no solo `GET /orders/{id}`.

## Hotfix post-implementación (2026-08-21, mismo día): mesa seguía "Listo" tras cobrarse y liberarse

Al probar en vivo contra el frontend corriendo con los cambios de arriba, el usuario reportó que
una mesa cuyo pedido ya se había cobrado seguía mostrando "Listo"/"listo para cobrar", con el
producto y el subtotal todavía visibles en la tarjeta de la mesa, y el panel lateral seguía
bloqueando con "No se puede cobrar esta mesa — Tiene pedidos sin cobrar, pero su sesión está
cerrada."

Investigado leyendo directamente el código de ambos repos (sin acceso a una BD real con datos —
el entorno de desarrollo no tenía tablas migradas). Causa raíz, un hueco **anterior a spec 029**,
no introducido por ella: `tableOrders()` (`pos-heladeria/.../pos-terminal.store.ts`, entonces
líneas 351-356) filtraba únicamente por `status !== 'pagada' && status !== 'cancelada'` — nunca
por `paid`. Como ya documenta research.md de esta misma spec (D2), un pedido QR/mostrador **nunca**
llega a `status === 'pagada'` (se queda en `'abierta'` a propósito, con la `Sale` ya emitida), así
que ese filtro nunca excluía nada para el camino vigente. El backend, en cambio, sí distingue esto
correctamente desde un hotfix de spec 028 (`has_billable_orders`/`_billable_orders`,
`table_sessions/service.py`, excluyen cualquier pedido con `Sale` asociada sin importar `status`) —
por eso, una vez el pedido se pagó y no quedaron comensales, el backend ya había liberado la mesa
por su cuenta (`try_release_if_empty`: sesión cerrada, `DiningTable.status = 'libre'`) mientras el
frontend seguía tratando ese mismo pedido, indefinidamente, como consumo activo.

Corregido en `pos-heladeria` (único repo afectado, sin cambios de backend):
- Nueva función pura `dropResolvedOrders(orders, tableStatus)` en `pos-terminal.store.ts`: si la
  mesa ya está `'libre'` en el backend, descarta los pedidos que además ya estén `paid` — un pedido
  sin pagar se conserva (es exactamente el caso huérfano que el aviso existe para señalar); si la
  mesa sigue `'ocupada'`, no filtra nada (un pedido pagado con cocina aún en curso debe seguir
  viéndose, spec 026/028).
- Aplicada en los tres consumidores de `tableOrders()` que alimentan vistas de mesa: la insignia y
  el conteo/subtotal de la tarjeta de mesa (`tablesView`), y la decisión de mostrar el panel vacío
  "mesa-libre" vs. "Pedido de la mesa" (`centralState`).
- `billOrphan` (el aviso "sesión está cerrada", en `loadSessionBill`) corregido para contar solo
  pedidos genuinamente sin pagar (`!o.paid`), en vez de cualquier pedido no terminal.

Tests nuevos en `pos-terminal.store.spec.ts`: `dropResolvedOrders` (mesa libre + pagado → excluido;
mesa libre + sin pagar → se conserva; mesa ocupada + pagado → se conserva; combinado con
`deriveTableStatus` confirma que una mesa ya liberada por el backend muestra "Libre", no "Listo") y
`loadSessionBill`/`billOrphan` (sin sesión + pagado → no marca huérfano; sin sesión + sin pagar →
sí marca huérfano).

Verificado: backend sin cambios, sigue 270/270. Frontend 320/323 (`ng test`), mismas 3 fallas
preexistentes sin relación verificadas antes de este hotfix.

## Hotfix post-implementación #2 (2026-08-21, mismo día): pedidos de una visita anterior reaparecían al reabrir la mesa por QR

Al probar el hotfix #1 en vivo, el usuario reportó un problema relacionado pero más profundo:
liberó la Mesa 1, volvió a escanear su QR para un pedido nuevo, y al confirmarlo la Terminal de
Mesas le mostró **3 pestañas de comensal**, 6 productos y $24.000 — mezclando visiblemente los
pedidos de la visita anterior (ya cobrada y cerrada) con el pedido nuevo.

Investigado con dos agentes de exploración en paralelo (uno por repo) más lectura directa del
código. Causa raíz, **anterior a spec 029** y más amplia que el hotfix #1: `GET /orders`
(`list_orders`, el único endpoint que alimenta toda la Terminal de Mesas) no filtraba por
`table_session_id` ni por `dining_table_id` — devolvía **todos los pedidos de todas las mesas de
toda la historia del tenant**, dejando que el frontend indexara todo únicamente por
`dining_table_id` (`activeOrders`/`ordersOfTable`/`tableOrders`, `pos-terminal.store.ts`). Como un
pedido QR/mostrador pagado nunca llega a `status === 'pagada'` (research.md D2 de esta spec) y
`CustomerOrder.table_session_id` no cambia después de creado, un pedido de cualquier visita
anterior a esa mesa física quedaba viviendo ahí para siempre y reaparecía en cuanto la mesa volvía
a ocuparse — el hotfix #1 (`dropResolvedOrders`) solo actuaba mientras la mesa seguía `libre`, así
que no alcanzaba a cubrir este caso.

Confirmado también que el backend crea una `TableSession` genuinamente **nueva** (UUID distinto)
al reabrir por QR tras un `release_paid_session` completo (`cart/service.py::
_get_or_create_table_session`, filtra solo `status == "active"`), y que ya existía el patrón de
filtro correcto — solo que no expuesto por `list_orders` — en `has_billable_orders`/
`_billable_orders` (`table_sessions/service.py`), que sí acotan por `table_session_id`.

Se presentaron dos opciones al usuario (acotar en el backend vs. parche solo en frontend); eligió
acotar en el backend, además porque corrige de raíz que `GET /orders` cargaba el historial
completo del tenant sin límite en cada sondeo.

Corregido:
- **Backend**: nueva función `service.list_orders(db, status_filter=None,
  active_sessions_only=False)` en `orders/service.py` (mismo patrón de subconsulta que
  `has_billable_orders`, expuesto aquí) — con `active_sessions_only=True`, excluye un pedido ya
  pagado cuya `TableSession` ya no está `active`; conserva uno **sin pagar** de una sesión cerrada
  (caso huérfano real que `billOrphan` existe para señalar, no algo que ocultar) y cualquier pedido
  sin `table_session_id` (mostrador puro). `router.py::list_orders` ahora delega en esa función y
  gana el parámetro `active_sessions_only: bool = Query(False)` — por defecto `False` conserva el
  comportamiento actual exacto.
- **Frontend**: `DiningSessionService.listOrders` gana un segundo parámetro opcional
  `activeSessionsOnly`; `PosTerminalStore.reloadOrders()` lo manda en `true`. El hotfix #1
  (`dropResolvedOrders` y sus 3 call sites en `tablesView`/`centralState`) quedó **revertido** —
  redundante una vez `this.orders()` nunca puede traer un pedido pagado de una sesión cerrada;
  `billOrphan` (`!o.paid`) no se tocó, sigue haciendo falta para el caso huérfano.

Tests nuevos: backend, `TestListOrdersActiveSessionsOnly` en `test_orders_service.py` (excluye
pagado de sesión cerrada; conserva sin pagar de sesión cerrada; conserva pagado de sesión activa;
conserva sin `table_session_id`; sin el parámetro no cambia nada). Frontend,
`dining-session.service.spec.ts` (pide `active_sessions_only=true` cuando se solicita; no lo manda
si no se pide) — se retiró el describe de `dropResolvedOrders` de `pos-terminal.store.spec.ts` por
quedar sin código que probar.

Verificado: backend 275/275 (`python -m unittest discover`). Frontend 318/321 (`ng test`), mismas
3 fallas preexistentes sin relación.

## Hotfix post-implementación #3 (2026-08-21, mismo día): un pedido de mesero ya `abierta` no tenía ninguna vía de cobro

El usuario reportó: al marcar los ítems de un pedido como "Listo", el pedido se quedaba en
`abierta` y no se generaba la factura de venta. La primera hipótesis —que "marcar listo" debería
disparar el cobro automáticamente— se descartó investigando a fondo (dos agentes en paralelo +
lectura directa de código y specs): **ningún spec pide esa automatización**; spec 009 documenta
que el negocio la consideró explícitamente y la rechazó ("es prácticamente imposible que
coincidan", cerrado como riesgo aceptado, no como requisito a construir). Al preguntarle, el
usuario aclaró el reporte real: "al confirmar el pago de la orden debe registrarse la venta... si
la orden tiene un ítem esta no debe quedar en estado abierta" — es decir, **intentó cobrar
explícitamente y el cobro no funcionó**. Eso sí era un bug real.

Causa raíz (`pos-heladeria`, sin cambios de backend): `saveOrder()`
(`pos-terminal.store.ts`, se dispara al agregar productos a una mesa que ya tiene consumo, a
diferencia de "+Crear Orden Manual") deja la orden en `'abierta'` directamente, sin pasar nunca
por `'recibida'`. `getSidebarMode(order)` (`dining.interface.ts`) decide el modo de la barra
lateral mirando solo el **canal** de la orden, nunca su `status` — así que esa orden en `'abierta'`
caía igual en modo `'terminal-pos'`, cuyo único botón de cobro ("Cobrar, Facturar y Enviar a
Cocina" → `checkoutAndSend()`) el backend rechaza con **409** si la orden no está en `'recibida'`.
La venta nunca se registraba; la orden quedaba `'abierta'` para siempre.

No había ninguna otra vía de cobro disponible en la UI para este caso. El mecanismo que sí sabe
cobrar una orden ya `'abierta'` —cerrar la **sesión de mesa** completa (`close_session`, ya
implementado y probado— vive en `<app-session-bill-panel>`, pero quedó huérfano tras el rediseño
de spec 028: su único uso en toda la app la instancia siempre en `[readOnly]="true"` (solo para el
resumen de pedidos QR); su modo de cobro real (botón "Cobrar y cerrar mesa") nunca se usaba. Su
`@Input() beforeCharge`, pensado para conectar `ensureReadyToCharge()` (auto-marcar-listo-y-cobrar
en un solo paso, ya implementada y comentada para ese propósito exacto en `pos-terminal.store.ts`),
tampoco se conectaba en ningún lado — la función existía pero nadie la llamaba.

Corregido en `pos-checkout-panel.component.ts`: dentro del modo `terminal-pos`, se ramifica ahora
por el `status` de la orden seleccionada (nuevo computed `showSessionCharge`), no solo por canal —
`status === 'recibida'` sigue el flujo actual (`checkoutAndSend`); cualquier otro status muestra en
su lugar `<app-session-bill-panel [readOnly]="false" [beforeCharge]="store.ensureReadyToCharge"
(charged)="store.onCharged($event)">`, reutilizando el mecanismo de cobro por sesión ya probado, con
el guard de "faltan ítems por marcar" conectado por fin. No se tocó el backend — `close_session`,
`_assert_closable`, `ensureReadyToCharge` y `session-bill-panel.component.ts` ya existían y hacían
exactamente lo que faltaba; el bug era puramente de conexión en el frontend.

Tests nuevos: `pos-checkout-panel.component.spec.ts` (orden `'abierta'` muestra "Cobrar y cerrar
mesa", no "Cobrar, Facturar y Enviar a Cocina"; al cobrar llama primero a `ensureReadyToCharge`
—`beforeCharge` conectado— y solo si resuelve `true` cierra la sesión; si resuelve `false` no
cierra nada) y `pos-terminal.store.spec.ts` (`ensureReadyToCharge` en sí, sin ningún test previo:
todos los ítems ya listos resuelve `true` de inmediato sin preguntar; con ítems sin marcar
pregunta y, si se confirma, los marca listos y recarga; si se cancela resuelve `false` sin marcar
nada).

Verificado: backend sin cambios, sigue 275/275. Frontend 324/327 (`ng test`), mismas 3 fallas
preexistentes sin relación.

## Hotfix post-implementación #4 (2026-08-21, mismo día): sin acción de "rechazar" al confirmar el cobro

El usuario reportó: "la orden sigue quedando abierta cuando se confirma el pago, debería quedar
pagada o rechazada de acuerdo a la acción que elija el cajero, si queda rechazada no se debe
registrar una venta ni tampoco movimiento en la caja." Tomado literalmente ("la orden debe pasar a
`status = 'pagada'` al confirmarse el pago") esto habría contradicho el diseño ya verificado varias
veces en esta misma spec: los caminos QR y mostrador dejan la orden en `'abierta'` a propósito tras
el pago —nunca pasan a `'pagada'`— precisamente porque `tableOrders()`/`activeOrders()` excluyen
`'pagada'` de "consumo activo" (ver hotfix #2); "pagado" se rastrea por la existencia de una `Sale`
(research.md D2), no por `status`.

Investigado con el usuario antes de tocar nada (`AskUserQuestion`): existía ya en el backend
exactamente el mecanismo que describe como "rechazada" —`checkout.cancel_order`— que cancela un
pedido en cualquier estado no terminal, ajusta inventario según lo que cocina ya alcanzó a
consumir, y **no toca `Sale` ni `CashShift` para nada**. Ya estaba expuesto en el frontend como
`DiningSessionService.cancelOrder(orderId, motivo)` → `POST /orders/{id}/cancel`, con test de
contrato propio. Pero era código huérfano: nadie lo llamaba desde la Terminal de Mesas (el único
`cancelOrder` en uso era uno distinto, del lado del comensal). El usuario confirmó que la lectura
correcta era agregar un botón "Rechazar pedido" junto a la acción de cobro.

Investigando `cancel_order` para conectarlo apareció un gap de consistencia propio: solo rechazaba
con 409 si `order.status` ya era terminal (`"pagada"`/`"cancelada"`), pero como el pago no mueve
`status` a `'pagada'` en los caminos QR/mostrador, nada le impedía cancelar un pedido que YA se
había cobrado, dejando su `Sale` huérfana — el mismo patrón de inconsistencia que el hotfix #1 ya
había cerrado para `void_item` con la misma guardia.

Corregido:
- **Backend** (`checkout.py::cancel_order`): nueva guardia, igual que la de `void_item` (hotfix
  #1) — si `order_has_sale(db, order_id)` es `True`, 409 "El pedido ya fue pagado y no puede
  rechazarse", antes de tocar inventario o marcar `'cancelada'`.
- **Frontend** (`pos-terminal.store.ts`): nuevo método `rejectOrder()`, mismo patrón de
  confirmación con motivo fijo que `voidPersistedItem`/`voidPersistedCombo` (`this.confirm.ask(...)`
  → `api.cancelOrder(id, 'Rechazado desde terminal')` → deselecciona y recarga).
- **Frontend** (`pos-checkout-panel.component.ts`): botón "Rechazar pedido" en los dos lugares
  donde antes solo existía "Cobrar" para un pedido individual —la rama `terminal-pos` (mostrador,
  junto a "Cobrar, Facturar y Enviar a Cocina") y la rama `showSessionCharge` (pedido de mesero ya
  en cocina, hotfix #3, junto a `<app-session-bill-panel>`). La rama `'resumen'` (QR de solo
  lectura) no se tocó: esos pedidos ya están pagados por diseño y la guardia nueva los rechazaría
  de todas formas.

Tests nuevos: `test_orders_checkout.py::test_cancel_order_409_si_orden_ya_tiene_sale_spec_029`
(backend); `pos-terminal.store.spec.ts` (`describe('PosTerminalStore.rejectOrder', ...)`: sin
pedido seleccionado no hace nada, si el cajero cancela el aviso no llama al backend, con
confirmación cancela con motivo fijo, y el 409 del backend se muestra con su propio mensaje);
`pos-checkout-panel.component.spec.ts` (el botón aparece y funciona en ambas ramas).

Verificado: backend 276/276 (`python -m unittest discover -s app/characterization_tests -p
'test_*.py'`). Frontend 330/333 (`ng test`), mismas 3 fallas preexistentes sin relación.

## Hotfix post-implementación #5 (2026-08-21, mismo día): "Órdenes" (Panel de Control) mostraba "Abierta" para pedidos ya cobrados

El usuario mostró dos capturas: en la Terminal de Mesas, Mesa 1 aparecía correctamente como
"Listo" con "Pedido pagado por el comensal desde el QR — nada que cobrar aquí" (mensaje que solo
se muestra con `paid = true`). Pero en **"Órdenes"** (`Panel de Control → Órdenes`,
`src/app/modules/orders/`, un módulo aparte de la Terminal de Mesas que ninguno de los hotfixes
#1-#4 tocó), el mismo pedido aparecía con badge "Abierta". Pidió que pasara a "Pagada".

Igual que en los hotfixes anteriores, tomar el reporte literalmente ("cambiar `status` a
`'pagada'`") habría reintroducido la regresión ya corregida en el hotfix #2: los caminos QR y
mostrador dejan `status = 'abierta'` a propósito tras el pago, porque `tableOrders()`/
`activeOrders()` de la Terminal de Mesas excluyen `'pagada'` de "consumo activo" (research.md D2).
La señal correcta de "ya pagado" es el campo `paid` (computado vía `order_has_sale`, ya presente
en `OrderResponse` y en `DiningOrder` desde la implementación original de esta spec) — la Terminal
de Mesas ya lo usa (`deriveTableStatus`, `billOrphan`); "Órdenes" simplemente nunca lo adoptó.

Causa raíz confirmada (sin cambios de backend, `paid` ya llegaba correcto en ambas respuestas que
usa este módulo): `orders-page.component.ts` filtraba las pestañas (`visibleOrders`) y pintaba el
badge de cada tarjeta comparando `order.status` crudo contra `DiningOrderStatus`; lo mismo hacía
el badge de `order-detail.component.ts`. Ninguno de los dos leía `order.paid`.

Corregido: nueva función pura `displayOrderStatus(order)` en `order-status.util.ts` (mismo
espíritu que `deriveTableStatus` de la Terminal de Mesas) — si `paid` es `true` y el pedido no está
`'cancelada'`, se muestra como `'pagada'` aunque `status` siga en `'abierta'`/`'bloqueada'`. Se usa
en el filtro de pestañas y el badge de `orders-page.component.ts`, y en el badge de
`order-detail.component.ts`. No se tocó el backend ni la Terminal de Mesas.

Tests nuevos: `order-status.util.spec.ts` (nuevo, el módulo no tenía ningún test antes de este
hotfix) — casos de `displayOrderStatus`; `orders-page.component.spec.ts` (nuevo) — un pedido
`abierta` con `paid: true` se ve "Pagada" y cae en la pestaña "Pagadas", uno genuinamente sin
pagar sigue "Abierta"; `order-detail.component.spec.ts` (nuevo) — mismo caso para el badge del
detalle.

Verificado: backend sin cambios, sigue 276/276. Frontend 337/340 (`ng test`), mismas 3 fallas
preexistentes sin relación.

## Hotfix post-implementación #6 (2026-08-21, mismo día): un pedido de mostrador no se auto-seleccionaba al reabrir la mesa

El usuario creó un pedido de mostrador (2 productos, $7.500) en Mesa 3, recargó la página y
seleccionó la mesa: la tarjeta de la izquierda se veía bien (badge "Pago pendiente", 2 productos,
$7.500), pero el panel central "Pedido de la mesa" decía "Pedido nuevo sin guardar" / "Aún no hay
productos en este pedido", y el de cobro "Pedido de mostrador" / "Agrega productos..." — como si no
hubiera ningún pedido, aunque existía uno persistido con productos.

No era un problema de orden de carga en la recarga (no hay ruta/`localStorage` que restaure la mesa
seleccionada) sino un bug de filtrado por `status` en `activeOrders`
(`pos-terminal.store.ts:333-338`, de la que depende `ordersOfTable()` → `selectTable()` para
auto-seleccionar el pedido de una mesa): excluía **todo** pedido `'recibida'` sin mirar el canal.
Un pedido de mostrador creado con `hold_for_payment: true` vive en `'recibida'` mientras el cajero
lo arma —nunca pasa por `confirmOrder()`, solo por `checkout-and-send` al cobrarlo— así que
`ordersOfTable()` salía vacía para esa mesa y `selectTable()` ponía `selectedOrderId` en `null`,
aunque el pedido existiera. `pendingOrders` (unas líneas arriba, línea 329-331) ya distinguía
correctamente `recibida && channel === 'qr'`; `activeOrders` nunca adoptó el mismo criterio.
`tableOrders()` (usada para la tarjeta/badge de la mesa) sí incluye `'recibida'` sin filtrar por
canal, por eso la tarjeta se veía bien mientras el panel de pedido no.

No es específico de la recarga: `createManualOrderFromDraft()` selecciona el pedido recién creado
directo en memoria (`this.selectedOrderId.set(order.id)`), sin pasar por `ordersOfTable()` — por
eso "funciona" justo después de crearlo. Cualquier `selectTable()` posterior (recarga, o simplemente
clic a otra mesa y volver) vuelve a pasar por el filtro roto.

Corregido en `activeOrders`: excluir `'recibida'` solo cuando `channel === 'qr'`, igual que
`pendingOrders`. Único consumidor de `activeOrders` es `ordersOfTable()` (confirmado por grep), así
que el cambio queda acotado a la selección de pedido por mesa, sin tocar `pendingOrders`,
`tableOrders()`/`deriveTableStatus` (ya correctos) ni el resto de la terminal.

Tests nuevos: `pos-terminal.store.spec.ts` (`describe('PosTerminalStore.selectTable', ...)`, no
existía ningún test de `selectTable()`/`ordersOfTable()` antes de este hotfix) — un pedido de
mostrador `'recibida'` sí se auto-selecciona; un pedido QR `'recibida'` (por confirmar) sigue sin
auto-seleccionarse (comportamiento existente preservado); una mesa sin pedidos no selecciona nada.

Verificado: backend sin cambios. Frontend 340/343 (`ng test`), mismas 3 fallas preexistentes sin
relación.

## Hotfix post-implementación #7 (2026-08-21, mismo día): pedido de mostrador ya cobrado seguía editable (dividir cuenta, método de pago, "Rechazar pedido")

El usuario mostró Mesa 3: un pedido creado manualmente, ya cobrado y con sus ítems "Listo" (badge
de mesa "Listo", que solo aparece con todos los pedidos de la mesa en `paid === true`). Pero el
panel de cobro seguía mostrando "Dividir la cuenta entre varias personas", el selector "Cuenta
única/Dividir por comensal", "Método de pago" y (desde el hotfix #4) "Rechazar pedido" — como si
siguiera pendiente de cobro. Pidió que se comportara igual que un pedido QR ya validado: solo
lectura, sin esas acciones.

Causa raíz: `getSidebarMode()` (`dining.interface.ts:220-222`), que decide entre el panel de solo
lectura (`'resumen'`) y el editable (`'terminal-pos'`), miraba únicamente el **canal** del pedido:

```ts
export function getSidebarMode(order) {
  return order?.channel === 'qr' ? 'resumen' : 'terminal-pos';
}
```

Nunca miraba `order.paid` —la señal real de "ya cobrado" en el resto de esta spec (`deriveTableStatus`,
`billOrphan`, `displayOrderStatus` del hotfix #5)—, así que un pedido de mostrador se quedaba en
`'terminal-pos'` para siempre, incluso ya pagado: `status` nunca llega a `'pagada'` en los caminos
QR/mostrador (research.md D2), así que no había ninguna otra señal que lo sacara de ese modo. Con
`showSessionCharge()` (hotfix #3) también en `true` para un pedido ya enviado a cocina, se
renderizaba `<app-session-bill-panel [readOnly]="false">` completo —selector de cuenta/método,
"Cobrar y cerrar mesa" y "Rechazar pedido" (hotfix #4)— en vez de la vista de solo lectura que ya
existe y que un pedido QR pagado sí usa correctamente ("Pedido pagado... nada que cobrar aquí").

Corregido: `getSidebarMode` ahora devuelve `'resumen'` cuando `order.paid` es `true`, sin importar
el canal, antes de mirar `channel === 'qr'`. Único consumidor de `getSidebarMode`
(`pos-checkout-panel.component.ts:249`), confirmado por grep — el cambio queda acotado a esa
decisión; no se tocó `session-bill-panel.component.ts` (su rama `readOnly` ya era correcta) ni el
backend.

Tests nuevos: `dining.interface.spec.ts` (nuevo, el archivo no tenía ningún test) — pedido de
mostrador pagado → `'resumen'`; pedido de mostrador sin pagar → `'terminal-pos'` (sin cambios);
pedido QR sin pagar → `'resumen'` (sin cambios); sin pedido → `'terminal-pos'`.
`pos-checkout-panel.component.spec.ts` — un pedido `waiter` con `paid: true` no muestra "Dividir
la cuenta", "Cuenta única" ni "Método de pago", no ofrece "Rechazar pedido", y sí muestra el
mensaje de solo lectura.

Verificado: backend sin cambios. Frontend 345/348 (`ng test`), mismas 3 fallas preexistentes sin
relación.
