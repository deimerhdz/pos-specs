---

description: "Task list for Pagos de Órdenes en Mesa (Skeilopos)"
---

# Tasks: Pagos de Órdenes en Mesa (Skeilopos)

**Input**: Design documents from `/specs/024-pagos-ordenes-mesa/` (plan.md, spec.md, research.md,
data-model.md, contracts/, quickstart.md)

**Tests**: incluidos — el propio `plan.md` (Project Structure) y `quickstart.md` fijan de antemano
qué ficheros de characterization test crea cada historia (Constitución, Principio X: Verificación
Obligatoria), así que no son opcionales para esta spec.

**Organization**: tareas agrupadas por historia de usuario (US1-US6, prioridades de `spec.md`) para
que cada una sea implementable y verificable de forma independiente, per `quickstart.md`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (ficheros distintos, sin dependencia de una tarea sin
  terminar)
- **[Story]**: historia de usuario a la que pertenece (US1..US6)
- Cada tarea incluye la ruta de fichero exacta, relativa a la raíz del repo sibling que corresponda
  (`pos-backend` o `pos-heladeria`)

## Path Conventions

Dos repositorios sibling de `pos-specs` (Constitución §Alcance, plan.md §Project Structure):

- Backend: `pos-backend/app/...` (rutas de este documento ya incluyen el prefijo `pos-backend/`)
- Frontend: `pos-heladeria/src/app/...` (rutas ya incluyen el prefijo `pos-heladeria/`)

---

## Phase 1: Setup

**Purpose**: confirmar que el entorno está listo — esta spec no agrega ninguna dependencia nueva
(plan.md Technical Context), así que no hay instalación ni configuración de herramientas que hacer.

- [X] T001 Confirmar entorno: `pos-backend` con el venv activado (`source env/bin/activate`,
  Python 3.14) y `pos-heladeria` con `npm install` ya corrido; verificar que ningún
  `requirements.txt`/`package.json` necesita cambio (plan.md confirma cero dependencias nuevas)

**Checkpoint**: entornos listos, sin instalar nada nuevo.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: modelo de datos y fixtures de test compartidos por todas las historias — nada de
`Phase 3+` puede empezar sin esto.

**⚠️ CRITICAL**: ninguna historia de usuario arranca hasta que esta fase esté completa.

- [X] T002 Crear migración Alembic `pos-backend/alembic/versions/{rev}_order_payment_attempts.py`:
  tabla `order_payment_attempts` (columnas, `CHECK`s e índice único parcial `WHERE status =
  'pendiente'` de data-model.md) + columna `payment_methods.payment_info` (JSONB, nullable), ambas
  vía `@for_each_tenant_schema` (mismo patrón que `a6b7c8d9e0f1_business_hours_audit.py`); incluir
  `downgrade()` con `op.drop_table`/`op.drop_column` (research.md, estrategia de rollback)
- [X] T003 [P] Agregar columna `payment_info: Mapped[Optional[dict]]` (JSONB) a `PaymentMethod` en
  `pos-backend/app/models/payment.py`
- [X] T004 [P] Crear modelo `OrderPaymentAttempt` en
  `pos-backend/app/models/order_payment_attempt.py` (todas las columnas, `CheckConstraint`s e
  índice único parcial de data-model.md; `{"schema": "tenant"}`)
- [X] T005 Agregar relación `payment_attempts: Mapped[List["OrderPaymentAttempt"]]` en
  `pos-backend/app/models/customer_order.py` (`back_populates="order"`, solo lectura) — depende de
  T004
- [X] T006 Crear helpers de fixtures compartidos — **desviación de ruta**: en vez de un fichero
  nuevo `payment_attempts_fixtures.py`, se extendieron `cart_fixtures.py` y `orders_fixtures.py`
  (`make_payment_method`/`make_payment_attempt` + tablas `payment_methods`/`order_payment_attempts`
  + remoción del índice único parcial para poder sembrar varios intentos no-pendientes por orden),
  porque esos dos ficheros ya traían el superconjunto de tablas que hacía falta (mesas/comensales/
  carrito/órdenes) y crear un tercero habría duplicado esa base — depende de T003, T004
- [ ] T007 Aplicar la migración de T002 contra una base de datos de prueba y verificar el
  `downgrade()` (rollback limpio, sin dejar columnas/tablas huérfanas) — depende de T002.
  **BLOQUEADA en este entorno**: sin acceso a Docker/Postgres en este sandbox (`docker ps` →
  "permission denied"; no hay `pg_isready` ni servidor local). La migración se escribió siguiendo
  al pie de la letra el patrón ya usado por `a6b7c8d9e0f1_business_hours_audit.py`
  (`@for_each_tenant_schema`, `_has_table` como ancla, `op.f()` con la convención de nombres del
  proyecto) y los 231 tests de la suite (que sí corren, sobre SQLite en memoria) ejercitan el mismo
  modelo de datos que esa migración crea — pero la migración en sí no se ejecutó contra un Postgres
  real. Pendiente: correrla en un entorno con Docker/Postgres antes de desplegar.

**Checkpoint**: modelo de datos y fixtures listos — las historias de usuario pueden empezar (en
paralelo si hay más de una persona).

---

## Phase 3: User Story 1 - El tenant configura sus métodos de pago (Priority: P1) 🎯 MVP mínimo

**Goal**: un administrador da de alta, activa, desactiva y edita métodos de pago (efectivo y
transferencias) desde la configuración del tenant.

**Independent Test**: dar de alta/activar/desactivar/editar métodos vía
`contracts/tenant-payment-methods.md`, sin que exista ninguna orden — verificar que los cambios
quedan reflejados y que nunca se puede llegar a cero métodos activos.

### Tests for User Story 1

- [X] T008 [P] [US1] Characterization test
  `pos-backend/app/characterization_tests/test_sales_payment_methods.py` — FR-001/FR-002/FR-003,
  Acceptance Scenarios 1-4 (quickstart.md §US1)

### Implementation for User Story 1

- [X] T009 [US1] Extender `pos-backend/app/api/v1/sales/schemas.py`: `payment_info` en
  `PaymentMethodCreate`/`PaymentMethodResponse`, schema nuevo `PaymentMethodUpdate` (`name?`,
  `payment_info?`, `active?`) — contracts/tenant-payment-methods.md
- [X] T010 [US1] Implementar en `pos-backend/app/api/v1/sales/service.py` la validación "al menos
  un método activo" (contar `active=true` excluyendo el que se desactiva, dentro de la misma
  transacción, research.md Decisión 10) y la lógica de actualización — depende de T009
- [X] T011 [US1] Agregar `PATCH /sales/payment-methods/{id}` en
  `pos-backend/app/api/v1/sales/router.py` (`require_tenant_admin`, `404`/`409` per contrato) y
  extender `POST /sales/payment-methods` para aceptar `payment_info` — depende de T009, T010
- [X] T012 [P] [US1] Agregar `payment_info`/`PaymentMethodUpdatePayload` a
  `pos-heladeria/src/app/modules/sales/interfaces/sales.interface.ts`
- [X] T013 [US1] Agregar método `update()` (PATCH) a
  `pos-heladeria/src/app/modules/sales/services/payment-method.service.ts` — depende de T012
- [X] T014 [US1] Extender
  `pos-heladeria/src/app/modules/sales/pages/payment-methods-page.component.ts` con el formulario
  de datos de transferencia (`payment_info`) y la guardia de UI "no puedes desactivar el último
  método activo" — depende de T013

**Checkpoint**: US1 completa y verificable de forma independiente (`python -m unittest
app.characterization_tests.test_sales_payment_methods -v`).

---

## Phase 4: User Story 2 - El participante paga por transferencia y el cajero revisa el comprobante (Priority: P1)

**Goal**: el comensal elige un método de transferencia, ve sus datos de pago, sube un comprobante;
el cajero lo ve vinculado a la orden y al intento, y lo aprueba o rechaza con motivo.

**Independent Test**: crear orden, seleccionar transferencia, subir comprobante, verificar que
aparece para el cajero vinculado a orden+intento, con opción de aprobar/rechazar con motivo
(quickstart.md §US2).

### Tests for User Story 2

- [X] T015 [P] [US2] Characterization test
  `pos-backend/app/characterization_tests/test_cart_payment_attempts.py` (lado comensal) —
  FR-004/FR-011/FR-012/FR-015a, Acceptance Scenarios 1-3 y 7 de US2
- [X] T016 [P] [US2] Characterization test
  `pos-backend/app/characterization_tests/test_orders_payment_gate.py` (lado cajero, casos de
  aprobar/rechazar) — FR-013/FR-014, Acceptance Scenarios 4-6 de US2

### Implementation for User Story 2 — lado comensal (`cart`)

- [X] T017 [US2] Agregar en `pos-backend/app/api/v1/cart/schemas.py`: ítem de método de pago,
  payload de crear intento, request/response de presign, payload de adjuntar comprobante, y
  `current_payment_attempt` para el detalle de orden — contracts/diner-payment-flow.md
- [X] T018 [US2] Implementar `GET /cart/payment-methods` en
  `pos-backend/app/api/v1/cart/router.py` + `service.py` (filtra `active=true`,
  `Depends(get_session_context)`) — depende de T017
- [X] T019 [US2] Implementar `POST /cart/orders/{order_id}/payment-attempts` en
  `pos-backend/app/api/v1/cart/router.py` + `service.py` (`404` si la orden no es del comensal,
  `409` si ya hay un intento `pendiente` o si `order.status != "recibida"`) — depende de T017
- [X] T020 [US2] Implementar `POST /cart/payment-attempts/{attempt_id}/receipt/presign` en
  `pos-backend/app/api/v1/cart/router.py` + `service.py` (llama directo a
  `generate_presigned_put_url`/`build_object_key` de `app/core/storage.py` con folder
  `"comprobantes"`, sin pasar por `require_tenant_admin` — research.md Decisión 6) — depende de T017
- [X] T021 [US2] Implementar `POST /cart/payment-attempts/{attempt_id}/receipt` en
  `pos-backend/app/api/v1/cart/router.py` + `service.py` (asocia `file_url`, `409` si el intento ya
  tiene comprobante o no es de transferencia) — depende de T017, T020
- [X] T022 [US2] Extender el detalle de orden del comensal en
  `pos-backend/app/api/v1/cart/service.py` con `current_payment_attempt` (sin `rejection_reason`,
  Clarification 3) — depende de T017

### Implementation for User Story 2 — lado cajero (`orders`)

- [X] T023 [US2] Agregar en `pos-backend/app/api/v1/orders/schemas.py`: ítem de historial de
  intentos (con `rejection_reason`), request/response de aprobar y de rechazar —
  contracts/cashier-payment-review.md
- [X] T024 [US2] Implementar `GET /orders/{order_id}/payment-attempts` en
  `pos-backend/app/api/v1/orders/router.py` + `checkout.py` (historial completo, FR-016) — depende
  de T023
- [X] T025 [US2] Implementar `POST /orders/payment-attempts/{attempt_id}/approve` en
  `pos-backend/app/api/v1/orders/router.py` + `checkout.py` (`SELECT ... WHERE status='pendiente'
  WITH FOR UPDATE`, solo transferencia con `receipt_file_url` presente) — depende de T023
- [X] T026 [US2] Implementar `POST /orders/payment-attempts/{attempt_id}/reject` en
  `pos-backend/app/api/v1/orders/router.py` + `checkout.py` (`reason` obligatorio, `422` si falta,
  mismo bloqueo pesimista que approve) — depende de T023

### Implementation for User Story 2 — frontend

- [X] T027 [P] [US2] Agregar `PaymentMethod`, `OrderPaymentAttempt` y los payloads nuevos a
  `pos-heladeria/src/app/modules/tables/interfaces/diner.interface.ts`
- [X] T028 [US2] Agregar llamadas de métodos de pago / crear intento / presign / adjuntar
  comprobante — **desviación de ruta**: se agregaron a `diner.service.ts` (el transporte HTTP real,
  donde ya viven `submitCart`/`myOrders`/`cancelMyOrder`), no a `dining-cart.service.ts` (que es
  estado derivado *solo* de las líneas del carrito — indexa el menú, no tiene relación con pagos ni
  con `DiningOrder`). Mismo patrón que ya usa el resto de `diner.service.ts` (`call()`,
  `DinerSessionExpiredError`) — depende de T027
- [X] T029 [US2] Construir la pantalla de pago del comensal (selección de método, datos de
  transferencia, carga de comprobante, estado del intento) en
  `pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts` — depende de T028
- [X] T030 [US2] Crear
  `pos-heladeria/src/app/modules/tables/components/payment-attempt-review-panel.component.ts`
  (cajero: comprobantes pendientes, aprobar / rechazar con modal de motivo obligatorio) — depende de
  T024, T025, T026

**Checkpoint**: US2 completa y verificable de forma independiente (`python -m unittest
app.characterization_tests.test_cart_payment_attempts
app.characterization_tests.test_orders_payment_gate -v`), sobre la base de US1 (métodos ya
configurables) + Foundational.

---

## Phase 5: User Story 3 - El cajero confirma el pago en efectivo y registra el cambio (Priority: P1)

**Goal**: el cajero registra el monto recibido en efectivo y el sistema calcula el cambio
automáticamente.

**Independent Test**: orden en efectivo, registrar monto recibido mayor al total desde caja,
verificar cambio correcto y orden confirmada (quickstart.md §US3).

### Tests for User Story 3

- [X] T031 [P] [US3] Extender
  `pos-backend/app/characterization_tests/test_orders_payment_gate.py` con los casos de
  confirmación en efectivo — FR-009/FR-010/FR-010a, Acceptance Scenarios 1-4 de US3

### Implementation for User Story 3

- [X] T032 [US3] Agregar request/response de `confirm-cash` en
  `pos-backend/app/api/v1/orders/schemas.py`
- [X] T033 [US3] Implementar `POST /orders/payment-attempts/{attempt_id}/confirm-cash` en
  `pos-backend/app/api/v1/orders/router.py` + `checkout.py` (mismo bloqueo `WITH FOR UPDATE` que
  approve/reject; `422` si `amount_received < total_orden`; `change_amount = amount_received -
  total_orden`) — depende de T032
- [X] T034 [US3] Agregar la UI de confirmación de efectivo (monto → cambio calculado) a
  `pos-heladeria/src/app/modules/tables/components/payment-attempt-review-panel.component.ts`,
  reutilizando el patrón de cálculo de `payment-draft.util.ts` — depende de T030, T033

**Checkpoint**: US3 completa y verificable de forma independiente.

---

## Phase 6: User Story 4 - Una orden solo avanza a comanda con el pago confirmado (Priority: P1)

**Goal**: `confirm_order` exige un intento de pago `confirmado`; ningún intento se confirma dos
veces.

**Independent Test**: intentar enviar a comanda una orden con pago pendiente, otra rechazada y otra
confirmada; confirmar dos veces el mismo intento (quickstart.md §US4).

### Tests for User Story 4

- [X] T035 [P] [US4] Extender
  `pos-backend/app/characterization_tests/test_orders_payment_gate.py` con los casos del gate de
  `confirm_order` — FR-017/FR-018, Acceptance Scenarios 1-4 de US4

### Implementation for User Story 4

- [X] T036 [US4] Agregar la precondición "existe `OrderPaymentAttempt` con `status='confirmado'`"
  dentro de `confirm_order` en `pos-backend/app/api/v1/orders/checkout.py`, entre la validación de
  `status=="recibida"` y el descuento de inventario — contracts/order-confirm-gate.md
- [X] T037 [US4] Reflejar el gate en
  `pos-heladeria/src/app/modules/tables/components/pending-orders-panel.component.ts` (botón
  "Confirmar" deshabilitado sin intento confirmado, mostrar el mensaje del `409` nuevo) — depende de
  T036

**Checkpoint**: US4 completa — combinada con US2/US3, el flujo de pago→comanda es end-to-end.

---

## Phase 7: User Story 5 - El participante reintenta el pago tras un comprobante rechazado (Priority: P2)

**Goal**: tras un rechazo, el comensal puede iniciar un nuevo intento sin perder sus productos.

**Independent Test**: rechazar un comprobante, verificar productos intactos, subir uno nuevo,
verificar que el cajero lo ve como intento nuevo separado del rechazado (quickstart.md §US5).

### Tests for User Story 5

- [X] T038 [P] [US5] Extender
  `pos-backend/app/characterization_tests/test_cart_payment_attempts.py` con los casos de
  reintento tras rechazo — FR-015/FR-015a, Acceptance Scenarios 1-3 de US5

### Implementation for User Story 5

- [X] T039 [US5] Agregar la afordancia "reintentar pago" (reabre la selección de método) tras un
  intento `rechazado` en `pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts` —
  depende de T029 (el backend ya soporta el reintento: es el mismo `POST
  /cart/orders/{order_id}/payment-attempts` de US2, sin endpoint nuevo)

**Checkpoint**: US5 completa. No requiere endpoints backend nuevos — solo verificación (T038) y UX
(T039) sobre lo construido en US2.

---

## Phase 8: User Story 6 - Un participante solo puede tener una orden activa a la vez (Priority: P3)

**Goal**: el sistema impide crear una segunda orden mientras la primera no esté finalizada.

**Independent Test**: crear orden, intentar una segunda antes de finalizar (rechazada), repetir tras
finalizar (permitida) (quickstart.md §US6).

### Tests for User Story 6

- [X] T040 [P] [US6] Characterization test
  `pos-backend/app/characterization_tests/test_cart_single_active_order.py` — FR-005/FR-006,
  Acceptance Scenarios 1-2 de US6

### Implementation for User Story 6

- [X] T041 [US6] Agregar a `submit_cart` en `pos-backend/app/api/v1/cart/service.py` la validación
  `409` si el comensal ya tiene una `CustomerOrder` con `status NOT IN ('pagada', 'cancelada')`
  (research.md Decisión 8)
- [X] T042 [US6] Mostrar el mensaje del `409` nuevo en el punto de envío del carrito en
  `pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts` — depende de T041

**Checkpoint**: las 6 historias de usuario funcionan de forma independiente y combinada.

---

## Phase 9: Polish & Cross-Cutting Concerns

**Purpose**: verificación de no-regresión y cierre de la cadena de trazabilidad (Constitución,
Principio X y XII).

- [X] T043 [P] Ejecutar `python -m unittest discover -s app/characterization_tests -p 'test_*.py'
  -v` completo desde `pos-backend` — confirmar que ningún characterization test preexistente
  (specs 007/008/015/016/017) quedó en rojo
- [X] T044 [P] Recorrer la tabla de verificación final SC-001 a SC-007 de `quickstart.md` y
  confirmar cada criterio con su comando/paso correspondiente
- [ ] T045 Validación manual end-to-end en `pos-heladeria` (`ng serve`): comensal escanea QR → paga
  por transferencia → cajero aprueba/rechaza; comensal paga en efectivo → cajero confirma con
  cambio; intento de enviar a comanda sin pago confirmado — depende de T014, T029, T030, T034, T037,
  T039, T042. **NO EJECUTADA en este entorno**: no hay backend corriendo (`ng serve` necesita la API
  real con Postgres/Redis/R2, ninguno disponible en este sandbox) ni navegador para operarlo. Sí se
  verificó `ng build --configuration development` en verde (compila y type-checkea todo el frontend
  de esta spec, cero warnings) — pero eso confirma tipos, no el comportamiento visual/interactivo.
  Pendiente: correrla contra un entorno real antes de dar la historia por completa de cara al
  usuario final.
- [ ] T046 [P] Agregar specs Vitest para los ficheros frontend nuevos/extendidos de esta spec
  (`payment-attempt-review-panel.component.spec.ts`, extensión de `dining-cart.service.spec.ts` y
  `payment-method.service.spec.ts`)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — arranca de inmediato.
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA todas las historias de usuario.
- **User Stories (Phase 3-8)**: todas dependen de Foundational. Dentro de las historias P1
  (US1-US4), hay una dependencia funcional de orden (no solo de prioridad):
  - **US1 antes que US2/US3**: el comensal necesita métodos de pago configurados (con
    `payment_info`) para poder elegirlos — T017-T030 asumen que `PATCH
    /sales/payment-methods/{id}` (T011) ya existe para tener datos de prueba realistas, aunque el
    modelo de datos (Foundational) ya alcanza para escribir los tests de US2 en paralelo.
  - **US2 y US3 antes que US4**: US4 (el gate de `confirm_order`) solo es observable de punta a
    punta si ya existe una vía para producir un intento `confirmado` (aprobar transferencia — US2 —
    o confirmar efectivo — US3). El test de US4 (T035) puede escribirse usando directamente los
    fixtures de Foundational (T006) sin depender de los endpoints de US2/US3, pero la validación
    manual (T045) sí requiere las tres.
  - **US5 depende de US2**: reutiliza el mismo endpoint de creación de intento (T019); solo agrega
    verificación y UX.
  - **US6 es independiente del resto**: solo toca `submit_cart`, no requiere ningún intento de pago
    existente.
- **Polish (Phase 9)**: depende de que todas las historias que se vayan a entregar estén completas.

### Dentro de cada historia

- Tests antes que implementación (T008 antes de T009-T014; T015/T016 antes de T017-T030; etc.) —
  escritos para fallar primero, per Constitución Principio X.
- Schemas antes que service/router del mismo endpoint (mismo fichero de router depende del de
  schemas correspondiente).
- Backend antes que frontend dentro de la misma historia (el frontend consume el contrato ya
  implementado).

### Parallel Opportunities

- Foundational: T003, T004 en paralelo (ficheros de modelo distintos); T006 tras ambos.
- US1: T008 y T012 en paralelo entre sí (test backend vs. interfaz frontend, ficheros distintos).
- US2: T015 y T016 en paralelo (dos ficheros de test distintos); T027 en paralelo con el bloque
  backend (T017-T026), ya que es un fichero de interfaces sin dependencia de implementación backend
  para escribirse (aunque sí para probarse end-to-end).
- US3, US4, US5, US6: cada tarea marcada [P] es el único test o el único fichero de interfaz de esa
  fase — paralelizable frente a cualquier otra tarea [P] de la misma fase.
- Distintas historias de usuario **no** se recomiendan en paralelo entre sí más allá de US1 con
  US6 (única pareja sin dependencia funcional cruzada) salvo que haya más de una persona
  implementando — todas comparten ficheros (`cart/router.py`, `orders/router.py`) entre US2-US5.

---

## Parallel Example: User Story 2

```bash
# Tests de la historia, en paralelo (ficheros distintos):
Task: "Characterization test test_cart_payment_attempts.py (lado comensal)"
Task: "Characterization test test_orders_payment_gate.py (lado cajero, approve/reject)"

# Interfaces de frontend, en paralelo con el bloque backend completo:
Task: "Agregar PaymentMethod/OrderPaymentAttempt a diner.interface.ts"
```

---

## Implementation Strategy

### MVP mínimo desplegable de forma autónoma: solo User Story 1

1. Completar Phase 1 (Setup) + Phase 2 (Foundational).
2. Completar Phase 3 (US1).
3. **DETENERSE y VALIDAR**: `python -m unittest app.characterization_tests.test_sales_payment_methods
   -v` + revisar `payment-methods-page.component.ts` en el navegador.
4. US1 es la única historia que entrega valor sin que exista ninguna orden (spec.md, "Why this
   priority" de US1) — es un incremento desplegable por sí solo.

### Objetivo de negocio completo: User Stories 1-4 (todas P1)

El "reemplazar la confirmación sin evidencia por verificación explícita del cajero" que motiva esta
spec (spec.md, resumen) solo está completo con US1+US2+US3+US4 juntas — son las cuatro historias
P1. US5 (P2) y US6 (P3) son mejoras sobre ese flujo ya completo, no prerrequisitos suyos.

### Entrega incremental

1. Setup + Foundational → base lista.
2. US1 → validar independientemente → demo (MVP mínimo).
3. US2 → validar independientemente → demo (transferencia con verificación real).
4. US3 → validar independientemente → demo (efectivo con cambio calculado).
5. US4 → validar independientemente → demo (el gate cierra el objetivo de negocio central).
6. US5 → validar independientemente → demo (reintento sin fricción).
7. US6 → validar independientemente → demo (guardarraíl de una orden activa).
8. Polish → verificación final de no-regresión y de SC-001 a SC-007.

---

## Notes

- `[P]` = ficheros distintos, sin dependencia pendiente.
- La etiqueta `[Story]` mapea cada tarea a su historia de `spec.md` para trazabilidad (Constitución,
  Principio XII).
- `test_orders_payment_gate.py` es un único fichero que crece a lo largo de US2 (T016), US3 (T031) y
  US4 (T035) — mismo patrón que agrupar todos los casos de `confirm_order`/resolución de intentos en
  un solo módulo, como ya anticipa quickstart.md.
- Verificar que los tests fallan antes de implementar (Constitución, Principio X).
- Ningún endpoint nuevo cambia el contrato de `POST /orders/{order_id}/confirm`
  (`response_model`/status codes) — solo su precondición interna (contracts/order-confirm-gate.md).
- Evitar: mezclar en una misma tarea cambios de más de un fichero de router/service cuando puedan
  separarse; tareas [P] que en realidad tocan el mismo fichero que otra tarea [P] de la misma fase.
