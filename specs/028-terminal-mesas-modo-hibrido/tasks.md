---

description: "Task list template for feature implementation"
---

# Tasks: Rediseño Híbrido de la Terminal de Mesas — Validación QR y Cobro Manual

**Input**: Design documents from `/specs/028-terminal-mesas-modo-hibrido/`

**Prerequisites**: [plan.md](./plan.md) (required), [spec.md](./spec.md) (required for user stories),
[research.md](./research.md), [data-model.md](./data-model.md), [contracts/api-contracts.md](./contracts/api-contracts.md)

**Tests**: Este proyecto ya tiene una convención de tests establecida (characterization tests en
backend, specs Vitest co-ubicados en frontend) y la Constitución exige verificación obligatoria
(Principio X) antes de dar una spec por completa — por eso cada historia incluye tareas de test,
sin ser TDD estricto "rojo antes que verde": se agregan junto a la implementación, citando el
spec/FR que verifican, igual que ya hacen los tests existentes con `RN-MESA-*`.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing
of each story. Ambas historias P1 (US1, US2) son necesarias para un MVP completo, porque el spec
describe un modelo híbrido: sin US2, la Terminal de Mesas seguiría sin soportar clientes sin QR; sin
US1, el bug reportado seguiría vivo.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4)
- Include exact file paths in descriptions. Rutas relativas a cada repo hermano de `pos-specs`:
  `../pos-backend/` y `../pos-heladeria/` (ver [plan.md](./plan.md), sección Project Structure).

## Path Conventions

- **Backend** (`../pos-backend/`): `app/api/v1/<módulo>/{service,router,schemas}.py`,
  `app/characterization_tests/test_*.py`.
- **Frontend** (`../pos-heladeria/`): `src/app/modules/tables/{pages,components,services,interfaces}/`,
  specs co-ubicados `*.component.spec.ts` / `*.service.spec.ts`.

---

## Phase 1: Setup

**Purpose**: Confirmar la línea base antes de tocar código.

- [X] T001 Verificar la rama `028-terminal-mesas-modo-hibrido` en ambos repos hermanos
  (`../pos-backend`, `../pos-heladeria`) y confirmar que la suite base pasa antes de empezar:
  backend `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` (desde
  `../pos-backend`), frontend `ng test` (desde `../pos-heladeria`).

**Checkpoint**: Línea base verde conocida antes de cualquier cambio.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Infraestructura de UI compartida que las historias P1 (US1 y US2) necesitan para
existir como contextos distintos — antes de esto, la pantalla solo tiene el layout viejo de dos
pestañas y una barra lateral de un solo modo.

**⚠️ CRITICAL**: Ninguna tarea de historia de usuario empieza antes de completar esta fase.

- [X] T002 Agregar el tipo `SidebarMode` (`'resumen' | 'terminal-pos'`) y la función
  `getSidebarMode(order: DiningOrder | null): SidebarMode` (deriva el modo de `order.channel`:
  `'qr'` → `'resumen'`, `'counter' | 'waiter'` → `'terminal-pos'`) en
  `../pos-heladeria/src/app/modules/tables/interfaces/dining.interface.ts` (spec FR-005; D7 de
  [research.md](./research.md)).
- [X] T003 [P] Refactorizar el área central de
  `../pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts` para ramificar por
  estado de la mesa/sesión (pago QR pendiente de validar / mesa libre / orden manual en
  construcción / ya enviada) en vez de la señal `centerTab` de dos pestañas — sin implementar
  todavía el contenido de cada rama (eso lo hacen US1/US2). Depende de T002.
- [X] T004 [P] Refactorizar
  `../pos-heladeria/src/app/modules/tables/components/pos-checkout-panel.component.ts` para
  renderizar en dos ramas exclusivas según `getSidebarMode(order)` (`resumen` vs. `terminal-pos`)
  en vez de renderizar siempre `<app-session-bill-panel>` — sin implementar todavía el contenido
  de cada rama. Depende de T002.

**Checkpoint**: La pantalla tiene los dos contextos (bloque central, barra lateral) listos para que
US1 y US2 llenen cada uno el suyo, en paralelo si hay más de una persona.

---

## Phase 3: User Story 1 - El cajero valida un pago QR desde un único bloque, sin botones que fallen (Priority: P1) 🎯 MVP (parte A)

**Goal**: Un único bloque "Validación de Pago Requerida" reemplaza las dos pestañas actuales; la
barra lateral en modo "Resumen de Cuenta" nunca ofrece el botón "Cobrar y cerrar mesa" que hoy
falla sobre órdenes QR.

**Independent Test**: abrir una mesa con un pedido QR pendiente de validación, confirmar que solo
existe un bloque de validación (no pestañas separadas), que la barra lateral no ofrece ningún
selector de método de pago ni botón de cobro para esa orden, y que aprobar el comprobante envía el
pedido a cocina y genera la factura sin ningún error.

Todos los endpoints que usa esta historia (`GET .../payment-attempts`,
`POST .../approve|reject|confirm-cash`) ya existen y no cambian (ver contracts/api-contracts.md,
sección "Endpoints reutilizados sin cambios") — esta historia es enteramente frontend.

### Implementation for User Story 1

- [X] T005 [US1] Crear
  `../pos-heladeria/src/app/modules/tables/components/payment-validation-block.component.ts` (+
  plantilla), que renderiza una tarjeta `<app-payment-attempt-review-panel>` independiente por cada
  intento de pago `pendiente` de la mesa (uno por comensal QR) — reemplaza el contenido combinado
  de las dos pestañas viejas (FR-001, FR-002; acceptance scenario 6: confirmar una tarjeta no debe
  afectar a las demás).
- [X] T006 [US1] Conectar `table-sessions.component.ts` para renderizar
  `<app-payment-validation-block>` en la rama "pago QR pendiente" del refactor de T003. Depende de
  T005, T003.
- [X] T007 [P] [US1] Retirar
  `../pos-heladeria/src/app/modules/tables/components/pending-orders-panel.component.ts` y su
  wiring de "Rechazar" a nivel de orden completa (`cancelOrder`) — queda superado por T005/D5 de
  research.md (el único "Rechazar" válido es el del intento de pago, con motivo).
- [X] T008 [P] [US1] Reemplazar el enlace `target="_blank"` de "Ver comprobante ↗" en
  `../pos-heladeria/src/app/modules/tables/components/payment-attempt-review-panel.component.ts`
  por un modal/previsualización en la misma pantalla (FR-002 escenario 2; D4 de research.md).
- [X] T009 [P] [US1] En
  `../pos-heladeria/src/app/modules/tables/components/session-bill-panel.component.ts`, quitar el
  botón "Cobrar y cerrar mesa" y el selector manual de método de pago cuando
  `getSidebarMode(order) === 'resumen'` (FR-006). Depende de T004.
- [X] T010 [P] [US1] En
  `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`, filtrar la cola de
  "pagos por confirmar" (o el signal equivalente) a `order.channel === 'qr'`, para que una orden
  manual `recibida` (aún sin cobrar, ver US2/T013) nunca aparezca en el bloque de validación.
- [X] T011 [P] [US1] Tests frontend: nuevo
  `payment-validation-block.component.spec.ts` (tarjetas independientes por comensal, FR-002) y
  actualización de `session-bill-panel.component.spec.ts` (ausencia de "Cobrar y cerrar mesa" en
  modo `resumen`, FR-006), en `../pos-heladeria/src/app/modules/tables/components/`.

**Checkpoint**: User Story 1 funcional y probable de forma independiente — el bug reportado ya no
puede reproducirse.

---

## Phase 4: User Story 2 - El cajero crea y cobra una orden manual para un cliente sin QR (Priority: P1) 🎯 MVP (parte B)

**Goal**: Desde una mesa libre, el cajero crea una orden manual, la cobra (efectivo con cambio, o
transferencia/datáfono sin comprobante) y la envía a cocina en una sola acción atómica.

**Independent Test**: seleccionar una mesa libre, crear una orden manual, agregar productos desde
el catálogo, cobrar en efectivo con cálculo de cambio (o con transferencia/datáfono), y verificar
que el pedido se marca pagado, se envía a cocina y queda una factura generada, todo sin usar el
flujo de validación de comprobantes.

### Backend for User Story 2

- [X] T012 [P] [US2] Agregar el campo opcional `hold_for_payment: bool = False` a `OrderCreate` en
  `../pos-backend/app/api/v1/orders/schemas.py` (D1 de research.md; contrato en
  contracts/api-contracts.md).
- [X] T013 [US2] En `create_order()` (`../pos-backend/app/api/v1/orders/service.py`), implementar
  la rama `hold_for_payment=true`: crea la orden en `status="recibida"` (no `"abierta"`), sin
  descuento de inventario ni visibilidad en cocina; `400` si se combina con `channel="qr"` (solo
  válido para `counter`/`waiter`). Depende de T012.
- [X] T014 [P] [US2] En el mismo `create_order()`
  (`../pos-backend/app/api/v1/orders/service.py`), agregar el chequeo de FR-013: `409` si la
  `table_session_id` resultante ya tiene una orden activa con `channel="qr"`.
- [X] T015 [P] [US2] En `submit_cart()`
  (`../pos-backend/app/api/v1/cart/service.py`, función que crea el pedido QR), agregar el chequeo
  simétrico de FR-013: `409` si la sesión de mesa ya tiene una orden activa con
  `channel in {"counter", "waiter"}`.
- [X] T016 [US2] Implementar `checkout_and_send()` en
  `../pos-backend/app/api/v1/orders/checkout.py` (D3 de research.md): valida `status == "recibida"`
  y el lock optimista `version`, construye la venta/factura vía `build_sale` (mismas líneas,
  promociones y descuentos que ya calcula `pay_order`), y ejecuta en la misma transacción la
  transición `recibida → abierta` (mismo camino que usa `_confirm_order_impl`: descuento de
  inventario + visibilidad en cocina); si el descuento de inventario falla, revierte toda la
  transacción sin dejar un estado intermedio (FR-011, spec 026 FR-002). Depende de T013.
- [X] T017 [US2] Agregar el endpoint `POST /orders/{order_id}/checkout-and-send` (+ schemas de
  request/response) en `../pos-backend/app/api/v1/orders/router.py` y
  `../pos-backend/app/api/v1/orders/schemas.py`, wireado a T016. Depende de T016.
- [X] T018 [P] [US2] Emitir desde `checkout_and_send()` los mismos eventos de tiempo real
  (`payment_completed`, `table_status_changed`) que ya emite el flujo QR equivalente, vía
  `../pos-backend/app/core/events.py` (D8 de research.md). Depende de T016.
- [X] T019 [P] [US2] Tests backend: `create_order(hold_for_payment=true)` (crea en `recibida`, sin
  descuento de inventario) y el chequeo bidireccional de FR-013 (T014, T015), en
  `../pos-backend/app/characterization_tests/test_orders_service.py` (o el fichero de
  characterization tests correspondiente a `orders/service.py`).
- [X] T020 [P] [US2] Tests backend: `checkout_and_send()` — atomicidad, protección contra doble
  ejecución vía `version` (spec 024 FR-018), y reversión completa si falla el descuento de
  inventario (spec 026 FR-002), en
  `../pos-backend/app/characterization_tests/test_orders_checkout.py` (o el fichero de
  characterization tests correspondiente a `orders/checkout.py`).

### Frontend for User Story 2

- [X] T021 [P] [US2] Crear
  `../pos-heladeria/src/app/modules/tables/components/manual-order-panel.component.ts` (+
  plantilla) para el estado vacío "+ Crear Orden Manual" (FR-004), reutilizando
  `pos-catalog-drawer.component.ts` y `cart.component.ts` ya existentes.
- [X] T022 [US2] Conectar `table-sessions.component.ts` para renderizar
  `<app-manual-order-panel>` en la rama "mesa libre" del refactor de T003, con el atajo de teclado
  F3 vinculado a la misma acción (FR-004). Depende de T021, T003.
- [X] T023 [P] [US2] Agregar `createManualOrder(payload)` (con `hold_for_payment: true`) a
  `../pos-heladeria/src/app/modules/tables/services/dining-session.service.ts`, wireado a
  `POST /orders` (T012/T013).
- [X] T024 [US2] Construir el contenido "Terminal POS / Cobro Inmediato" de la barra lateral (en
  `pos-checkout-panel.component.ts` o un componente hijo nuevo en
  `../pos-heladeria/src/app/modules/tables/components/`) para `getSidebarMode(order) ===
  'terminal-pos'`: desglose de cuenta, selector de método de pago reutilizando
  `payment-input.component.ts` (efectivo con cambio FR-008, o transferencia/datáfono sin
  comprobante FR-009), y campo de nombre de facturación con "Consumidor Final" por defecto
  (FR-010). Depende de T004.
- [X] T025 [US2] Agregar `checkoutAndSend(orderId, payload)` a `dining-session.service.ts`, wireado
  a `POST /orders/{id}/checkout-and-send` (T017); conectar el botón único "Cobrar, Facturar y
  Enviar a Cocina" en T024 con protección contra doble clic (deshabilitar mientras está en curso,
  más el `version` de T016/T017 como respaldo de servidor) (FR-011). Depende de T017, T024.
- [X] T026 [P] [US2] Tests frontend: nuevo `manual-order-panel.component.spec.ts` (F3, creación) y
  actualización del componente de T024 (`*.spec.ts`) cubriendo cambio en efectivo, ausencia de
  paso de comprobante en transferencia/datáfono, y "Consumidor Final" por defecto.

**Checkpoint**: User Story 2 funcional de forma independiente. Junto con US1, el MVP híbrido
completo (flujo QR + flujo manual) ya opera.

---

## Phase 5: User Story 3 - El cajero reimprime la factura, adelanta la pre-cuenta, y libera la mesa cuando ya está todo pagado (Priority: P2)

**Goal**: Acciones secundarias de la operación diaria — pre-cuenta antes de pagar, reimpresión
después de facturar, y una liberación manual de mesa que no dependa del barrido automático.

**Independent Test**: sobre una mesa con orden QR ya pagada, usar "Reimprimir Factura POS" y
verificar que se reimprime el mismo documento sin alterar el pago ni la orden; sobre una mesa con
orden aún sin pagar, usar "Imprimir Pre-cuenta" y verificar que imprime el detalle de consumo sin
marcar la orden como pagada; sobre una mesa ya pagada y sin comensales, usar "Cerrar Mesa" y
verificar que vuelve a "Libre" de inmediato.

### Backend for User Story 3

- [X] T027 [US3] Implementar `release_paid_session()` en
  `../pos-backend/app/api/v1/table_sessions/service.py` (D2 de research.md): reutiliza
  `_load(..., lock=True)`, rechaza con `409` si `has_billable_orders(...)` es verdadero (condición
  inversa a `close_session`), reutiliza `_assert_closable(...)` sin modificarla, y llama
  `checkout.close_table_sessions(...)` + `table.status = "libre"` (FR-016).
- [X] T028 [US3] Agregar el endpoint `POST /table-sessions/{table_session_id}/release` (+ schema de
  respuesta) en `../pos-backend/app/api/v1/table_sessions/router.py` y
  `../pos-backend/app/api/v1/table_sessions/schemas.py`, wireado a T027. Depende de T027.
- [X] T029 [P] [US3] Emitir desde `release_paid_session()` los mismos eventos
  (`session_closed`, `table_status_changed`) que ya emite `close_session` (D8 de research.md).
  Depende de T027.
- [X] T030 [P] [US3] Tests backend: `release_paid_session()` — rechazo cuando queda algo por
  cobrar, rechazo cuando hay ítems de cocina sin terminar, y seguridad ante doble clic vía el
  mismo lock de fila (`RN-MESA-01`), en
  `../pos-backend/app/characterization_tests/test_table_sessions_service.py`.

### Frontend for User Story 3

- [X] T031 [P] [US3] Agregar `sessionBillToReceipt(bill, ctx)` a
  `../pos-heladeria/src/app/modules/tables/services/receipt.util.ts` (D6 de research.md),
  construyendo una plantilla de pre-cuenta desde `SessionBillResponse`/`BillResponse` (sin `Sale`
  todavía) y reutilizando `printReceiptHtml()` sin cambios.
- [X] T032 [US3] Agregar el botón "Imprimir Pre-cuenta" (visible antes del pago, en ambos modos de
  la barra lateral) wireado a T031, en `pos-checkout-panel.component.ts` (FR-007). Depende de T031,
  T004.
- [X] T033 [P] [US3] Agregar el botón "Reimprimir Factura POS" reutilizando
  `saleToReceipt()`/`printReceiptHtml()` ya existentes (sin cambio de backend), visible solo si la
  orden ya tiene una `Sale`/`Invoice` emitida, en `pos-checkout-panel.component.ts` o la cabecera
  de la mesa (FR-007, FR-012).
- [X] T034 [P] [US3] Agregar `release(tableSessionId)` a
  `../pos-heladeria/src/app/modules/tables/services/table-session.service.ts`, wireado a
  `POST /table-sessions/{id}/release` (T028).
- [X] T035 [US3] Agregar el botón "Cerrar Mesa" / "Liberar Mesa" en la cabecera de la mesa o la
  barra lateral (visible en ambos modos), wireado a T034, mostrando el motivo de rechazo (`409`) de
  forma clara para el cajero cuando la mesa aún no puede liberarse (FR-016). Depende de T034, T028.
- [X] T036 [P] [US3] Tests frontend: pre-cuenta/reimpresión/cerrar-mesa en el componente de T032/
  T033/T035, y `table-session.service.spec.ts` para `release()`.

**Checkpoint**: User Stories 1, 2 y 3 funcionan de forma independiente y en conjunto.

---

## Phase 6: User Story 4 - El personal identifica el estado de cada mesa por su insignia en el listado (Priority: P3)

**Goal**: El listado lateral de mesas muestra una insignia de color + texto por estado.

**Independent Test**: mostrar el listado de mesas con al menos una mesa en cada estado (por
confirmar, en preparación, libre) y verificar que cada una muestra la insignia de color y texto
correspondiente, sin ambigüedad entre estados.

### Implementation for User Story 4

- [X] T037 [US4] Agregar la lógica de derivación de insignia en
  `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`: mapea
  `DiningTable.status` + presencia de algún intento de pago `pendiente` en la sesión a
  `{ label, color }` — "Por confirmar" (amarillo) si queda al menos un comensal pendiente,
  "En preparación" (azul) si todos los comensales ya están resueltos y pagados, "Libre"
  (gris/verde) sin consumo activo (FR-014; clarificación de la sesión 2026-08-20 sobre envío por
  comensal).
- [X] T038 [P] [US4] Renderizar la insignia (color + texto, nunca solo color) en el ítem de mesa
  del listado lateral dentro de `table-sessions.component.ts` (o su componente hijo de lista de
  mesas), consumiendo T037.
- [X] T039 [P] [US4] Tests frontend: `pos-terminal.store.spec.ts` cubriendo los tres estados y el
  caso mixto (un comensal confirmado, otro pendiente → sigue "Por confirmar"), en
  `../pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts`.

**Checkpoint**: Las cuatro historias de usuario funcionan de forma independiente y en conjunto.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Verificación final contra el spec completo, no ligada a una sola historia.

- [ ] T040 [P] Ejecutar los 6 escenarios de [quickstart.md](./quickstart.md) de punta a punta contra
  ambos repos corriendo localmente, registrando el resultado de cada uno. **Pendiente**: no se
  levantaron los servidores reales (backend + Postgres + frontend) durante esta implementación;
  la cobertura de esta iteración es por tests automatizados (T041/T042) y revisión de código, no
  por click-through manual en navegador. Ver "Notas de implementación" al final de este documento.
- [X] T041 [P] Verificar la suite completa de characterization tests en verde:
  `python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v` desde
  `../pos-backend` (ninguno de los tests ya congelados debe cambiar de resultado).
- [X] T042 [P] Verificar la suite completa de tests frontend en verde: `ng test` desde
  `../pos-heladeria`.
- [X] T043 Revisión manual de FR-015 (ninguna información hoy visible sobre pedido/pago/factura
  desaparece): comparar la nueva UI contra la captura de referencia del spec, ítem por ítem.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — empieza de inmediato.
- **Foundational (Phase 2)**: depende de Setup — **bloquea** a US1 y US2 (ambas escriben sobre los
  contenedores que T003/T004 refactorizan).
- **User Story 1 (Phase 3)**: depende de Foundational (T003, T004). No depende de US2.
- **User Story 2 (Phase 4)**: depende de Foundational (T003, T004). No depende de US1 — puede
  avanzar en paralelo por otra persona una vez completada la Fase 2.
- **User Story 3 (Phase 5)**: depende de Foundational (T004, para T032); no depende de US1 en el
  backend, pero su UI (T032, T035) se ubica en el mismo `pos-checkout-panel.component.ts` que US1
  (T009) y US2 (T024) ya modificaron — conviene completar US1/US2 antes para minimizar conflictos
  de edición, aunque no hay dependencia funcional dura.
- **User Story 4 (Phase 6)**: depende funcionalmente de que existan los estados que US1/US2 ya
  dejan correctos en `DiningTable.status`/intentos de pago — no depende de código nuevo de US1/US2,
  solo de que esos flujos ya muevan el estado correctamente. Puede implementarse en paralelo, pero
  se prueba mejor después.
- **Polish (Phase 7)**: depende de todas las historias que se vayan a entregar en este incremento.

### Dentro de cada historia

- Los tests (T011, T019-T020, T026, T030, T036, T039) se agregan junto a su implementación
  correspondiente, no estrictamente antes (ver nota en "Tests" arriba).
- Backend antes que frontend dentro de US2 y US3, porque el frontend de esas historias llama a los
  endpoints nuevos (T017, T028) que el backend debe exponer primero.

### Parallel Opportunities

- T003 y T004 (Foundational) son paralelos entre sí una vez completado T002 (tocan archivos
  distintos).
- Dentro de US1: T007, T008, T009, T010 son paralelos entre sí (archivos distintos); T011 al final.
- Dentro de US2 (backend): T014 y T015 son paralelos entre sí (archivos distintos); T019 y T020 son
  paralelos entre sí.
- Dentro de US2 (frontend): T021 y T023 son paralelos entre sí.
- US1 y US2 completas son paralelas entre sí una vez terminada la Fase 2 (dos personas distintas).
- US3 y US4 son paralelas entre sí una vez terminadas US1/US2.

---

## Parallel Example: User Story 2 (backend)

```bash
# Una vez completado T013 (create_order con hold_for_payment):
Task: "Agregar chequeo FR-013 en create_order() (../pos-backend/app/api/v1/orders/service.py)"
Task: "Agregar chequeo simétrico FR-013 en submit_cart() (../pos-backend/app/api/v1/cart/service.py)"

# Una vez completado T016 (checkout_and_send):
Task: "Tests de checkout_and_send() en ../pos-backend/app/characterization_tests/test_orders_checkout.py"
Task: "Emitir eventos de tiempo real desde checkout_and_send()"
```

---

## Implementation Strategy

### MVP First (User Stories 1 + 2)

Ambas son P1 y juntas forman el MVP real del modelo híbrido — ninguna por sí sola resuelve el
problema completo que motivó el spec (el bug de US1 sin US2 deja sin resolver la atención a
clientes sin QR; US2 sin US1 deja el bug original intacto):

1. Completar Fase 1: Setup.
2. Completar Fase 2: Foundational (bloqueante).
3. Completar Fase 3 (US1) y Fase 4 (US2) — en paralelo si hay dos personas, o US1 primero (corrige
   el bug reportado) y luego US2.
4. **STOP and VALIDATE**: correr los escenarios 1-3 y 5 de [quickstart.md](./quickstart.md).
5. Desplegar/demostrar el MVP híbrido.

### Incremental Delivery

1. Setup + Foundational → base lista.
2. US1 → probar de forma independiente → corrige el bug reportado (valor inmediato aunque US2 no
   esté lista).
3. US2 → probar de forma independiente → MVP híbrido completo.
4. US3 → probar de forma independiente → operación diaria (pre-cuenta, reimpresión, liberar mesa).
5. US4 → probar de forma independiente → reconocimiento visual en el listado.
6. Polish → verificación final contra el spec completo.

### Parallel Team Strategy

Con más de una persona disponible:

1. El equipo completa Setup + Foundational junto.
2. Una vez lista la Fase 2:
   - Persona A: User Story 1 (frontend puro).
   - Persona B: User Story 2, backend primero (T012-T020) y luego frontend (T021-T026).
3. Con US1/US2 ya integradas, Persona A o B toman User Story 3 (backend + frontend) y User Story 4
   (frontend puro) en paralelo.

---

## Notes

- [P] tasks = distintos archivos, sin dependencias entre sí.
- [Story] label mapea cada tarea a su historia de usuario para trazabilidad (Principio XII de la
  Constitución).
- Ningún endpoint reutilizado (aprobar/rechazar/confirmar-efectivo, `close_session`,
  `block_order`/`pay_order` fuera de este flujo) cambia de comportamiento — ver Constitution Check
  en [plan.md](./plan.md).
- `checkout_and_send()` (T016) y `release_paid_session()` (T027) son las dos piezas de servicio
  nuevas identificadas en research.md (D2, D3) — ningún otro cambio de backend es necesario.
- Confirmar que los tests fallan por la razón esperada antes de implementar, cuando se sigan en
  orden test-primero dentro de una tarea.
- Commit después de cada tarea o grupo lógico.
- Detenerse en cualquier checkpoint para validar una historia de forma independiente.
- Evitar: tareas vagas, conflictos de archivo simultáneos, dependencias cruzadas entre historias
  que rompan su independencia.

---

## Notas de implementación (2026-08-20)

Ejecutado vía `/speckit-implement` en dos agentes en paralelo (backend en `../pos-backend`,
frontend en `../pos-heladeria`), ambos sobre la rama `028-terminal-mesas-modo-hibrido` creada
desde `develop` en cada repo, sin commits (working tree para revisión). Verificación posterior de
los diffs y ambas suites completas:

- **Backend**: 260/260 tests (`app/characterization_tests`), incluidos los nuevos de T019/T020/T030.
  Ningún test `"""CONGELA comportamiento actual"""` existente fue tocado.
- **Frontend**: 296/299 tests (`ng test`); las 3 fallas restantes (`app.spec.ts`,
  `auth.service.spec.ts`, `payment-method.service.spec.ts`) son preexistentes en la rama antes de
  esta feature — verificado con `git stash` contra el baseline.

**Hallazgo corregido durante la revisión (no delegado a un agente)**: la implementación inicial de
T033 ("Reimprimir Factura POS") solo funcionaba para pedidos de mostrador cobrados en la misma
pestaña del navegador (caché en memoria), sin cubrir pedidos de origen QR ni sobrevivir a un
recargo de página — un incumplimiento real de FR-012 ("disponible para cualquier orden QR o
manual... sin límite de veces"). Se corrigió agregando
`DiningSessionService.findSaleForOrder()` (`GET /invoices?order_id=` → `GET /sales/{sale_id}`,
ambos ya existentes en el backend, sin cambios ahí) y `PosTerminalStore.resolveSaleForOrder()`
como respaldo cuando la caché local no tiene la venta; el botón ahora vive en el pie compartido de
la barra lateral (visible en ambos modos) en vez de solo en el modo `terminal-pos`. Al mover la
lógica de resolución a un método separado (`resolveSaleForOrder`) se pudo cubrir con tests sin
disparar el mecanismo real de impresión (iframe + `window.print`, que el builder de tests de este
proyecto —Angular + Vitest— no soporta interceptar vía `vi.mock` para imports relativos, y que
además deja un listener asíncrono capaz de romper el test siguiente si se ejercita de punta a
punta) — de ahí que los tests de T033 vivan en `pos-terminal.store.spec.ts` en vez de en
`pos-checkout-panel.component.spec.ts`.

## Hotfix post-implementación (2026-08-20, mismo día): confirmar pago en efectivo fallaba siempre

Al probar T040 manualmente contra un servidor real, el usuario reportó que "Confirmar efectivo"
fallaba siempre con `409 "La orden no tiene un pago confirmado"`, y que el cambio nunca se mostraba.
Investigado con dos agentes de exploración (uno por repo) y confirmado leyendo el código: **era un
bug preexistente en `pos-backend`, ya presente en `develop`**, no introducido por esta feature —
`confirm_cash_payment_attempt`/`approve_payment_attempt` (`orders/checkout.py`) mutan
`attempt.status = "confirmado"` y llaman de inmediato a `_confirm_order_impl`, cuya primera
verificación es una `SELECT` fresca del mismo estado; la sesión real de producción usa
`autoflush=False` (`app/core/db.py`), así que esa `SELECT` nunca veía el `UPDATE` todavía en
memoria y rechazaba siempre. Los characterization tests nunca lo detectaron porque su fixture
(`orders_fixtures.py::new_session()`) usaba el default de SQLAlchemy, `autoflush=True` — lo
opuesto a producción.

Corregido:
- `db.flush()` explícito entre la mutación y `_confirm_order_impl` en ambas funciones (mismo patrón
  ya usado en `table_sessions/service.py`/`sales/service.py`).
- `orders_fixtures.py::new_session()` gana un parámetro opcional `autoflush: bool = True` (default
  sin cambios, cero impacto en los 261 tests existentes) para poder reproducir la configuración real
  de producción cuando un test lo necesite.
- Nuevo test de regresión `test_confirm_cash_y_approve_funcionan_con_autoflush_false` — verificado
  que falla exactamente con el 409 reportado sin el fix, y pasa con él.
- Frontend: `payment-attempt-review-panel.component.ts` ganó una vista previa del cambio calculada
  en vivo mientras el cajero escribe el monto (antes solo se mostraba después de confirmar, y nunca
  se llegaba a ver porque la confirmación siempre fallaba) — mismo criterio de visibilidad
  (`> 0`) que ya usa `payment-input.component.ts` para el cobro de mostrador.

Verificado: backend 261/261 (`python -m unittest discover -s app/characterization_tests`), frontend
298/301 (`ng test`, mismas 3 fallas preexistentes sin relación). Plan detallado en
`~/.claude/plans/sigue-sin-dejarme-confirmar-resilient-lemon.md`.

## Hotfix post-implementación #2 (2026-08-20, mismo día): un pedido QR nunca llegaba a facturarse

Al probar el hotfix #1, el usuario notó que "Reimprimir Factura POS" seguía fallando ("Este pedido
todavía no tiene una factura emitida") y preguntó si valía la pena mantener ese botón frente a
"Imprimir Pre-cuenta". Investigando la pregunta se encontró la causa real, más profunda que un botón
redundante: **ningún pedido de origen QR llegaba jamás a tener una `Sale`/`Invoice`**.

Antes de esta spec, la única vía que generaba la venta de un pedido QR era "Cobrar y cerrar mesa"
(`close_session`, spec 010) — botón que esta spec retiró correctamente del modo `resumen` (FR-006,
era la causa del hotfix #1). Pero `approve_payment_attempt`/`confirm_cash_payment_attempt`
(`_confirm_order_impl`) nunca generaron una `Sale` por sí mismos — solo envían a cocina y descuentan
inventario. Sin ningún reemplazo, un pedido QR quedaba pagado y en cocina, pero **sin factura para
siempre**. El mismo hueco rompía "Liberar Mesa" (`release_paid_session`, T027): `has_billable_orders`
solo miraba `status` (`pagada`/`cancelada`), y un pedido QR nunca llega a `"pagada"` bajo este diseño
(se mantiene en `"abierta"` a propósito, para seguir siendo visible como consumo activo mientras
cocina lo termina — `activeOrders`/`tableOrders` del frontend excluyen `"pagada"`) — así que
`release_paid_session` rechazaba siempre, para cualquier mesa QR, con "todavía hay algo por cobrar".

Decisión del usuario: generar la venta/factura **al confirmar el pago** (aprobar comprobante o
confirmar efectivo), en la misma llamada — mismo momento que ya usa el flujo manual
(`checkout_and_send`), sin volver a status=`"pagada"` para no romper la visibilidad en cocina.

Corregido en `pos-backend`:
- `approve_payment_attempt`/`confirm_cash_payment_attempt` (`orders/checkout.py`) ahora construyen
  la venta (`build_sale`, mismas líneas/promociones/combos que `pay_order`/`checkout_and_send`) en
  la misma transacción que confirma el intento y envía a cocina. Requieren `cash_shift_id` nuevo
  (`PaymentAttemptApproveIn` nuevo; `PaymentAttemptConfirmCashIn` +campo).
- `has_billable_orders`/`_billable_orders` (`table_sessions/service.py`) ahora también excluyen
  pedidos que ya tienen una `Sale` asociada, sin importar su `status` — corrige tanto
  `release_paid_session` como el barrido automático (`try_release_if_empty`), que tenían el mismo
  hueco silenciosamente.
- `cart_fixtures.py` gana la tabla `sales` en su subconjunto de esquema (la necesitaba
  `has_billable_orders`, antes ausente ahí).
- Tests nuevos: aserciones de `Sale`/`Invoice` en los tests existentes de approve/confirm-cash, y un
  test de regresión en `test_table_sessions_service.py` que siembra una `Sale` real sobre un pedido
  `"abierta"` (sin pasar por `"pagada"`) y confirma que `release_paid_session` ya lo libera — el
  escenario exacto del bug.

Corregido en `pos-heladeria`: `DiningSessionService.approvePaymentAttempt`/
`confirmCashPaymentAttempt` mandan `cash_shift_id`; `PaymentAttemptReviewPanelComponent` gana el
`@Input() cashShiftId`, deshabilita "Aprobar"/"Confirmar efectivo" y avisa si no hay turno abierto;
threadeado vía `PaymentValidationBlockComponent` desde `table-sessions.component.ts`
(`store.cashShiftId()`).

Verificado: backend 262/262, frontend 302 (299 pasan + 3 fallas preexistentes sin relación).

**T040 queda sin marcar** deliberadamente: nada en este repo (`pos-specs`) puede levantar los
servidores reales de `pos-backend`/`pos-heladeria`; se recomienda ejecutarlo manualmente antes de
mergear — ahora sí con el flujo de facturación QR completo de punta a punta.

**Seguimiento pendiente, fuera de alcance de este hotfix**: FR-003 (spec.md) pide un toggle de
"imprimir factura/comanda automáticamente al confirmar" en el bloque de validación — no se encontró
ninguna tarea que lo implementara en `tasks.md` ni rastro de él en el frontend; queda como hueco
conocido para una iteración futura.
