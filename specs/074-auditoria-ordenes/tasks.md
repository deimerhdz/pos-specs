---

description: "Task list for feature implementation"
---

# Tasks: Auditoría del ciclo de vida de una orden

**Input**: Design documents from `/specs/074-auditoria-ordenes/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/order-audit-log-event.md, quickstart.md

**Repositorio de implementación**: todas las rutas de código son relativas a `pos-backend/` (repositorio independiente de `pos-specs`, ver plan.md § Project Structure).

**Tests**: incluidos — el spec no los pide explícitamente, pero la constitución del proyecto (Principio X, Verificación Obligatoria) y `research.md` § 6 ya comprometieron una estrategia de test concreta (`unittest`, mockeando el punto de salida a Sentry) como parte del diseño de este feature.

**Organización**: las tareas se agrupan por historia de usuario (spec.md), para que cada una sea implementable y verificable de forma incremental.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: a qué historia de usuario pertenece (US1, US2, US3)

## Phase 1: Setup

**Purpose**: preparar la configuración compartida antes de tocar la lógica de negocio.

- [X] T001 Añadir el setting **opcional** `AUDIT_HASH_SECRET` (`Optional[str] = Field(default=None, env="AUDIT_HASH_SECRET")`) a la clase `Settings` en `pos-backend/app/core/config.py`, siguiendo el patrón ya usado por `QR_TOKEN_SECRET` (no el de `JWT_SECRET`, que es requerido) — DEBE ser opcional a nivel de `Settings` para no romper el arranque de la app (y de toda la suite de tests existente) en cualquier entorno que aún no tenga la variable configurada; la falla explícita por ausencia se implementa en `_hash_sensitive`, no aquí (ver T006)
- [X] T002 [P] Documentar `AUDIT_HASH_SECRET` en `pos-backend/.env.example`, con un comentario breve indicando que es la clave HMAC para el log de auditoría de órdenes (spec 074) y que nunca debe reutilizar `JWT_SECRET`
- [X] T003 [P] Crear el módulo vacío `pos-backend/app/core/order_audit.py` con un docstring de módulo que explique su propósito (helper de emisión de eventos de auditoría de orden hacia Sentry Logs, sin persistencia propia — spec 074) y las referencias a `data-model.md`/`contracts/order-audit-log-event.md`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: el helper de emisión y sus tipos, compartidos por las 3 historias de usuario.

**⚠️ CRITICAL**: ninguna historia de usuario puede empezar hasta que esta fase esté completa.

- [X] T004 Definir el enum `OrderAuditEventType` (str enum, los 7 valores fijos de `data-model.md` § `OrderAuditEventType`) en `pos-backend/app/core/order_audit.py`
- [X] T005 Definir el enum `ActorType` (`comensal`/`cajero`/`sistema`) y el dataclass congelado `OrderAuditActor` (`type`, `id: str | None`, `role: str | None`) en `pos-backend/app/core/order_audit.py`, según `data-model.md` § Actor
- [X] T006 Implementar `_hash_sensitive(value: str) -> str` en `pos-backend/app/core/order_audit.py`: HMAC-SHA256 usando `settings.AUDIT_HASH_SECRET` como clave (research.md § 3) — misma entrada produce siempre la misma salida; DEBE lanzar un `RuntimeError` explícito si `settings.AUDIT_HASH_SECRET is None` (dado que T001 lo define como opcional a nivel de `Settings`, este es el único punto donde su ausencia se detecta — nunca un fallback silencioso a otro secreto)
- [X] T007 Implementar `record_order_audit_event(*, event_type: OrderAuditEventType, order_id, tenant_id: int, actor: OrderAuditActor, details: dict) -> None` en `pos-backend/app/core/order_audit.py`: **aplana** `actor`/`details` en un diccionario de atributos planos (`order_id`, `tenant_id`, `event_type` (el `.value` del enum), `occurred_at` en UTC, `actor_type`, `actor_id`, `actor_role`, y cada clave de `details` como atributo propio de nivel superior — nunca un objeto anidado, ver `contracts/order-audit-log-event.md` § Envoltorio común), omitiendo del diccionario cualquier clave cuyo valor sea `None` en vez de incluirla; aplica el gate `if settings.ENVIRONMENT != "prod": return` (research.md § 2); llama a `sentry_sdk.logger.info(event_type.value, attributes={...})` con esos atributos planos; y envuelve toda la función en un `try/except` que nunca deja propagar una excepción hacia quien la invoca (FR-011)
- [X] T008 [P] Tests unitarios de `_hash_sensitive` en `pos-backend/app/characterization_tests/test_order_audit_log.py`: mismo valor de entrada → mismo hash (FR-012); valores distintos → hashes distintos; falla de forma explícita si `AUDIT_HASH_SECRET` no está configurado
- [X] T009 Tests unitarios de `record_order_audit_event` en `pos-backend/app/characterization_tests/test_order_audit_log.py` (mismo archivo que T008, secuencial), mockeando `sentry_sdk.logger.info`: el diccionario `attributes` recibido por el mock siempre incluye `event_type`/`order_id`/`tenant_id`/`actor_type`/`occurred_at` como valores planos (nunca un objeto/dict anidado bajo una clave `actor` o `details` — cubre F2 de `/speckit-analyze`); ninguna clave del diccionario tiene valor `None` (los campos no aplicables se omiten, no se envían como `None`); el primer argumento posicional (`template`) pasado al mock es el valor de `event_type` (p. ej. `"order.created"`), evidenciando la categoría distinguible exigida por FR-007; una excepción lanzada dentro del mock nunca se propaga fuera de `record_order_audit_event` (FR-011); no se envía nada cuando `settings.ENVIRONMENT != "prod"` (mockeando `settings.ENVIRONMENT`)

**Checkpoint**: helper de auditoría listo y probado — puede empezar la integración por historia de usuario.

---

## Phase 3: User Story 1 - Reconstruir el historial completo de una orden para resolver disputas (Priority: P1) 🎯 MVP

**Goal**: los 7 tipos de evento del ciclo de vida de una orden se emiten, correlacionados por `order_id`/`tenant_id`, con el actor disponible en cada punto (sin distinguir todavía el caso automático/sistema — eso lo añade US2 — y sin los campos de hash sensibles — eso lo añade US3).

**Independent Test**: ejecutar la secuencia de los Acceptance Scenarios 1 y 2 de US1 (creación QR → confirmación → pago en efectivo; creación manual → cancelación) contra el mock de `record_order_audit_event` y verificar que cada paso emitió su evento, todos con el mismo `order_id`, en orden cronológico.

### Tests for User Story 1 ⚠️

> Escribir estos tests primero; deben fallar hasta completar la Implementation de esta fase.

- [X] T010 [US1] Integration test: `submit_cart` (creación vía QR) invoca `record_order_audit_event` con `event_type=ORDER_CREATED`, `order_id`/`tenant_id` correctos y `actor=OrderAuditActor(ActorType.COMENSAL, participant_id)` — en `pos-backend/app/characterization_tests/test_order_audit_log.py`
- [X] T011 [US1] Integration test: `create_order` (creación manual por staff) invoca `record_order_audit_event` con `event_type=ORDER_CREATED` y `actor=OrderAuditActor(ActorType.CAJERO, user_id, role)` — mismo archivo
- [X] T012 [US1] Integration test: `create_payment_attempt` invoca `record_order_audit_event` con `event_type=PAYMENT_ATTEMPT_CREATED` y `details.payment_method_type`/`details.payment_method_name` correctos — mismo archivo
- [X] T013 [US1] Integration test: `_confirm_order_impl` invoca `record_order_audit_event` con `event_type=ORDER_CONFIRMED` solo después de que la transición `recibida → abierta` se completó — mismo archivo
- [X] T014 [US1] Integration test: `confirm_cash_payment_attempt` invoca `record_order_audit_event` con `event_type=PAYMENT_CASH_CONFIRMED` y `details.amount_received`/`details.change` correctos — mismo archivo
- [X] T015 [US1] Integration test: `approve_payment_attempt` invoca `record_order_audit_event` con `event_type=PAYMENT_TRANSFER_APPROVED` — mismo archivo
- [X] T016 [US1] Integration test: `reject_payment_attempt` invoca `record_order_audit_event` con `event_type=PAYMENT_TRANSFER_REJECTED` y `details.rejection_reason` correcto — mismo archivo
- [X] T017 [US1] Integration test: `cancel_order` invoca `record_order_audit_event` con `event_type=ORDER_CANCELLED` tanto para cancelación iniciada por staff (vía `orders/router.py`) como por comensal (vía `cart/service.py::cancel_my_order`), con `details.initiated_by`/`details.inventory_loss` correctos (FR-009) — mismo archivo
- [X] T018 [US1] Integration test: ninguna de las funciones anteriores invoca `record_order_audit_event` cuando su transición falla la validación de negocio (p. ej. `confirm_cash_payment_attempt` con `amount_received` insuficiente) — cubre FR-010 — mismo archivo

### Implementation for User Story 1

- [X] T019 [US1] Integrar la llamada a `record_order_audit_event(ORDER_CREATED, ...)` en `submit_cart`, justo después de su `commit`, en `pos-backend/app/api/v1/cart/service.py` — `actor=OrderAuditActor(ActorType.COMENSAL, participant_id)`, `details={"channel": "QR_MENU", "order_type": ...}`
- [X] T020 [US1] Integrar la llamada a `record_order_audit_event(ORDER_CREATED, ...)` en `create_order`, justo después de su `commit`, en `pos-backend/app/api/v1/orders/service.py` — `actor=OrderAuditActor(ActorType.CAJERO, user.id, user.role_name)`, `details={"channel": "POS", "order_type": ..., "hold_for_payment": ...}`
- [X] T021 [US1] Integrar la llamada a `record_order_audit_event(PAYMENT_ATTEMPT_CREATED, ...)` en `create_payment_attempt`, justo después de su `commit`, en `pos-backend/app/api/v1/cart/service.py` — `details={"payment_method_type": ..., "payment_method_name": ...}`
- [X] T022 [US1] Integrar la llamada a `record_order_audit_event(ORDER_CONFIRMED, ...)` en `_confirm_order_impl`, justo después de su `commit`, en `pos-backend/app/api/v1/orders/checkout.py` — `actor` construido a partir del `user` disponible en el llamador (la distinción manual/automática/sistema se añade en US2, T029-T030)
- [X] T023 [US1] Integrar la llamada a `record_order_audit_event(PAYMENT_CASH_CONFIRMED, ...)` en `confirm_cash_payment_attempt`, justo después de su `commit`, en `pos-backend/app/api/v1/orders/checkout.py` — `details={"amount_received": ..., "change": ...}`, `actor=OrderAuditActor(ActorType.CAJERO, user.id, user.role_name)`
- [X] T024 [US1] Integrar la llamada a `record_order_audit_event(PAYMENT_TRANSFER_APPROVED, ...)` en `approve_payment_attempt`, justo después de su `commit`, en `pos-backend/app/api/v1/orders/checkout.py`
- [X] T025 [US1] Integrar la llamada a `record_order_audit_event(PAYMENT_TRANSFER_REJECTED, ...)` en `reject_payment_attempt`, justo después de su `commit`, en `pos-backend/app/api/v1/orders/checkout.py` — `details={"rejection_reason": ...}`
- [X] T026 [US1] Integrar la llamada a `record_order_audit_event(ORDER_CANCELLED, ...)` en `cancel_order`, justo después de su `commit`, en `pos-backend/app/api/v1/orders/checkout.py` — `actor` según cuál de `user`/`participant` no sea nulo, `details={"initiated_by": ..., "reason": ..., "inventory_loss": ...}`

**Checkpoint**: User Story 1 completa y verificable de forma independiente — corre T010-T018 y confirma que pasan (FR-001, FR-002, FR-004, FR-009, FR-010, SC-001, SC-004, SC-005 quedan satisfechos con esta fase).

---

## Phase 4: User Story 2 - Identificar con certeza quién ejecutó cada acción sensible (Priority: P1)

**Goal**: cuando la confirmación de la orden ocurre como efecto automático de un pago (no vía una llamada manual a `confirm_order`), su actor queda marcado explícitamente como `sistema`, distinguible del caso manual — y el resto de eventos ya emitidos en US1 quedan verificados como inequívocos (SC-002).

**Independent Test**: ejecutar el Acceptance Scenario 3 de US2 (una confirmación de pago dispara automáticamente la confirmación de la orden) y verificar que el evento `order.confirmed` resultante tiene `actor.type == "sistema"`, mientras que una llamada directa a `confirm_order` produce `actor.type == "cajero"`.

### Tests for User Story 2 ⚠️

- [X] T027 [US2] Integration test: cuando `_confirm_order_impl` se dispara desde dentro de `confirm_cash_payment_attempt`/`approve_payment_attempt` (ruta automática), el evento `order.confirmed` tiene `actor.type == "sistema"` y `details.trigger == "automatic_payment"`; cuando se dispara desde `confirm_order` (ruta manual/recuperación), tiene `actor.type == "cajero"` y `details.trigger == "manual"` — en `pos-backend/app/characterization_tests/test_order_audit_log.py`
- [X] T028 [US2] Test: al ejecutar en secuencia los 7 tipos de evento — (a) una orden vía QR pagada en efectivo: `order.created`, `order.payment_attempt.created`, `order.confirmed`, `order.payment.cash_confirmed`; (b) una orden manual cancelada: `order.created`, `order.cancelled`; (c) una orden con un intento de pago por transferencia aprobado y, en otra orden, uno rechazado: `order.payment.transfer_approved`, `order.payment.transfer_rejected` —, el 100% de los eventos capturados por el mock de `record_order_audit_event` tiene `actor.type` en {`comensal`, `cajero`, `sistema`} con `actor.id` no nulo salvo cuando `actor.type == "sistema"` — cubre SC-002 — mismo archivo

### Implementation for User Story 2

- [X] T029 [US2] Añadir un parámetro `trigger: Literal["manual", "automatic_payment"]` a `_confirm_order_impl` en `pos-backend/app/api/v1/orders/checkout.py`; cuando `trigger == "automatic_payment"`, el `actor` pasado a `record_order_audit_event` es `OrderAuditActor(ActorType.SISTEMA, id=None, role=None)` en vez del cajero que confirmó el pago, y `details["trigger"]` refleja el valor recibido (research.md § 5)
- [X] T030 [US2] Actualizar las 3 llamadas existentes a `_confirm_order_impl` (desde `confirm_order`, `confirm_cash_payment_attempt` y `approve_payment_attempt`) en `pos-backend/app/api/v1/orders/checkout.py` para pasar el `trigger` correcto (`"manual"` en `confirm_order`, `"automatic_payment"` en los otros dos)

**Checkpoint**: User Story 2 completa — actor sin ambigüedad en el 100% de los eventos, incluida la transición automática (FR-003, SC-002).

---

## Phase 5: User Story 3 - Mantener los datos sensibles del comensal fuera de texto plano (Priority: P2)

**Goal**: el nombre del comensal y el comprobante de pago, cuando aparecen en un evento, viajan como su hash HMAC-SHA256 — nunca en texto plano.

**Independent Test**: ejecutar el Acceptance Scenario 1 y 2 de US3 (comensal con nombre real; comprobante de transferencia aprobado) y verificar, sobre el payload capturado por el mock, que ni el nombre ni el comprobante aparecen en texto plano — solo su hash.

### Tests for User Story 3 ⚠️

- [X] T031 [US3] Test: el evento `order.created` emitido por `submit_cart` incluye, en el `details` pasado a `record_order_audit_event` (y por lo tanto en el atributo plano `diner_name_hash` que `record_order_audit_event` envía a Sentry — ver T007), un valor igual al resultado de `_hash_sensitive(display_name)`; en ningún atributo enviado a `sentry_sdk.logger.info` aparece `display_name` en texto plano — en `pos-backend/app/characterization_tests/test_order_audit_log.py`
- [X] T032 [US3] Test: los eventos `order.payment_attempt.created`, `order.payment.transfer_approved` y `order.payment.transfer_rejected`, para el mismo intento de pago, producen el mismo atributo plano `receipt_hash` (`_hash_sensitive(receipt_file_url)`) en lo enviado a `sentry_sdk.logger.info`, y en ningún atributo aparece la URL del comprobante en texto plano (FR-012) — mismo archivo

### Implementation for User Story 3

- [X] T033 [US3] Añadir `details["diner_name_hash"] = _hash_sensitive(display_name)` a la llamada de `record_order_audit_event(ORDER_CREATED, ...)` en `submit_cart`, `pos-backend/app/api/v1/cart/service.py` (solo cuando el `display_name` está disponible, es decir origen QR)
- [X] T034 [US3] Añadir `details["receipt_hash"] = _hash_sensitive(receipt_file_url)` a la llamada de `record_order_audit_event(PAYMENT_ATTEMPT_CREATED, ...)` en `create_payment_attempt`, `pos-backend/app/api/v1/cart/service.py`, cuando el método de pago es de transferencia
- [X] T035 [US3] Añadir el mismo `details["receipt_hash"]` a las llamadas de `record_order_audit_event(PAYMENT_TRANSFER_APPROVED, ...)` y `record_order_audit_event(PAYMENT_TRANSFER_REJECTED, ...)` en `approve_payment_attempt` y `reject_payment_attempt`, `pos-backend/app/api/v1/orders/checkout.py`

**Checkpoint**: User Story 3 completa — SC-003 satisfecho (0% de eventos con nombre de comensal o comprobante en texto plano).

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T036 [P] Correr la suite completa de characterization tests para confirmar que no hay regresión: `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` (desde `pos-backend/`)
- [X] T037 Ejecutar la validación automatizada de `specs/074-auditoria-ordenes/quickstart.md` § 1 (`python -m unittest app.characterization_tests.test_order_audit_log -v`) y confirmar que todas las aserciones descritas ahí pasan
- [X] T038 [P] Revisar que `pos-backend/app/core/order_audit.py` no reutiliza `JWT_SECRET` en ningún camino (verificación manual del research.md § 3 — ningún código de fallback silencioso a otro secreto)
- [ ] T039 Validación manual end-to-end contra un proyecto Sentry de prueba, siguiendo `specs/074-auditoria-ordenes/quickstart.md` § 2 (opcional, antes de cerrar el feature)

---

## Phase 7: Adenda — cobro y envío en un solo paso (FR-014, descubierto durante `/speckit-implement`)

**Contexto**: `checkout_and_send` (`POST /orders/{order_id}/checkout-and-send`) no estaba en el mapeo original de `research.md` § 4. Cobra y envía a cocina una comanda `hold_for_payment` en un solo paso (`recibida → pagada` directo), sin pasar por `_confirm_order_impl` ni por un `OrderPaymentAttempt`. Ver `research.md` § 4 (fila `order.payment.checkout_and_send`) y `contracts/order-audit-log-event.md` § `order.payment.checkout_and_send` para el diseño ya resuelto.

### Tests

- [X] T040 Integration test: `checkout_and_send` invoca, tras su único `commit`, tanto `record_order_audit_event(ORDER_CONFIRMED, ..., details={"trigger": "automatic_payment"})` con actor `sistema`, como `record_order_audit_event(PAYMENT_CHECKOUT_AND_SEND, ...)` con `details.payment_method_types`/`details.total_amount`/`details.payment_count` correctos a partir de `data.payments`; y que ninguno de los dos se emite si la transacción falla (p. ej. `version` desactualizada, o falta de stock al enviar a cocina) — en `pos-backend/app/characterization_tests/test_order_audit_log.py`

### Implementation

- [X] T041 Añadir `PAYMENT_CHECKOUT_AND_SEND = "order.payment.checkout_and_send"` a `OrderAuditEventType` en `pos-backend/app/core/order_audit.py` (data-model.md ya lo define como el 8º valor)
- [X] T042 Integrar, en `checkout_and_send` (`pos-backend/app/api/v1/orders/checkout.py`), justo después de su `commit`: una llamada a `_record_order_confirmed(order.id, cashier, "automatic_payment")` y una llamada a `record_order_audit_event(PAYMENT_CHECKOUT_AND_SEND, ...)` con `actor=_cajero_actor(cashier)` y `details={"payment_method_types": [...], "total_amount": float(...), "payment_count": len(data.payments)}` calculados a partir de `data.payments` (resolviendo el tipo de cada `payment_method_id` contra `PaymentMethod`)
- [X] T043 Correr `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` (desde `pos-backend/`) y confirmar cero regresiones, incluyendo T040

**Checkpoint**: FR-014 satisfecho — el cuarto camino de pago queda auditado igual que los otros tres.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato.
- **Foundational (Phase 2)**: depende de Setup (T006 depende de T001 para leer `settings.AUDIT_HASH_SECRET`). Bloquea las 3 historias de usuario.
- **US1 (Phase 3)**: depende de Foundational (T007). Es el MVP.
- **US2 (Phase 4)**: depende de Foundational, y específicamente de que T022 (la integración base de `_confirm_order_impl`) ya exista, porque T029/T030 modifican esa misma integración para añadir el parámetro `trigger`.
- **US3 (Phase 5)**: depende de Foundational (T006, el hash) y de que T019/T021/T024/T025 (las integraciones base que construyen `details`) ya existan, porque T033-T035 añaden campos a esos mismos `details`.
- **Polish (Phase 6)**: depende de que las historias que se vayan a entregar ya estén completas.

### Notas de archivo compartido (afecta el uso de [P])

`pos-backend/app/api/v1/orders/checkout.py` concentra 5 de los 7 puntos de integración (T022-T026, T029-T030, T035) — esas tareas son secuenciales entre sí (mismo archivo), nunca `[P]`, aunque pertenezcan a historias distintas. Lo mismo aplica a `pos-backend/app/characterization_tests/test_order_audit_log.py` (T008-T018, T027-T028, T031-T032): todas las tareas de test comparten ese único archivo y se ejecutan en el orden listado, no en paralelo entre sí.

### Dentro de cada historia

- Tests antes que Implementation (los tests deben fallar hasta que la Implementation de esa fase esté lista).
- T019-T021 (creación/intento de pago, en `cart/service.py` y `orders/service.py`) son independientes entre sí en cuanto a archivo, pero como cada una es una única tarea de una sola línea de integración, no se marcan `[P]` salvo que se confirme que no comparten archivo (T019/T021 sí comparten `cart/service.py` — tampoco son `[P]` entre sí; T020 sí podría ejecutarse en paralelo con T019/T021 por estar en `orders/service.py`, pero se deja secuencial por simplicidad de revisión).

### Parallel Opportunities

- T002 y T003 (Setup) — archivos distintos.
- T008 (test del hash) puede escribirse en paralelo con el desarrollo de T007 (la implementación de `record_order_audit_event`), ya que prueba una función distinta (`_hash_sensitive`, T006) — se marca `[P]`.
- T036 y T038 (Polish) — no dependen entre sí ni tocan el mismo archivo.

---

## Parallel Example: Setup

```bash
Task: "Documentar AUDIT_HASH_SECRET en pos-backend/.env.example"
Task: "Crear el módulo vacío pos-backend/app/core/order_audit.py con su docstring"
```

---

## Implementation Strategy

### MVP First (User Story 1 solamente)

1. Completar Phase 1: Setup
2. Completar Phase 2: Foundational (bloqueante — helper + tipos + hash)
3. Completar Phase 3: User Story 1
4. **Parar y validar**: correr T010-T018, confirmar que los 7 tipos de evento se emiten correlacionados por `order_id`
5. Esto ya entrega el valor central del spec (User Story 1, P1): reconstruir el historial de una orden para resolver disputas — aunque todavía sin la distinción fina de actor automático (US2) ni el hash de datos sensibles (US3)

### Incremental Delivery

1. Setup + Foundational → base lista
2. + User Story 1 → validar independientemente → ya es una entrega útil (MVP)
3. + User Story 2 → validar independientemente → cierra la ambigüedad de actor en el caso automático
4. + User Story 3 → validar independientemente → cierra el requisito de protección de datos antes de ir a producción (bloqueante para producción real, aunque US1+US2 ya sean funcionalmente completas)

**Importante**: aunque US1 y US2 son ambas P1, no se debe considerar el feature listo para producción sin US3 — enviar `display_name`/`receipt_file_url` en texto plano a un proveedor externo (Sentry) incumple FR-005, un requisito no negociable del spec original.
