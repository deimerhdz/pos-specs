---

description: "Task list for spec 056 — tipo de orden Domicilio"
---

# Tasks: Habilitación del tipo de orden "Domicilio" en la creación manual de pedidos

**Input**: Design documents from `/specs/056-domicilio-orden-manual/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/orders-create.md,
contracts/orders-checkout-total.md, quickstart.md

**Tests**: incluidos — el proyecto exige mantener en verde sus characterization tests (Principio
III/X de la constitución), y spec 055 (el precedente directo de esta misma pantalla) los incluyó
como parte del trabajo, no como opcional.

**Organización**: por historia de usuario de spec.md (US1/US2/US3, las tres P1, en el orden en que
aparecen en spec.md). El código vive en los repositorios hermanos `../pos-backend` y
`../pos-heladeria` (no en `pos-specs`).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: US1, US2 o US3 — solo en fases de historia de usuario

---

## Phase 1: Setup

- [X] T001 Confirmar el head real de Alembic en `pos-backend/alembic/versions/` (`alembic heads`
      dentro de `pos-backend`) para encadenar `down_revision` de la migración nueva
      (research.md Decisión 10)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Propósito**: dejar el modelo de datos, los schemas de API y los tipos/signals de frontend en un
estado consistente y ya verificado en verde, **antes** de construir ninguna historia de usuario
nueva. Ninguna historia (US1/US2/US3) puede probarse de forma independiente sin estas columnas y
campos ya existiendo.

- [X] T002 Modificar `CustomerOrder` en `pos-backend/app/models/customer_order.py`: columnas nuevas
      `delivery_address` (`String(255)`, nulable), `delivery_phone` (`String(30)`, nulable),
      `delivery_fee` (`Numeric(12, 2)`, nulable, sin default), y `CheckConstraint`
      `ck_customer_order_delivery_fee_non_negative`
      (`"delivery_fee IS NULL OR delivery_fee >= 0"`) (data-model.md, research.md Decisión 1-3)
- [X] T003 [P] Modificar `Sale` en `pos-backend/app/models/sale.py`: columna nueva `delivery_fee`
      (`Numeric(12, 2)`, nulable, sin default), mismo patrón que `discount`/`tax`/`tip`
      (data-model.md, research.md Decisión 5)
- [X] T004 [P] Actualizar `pos-backend/app/api/v1/orders/schemas.py`: `OrderCreate` gana
      `delivery_address: str | None = None`, `delivery_phone: str | None = None`,
      `delivery_fee: Decimal | None = None`; `OrderResponse` gana los mismos tres campos como
      salida (data-model.md, contracts/orders-create.md)
- [X] T005 Escribir la migración Alembic nueva en
      `pos-backend/alembic/versions/<rev>_domicilio_orden_manual.py`, encadenada al head de T001,
      siguiendo el patrón `@for_each_tenant_schema` + `_has_table(schema, "customer_orders")` /
      `_has_table(schema, "sales")`: agrega las 3 columnas + el `CheckConstraint` a
      `customer_orders`, agrega `delivery_fee` a `sales` — sin backfill (columnas puramente
      aditivas, ningún pedido histórico es `DELIVERY`); `downgrade()` simétrico (elimina
      constraint + 4 columnas) (research.md Decisión 10, data-model.md). Depende de T002, T003.
- [X] T006 Aplicar la migración en un entorno de desarrollo (`alembic upgrade head` en
      `pos-backend`) y confirmar manualmente que las 4 columnas nuevas existen y que
      `delivery_fee` negativo es rechazado por el `CheckConstraint`. Depende de T005.
- [X] T007 [P] Actualizar `pos-heladeria/src/app/modules/tables/interfaces/dining.interface.ts`:
      `OrderCreatePayload` gana `delivery_address?: string | null; delivery_phone?: string | null;
      delivery_fee?: number | null;`; `DiningOrder` gana los mismos tres campos como opcionales de
      respuesta (data-model.md)
- [X] T008 [P] Agregar en `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`
      tres signals nuevos junto a `customerName` (línea ~314): `readonly deliveryAddress =
      signal('')`, `readonly deliveryPhone = signal('')`, `readonly deliveryFee = signal<number |
      null>(null)` (research.md Decisión 7) — sin wiring todavía a `totals()` ni a
      `createManualOrderFromDraft()` (eso ocurre en US1/US3)
- [X] T009 Ejecutar la suite completa de tests de `pos-backend` (`pytest
      app/characterization_tests/`) y de `pos-heladeria` (`ng test`) y confirmar 100% en verde
      antes de continuar (Principio III/X — gate obligatorio antes de tocar cualquier historia de
      usuario). Depende de T002-T008.

**Checkpoint**: modelo de datos, schemas de API, tipos y signals de frontend listos; sin ningún
cambio de comportamiento observable todavía (nada los usa aún); todos los tests existentes en
verde. Las historias de usuario pueden empezar.

---

## Phase 3: User Story 1 - Crear un pedido "Domicilio" con sus datos de entrega (Priority: P1) 🎯 MVP

**Goal**: habilitar la pestaña "🛵 Domicilio" en el panel de tipo de orden de la creación de orden
manual; sin mesa, con los campos "Cliente" (vacío, sin valor por defecto), "Dirección", "Teléfono"
y "Valor del domicilio" visibles; al confirmar, la orden queda `channel=POS`,
`order_type=DELIVERY`, sin mesa asociada, con esos cuatro datos guardados.

**Independent Test**: abrir la creación de orden manual, seleccionar "Domicilio", diligenciar
cliente/dirección/valor del domicilio (con o sin teléfono), agregar productos, confirmar sin
elegir mesa, y verificar que la orden se crea correctamente con esos datos (quickstart.md
Escenario 1).

### Tests for User Story 1

- [X] T010 [P] [US1] Test backend: `POST /orders` con `order_type: "DELIVERY"` y
      `customer_name`/`delivery_address`/`delivery_fee` completos (con y sin `delivery_phone`)
      devuelve `201` con los cuatro campos reflejados en la respuesta — nuevo caso en
      `pos-backend/app/characterization_tests/test_orders_service.py`
      (contracts/orders-create.md)
- [X] T011 [P] [US1] Test frontend en
      `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.spec.ts`: reescribir
      el caso existente que afirma "Domicilio sigue deshabilitado" (heredado de spec 055) para
      confirmar que la pestaña "🛵 Domicilio" ya **no** está `disabled` y puede seleccionarse;
      agregar casos nuevos: al seleccionarla se oculta el bloque "Mesas", y se muestran los campos
      "Cliente" (vacío, **sin** `readOnly` ni "Consumidor final"), "Dirección" (vacío), "Teléfono"
      (vacío) y "Valor del domicilio" (vacío) (research.md Decisión 8, 11)
- [X] T012 [P] [US1] Test frontend en
      `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.spec.ts`:
      `createManualOrderFromDraft()` con `orderTypeTab() === 'domicilios'` no exige
      `selectedTableId()` y envía `{ channel: 'POS', order_type: 'DELIVERY', dining_table_id: null,
      delivery_address, delivery_phone, delivery_fee }` leídos de los signals correspondientes
      (research.md Decisión 7)

### Implementation for User Story 1

- [X] T013 [US1] En `create_order` (`pos-backend/app/api/v1/orders/service.py`): pasar
      `delivery_address`, `delivery_phone`, `delivery_fee` de `data` (`OrderCreate`) como kwargs a
      la construcción de `CustomerOrder(...)` (~línea 200-213) (research.md Decisión 4,
      data-model.md). Contribuye a hacer pasar T010 (junto con T021 de US2 para el caso con
      validación). Depende de T002, T004.
- [X] T014 [US1] En
      `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`: habilitar el
      botón "🛵 Domicilio" (quitar `disabled`/`title="Todavía no disponible..."`, líneas ~136-143),
      conectado a `store.setOrderTypeTab('domicilios')`, con la misma clase activa condicionada a
      `store.orderTypeTab()` que ya usan "En Mesa"/"Para Llevar" (research.md Decisión 6, 8)
- [X] T015 [US1] En el mismo archivo: extender la condición que oculta el bloque "Mesas" (buscador
      de mesa, spec 053) para que también se oculte con `orderTypeTab() === 'domicilios'` (hoy solo
      se oculta con `'para-llevar'`)
- [X] T016 [US1] En el mismo archivo: agregar los campos "Cliente" (input simple siempre editable,
      sin el patrón de solo-lectura+botón editar de "En Mesa"/"Para Llevar"), "Dirección",
      "Teléfono" y "Valor del domicilio" (input numérico, `min="0"`), visibles solo con `@if
      (store.orderTypeTab() === 'domicilios')`, enlazados a `store.customerName`,
      `store.deliveryAddress`, `store.deliveryPhone`, `store.deliveryFee` respectivamente
      (research.md Decisión 8). Depende de T008.
- [X] T017 [US1] En `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`:
      `createManualOrderFromDraft()` — agregar `const esDomicilio = this.orderTypeTab() ===
      'domicilios'`; extender el guard de mesa a `if ((!esParaLlevar && !esDomicilio && !tableId)
      || this.draftLines().length === 0) return false`; extender el payload de
      `api.createManualOrder(...)` con `order_type: esDomicilio ? 'DELIVERY' : esParaLlevar ?
      'TAKEAWAY' : 'DINE_IN'`, `dining_table_id: (esParaLlevar || esDomicilio) ? null : tableId`,
      `delivery_address: esDomicilio ? this.deliveryAddress().trim() : null`, `delivery_phone:
      esDomicilio ? (this.deliveryPhone().trim() || null) : null`, `delivery_fee: esDomicilio ?
      this.deliveryFee() : null` (research.md Decisión 7). Hace pasar T012. Depende de T007, T008.
- [ ] T018 [US1] Ejecutar manualmente quickstart.md Escenario 1 completo (UI) y verificar en base
      de datos el resultado (`channel='POS'`, `order_type='DELIVERY'`, `dining_table_id IS NULL`,
      `customer_name`/`delivery_address`/`delivery_phone`/`delivery_fee` correctos)

**Checkpoint**: "Domicilio" es un flujo completo y funcional de punta a punta (creación con datos
correctos), verificable de forma independiente — aún sin la validación de campos obligatorios de
US2 ni la suma al total facturado de US3.

---

## Phase 4: User Story 2 - Impedir confirmar un pedido "Domicilio" incompleto (Priority: P1)

**Goal**: el sistema bloquea la confirmación de un pedido "Domicilio" si falta el nombre del
cliente, la dirección, o el valor del domicilio — tanto en la UI (botón deshabilitado) como en el
backend (`422`), sin bloquear nunca por falta de teléfono.

**Independent Test**: seleccionar "Domicilio", dejar vacío el nombre del cliente (o la dirección, o
el valor del domicilio), intentar confirmar, y verificar que el sistema bloquea el envío y señala
el campo faltante; con los tres presentes y el teléfono vacío, verificar que sí se puede confirmar
(quickstart.md Escenario 2). No depende de que US3 esté implementada.

### Tests for User Story 2

- [X] T019 [P] [US2] Test backend: `POST /orders` con `order_type: "DELIVERY"` y, por separado,
      `customer_name`/`delivery_address`/`delivery_fee` faltante devuelve `422`; con los tres
      presentes y `delivery_phone` ausente devuelve `201` — nuevos casos en
      `pos-backend/app/characterization_tests/test_orders_service.py` (data-model.md, "Reglas de
      validación")
- [X] T020 [P] [US2] Test frontend en `manual-order-page.component.spec.ts`: con "Domicilio"
      seleccionado, el botón "Confirmar y Enviar" está deshabilitado si "Cliente", "Dirección" o
      "Valor del domicilio" están vacíos (uno por uno), y habilitado si solo falta "Teléfono"
      (research.md Decisión 8)

### Implementation for User Story 2

- [X] T021 [US2] En `create_order` (`pos-backend/app/api/v1/orders/service.py`), inmediatamente
      después del guard de mesa existente (~línea 146): agregar el rechazo `422` cuando
      `data.order_type is OrderType.DELIVERY` y falta `customer_name`, `delivery_address`, o
      `delivery_fee` (no vacíos tras `.strip()` para los textos, no `None` para el valor) — mensaje
      explícito indicando qué falta; `delivery_phone` queda fuera de esta condición (research.md
      Decisión 4). Hace pasar T019. Depende de T013.
- [X] T022 [US2] En `manual-order-page.component.ts`: modificar `applyDefaultCustomerName()`
      (líneas ~312-317) para que retorne de inmediato (sin rellenar "Consumidor final") cuando
      `store.orderTypeTab() === 'domicilios'` — sin este cambio, el valor por defecto se aplicaría
      silenciosamente justo antes de enviar (dentro de `confirm()`, línea ~328), anulando FR-003
      (research.md Decisión 8, hallazgo crítico). Depende de T014.
- [X] T023 [US2] En el mismo archivo: extender el `[disabled]` del botón "Confirmar y Enviar"
      (líneas ~234-238) con el disyunto `(store.orderTypeTab() === 'domicilios' && (
      !store.customerName().trim() || !store.deliveryAddress().trim() || store.deliveryFee() ==
      null))` (research.md Decisión 8). Hace pasar T020. Depende de T016.
- [X] T024 [US2] En `pos-terminal.store.ts`: dentro de `createManualOrderFromDraft()`, agregar un
      guard defensivo adicional — `if (esDomicilio && (!this.customerName().trim() ||
      !this.deliveryAddress().trim() || this.deliveryFee() == null)) return false;` justo después
      del guard de mesa (research.md Decisión 7 — segunda capa de protección, mismo criterio que ya
      protege `tableId`). Depende de T017.
- [ ] T025 [US2] Ejecutar manualmente quickstart.md Escenario 2 (UI y `curl`) y confirmar los
      bloqueos/códigos de respuesta esperados

**Checkpoint**: un pedido "Domicilio" nunca puede confirmarse sin nombre de cliente, dirección y
valor del domicilio — verificable de forma independiente en UI y API.

---

## Phase 5: User Story 3 - El valor del domicilio se suma al total y queda facturado (Priority: P1)

**Goal**: el valor del domicilio se refleja de inmediato en el total mostrado en pantalla durante
la creación del pedido, y queda incluido en el total de la venta/factura generada al cobrar ese
pedido, en los 4 caminos de checkout existentes — sin afectar el total de ningún pedido que no sea
"Domicilio", nuevo o histórico.

**Independent Test**: escribir un valor de domicilio en una orden "Domicilio" y verificar que el
total en pantalla aumenta en esa cantidad; confirmar el pedido, facturarlo, y verificar que el
total de la factura resultante incluye ese mismo valor (quickstart.md Escenario 3); verificar que
pedidos de otro tipo no cambian su total (Escenario 4). Depende de que US1 exista (para tener un
pedido "Domicilio" que facturar), pero la lógica de cálculo de total es verificable de forma
aislada vía los tests backend de esta historia.

### Tests for User Story 3

- [X] T026 [P] [US3] Test backend: `build_sale(...)` con `delivery_fee > 0` produce
      `Sale.total = subtotal - discount + tax + tip + delivery_fee` y persiste
      `Sale.delivery_fee` — nuevo caso en
      `pos-backend/app/characterization_tests/test_orders_checkout.py`, cubriendo `pay_order` y
      `checkout_and_send` (contracts/orders-checkout-total.md)
- [X] T027 [P] [US3] Test backend en el mismo archivo: `approve_payment_attempt` y
      `confirm_cash_payment_attempt` sobre una orden `DELIVERY` — el pago autogenerado/el chequeo
      de efectivo cubren `subtotal + delivery_fee` (no solo `subtotal`) y el checkout se completa
      con éxito, sin `422` por pago insuficiente (research.md Decisión 5, punto de mayor riesgo
      encontrado; contracts/orders-checkout-total.md)
- [X] T028 [P] [US3] Test frontend en `pos-terminal.store.spec.ts`: `totals()` incluye
      `deliveryFee` (leído de `store.deliveryFee()`) sumado al `total` únicamente cuando
      `orderTypeTab() === 'domicilios'`; con `'mesas'`/`'para-llevar'` el total no cambia
      (research.md Decisión 7)

### Implementation for User Story 3

- [X] T029 [US3] En `pos-backend/app/api/v1/sales/builder.py::build_sale`: agregar parámetro
      `delivery_fee: Decimal = Decimal("0")`; extender la fórmula de la línea 132 a `total =
      subtotal - Decimal(discount) + Decimal(tax) + Decimal(tip) + Decimal(delivery_fee)`; pasar
      `delivery_fee=delivery_fee` a la construcción de `Sale(...)` (research.md Decisión 5). Hace
      pasar T026. Depende de T003.
- [X] T030 [US3] En `pos-backend/app/api/v1/orders/checkout.py::_order_total()` (líneas ~784-792):
      sumar `order.delivery_fee or Decimal("0")` al resultado — usado por el chequeo previo de
      efectivo en `confirm_cash_payment_attempt` (research.md Decisión 5)
- [X] T031 [US3] En `checkout.py::approve_payment_attempt` (línea ~874): sumar `order.delivery_fee
      or Decimal("0")` al `total` computado localmente **antes** de construir el `PaymentIn`
      automático que se le pasa a `build_sale` — sin este cambio, el pago autogenerado queda corto
      exactamente en el valor del domicilio y `build_sale` lo rechazaría con `422` (research.md
      Decisión 5, hallazgo crítico). Hace pasar la mitad de T027. Depende de T029.
- [X] T032 [US3] En `checkout.py`: actualizar las 4 llamadas a `build_sale(...)` (`pay_order`
      ~línea 280, `checkout_and_send` ~línea 471, `approve_payment_attempt` ~línea 876,
      `confirm_cash_payment_attempt` ~línea 995) para pasar `delivery_fee=order.delivery_fee or
      Decimal("0")` (research.md Decisión 5). Hace pasar T026 y la otra mitad de T027. Depende de
      T029, T030, T031.
- [X] T033 [US3] En `pos-terminal.store.ts`: extender `totals` (líneas ~792-801) con `const
      deliveryFee = this.orderTypeTab() === 'domicilios' ? (this.deliveryFee() ?? 0) : 0;` sumado
      al cálculo de `total`, y expuesto en el objeto devuelto (research.md Decisión 7). Hace pasar
      T028. Depende de T008.
- [X] T034 [US3] En `manual-order-page.component.ts`: agregar una fila "Domicilio" al bloque de
      totales (líneas ~221-230), leyendo `tot.deliveryFee`, antes de la fila "Total" — visible
      cuando la pestaña activa es `'domicilios'` (research.md Decisión 8). Depende de T033.
- [ ] T035 [US3] Ejecutar manualmente quickstart.md Escenario 3 (total facturado incluye el
      domicilio, en particular el camino de aprobación por transferencia) y Escenario 4 (sin efecto
      sobre pedidos que no son "Domicilio", ni sobre ventas ya emitidas)

**Checkpoint**: el valor del domicilio queda reflejado en el total en pantalla y en la factura
final, en los 4 caminos de checkout, sin afectar ningún otro tipo de pedido ni venta histórica —
verificable de forma independiente.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [ ] T036 Ejecutar los 4 escenarios de quickstart.md de punta a punta como validación final
- [X] T037 Ejecutar la suite completa de tests de `pos-backend` y `pos-heladeria` una vez más,
      confirmando 100% en verde (Principio X)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: depende de Setup (T001, para T005). Bloquea todas las historias.
- **US1 (Phase 3)**: depende solo de Foundational — es la historia base (habilita la pestaña y crea
  el pedido con los datos correctos).
- **US2 (Phase 4)**: depende de Foundational y de partes específicas de US1 (T013 para el backend,
  T014/T016 para el frontend) — no puede validarse "campo obligatorio" antes de que el campo exista
  en pantalla/backend.
- **US3 (Phase 5)**: depende de Foundational; su implementación de backend (T029-T032) es
  independiente de US1/US2 y puede avanzar en paralelo con ellas, pero su verificación manual
  completa (T035) necesita poder crear un pedido "Domicilio" real (US1) para facturarlo.
- **Polish (Phase 6)**: depende de que las tres historias estén completas.

### Dentro de Foundational

T001 → T005 (necesita el head real). T002, T003 → T005 (la migración refleja las columnas ya
definidas en los modelos). T005 → T006. T002-T008 → T009 (gate final).

### Dentro de cada historia

- US1: T010-T012 (tests) antes de T013-T017 (implementación) — T013 contribuye a T010 (con T021 de
  US2 para el rechazo); T014-T016 hacen pasar T011; T017 hace pasar T012. T018 al final.
- US2: T019-T020 (tests) antes de T021-T024 (implementación) — T021 hace pasar T019; T022-T023
  hacen pasar T020; T024 es protección adicional sin test dedicado propio (cubierta indirectamente
  por T012/T020). T025 al final.
- US3: T026-T028 (tests) antes de T029-T034 (implementación) — T029, T031, T032 hacen pasar T026 y
  T027; T033 hace pasar T028; T034 es la fila visual, sin test dedicado (cubierta por T028 a nivel
  de store). T035 al final.

### Parallel Opportunities

- Foundational: T003, T004 en paralelo con T002 (archivos distintos); T007, T008 en paralelo entre
  sí y con T002-T004.
- US1: T010, T011, T012 en paralelo entre sí (archivos de test distintos, backend vs. dos archivos
  de frontend).
- US2: T019, T020 en paralelo entre sí.
- US3: T026, T027, T028 en paralelo entre sí (T026/T027 mismo archivo backend pero casos
  independientes; T028 archivo de frontend distinto).
- Con más de una persona disponible: una vez terminado Foundational, US1 (frontend) y el backend de
  US3 (T029-T032, no depende de UI) pueden trabajarse en paralelo; US2 debe esperar a que al menos
  T013/T014/T016 de US1 existan.

---

## Parallel Example: Foundational

```bash
# En paralelo, tras T001:
Task: "Modificar CustomerOrder en pos-backend/app/models/customer_order.py (T002)"
Task: "Modificar Sale en pos-backend/app/models/sale.py (T003)"
Task: "Actualizar OrderCreate/OrderResponse en orders/schemas.py (T004)"

# En paralelo, tras T002-T004:
Task: "Actualizar dining.interface.ts (T007)"
Task: "Agregar signals nuevos en pos-terminal.store.ts (T008)"
```

## Parallel Example: User Story 1

```bash
Task: "Test backend: POST /orders DELIVERY completo (T010)"
Task: "Test frontend: pestaña Domicilio habilitada + campos nuevos (T011)"
Task: "Test frontend: payload DELIVERY en pos-terminal.store.spec.ts (T012)"
```

## Parallel Example: User Story 3

```bash
Task: "Test backend: build_sale con delivery_fee, pay_order/checkout_and_send (T026)"
Task: "Test backend: approve_payment_attempt/confirm_cash_payment_attempt con delivery_fee (T027)"
Task: "Test frontend: totals() con deliveryFee solo en pestaña Domicilio (T028)"
```

---

## Implementation Strategy

### MVP First (Foundational + User Story 1)

1. Completar Phase 1 (Setup) y Phase 2 (Foundational) — sin esto no hay base segura para nada.
2. Completar Phase 3 (US1) — entrega el resultado de negocio explícitamente pedido ("habilitar
   Domicilio").
3. **Detener y validar**: quickstart.md Escenario 1.
4. US2 (protección de datos incompletos) y US3 (el domicilio efectivamente se cobra) son ambas P1
   y necesarias para que el MVP sea utilizable en producción sin riesgo — no se recomienda entregar
   US1 solo, sin al menos US2, fuera de un entorno de prueba.

### Incremental Delivery

1. Foundational → base lista, sin cambio de comportamiento observable, tests en verde.
2. + US1 → "Domicilio" crea pedidos completos → validar → demo.
3. + US2 → pedidos incompletos ya no pueden confirmarse → validar.
4. + US3 → el valor del domicilio se cobra de verdad, en los 4 caminos de checkout → validar
   (especial atención al camino de aprobación por transferencia, research.md Decisión 5).
5. + Polish.

---

## Notes

- Ningún task de este documento agrega dependencias nuevas ni toca `pos-tables-panel.component.ts`
  (research.md Decisión 6) ni recalcula ninguna venta/factura ya emitida (Principio VII).
- T031 (el `total` local de `approve_payment_attempt`) es el punto más sensible de todo el plan
  (research.md Decisión 5): sin este task, US3 parecería funcionar en los caminos de pago directo
  (`pay_order`, `checkout_and_send`, `confirm_cash_payment_attempt`) pero fallaría con `422` real
  al aprobar cualquier pago por transferencia de una orden "Domicilio" — confirmar T027 en verde
  específicamente, no solo T026.
- T022 (excepción de `applyDefaultCustomerName()` para Domicilio) es el segundo punto más sensible:
  sin él, FR-003 queda anulado en silencio por código que ya existe, no por código ausente.
- Commitear después de cada tarea o grupo lógico; detenerse en cada checkpoint para validar la
  historia de forma independiente antes de continuar con la siguiente.
