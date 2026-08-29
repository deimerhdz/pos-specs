---

description: "Task list for spec 055 — canal y tipo de orden"
---

# Tasks: Estandarización de canal y tipo de orden — habilitación de pedidos "Para Llevar"

**Input**: Design documents from `/specs/055-canal-tipo-orden/`
**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/orders-create.md, quickstart.md

**Tests**: incluidos — el proyecto exige mantener en verde sus characterization tests (Principio
III/X de la constitución) y todas las specs anteriores (051-054) los incluyen como parte del
trabajo, no como opcional.

**Organización**: por historia de usuario de spec.md (US1/US2/US3, en orden de prioridad). El
código vive en los repositorios hermanos `../pos-backend` y `../pos-heladeria` (no en `pos-specs`).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo distinto, sin dependencias pendientes)
- **[Story]**: US1, US2 o US3 — solo en fases de historia de usuario

---

## Phase 1: Setup

- [X] T001 Confirmar el head real de Alembic en `pos-backend/alembic/versions/` (`alembic heads`
      dentro de `pos-backend`) para encadenar `down_revision` de la migración nueva
      (research.md D7 — el historial tenía múltiples heads sueltos al momento de planear)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Propósito**: dejar el modelo de datos, los schemas y todos los puntos que hoy dependen de los
valores libres de `channel` en un estado consistente y ya verificado en verde, **antes** de
construir ninguna historia de usuario nueva. Ninguna historia (US1/US2/US3) puede empezar sin esto.

**⚠️ CRÍTICO**: el rename `counter`/`waiter` → `POS` y el fix de `is_consolidation_order`
(research.md Decisión 2) no son opcionales ni pueden diferirse — sin ellos, fusionar el canal
introduce un bug de facturación real (reabrir una comanda de mostrador ya cobrada). Deben quedar
completos y con tests en verde antes de tocar cualquier historia de usuario.

- [X] T002 Modificar `CustomerOrder` en `pos-backend/app/models/customer_order.py`: nuevo
      `CheckConstraint` de `channel` (`'POS'`, `'QR_MENU'`, `'WHATSAPP'`, `'API'`), columna nueva
      `order_type` (`String(10)`, nulable, `CheckConstraint` con `'DINE_IN'`/`'TAKEAWAY'`/
      `'DELIVERY'`), columna nueva `is_consolidation_order` (`Boolean`, `NOT NULL`,
      `server_default='false'`), e índices `idx_customer_orders_channel` e
      `idx_customer_orders_order_type` (data-model.md)
- [X] T003 [P] Actualizar `pos-backend/app/api/v1/orders/schemas.py`: `OrderChannel` con los 4
      valores nuevos (`POS`, `QR_MENU`, `WHATSAPP`, `API`), `OrderType(str, Enum)` nuevo
      (`DINE_IN`, `TAKEAWAY`, `DELIVERY`), `OrderCreate.channel` default `OrderChannel.POS`,
      `OrderCreate.order_type: OrderType = OrderType.DINE_IN` nuevo,
      `OrderResponse.order_type: str | None = None` nuevo (data-model.md, contracts/orders-create.md)
- [X] T004 Escribir la migración Alembic nueva en
      `pos-backend/alembic/versions/<rev>_estandariza_canal_tipo_orden.py`, encadenada al head de
      T001, siguiendo el patrón `@for_each_tenant_schema` + `_has_table(schema, "products")`:
      agrega columnas → backfill (`order_type` según `dining_table_id`,
      `is_consolidation_order` según `channel = 'waiter'`) → remapeo de `channel` (`CASE`) →
      reemplaza `CheckConstraint` de `channel` + agrega el de `order_type` + los 2 índices; con
      `downgrade()` completo en orden inverso (research.md D7, data-model.md)
- [X] T005 Aplicar la migración en un entorno de desarrollo (`alembic upgrade head` en
      `pos-backend`) contra un tenant con datos de prueba y confirmar manualmente: sin valores
      libres de `channel` remanentes, `order_type` poblado según `dining_table_id`
      (quickstart.md Escenario 4)
- [X] T006 [P] Renombrar comparaciones `OrderChannel.QR` → `OrderChannel.QR_MENU` en
      `pos-backend/app/api/v1/orders/service.py` (líneas ~121 y ~155) — sin cambio de lógica
      (research.md D2)
- [X] T007 [P] En `pos-backend/app/api/v1/cart/service.py`: renombrar el filtro de canal de
      mostrador/mesero en la línea 589 de `channel.in_(("counter", "waiter"))` a
      `channel == "POS"` (sin cambio de conjunto resultante), y en la construcción del
      `CustomerOrder(...)` de `submit_cart` (el pedido que el comensal envía desde el menú QR)
      agregar `order_type="DINE_IN"` explícito — hoy no se fija ningún valor, y la columna no
      tiene default de base de datos (data-model.md), así que sin esto todo pedido nuevo por QR
      quedaría con `order_type` vacío, violando FR-015 (research.md D2, D4)
- [X] T008 Corregir `get_or_create_open_order` en
      `pos-backend/app/api/v1/orders/consolidation.py` (líneas ~83 y ~93): reemplazar el filtro y
      la creación por `channel == "waiter"` por `channel == "POS"` +
      `is_consolidation_order == True` / `is_consolidation_order=True` — mismo conjunto de
      órdenes encontradas/creadas que antes (research.md D2) — y agregar `order_type="DINE_IN"`
      explícito a esa misma construcción de `CustomerOrder(...)`: hoy no se fija ningún valor, y
      sin esto todo pedido nuevo abierto por el mesero (consolidación / ítem directo a mesa)
      quedaría con `order_type` vacío, violando FR-015 (research.md D2, D4). Depende de T002.
- [X] T009 [P] Actualizar el valor por defecto de los fixtures de test
      (`kw.setdefault("channel", "waiter")` → `"POS"`) en
      `pos-backend/app/characterization_tests/orders_fixtures.py` y
      `pos-backend/app/characterization_tests/table_sessions_fixtures.py` (research.md D2)
- [X] T010 Actualizar las aserciones `assertEqual(order.channel, "waiter")` en
      `pos-backend/app/characterization_tests/test_orders_consolidation.py:241,263` a
      `assertEqual(order.channel, "POS")` + `assertTrue(order.is_consolidation_order)` — mismo
      comportamiento protegido, verificado con las dos aserciones en vez de una (research.md D2,
      Principio III). Depende de T008, T009.
- [X] T011 [P] Renombrar los literales `"waiter"`/`"counter"` restantes (valores de setup, sin
      aserción sobre el canal en sí) a `"POS"` en
      `pos-backend/app/characterization_tests/test_orders_service.py`,
      `test_orders_checkout.py`, `test_orders_kitchen.py`, `test_cart_single_active_order.py`,
      `test_scheduler.py`, `test_orders_timezone.py`, y
      `pos-backend/app/scripts/test_split_blindaje.py` (research.md D2)
- [X] T012 Ejecutar la suite completa de characterization tests de `pos-backend`
      (`pytest app/characterization_tests/`) y confirmar 100% en verde antes de continuar
      (Principio III/X — gate obligatorio antes de tocar cualquier historia de usuario)
- [X] T013 [P] Actualizar `pos-heladeria/src/app/modules/tables/interfaces/dining.interface.ts`:
      `OrderChannel` a `'POS' | 'QR_MENU' | 'WHATSAPP' | 'API'`, `OrderType` nuevo
      (`'DINE_IN' | 'TAKEAWAY' | 'DELIVERY'`), `OrderCreatePayload.order_type?: OrderType` nuevo,
      y renombrar el literal `'qr'` de `getSidebarMode` (línea ~234) a `'QR_MENU'`
      (data-model.md, research.md D6)
- [X] T014 [P] Renombrar los 3 literales `'qr'` restantes a `'QR_MENU'` en
      `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts` (líneas ~162, ~425,
      ~450) — sin cambio de lógica (research.md D2/D6)
- [X] T015 Ejecutar la suite de tests de `pos-heladeria` (`ng test`) y confirmar 100% en verde
      antes de continuar (Principio III/X — gate obligatorio). Depende de T013, T014.

**Checkpoint**: canal estandarizado de punta a punta (backend + frontend), sin ningún cambio de
comportamiento observable, todos los tests existentes en verde. Las historias de usuario pueden
empezar.

---

## Phase 3: User Story 1 - Crear un pedido "Para Llevar" sin seleccionar mesa (Priority: P1) 🎯 MVP

**Goal**: habilitar la pestaña "Para Llevar" en el panel de tipo de orden de la creación de orden
manual; sin mesa, con el campo "Cliente" (Consumidor final por defecto, editable) reusado de
spec 054; la orden queda `channel=POS`, `order_type=TAKEAWAY`, sin mesa asociada.

**Independent Test**: abrir la creación de orden manual, seleccionar "Para Llevar", agregar
productos, confirmar sin elegir mesa, y verificar que la orden se crea correctamente
(quickstart.md Escenario 1).

### Tests for User Story 1

- [X] T016 [P] [US1] Test backend: `POST /orders` con `order_type: "TAKEAWAY"` (o `"DELIVERY"`) y
      `dining_table_id` no nulo devuelve `422` — nuevo caso en
      `pos-backend/app/characterization_tests/test_orders_service.py` (quickstart.md Escenario 3)
- [X] T017 [P] [US1] Test frontend: `manual-order-page.component.spec.ts` — pestaña "Para Llevar"
      seleccionable (ya no `disabled`), oculta el bloque "Mesas", botón "Confirmar" habilitado sin
      mesa seleccionada, campo "Cliente" visible con "Consumidor final" por defecto, y el botón
      "🛵 Domicilio" **sigue** `disabled` (FR-012, no regresión)
- [X] T018 [P] [US1] Test frontend: `pos-terminal.store.spec.ts` — `createManualOrderFromDraft()`
      con `orderTypeTab() === 'para-llevar'` no exige `selectedTableId()` y envía
      `{ channel: 'POS', order_type: 'TAKEAWAY', dining_table_id: null }`; con `'mesas'` sigue
      exigiendo mesa y envía `order_type: 'DINE_IN'`

### Implementation for User Story 1

- [X] T019 [US1] Agregar en `create_order`
      (`pos-backend/app/api/v1/orders/service.py`) el rechazo `422` cuando
      `order_type in (TAKEAWAY, DELIVERY)` y `dining_table_id is not None` (research.md D5,
      data-model.md). Hace pasar T016.
- [X] T020 [US1] En `pos-heladeria/src/app/modules/tables/pages/manual-order-page.component.ts`:
      conectar el botón "🛍️ Para Llevar" a `store.setOrderTypeTab('para-llevar')` (quitar
      `disabled`/`title`), y el botón "🍽️ En Mesa" a `store.setOrderTypeTab('mesas')`; clase
      activa condicionada a `store.orderTypeTab()` en ambos (research.md D6)
- [X] T021 [US1] En el mismo archivo: ocultar el bloque "Mesas" (título + `app-searchable-select`)
      con `@if (store.orderTypeTab() === 'mesas')`
      (`manual-order-page.component.ts` ~136-145, FR-009)
- [X] T022 [US1] En el mismo archivo: cambiar el `[disabled]` del botón "Confirmar y Enviar" de
      `store.cartEmpty() || store.submitting() || !store.selectedTableId()` a
      `store.cartEmpty() || store.submitting() || (store.orderTypeTab() === 'mesas' &&
      !store.selectedTableId())` (`manual-order-page.component.ts` ~217)
- [X] T023 [US1] En el mismo archivo: llamar `applyDefaultCustomerName()` también al cambiar a la
      pestaña "Para Llevar" (mismo método ya usado en `ngOnInit`/`selectTable`/`onClienteBlur`,
      spec 054) — el campo "Cliente" queda visible y diligenciado en ambas pestañas
- [X] T024 [US1] En `pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts`:
      `createManualOrderFromDraft()` — el guard `if (!tableId || draftLines().length === 0)` pasa
      a exigir `tableId` solo si `orderTypeTab() === 'mesas'`; el payload de
      `api.createManualOrder(...)` pasa a `channel: 'POS'`, `order_type: orderTypeTab() ===
      'para-llevar' ? 'TAKEAWAY' : 'DINE_IN'`, `dining_table_id: orderTypeTab() === 'para-llevar'
      ? null : tableId` (research.md D6). Depende de T013.
- [ ] T025 [US1] Ejecutar manualmente quickstart.md Escenario 1 completo (UI) y verificar en base
      de datos el resultado (`channel='POS'`, `order_type='TAKEAWAY'`, `dining_table_id IS NULL`,
      `customer_name` correcto)

**Checkpoint**: "Para Llevar" es un flujo completo y funcional de punta a punta, verificable de
forma independiente.

---

## Phase 4: User Story 2 - Impedir combinaciones ilógicas de canal y tipo de orden (Priority: P1)

**Goal**: `POST /orders` rechaza combinaciones de canal/tipo de orden que no tienen sentido de
negocio (p. ej. `WHATSAPP` + `DINE_IN`), según la matriz de data-model.md.

**Independent Test**: llamar `POST /orders` con `channel: WHATSAPP, order_type: DINE_IN` y
verificar `400` sin registro creado; con `channel: POS, order_type: TAKEAWAY` verificar `201`
(quickstart.md Escenario 2). No depende de la UI de US1.

### Tests for User Story 2

- [X] T026 [P] [US2] Tests backend en
      `pos-backend/app/characterization_tests/test_orders_service.py`: `400` para
      `WHATSAPP+DINE_IN`, `API+DINE_IN`, `QR_MENU+TAKEAWAY`, `QR_MENU+DELIVERY`; `201`/éxito para
      `POS+DINE_IN`, `POS+TAKEAWAY`, `POS+DELIVERY`, `WHATSAPP+TAKEAWAY`, `WHATSAPP+DELIVERY`,
      `API+TAKEAWAY`, `API+DELIVERY` (data-model.md, tabla de combinaciones)

### Implementation for User Story 2

- [X] T027 [US2] En `pos-backend/app/api/v1/orders/service.py`: agregar el mapa de combinaciones
      permitidas (`dict[OrderChannel, frozenset[OrderType]]`, module-level) y la validación al
      inicio de `create_order` (mismo bloque que la validación existente de `hold_for_payment` +
      `channel is QR_MENU`) — `400` con mensaje explícito citando ambos valores recibidos cuando
      la combinación no está permitida (research.md D4, FR-006, FR-007). Hace pasar T026.
- [X] T028 [US2] Ejecutar manualmente quickstart.md Escenario 2 (curl) y confirmar los códigos de
      respuesta esperados

**Checkpoint**: la validación de combinaciones funciona de forma independiente, verificable sin
tocar la UI de "Para Llevar".

---

## Phase 5: User Story 3 - Filtrar y clasificar pedidos por canal y tipo, incluyendo el histórico (Priority: P2)

**Goal**: confirmar que tanto los pedidos históricos (reclasificados en Foundational, T004/T005)
como los nuevos (US1/US2) quedan siempre con un canal y tipo de orden del catálogo estandarizado,
listos para filtrar/reportar — y que la corrección de `consolidation.py` (Foundational, T008) no
introdujo ninguna regresión de negocio real.

**Independent Test**: consultar el conjunto completo de pedidos (históricos + nuevos) y verificar
que todos tienen canal estandarizado, y que los históricos con mesa quedaron `DINE_IN`
(quickstart.md Escenario 4). No depende de US1 ni US2.

### Tests for User Story 3

- [X] T029 [P] [US3] Test backend nuevo en
      `pos-backend/app/characterization_tests/test_orders_consolidation.py`: con una mesa que
      tiene una orden `POS`/`DINE_IN` ya cobrada (`status='abierta'` + `Sale` emitida) y una nueva
      orden de consolidación agregada después, verificar que **no** se agregan ítems a la orden ya
      cobrada — regresión guardada por `is_consolidation_order` (research.md D2,
      quickstart.md Escenario 5). Verificar además que la orden de consolidación recién
      creada/reusada tiene `order_type == "DINE_IN"` (cierra el hallazgo del análisis sobre la
      ausencia de este valor en las órdenes de mesero — research.md D4).
- [X] T030 [P] [US3] Test backend: un pedido nuevo enviado por el flujo QR (`submit_cart`) queda
      creado con `order_type == "DINE_IN"` — nuevo caso en
      `pos-backend/app/characterization_tests/test_cart_single_active_order.py` (o el archivo de
      characterization tests de `cart/service.py` que corresponda) (research.md D4)

### Implementation for User Story 3

- [X] T031 [US3] Ejecutar manualmente quickstart.md Escenario 4 contra los datos migrados en T005
      y documentar el resultado (conteo de valores de `channel` distintos, conteo de
      `order_type`/`dining_table_id` consistentes) — sin código nuevo: valida el trabajo ya hecho
      en Foundational
- [ ] T032 [US3] Ejecutar manualmente quickstart.md Escenario 5 (UI: ítem directo de mesero +
      orden de mostrador cobrada en la misma mesa) como verificación cruzada de T029

**Checkpoint**: histórico y nuevos pedidos verificablemente filtrables por canal/tipo de orden, sin
regresión en el flujo de consolidación del mesero.

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T033 [P] Revisar comentarios/docstrings en los archivos tocados
      (`consolidation.py`, `cart/service.py`, `orders/service.py`, `pos-terminal.store.ts`,
      `dining.interface.ts`) por referencias residuales a los valores de canal antiguos
      (`'qr'`/`'counter'`/`'waiter'`) y actualizarlas
- [ ] T034 Ejecutar los 5 escenarios de quickstart.md de punta a punta como validación final
- [X] T035 Ejecutar la suite completa de tests de `pos-backend` y `pos-heladeria` una vez más,
      confirmando 100% en verde (Principio X)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias.
- **Foundational (Phase 2)**: depende de Setup (T001, para T004). Bloquea todas las historias.
- **US1 (Phase 3)** y **US2 (Phase 4)**: dependen solo de Foundational — pueden avanzar en
  paralelo entre sí (tocan bloques distintos de `orders/service.py`, sin conflicto real de líneas
  si se secuencian con cuidado; ver nota en Parallel Opportunities).
- **US3 (Phase 5)**: depende solo de Foundational (T005 en particular) — es principalmente
  verificación, no requiere que US1/US2 estén terminadas.
- **Polish (Phase 6)**: depende de que las historias que se vayan a entregar estén completas.

### Dentro de Foundational

T001 → T004 (necesita el head real). T002/T003 → T004 (la migración refleja las columnas ya
definidas en el modelo/schema). T002 → T008 (la columna `is_consolidation_order` debe existir
antes de usarla en la consulta). T008, T009 → T010. T009, T010, T011 → T012. T013, T014 → T015.

### Dentro de cada historia

- US1: T016-T018 (tests) antes de T019-T024 (implementación) — T019 hace pasar T016; T020-T023
  hacen pasar T017; T024 hace pasar T018 (depende de T013 de Foundational). T025 al final.
- US2: T026 antes de T027 (hace pasarlo). T028 al final.
- US3: T029/T030 validan el resultado de Foundational directamente; T031/T032 son verificación
  manual, sin dependencia de código nuevo de US1/US2.

### Parallel Opportunities

- Foundational: T003 puede correr en paralelo con T002 (archivos distintos); T006, T007 en
  paralelo entre sí y con T002/T003 (archivos distintos); T009 en paralelo con T006/T007; T011 en
  paralelo con T009 (T010 sí depende de T008/T009). T013, T014 en paralelo entre sí.
- US1: T016, T017, T018 en paralelo entre sí (archivos de test distintos, backend vs. frontend).
- US2: T026 es el único test de la historia.
- US3: T029, T030 en paralelo entre sí (archivos de test distintos).
- Con más de una persona disponible: US1 y US2 pueden trabajarse en paralelo una vez terminado
  Foundational (tocan secciones distintas del mismo `orders/service.py`; coordinar el orden de
  merge de T019 y T027 para evitar conflicto de líneas, no de lógica).

---

## Parallel Example: Foundational

```bash
# En paralelo, tras T001:
Task: "Modificar CustomerOrder en pos-backend/app/models/customer_order.py (T002)"
Task: "Actualizar OrderChannel/OrderType en pos-backend/app/api/v1/orders/schemas.py (T003)"

# En paralelo, tras T002/T003:
Task: "Renombrar OrderChannel.QR en orders/service.py (T006)"
Task: "Renombrar filtro de canal en cart/service.py:589 (T007)"
Task: "Actualizar fixtures de test (T009)"
```

## Parallel Example: User Story 1

```bash
Task: "Test backend: 422 en TAKEAWAY/DELIVERY + mesa (T016)"
Task: "Test frontend: pestaña Para Llevar en manual-order-page.component.spec.ts (T017)"
Task: "Test frontend: payload TAKEAWAY en pos-terminal.store.spec.ts (T018)"
```

---

## Implementation Strategy

### MVP First (Foundational + User Story 1)

1. Completar Phase 1 (Setup) y Phase 2 (Foundational) — sin esto no hay base segura para nada.
2. Completar Phase 3 (US1) — entrega el resultado de negocio explícitamente pedido ("habilitar
   Para Llevar").
3. **Detener y validar**: quickstart.md Escenario 1 y 3.
4. US2 y US3 pueden entregarse después, de forma incremental, sin romper US1.

### Incremental Delivery

1. Foundational → base lista, sin cambio de comportamiento observable, tests en verde.
2. + US1 → "Para Llevar" funcional → validar → demo (MVP).
3. + US2 → validación de combinaciones activa → validar.
4. + US3 → verificación de histórico + regresión de consolidación documentada → validar.
5. + Polish.

---

## Notes

- Ningún task de este documento agrega dependencias nuevas ni toca `pos-tables-panel.component.ts`
  (research.md D6) ni ningún dato de facturación histórico (Principio VII).
- T008/T010 son el punto más sensible de todo el plan (research.md Decisión 2, Complexity
  Tracking del plan.md): confirmar T012 (suite completa en verde) antes de considerar Foundational
  terminado, no solo el archivo tocado.
- Commitear después de cada tarea o grupo lógico; detenerse en cada checkpoint para validar la
  historia de forma independiente antes de continuar con la siguiente.
