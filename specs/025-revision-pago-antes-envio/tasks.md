---

description: "Task list for Revisión y Pago Antes de Enviar el Pedido (Skeilopos)"
---

# Tasks: Revisión y Pago Antes de Enviar el Pedido (Skeilopos)

**Input**: Design documents from `/specs/025-revision-pago-antes-envio/` (plan.md, spec.md,
research.md, data-model.md, contracts/, quickstart.md)

**Tests**: incluidos — `plan.md` (Technical Context) y `quickstart.md` fijan de antemano qué
ficheros de characterization test extiende cada historia (Constitución, Principio X: Verificación
Obligatoria); no se crean ficheros de test nuevos, se amplían `test_cart_payment_attempts.py` y
`test_cart_single_active_order.py` (spec 024), citando esta spec.

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

- Backend: `pos-backend/app/...` (rutas de este documento ya incluyen el prefijo `pos-backend/`)
- Frontend: `pos-heladeria/src/app/...` (rutas ya incluyen el prefijo `pos-heladeria/`)

---

## Phase 1: Setup

**Purpose**: confirmar que el entorno está listo — esta spec no agrega ninguna dependencia nueva
(plan.md Technical Context), así que no hay instalación ni configuración de herramientas que hacer.

- [X] T001 Confirmar entorno: `pos-backend` con el venv activado (`source env/bin/activate`,
  Python 3.14) sobre el código de spec 024 ya implementado, y `pos-heladeria` con `npm install` ya
  corrido; verificar que ningún `requirements.txt`/`package.json` necesita cambio (plan.md confirma
  cero dependencias nuevas)

**Checkpoint**: entornos listos, sin instalar nada nuevo.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: el índice único parcial y los schemas base que `submit_cart`/`presign_payment_receipt`
necesitan — nada de `Phase 3+` puede empezar sin esto.

**⚠️ CRITICAL**: ninguna historia de usuario arranca hasta que esta fase esté completa.

- [X] T002 Crear migración Alembic
  `pos-backend/alembic/versions/{rev}_active_order_per_participant.py`: índice único parcial
  `idx_active_order_per_participant` en `customer_orders(participant_id) WHERE status NOT IN
  ('pagada', 'cancelada')`, vía `@for_each_tenant_schema` (mismo patrón que
  `a6b7c8d9e0f1_business_hours_audit.py`); `downgrade()` solo elimina el índice (data-model.md,
  research.md Decisión 4)
- [X] T003 [P] Agregar el índice único parcial equivalente en `__table_args__` de `CustomerOrder`
  (`pos-backend/app/models/customer_order.py`) — mismo predicado que T002, para que el modelo ORM
  documente la restricción que ya existe en la base de datos
- [X] T004 [P] Agregar schema `SubmitCartIn` (`payment_method_id: UUID`, `receipt_file_url: str |
  None = None`) en `pos-backend/app/api/v1/cart/schemas.py` — contracts/submit-cart-with-payment.md
- [X] T005 [P] Agregar schema `PaymentReceiptPresignIn` (reexporta `ReceiptPresignIn`, mismo shape,
  sin campos nuevos) en `pos-backend/app/api/v1/cart/schemas.py` —
  contracts/payment-receipt-presign.md
- [X] T006 [P] Extender `pos-backend/app/characterization_tests/cart_fixtures.py` con un helper para
  sembrar el índice nuevo cuando algún test necesite dos participantes/dos órdenes a propósito
- [ ] T007 Aplicar la migración de T002 contra una base de datos de prueba y verificar el
  `downgrade()` (rollback limpio, sin dejar el índice huérfano) — depende de T002

**Checkpoint**: índice y schemas base listos — las historias de usuario pueden empezar.

---

## Phase 3: User Story 1 - El comensal revisa su pedido y elige cómo pagar antes de enviarlo (Priority: P1) 🎯 MVP

**Goal**: al presionar "Enviar pedido", se muestra una pantalla de revisión (resumen del carrito +
métodos de pago activos del tenant) en lugar de enviar el pedido de inmediato; ningún pedido queda
registrado para el staff en este punto.

**Independent Test**: armar un carrito, presionar "Enviar pedido", y verificar que aparece la
pantalla de revisión con el resumen correcto y los métodos de pago activos del tenant, sin que el
pedido aparezca todavía en el panel del staff (quickstart.md §US1).

### Tests for User Story 1

- [X] T008 [P] [US1] Extender
  `pos-backend/app/characterization_tests/test_cart_payment_attempts.py`: verificar que no existe
  ninguna `CustomerOrder` para el comensal mientras no complete el pago (ausencia de fila, no un
  estado nuevo) y que `service.list_payment_methods(db)` (spec 024, sin cambios) solo devuelve
  métodos `active=true` — FR-001/FR-002/FR-003, Acceptance Scenarios 1-3 de US1

### Implementation for User Story 1

- [X] T009 [US1] Construir la pantalla de revisión (resumen: ítems, cantidades, notas por ítem,
  subtotal, total + selección del método de pago entre los activos del tenant) en
  `pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts`, reemplazando el envío
  inmediato de "Enviar pedido" — reutiliza el modal de selección de método que spec 024 ya
  construyó, sin llamar todavía a ningún endpoint que cree el pedido; permitir volver a la carta sin
  completar el pago (FR-008)

**Checkpoint**: US1 completa y verificable de forma independiente (`python -m unittest
app.characterization_tests.test_cart_payment_attempts -v`).

---

## Phase 4: User Story 2 - El comensal paga en efectivo y el pedido queda para el cajero (Priority: P1)

**Goal**: al elegir efectivo y confirmar, el pedido se crea junto con su primer intento de pago en
efectivo, pendiente de que el cajero registre el monto recibido.

**Independent Test**: elegir efectivo en la pantalla de revisión, confirmar, y verificar que el
pedido aparece para el staff con el pago en efectivo pendiente de registrar; que el cajero pueda
confirmarlo con el flujo ya existente (quickstart.md §US2).

### Tests for User Story 2

- [X] T010 [P] [US2] Extender `test_cart_payment_attempts.py`:
  `submit_cart(db, participant, efectivo.id)` crea `CustomerOrder` (`status='recibida'`) con
  `current_payment_attempt.status='pendiente'`, `is_cash=True`, `receipt_file_url=None`; y falla con
  `422` si se envía `receipt_file_url` con un método en efectivo — FR-004, Acceptance Scenario 1 de
  US2
- [X] T011 [P] [US2] Extender
  `pos-backend/app/characterization_tests/test_cart_single_active_order.py` con el caso de
  confirmación duplicada: llamar `submit_cart(db, participant, metodo.id)` dos veces seguidas sobre
  la misma sesión (doble toque/reintento de red) — la segunda falla `409` y solo queda **una**
  `CustomerOrder` para ese comensal — FR-013, SC-006, edge case "confirmación duplicada" (depende de
  T002 aplicado o al menos del `IntegrityError` que capturará T012)

### Implementation for User Story 2

- [X] T012 [US2] Modificar `submit_cart` en `pos-backend/app/api/v1/cart/service.py`: nueva firma
  `submit_cart(db, participant, payment_method_id, receipt_file_url=None)`; valida en orden carrito
  no vacío (`409`), orden activa ya existente (`409`), `payment_method_id` existe (`404`)/activo
  (`409`), efectivo sin `receipt_file_url` (`422`), disponibilidad de stock (sin cambio); crea
  `CustomerOrder` + `OrderItem` + primer `OrderPaymentAttempt` en una sola transacción (`commit()`
  único); captura `IntegrityError` de `idx_active_order_per_participant` alrededor del `commit()` y
  la traduce al mismo `409` de orden activa (research.md Decisión 4) — depende de T004, T005, T007
- [X] T013 [US2] Actualizar `POST /cart/submit` en `pos-backend/app/api/v1/cart/router.py` para
  exigir el body `SubmitCartIn` (antes vacío) y pasar `payment_method_id`/`receipt_file_url` a
  `submit_cart` — depende de T012
- [X] T014 [US2] Retirar el botón "Elegir cómo pagar" del historial de pedidos en
  `pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts` — ya no aplica, todo pedido
  nace con su intento de pago adjunto una vez `POST /cart/submit` exige el body — depende de T013
- [X] T015 [P] [US2] Agregar `SubmitCartPayload`
  (`payment_method_id`, `receipt_file_url?`) a
  `pos-heladeria/src/app/modules/tables/interfaces/diner.interface.ts`
- [X] T016 [US2] Actualizar `submitCart()` en
  `pos-heladeria/src/app/modules/tables/services/diner.service.ts` para enviar
  `payment_method_id`/`receipt_file_url` en el body — depende de T015
- [X] T017 [US2] Conectar la confirmación de efectivo en la pantalla de revisión (T009): al elegir
  efectivo y confirmar, llama `submitCart(methodId)` y muestra el pedido creado (sin número/ID hasta
  que la respuesta llega, FR-006a) — depende de T009, T013 (contrato), T016

**Checkpoint**: US2 completa y verificable de forma independiente (`python -m unittest
app.characterization_tests.test_cart_payment_attempts
app.characterization_tests.test_cart_single_active_order -v`).

---

## Phase 5: User Story 3 - El comensal paga por transferencia y el pedido solo se registra tras cargar el comprobante (Priority: P1)

**Goal**: al elegir un método de transferencia, el comensal ve sus datos de pago y debe cargar un
comprobante; el pedido solo se crea una vez el comprobante ya se subió con éxito.

**Independent Test**: elegir un método de transferencia, ver sus datos de pago, cargar un
comprobante, y verificar que el pedido solo aparece para el staff después de completar esa carga
(quickstart.md §US3).

### Tests for User Story 3

- [X] T018 [P] [US3] Extender `test_cart_payment_attempts.py`:
  `service.presign_payment_receipt(db, "tenant_test", participant.id, "image/jpeg")` no exige
  ningún `attempt_id` y devuelve `upload_url`/`public_url`; sigue sin existir ninguna `CustomerOrder`
  después de llamarlo; `submit_cart(db, participant, nequi.id, receipt_file_url=public_url)` crea la
  orden con `current_payment_attempt.receipt_file_url == public_url`; `submit_cart(db, participant,
  nequi.id)` sin `receipt_file_url` falla `422` — FR-005/FR-006/FR-007, Acceptance Scenarios 1 y 3
  de US3
- [X] T019 [P] [US3] Extender `test_cart_payment_attempts.py` con el caso de reintento sin volver a
  subir el archivo (FR-012): forzar un fallo en `submit_cart` (mockear `db.commit` para que lance la
  primera vez) tras un `presign_payment_receipt` exitoso — verificar que no queda ninguna
  `CustomerOrder` ni `OrderPaymentAttempt` creados; reintentar `submit_cart` con el mismo
  `public_url` (sin llamar de nuevo al presign) — verificar que esta vez sí se crea la orden con ese
  mismo `receipt_file_url`
- [X] T020 [US3] Extender
  `pos-backend/app/characterization_tests/test_orders_payment_gate.py` con un caso que arranca desde
  `submit_cart(..., receipt_file_url=...)` de esta spec — confirmar que
  `checkout.approve_payment_attempt`/`reject_payment_attempt` (spec 024, sin cambios) siguen
  funcionando igual sobre una orden creada por este flujo, incluyendo el reintento tras rechazo
  (`POST /cart/orders/{order_id}/payment-attempts`, sin cambios) — Acceptance Scenario 4 de US3

### Implementation for User Story 3

- [X] T021 [US3] Implementar `presign_payment_receipt(db, tenant_schema, participant_id,
  content_type)` en `pos-backend/app/api/v1/cart/service.py` — reutiliza `build_object_key`,
  `generate_presigned_put_url`, `public_url_for` de `app/core/storage.py` (mismas primitivas que
  `presign_receipt`, spec 024), sin validar ningún recurso previo — `422` si `content_type` no está
  en la whitelist — depende de T005
- [X] T022 [US3] Agregar `POST /cart/payment-receipt/presign` en
  `pos-backend/app/api/v1/cart/router.py` (`Depends(get_session_context)`, mismo patrón que el resto
  de `cart/router.py`, invariante A-24) — depende de T021
- [X] T023 [US3] Extender la validación de `submit_cart`
  (`pos-backend/app/api/v1/cart/service.py`, T012) para exigir `receipt_file_url` cuando el método
  no es efectivo (`422` si falta, FR-006) — depende de T012
- [X] T024 [P] [US3] Agregar `presignPaymentReceipt()` a
  `pos-heladeria/src/app/modules/tables/services/diner.service.ts` (sube el `content_type`, recibe
  `upload_url`/`public_url`, sube el archivo directo a R2 con `PUT`) — depende de T015
- [X] T025 [US3] Construir el paso de "método de transferencia" en la pantalla de revisión (T009):
  datos de pago del método elegido + control de carga de comprobante (llama
  `presignPaymentReceipt()`, sube a R2, conserva `public_url` en memoria) + confirmación final que
  llama `submitCart(methodId, receiptUrl)` — depende de T009, T024, T016, T022

**Checkpoint**: US3 completa y verificable de forma independiente (`python -m unittest
app.characterization_tests.test_cart_payment_attempts
app.characterization_tests.test_orders_payment_gate -v`).

---

## Phase 6: User Story 4 - El comensal cambia de método de transferencia antes de subir el comprobante (Priority: P2)

**Goal**: mientras no se haya cargado ningún comprobante, el comensal puede volver y elegir otro
método (de transferencia o efectivo) sin restricción, porque el pedido todavía no existe.

**Independent Test**: abrir la pantalla de un método de transferencia sin cargar nada, volver y
elegir un método distinto, y verificar que solo termina existiendo un pedido, con el método
finalmente elegido (quickstart.md §US4).

### Implementation for User Story 4

- [X] T026 [US4] Permitir volver desde el paso de un método de transferencia (T025) a la selección
  de método sin ninguna restricción (sin llamar a `submitCart` hasta la confirmación final; ningún
  rastro del método abandonado ni de su `public_url` sin usar) en
  `pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts` — depende de T025

**Checkpoint**: las 4 historias de usuario funcionan de forma independiente y combinada — el flujo
de revisión+pago queda completo.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: verificación de no-regresión y cierre de la cadena de trazabilidad (Constitución,
Principio X y XII).

- [X] T027 [P] Ejecutar `python -m unittest discover -s app/characterization_tests -p 'test_*.py'
  -v` completo desde `pos-backend` — confirmar que toda la suite (spec 024 y anteriores, actualizada
  a la nueva firma de `submit_cart`) queda en verde
- [X] T028 [P] Recorrer la tabla de verificación final SC-001 a SC-006 de `quickstart.md` y
  confirmar cada criterio con su comando/paso correspondiente
- [ ] T029 Validación manual end-to-end en `pos-heladeria` (`ng serve`): revisión → efectivo →
  pedido visible para el cajero → confirmación con cambio; revisión → transferencia → carga de
  comprobante → pedido visible para el cajero → aprobación/rechazo; cambio de método de transferencia
  antes de cargar comprobante — depende de T017, T025, T026
- [ ] T030 [P] Agregar specs Vitest para los ficheros frontend nuevos/extendidos de esta spec
  (pantalla de revisión en `public-menu.component.ts`, extensión de `diner.service.ts`) — spec 024
  dejó esto pendiente, no se agrava aquí (plan.md Testing)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — arranca de inmediato.
- **Foundational (Phase 2)**: depende de Setup — BLOQUEA todas las historias de usuario.
- **User Stories (Phase 3-6)**: todas dependen de Foundational. Entre las historias P1 (US1-US3) hay
  una dependencia funcional de orden, no solo de prioridad:
  - **US1 antes que US2/US3**: la pantalla de revisión (T009) es donde se conectan después la
    confirmación de efectivo (US2) y el paso de transferencia (US3) — ambas historias extienden el
    mismo componente que US1 construye.
  - **US2 antes que US3 en el backend**: `submit_cart` (T012, US2) es la función que US3 extiende
    (T023) con la validación de transferencia — no son funciones separables, es la misma
    fusionada por research.md Decisión 1.
  - **US4 depende de US3**: reutiliza el mismo paso de método de transferencia (T025); solo agrega
    la libertad de volver atrás sin restricción.
- **Polish (Phase 7)**: depende de que todas las historias que se vayan a entregar estén completas.

### Dentro de cada historia

- Tests antes que implementación (T008 antes de T009; T010/T011 antes de T012-T017; T018-T020 antes
  de T021-T025) — escritos para fallar primero, per Constitución Principio X.
- Schemas (Foundational) antes que service; service antes que router del mismo endpoint.
- Backend antes que frontend dentro de la misma historia (el frontend consume el contrato ya
  implementado).

### Parallel Opportunities

- Foundational: T003, T004, T005, T006 en paralelo (ficheros distintos); T007 tras T002.
- US1: T008 es la única tarea de test, sin nada más con quien paralelizar dentro de la fase.
- US2: T010 y T011 en paralelo (dos ficheros de test distintos); T015 en paralelo con el bloque
  backend (T012-T014), ya que es un fichero de interfaces sin dependencia de implementación backend
  para escribirse.
- US3: T018 y T019 en paralelo (mismo fichero pero casos independientes — coordinar si se trabajan a
  la vez); T024 en paralelo con el bloque backend (T021-T023).
- Distintas historias de usuario **no** se recomiendan en paralelo entre sí más allá de Foundational,
  ya que US1-US4 comparten el mismo componente frontend (`public-menu.component.ts`) y la misma
  función backend (`submit_cart`) — salvo que haya más de una persona coordinando esos ficheros.

---

## Parallel Example: User Story 2

```bash
# Tests de la historia, en paralelo (ficheros distintos):
Task: "Extender test_cart_payment_attempts.py con el caso de efectivo"
Task: "Extender test_cart_single_active_order.py con la confirmación duplicada"

# Interfaz de frontend, en paralelo con el bloque backend completo:
Task: "Agregar SubmitCartPayload a diner.interface.ts"
```

---

## Implementation Strategy

### MVP mínimo desplegable de forma autónoma: User Story 1 + User Story 2

1. Completar Phase 1 (Setup) + Phase 2 (Foundational).
2. Completar Phase 3 (US1) + Phase 4 (US2).
3. **DETENERSE y VALIDAR**: `python -m unittest app.characterization_tests.test_cart_payment_attempts
   app.characterization_tests.test_cart_single_active_order -v` + revisar la pantalla de revisión y
   el pago en efectivo en el navegador.
4. US1 sola no es desplegable (no hay ningún camino de pago que complete el flujo) — el mínimo
   entregable real es US1+US2, que cubre el camino de efectivo de punta a punta.

### Objetivo de negocio completo: User Stories 1-3 (todas P1)

El cambio central que pide `spec.md` — que ningún pedido llegue al staff sin su método de pago
resuelto — solo está completo con US1+US2+US3 juntas, que cubren los dos únicos caminos de pago
(efectivo y transferencia). US4 (P2) es una mejora de fricción sobre US3, no un prerrequisito suyo.

### Entrega incremental

1. Setup + Foundational → índice y schemas listos.
2. US1 → pantalla de revisión visible (sin camino de pago completo todavía).
3. US2 → validar independientemente → demo (efectivo de punta a punta).
4. US3 → validar independientemente → demo (transferencia de punta a punta, objetivo de negocio
   completo).
5. US4 → validar independientemente → demo (fricción reducida al cambiar de método).
6. Polish → verificación final de no-regresión y de SC-001 a SC-006.

---

## Notes

- `[P]` = ficheros distintos, sin dependencia pendiente.
- La etiqueta `[Story]` mapea cada tarea a su historia de `spec.md` para trazabilidad (Constitución,
  Principio XII).
- `test_cart_payment_attempts.py` es un único fichero que crece a lo largo de US1 (T008), US2 (T010)
  y US3 (T018, T019) — mismo patrón que agrupar todos los casos de `submit_cart` en un solo módulo,
  como ya anticipa quickstart.md.
- Verificar que los tests fallan antes de implementar (Constitución, Principio X).
- `checkout.py`, `orders/router.py` y el panel del cajero (spec 024) NO se tocan en ninguna tarea —
  reciben una orden con su intento de pago ya creado, exactamente como antes (plan.md, Principio V).
- Evitar: mezclar en una misma tarea cambios de más de un fichero de router/service cuando puedan
  separarse; tareas [P] que en realidad tocan el mismo fichero que otra tarea [P] de la misma fase
  (T018/T019 comparten fichero — señalado explícitamente arriba, no ejecutar a ciegas en paralelo sin
  coordinar).
</content>
