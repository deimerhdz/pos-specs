---

description: "Task list for Rediseño UX de Confirmación de Pago y Comanda en Terminal de Mesas (Skeilopos)"
---

# Tasks: Rediseño UX de Confirmación de Pago y Comanda en Terminal de Mesas (Skeilopos)

**Input**: Design documents from `/specs/026-mejoras-ux-comanda/` (plan.md, spec.md, research.md,
data-model.md, contracts/, quickstart.md)

**Tests**: incluidos — `plan.md` (Project Structure) y `quickstart.md` fijan de antemano qué
ficheros de test crea o extiende cada historia (Constitución, Principio X: Verificación
Obligatoria; Principio III: el CONGELA de `confirm_order` se actualiza citando esta spec), así que
no son opcionales para esta spec.

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

**Purpose**: confirmar que el entorno está listo y que hay una línea base verde antes de tocar
nada — esta spec no agrega ninguna dependencia nueva ni migración (plan.md Technical Context;
data-model.md).

- [X] T001 Confirmar entorno: `pos-backend` con el venv activado (Python 3.14) y `pos-heladeria`
  con `npm install` ya corrido; correr `python -m unittest discover app/characterization_tests -v`
  en `pos-backend` como línea base y confirmar que toda la suite pasa antes de empezar (research.md
  no exige ninguna instalación nueva)

**Checkpoint**: entornos listos, línea base verde confirmada, sin instalar nada nuevo.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: verificar si existe algún prerrequisito compartido por las 4 historias.

**Resultado de esa verificación**: ninguno. Cada historia toca un subconjunto disjunto de ficheros
(plan.md, Structure Decision) y ninguna depende de un modelo, tabla o servicio nuevo compartido —
esta spec no agrega ninguno (data-model.md). El único prerrequisito real es específico de la
Historia 1 (extraer `_confirm_order_impl`) y vive dentro de su propia fase, no aquí, porque no lo
necesitan las Historias 2-4.

**Checkpoint**: sin tareas en esta fase — se pasa directo a la Fase 3.

---

## Phase 3: User Story 1 - Confirmar el pago envía el pedido a cocina en un solo paso (Priority: P1) 🎯 MVP

**Goal**: que confirmar un pago (efectivo recibido o comprobante aprobado) descuente el inventario
y envíe el pedido a cocina en la misma acción, sin el botón "Confirmar" manual que hoy vive en la
pestaña "Por confirmar" (spec FR-001/FR-002/FR-003, research.md Decisiones 1-3, 5).

**Independent Test**: confirmar el pago de un pedido (efectivo o transferencia) y verificar que,
sin ninguna acción adicional del cajero, el pedido queda con el inventario descontado y visible
para cocina; y que un fallo de stock deja el intento sin confirmar y la orden sin tocar.

### Implementación backend para User Story 1

- [X] T002 [US1] En `pos-backend/app/api/v1/orders/checkout.py`, extraer el cuerpo de
  `confirm_order` (desde la carga con `with_for_update` hasta antes de `db.commit()`,
  aproximadamente líneas 311-364) a una función interna `_confirm_order_impl(db: Session, order_id:
  UUID, user: User) -> CustomerOrder` que hace exactamente lo mismo (precondición de intento
  confirmado, lock de la orden, `deduct_order_items`, `order.status = "abierta"`, `order.version +=
  1`) pero **sin** `db.commit()`/`db.rollback()` propios — solo levanta la `HTTPException` que
  corresponda. `confirm_order` pasa a ser un wrapper delgado: llama `_confirm_order_impl` dentro de
  su mismo `try/except` ya existente, que sigue haciendo `commit`/`rollback` exactamente como hoy
  (research.md, Decisión 1). El endpoint público `POST /orders/{id}/confirm` no cambia de
  contrato (contracts/order-confirm-manual.md).
- [X] T003 [US1] En el mismo fichero, modificar `confirm_cash_payment_attempt`
  (aprox. líneas 697-728): justo después de fijar `attempt.status = "confirmado"` y sus campos
  (`amount_received`, `change_amount`, `resolved_by_user_id`, `resolved_at`) y **antes** de su
  `db.commit()` propio, invocar `_confirm_order_impl(db, attempt.order_id, user)` (T002). Si
  `_confirm_order_impl` levanta una excepción (p. ej. stock insuficiente), debe propagarse al mismo
  `except`/`db.rollback()` que ya existe en esta función, de forma que el intento de pago **no**
  quede confirmado (contracts/payment-confirmation.md, research.md Decisión 5)
- [X] T004 [US1] En el mismo fichero, aplicar el mismo cambio a `approve_payment_attempt`
  (aprox. líneas 640-663): invocar `_confirm_order_impl(db, attempt.order_id, user)` justo después
  de fijar `attempt.status = "confirmado"` y antes de su `db.commit()` propio, con la misma
  propagación de errores al `rollback` existente. `reject_payment_attempt` no se toca (nunca ha
  disparado envío a cocina)

### Tests para User Story 1

- [X] T005 [P] [US1] Actualizar el characterization test CONGELA de `confirm_order` en
  `pos-backend/app/characterization_tests/test_orders_checkout.py` (línea ~288): ajustar la cita de
  líneas al nuevo `_confirm_order_impl` y dejar constancia, en el mismo commit, de la autorización
  de esta spec (026, FR-001) para el cambio de *quién* invoca esta lógica — el comportamiento
  observable del endpoint público `confirm_order` no cambia, así que las aserciones existentes
  deben seguir pasando sin modificación de fondo (Constitución, Principio III)
- [X] T006 [P] [US1] Agregar casos nuevos en
  `pos-backend/app/characterization_tests/test_orders_payment_gate.py`: (a) confirmar un pago en
  efectivo con stock suficiente deja, en una sola llamada a `confirm_cash_payment_attempt`, el
  intento `confirmado` y la orden en `abierta` con el inventario descontado; (b) lo mismo para
  `approve_payment_attempt` con un comprobante; (c) confirmar un pago sobre una orden cuyo stock ya
  no alcanza deja el intento **sin** confirmar (`pendiente`) y la orden **sin** avanzar
  (`recibida`) — ninguna de las dos cosas ocurre a medias (FR-002); (d) doble confirmación casi
  simultánea del mismo intento no duplica el envío a cocina (reutiliza la garantía de idempotencia
  de spec 024, FR-018)

### Implementación frontend para User Story 1

- [X] T007 [US1] En
  `pos-heladeria/src/app/modules/tables/components/pending-orders-panel.component.ts`, eliminar el
  botón "Confirmar" (aprox. líneas 106-115), el método `confirm()` (aprox. línea 203) y
  `isPaymentConfirmed()` (aprox. línea 191); conservar el botón "Rechazar" y `reject()`/
  `cancelOrder(...)` sin cambios (research.md Decisión 3) — depende de T003/T004 ya mergeados en el
  backend
- [X] T008 [US1] En el mismo componente (y en
  `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` /
  `pos-heladeria/src/app/modules/tables/pages/table-sessions.component.ts` si el conteo/rótulo de
  la pestaña vive ahí), ajustar el texto de la pestaña "🔔 Por confirmar" y su descripción ("Confirmar
  descuenta el inventario y manda el pedido a cocina", línea ~36) para reflejar que ahora agrupa
  comprobantes de transferencia pendientes de revisión del cajero, no pedidos pendientes de enviar
  a cocina — depende de T007
- [X] T009 [P] [US1] Crear
  `pos-heladeria/src/app/modules/tables/components/pending-orders-panel.component.spec.ts`
  verificando: no existe ningún botón "Confirmar" ni llamada a `confirmOrder()` disparada desde este
  panel; el botón "Rechazar" sigue invocando `cancelOrder(...)` igual que antes — depende de T007,
  T008

**Checkpoint**: la Historia 1 es funcional y verificable de forma independiente — confirmar un pago
ya envía el pedido a cocina sin ningún clic adicional, y un fallo de stock no deja nada a medias.

---

## Phase 4: User Story 2 - El cajero ve el cambio a entregar al confirmar un pago en efectivo (Priority: P1)

**Goal**: mostrar el monto recibido y el cambio calculado de forma inmediata y permanente al
confirmar un pago en efectivo — el dato ya existe en backend, el defecto es de presentación
(spec FR-004/FR-005, research.md Decisión 6).

**Independent Test**: registrar un monto recibido mayor al total de una orden en efectivo,
confirmar el pago, y verificar que el cambio aparece de inmediato y permanece visible al reabrir el
pedido más tarde; con monto exacto, verificar que se muestra "$0" explícitamente.

### Implementación para User Story 2

- [X] T010 [US2] En
  `pos-heladeria/src/app/modules/tables/components/payment-attempt-review-panel.component.ts`,
  extender el bloque `@if (last.status === 'confirmado')` (aprox. líneas 111-112) para que, cuando
  `last.is_cash` sea verdadero, muestre también `last.amount_received` y `last.change_amount` de
  forma permanente (no solo en el toast de `confirmCash()`, aprox. línea 200-202, que se conserva
  como aviso inmediato adicional) — incluir explícitamente el caso `change_amount === '0.00'` como
  "Cambio: $0", nunca omitido (research.md Decisión 6; ambos campos ya están tipados en
  `PaymentAttempt`, `dining.interface.ts:144-158`, sin cambios de backend necesarios)

### Tests para User Story 2

- [X] T011 [P] [US2] Crear
  `pos-heladeria/src/app/modules/tables/components/payment-attempt-review-panel.component.spec.ts`
  cubriendo: cambio > 0 se muestra en la vista persistente tras confirmar; cambio = 0 se muestra
  explícitamente como "$0"; monto recibido y cambio siguen visibles al volver a renderizar el
  componente con el mismo intento ya confirmado (simulando "reabrir el pedido más tarde") — depende
  de T010

**Checkpoint**: las Historias 1 y 2 funcionan juntas de forma independiente — confirmar efectivo
envía a cocina y muestra el cambio, sin pasos ni pantallas adicionales.

---

## Phase 5: User Story 3 - El cajero divide la cuenta de una mesa y la cobra con varios métodos de pago desde la Terminal de Mesas (Priority: P2)

**Goal**: verificar que el panel "Cuenta de la mesa" ya cumple división por ítem/unidad y cobro
combinado (spec FR-006-009) y cerrar únicamente los gaps reales encontrados — no reconstruir la
lógica ya existente (research.md Decisión 7).

**Independent Test**: abrir una mesa con consumo, ver el detalle de su cuenta, dividirla asignando
ítems a dos comensales, cobrar cada parte con métodos combinados, y verificar que el total y el
cambio son correctos y que la factura queda generada al cerrar.

### Auditoría e implementación para User Story 3

- [X] T012 [US3] Recorrer los 4 acceptance scenarios de la Historia 3 de `spec.md` contra el
  comportamiento actual de
  `pos-heladeria/src/app/modules/tables/components/session-bill-panel.component.ts` y
  `split-bill-panel.component.ts` (apoyándose en `contracts/table-bill-multi-payment.md`),
  documentando en el PR/commit qué escenario ya pasa tal cual y cuál no — sin modificar código en
  esta tarea
- [X] T013 [US3] Corregir o completar únicamente los gaps reales identificados en T012 en
  `session-bill-panel.component.ts`/`split-bill-panel.component.ts` (por ejemplo, algún dato del
  detalle de cuenta que falte mostrar, o algún caso de cobro combinado no cubierto) — sin
  reescribir la lógica de reparto (`AssignRow.units[]`) ni la de cobro (`PaymentInputComponent`/
  `payment-draft.util.ts`), que ya funcionan; depende de T012

### Tests para User Story 3

- [X] T014 [P] [US3] Crear
  `pos-heladeria/src/app/modules/tables/components/split-bill-panel.component.spec.ts` cubriendo
  los 4 acceptance scenarios de la Historia 3 (detalle de cuenta visible sin salir del panel,
  división exclusivamente por ítem/unidad nunca porcentual, cobro combinando efectivo + otro
  método con cambio solo del excedente en efectivo, factura generada al cerrar) — depende de T013
- [X] T015 [US3] Correr
  `pos-heladeria/src/app/modules/tables/components/session-bill-panel.component.spec.ts` (ya
  existente) y confirmar que sigue en verde tras cualquier ajuste de T013, incluida la regresión ya
  cubierta del placeholder "Selecciona una mesa con consumo" — depende de T013

**Checkpoint**: las Historias 1-3 funcionan juntas de forma independiente — la Terminal de Mesas
cubre pago→cocina, cambio visible, y cuenta de mesa dividida/multi-pago con factura.

---

## Phase 6: User Story 4 - El personal lee la comanda con claridad, en tablet o en escritorio (Priority: P3)

**Goal**: subir el tamaño de texto/controles de los tres paneles a los mínimos medibles de
FR-010/FR-011 (texto ≥16px equivalente, controles táctiles ≥44×44pt) y asegurar que ningún estado
dependa solo del color (FR-003), sin romper ninguna información ya visible (FR-012).

**Independent Test**: mostrar la Terminal de Mesas a personal nuevo en tablet y en escritorio y
verificar que identifica mesa, productos, total y estado sin zoom ni ayuda, que los controles se
tocan sin error, y que los estados se distinguen incluso en escala de grises.

### Implementación para User Story 4

- [X] T016 [P] [US4] En
  `pos-heladeria/src/app/modules/tables/components/pos-tables-panel.component.ts`, subir las clases
  Tailwind de texto esencial (nombre de mesa, total, estado) de `text-xs`/`text-sm` a `text-base`
  como mínimo (`text-lg`/`text-xl`/`font-bold` para total y estado), los controles de acción a
  `min-h-11 min-w-11`, y verificar/agregar una etiqueta de texto o ícono junto a cualquier estado de
  mesa que hoy dependa solo del color (research.md Decisiones 8-9) — sin dependencia de otras
  tareas, este componente no lo toca ninguna otra historia
- [X] T017 [P] [US4] Mismo ajuste de tamaño y de etiqueta/ícono no dependiente del color en
  `pos-heladeria/src/app/modules/tables/components/pending-orders-panel.component.ts` — depende de
  T007/T008 (US1) para no chocar con esos cambios en el mismo fichero
- [X] T018 [P] [US4] Mismo ajuste en
  `pos-heladeria/src/app/modules/tables/components/payment-attempt-review-panel.component.ts` —
  depende de T010 (US2) para no chocar con esos cambios en el mismo fichero
- [X] T019 [P] [US4] Mismo ajuste en
  `pos-heladeria/src/app/modules/tables/components/session-bill-panel.component.ts` — depende de
  T013 (US3)
- [X] T020 [P] [US4] Mismo ajuste en
  `pos-heladeria/src/app/modules/tables/components/split-bill-panel.component.ts` — depende de T013
  (US3)

### Validación manual para User Story 4

- [ ] T021 [US4] Ejecutar el checklist manual de `quickstart.md` (Historia 4) sobre el build
  servido (`npm start` en `pos-heladeria`) en un viewport de tablet y uno de escritorio: tamaño de
  texto computado ≥16px en DevTools, controles ≥44×44px, estados distinguibles bajo un filtro de
  escala de grises, y ninguna información antes visible desapareció (FR-012) — depende de
  T016-T020

**Checkpoint**: las 4 historias funcionan juntas — la Terminal de Mesas queda legible en tablet y
escritorio sin perder ninguna funcionalidad ni información previa.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de no-regresión sobre todo lo tocado.

- [X] T022 [P] Correr `python -m unittest discover app/characterization_tests -v` en
  `pos-backend` completo y confirmar que ningún módulo distinto de `test_orders_checkout.py`
  (T005) cambia de resultado esperado — en particular `test_table_sessions_service.py` y
  `test_table_sessions_split_blindaje.py` deben seguir exactamente igual, sin tocar (research.md
  Decisión 7)
- [X] T023 Ejecutar `quickstart.md` completo (las 4 historias + la sección de Regresión general) de
  principio a fin como validación final antes de dar la funcionalidad por completa (Constitución,
  Principio X)
- [X] T024 [P] Revisión cruzada de FR-012 sobre los 5 componentes tocados
  (`pending-orders-panel`, `payment-attempt-review-panel`, `pos-tables-panel`,
  `session-bill-panel`, `split-bill-panel`): confirmar que ítems, notas por producto, método de
  pago y estado del pedido siguen todos visibles, ninguno se perdió al subir tamaños ni al quitar
  el botón "Confirmar"

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede arrancar de inmediato.
- **Foundational (Phase 2)**: vacía — no bloquea nada (ver explicación en la fase).
- **User Stories (Phase 3-6)**: todas pueden arrancar apenas termina el Setup (Phase 1). US1, US2 y
  US3 son independientes entre sí a nivel de código (ficheros disjuntos); US4 toca los mismos
  ficheros que US1/US2/US3 tocan primero (ver "Dependencias entre historias" abajo).
- **Polish (Phase 7)**: depende de que las historias que se vayan a entregar ya estén completas.

### Dependencias entre historias

- **US1 (P1)**: sin dependencia de otra historia — el MVP mínimo de esta spec.
- **US2 (P1)**: sin dependencia de código de US1 (toca un componente distinto), pero comparte el
  mismo objetivo de negocio ("el flujo de confirmación de pago queda claro") — se recomienda
  implementarla junto con US1 antes de entregar, aunque técnicamente es independiente.
- **US3 (P2)**: sin dependencia de US1/US2 — toca únicamente los componentes de "Cuenta de la
  mesa".
- **US4 (P3)**: independiente en su *objetivo* (legibilidad), pero sus tareas T017-T020 tocan los
  mismos ficheros que US1 (T007/T008), US2 (T010) y US3 (T013) modifican primero — por eso quedan
  con esa dependencia de fichero explícita, no de comportamiento, para evitar conflictos de merge
  (Notes: "Avoid... same file conflicts"). Solo T016 (`pos-tables-panel`, no tocado por ninguna
  otra historia) es realmente independiente desde el arranque.

### Dentro de cada historia

- Tests (T005-T006, T009, T011, T014-T015) se escriben junto con o inmediatamente después de la
  implementación que verifican — este proyecto usa characterization tests que documentan
  comportamiento real, no TDD estricto (mismo patrón que spec 024).
- Backend antes que frontend dentro de US1 (T002-T004 antes de T007-T009), porque el frontend deja
  de llamar un endpoint que antes llamaba manualmente — depende de que el backend ya garantice el
  efecto combinado.

### Parallel Opportunities

- T005 y T006 (tests de US1) en paralelo entre sí, tras T002-T004.
- T016, T017, T018, T019, T020 (las 5 tareas de tamaño de US4) en paralelo entre sí — cada una es
  un fichero distinto, aunque cada una individualmente depende de que su historia correspondiente
  ya haya tocado ese mismo fichero (ver arriba).
- T022 y T024 (Polish) en paralelo entre sí.

---

## Parallel Example: User Story 1

```bash
# Tras T002-T004 (backend fusionado), en paralelo:
Task: "Actualizar CONGELA de confirm_order en pos-backend/app/characterization_tests/test_orders_checkout.py"
Task: "Agregar casos nuevos en pos-backend/app/characterization_tests/test_orders_payment_gate.py"
```

## Parallel Example: User Story 4

```bash
# Tras T007/T008 (US1), T010 (US2) y T013 (US3) ya mergeados, en paralelo:
Task: "Ajustar tamaño/etiquetas en pos-tables-panel.component.ts"
Task: "Ajustar tamaño/etiquetas en pending-orders-panel.component.ts"
Task: "Ajustar tamaño/etiquetas en payment-attempt-review-panel.component.ts"
Task: "Ajustar tamaño/etiquetas en session-bill-panel.component.ts"
Task: "Ajustar tamaño/etiquetas en split-bill-panel.component.ts"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup (T001).
2. Phase 2: Foundational — vacía, sin esperar nada.
3. Completar Phase 3: User Story 1 (T002-T009).
4. **DETENER y VALIDAR**: correr el bloque de Historia 1 de `quickstart.md` de forma independiente.
5. Esto ya resuelve el defecto central reportado ("pago verificado, pedido no pasó a la mesa").

### Incremental Delivery

1. Setup → Historia 1 (T002-T009) → validar → esto ya es el MVP que corrige el flujo roto.
2. Agregar Historia 2 (T010-T011) → validar → corrige el defecto del cambio en efectivo.
3. Agregar Historia 3 (T012-T015) → validar → cuenta de mesa dividida/multi-pago clara.
4. Agregar Historia 4 (T016-T021) → validar → legibilidad transversal, cierre de la spec.
5. Phase 7 (T022-T024) → verificación final de no-regresión antes de dar la spec por completa.

### Parallel Team Strategy

Con más de una persona: una vez completado el Setup, US1, US2 y US3 pueden repartirse entre
personas distintas de inmediato (ficheros disjuntos); US4 se reparte igual, pero cada una de sus
tareas de tamaño debe esperar a que la historia dueña de ese fichero (US1/US2/US3) termine su
propio cambio, para evitar tocar el mismo fichero a la vez.

---

## Notes

- [P] tasks = ficheros distintos, sin dependencias entre sí.
- [Story] mapea cada tarea a su historia de usuario para trazabilidad (Constitución, Principio
  XII).
- Cero tablas, columnas, endpoints o dependencias nuevas en toda esta spec (plan.md, data-model.md)
  — todas las tareas son edición de ficheros ya existentes.
- El único characterization test CONGELA que se toca es `test_orders_checkout.py` (T005), citando
  esta spec como autorización (Constitución, Principio III) — ningún otro CONGELA se modifica.
- Verificar `quickstart.md` tras cada historia, no solo al final.
- Evitar: reescribir `session-bill-panel`/`split-bill-panel` desde cero (research.md Decisión 7);
  reintroducir un botón "Confirmar" "por si acaso" en `pending-orders-panel` (contradice FR-001).
