---

description: "Task list template for feature implementation"
---

# Tasks: Vaciado del Carrito del Participante al Crear el Pedido (Menú QR)

**Input**: Design documents from `/specs/038-vaciado-carrito-pedido/`

**Prerequisites**: [plan.md](./plan.md) (required), [spec.md](./spec.md) (requerido para historias de
usuario), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/](./contracts/),
[quickstart.md](./quickstart.md)

**Tests**: Esta spec modifica un comportamiento protegido por characterization tests (Principio III de
la Constitución) y su plan/quickstart exigen explícitamente reescribir 2 tests `CONGELA` y agregar
tests nuevos por historia — **los tests no son opcionales en este feature**, están incluidos abajo
como tareas de implementación.

**Organization**: Las tareas están agrupadas por historia de usuario para permitir implementación y
prueba independiente de cada una. El código de este feature vive en los repos sibling
`../pos-backend` y `../pos-heladeria` (rutas relativas a la raíz de cada uno, ver plan.md
§Project Structure).

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (archivos distintos, sin dependencias entre sí)
- **[Story]**: A qué historia de usuario pertenece la tarea (US1, US2, US3, US4)
- Cada tarea incluye la ruta de archivo exacta

## Path Conventions

- **Backend** (`pos-backend/`): `app/api/v1/cart/service.py`, `app/api/v1/orders/schemas.py`,
  `app/models/order_item.py`, `app/characterization_tests/*.py`, `alembic/versions/*.py`
- **Frontend** (`pos-heladeria/`): `src/app/modules/tables/interfaces/dining.interface.ts`

---

## Phase 1: Setup

**Purpose**: Confirmar la línea base antes de tocar código (quickstart.md Paso 0)

- [X] T001 Ejecutar la suite de characterization tests de `cart`/`orders` en `pos-backend` y
  confirmar que todo está en verde, incluidos los 2 tests `CONGELA` que esta spec va a modificar y
  el tercero que no se toca: `python -m unittest app.characterization_tests.test_cart_service -v`,
  `python -m unittest app.characterization_tests.test_cart_router -v`,
  `python -m unittest app.characterization_tests.test_orders_consolidation -v`
  (quickstart.md Paso 0)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Nota**: no hay infraestructura compartida que bloquee a *todas* las historias más allá de lo que
ya existe en el repo (sin dependencias nuevas, sin módulos nuevos — Technical Context del plan). El
único prerrequisito real ("el carrito se borra físicamente") está etiquetado por `spec.md` como
`[Historia 1]` (FR-003/FR-004) y se implementa dentro de la Fase 3 — US2/US3/US4 dependen de que esa
fase se complete primero (ver Dependencies & Execution Order), pero no hay tareas foundational-only
independientes de ninguna historia. Se pasa directo a la Fase 3.

---

## Phase 3: User Story 1 - El carrito desaparece en cuanto el pedido se confirma (Priority: P1) 🎯 MVP

**Goal**: al confirmar el pedido con éxito, el carrito del participante se elimina físicamente
(no se archiva) en la misma transacción del pedido, y cada línea del pedido persiste su propio
snapshot de descuento — visible de inmediato en la respuesta y tras recargar la página.

**Independent Test**: agregar ítems al carrito, confirmar el pedido con un método de pago válido, y
verificar que (a) la respuesta del pedido confirmado incluye esas líneas con sus datos (incluido el
descuento aplicado) y (b) una consulta posterior del carrito (incluida tras recargar la página)
devuelve cero líneas.

### Implementation for User Story 1

- [X] T002 [P] [US1] Agregar columnas nullable `discounted_unit_price`/`discounted_line_total`
  (`Numeric(12, 2)`, sin default) al modelo `OrderItem` en
  `pos-backend/app/models/order_item.py` (data-model.md §OrderItem, research.md Decisión 7)

- [X] T003 [P] [US1] Crear la migración Alembic que agrega esas mismas 2 columnas a `order_items`,
  `down_revision='205f518df786'` (head actual), siguiendo el patrón `@for_each_tenant_schema` de
  `pos-backend/alembic/versions/a63ddb1f0d97_combos_de_promociones_dias_del_mes_.py`, sin
  `server_default`, en
  `pos-backend/alembic/versions/{rev}_order_item_discount_snapshot.py`
  (research.md Decisión 7, FR-015)

- [X] T004 [US1] En `submit_cart`, reemplazar `cart.status = "confirmado"` por `db.delete(cart)`
  dentro del mismo bloque `try`/`commit()` que ya crea `CustomerOrder`/`OrderItem`/
  `OrderPaymentAttempt`, en `pos-backend/app/api/v1/cart/service.py` (línea ~605) — apoyado en las
  cascadas ya declaradas (`Cart.items` `cascade="all, delete-orphan"`,
  `cart_items.cart_id`/`cart_item_options.cart_item_id` `ondelete="CASCADE"`), sin migración de
  esquema para `Cart`/`CartItem` (research.md Decisión 1, FR-003, FR-004)

- [X] T005 [US1] Extraer el bloque de cómputo de descuento por línea que hoy vive inline en
  `serialize_cart` (`pos-backend/app/api/v1/cart/service.py`, líneas ~224-259: construcción del
  `catalog` y el cálculo de `discounted_unit_price`/`discounted_line_total` con
  `promotions.best_line_discount`) a funciones privadas compartidas (p. ej.
  `_line_discount(...)` y un helper de catálogo); `serialize_cart` pasa a llamarlas, sin cambiar su
  resultado observable (`CartResponse` idéntica a hoy) (research.md Decisión 4)

- [X] T006 [US1] En `submit_cart`, poblar `discounted_unit_price`/`discounted_line_total` de cada
  `OrderItem` creado, usando las funciones extraídas en T005 con
  `promotions.active_discount_promotions(db, now)` calculado una sola vez antes del bucle (mismo
  patrón que `serialize_cart`), en `pos-backend/app/api/v1/cart/service.py` (research.md Decisión
  4, FR-002, FR-013)

- [X] T007 [P] [US1] Agregar `discounted_unit_price: Decimal | None = None` y
  `discounted_line_total: Decimal | None = None` a `OrderItemResponse` en
  `pos-backend/app/api/v1/orders/schemas.py` (líneas ~139-157), mismos nombres/semántica que
  `CartItemResponse` (contracts/cart-orders.md, FR-013)

- [X] T008 [P] [US1] Agregar `discounted_unit_price?: string | null` y
  `discounted_line_total?: string | null` a `DiningOrderItem` en
  `pos-heladeria/src/app/modules/tables/interfaces/dining.interface.ts` (líneas ~70-99), espejo de
  `CartItemResponse` en `diner.interface.ts` — sin tocar ningún componente/plantilla
  (contracts/cart-orders.md, research.md Decisión 8)

- [X] T009 [P] [US1] Reescribir y renombrar el test `CONGELA`
  `test_submit_cart_confirma_pedido_y_abre_carrito_nuevo` →
  `test_submit_cart_elimina_carrito_y_abre_uno_nuevo_tras_pedido` en
  `pos-backend/app/characterization_tests/test_cart_service.py` (líneas ~484-522): reemplazar
  `db.refresh(old_cart); self.assertEqual(old_cart.status, "confirmado")` por
  `self.assertIsNone(db.get(Cart, old_cart.id))`, citando esta spec en el docstring; el resto del
  test (creación del pedido, `current_payment_attempt`, `new_cart.id != old_cart.id`) queda igual
  (research.md Decisión 9, depende de T004)

- [X] T010 [P] [US1] Reescribir la aserción del test `CONGELA`
  `test_submit_cart_endpoint_evento_tras_commit` (mismo nombre) en
  `pos-backend/app/characterization_tests/test_cart_router.py` (líneas ~216-252): dentro de `_spy`,
  reemplazar la consulta `Cart.status == "confirmado"`/`seen["cart_confirmado"]` por una consulta
  `select(Cart.id).where(Cart.participant_id == participant.id)`/`seen["cart_exists"]`, y
  `self.assertEqual(seen["cart_confirmado"], "confirmado")` por
  `self.assertIsNone(seen["cart_exists"])`; el resto del test (orden `recibida`, el evento se
  publica exactamente una vez después del commit) no cambia (research.md Decisión 9, depende de
  T004)

- [X] T011 [US1] Agregar un test que confirme que el snapshot de descuento persistido en los
  `OrderItem` coincide exactamente con lo que `serialize_cart`/`GET /cart` mostraba justo antes de
  confirmar: un carrito con 2 ítems, uno con una promoción de descuento activa sobre su línea →
  `submit_cart` → verificar `discounted_unit_price`/`discounted_line_total` no nulos en la línea
  con promoción y nulos en la otra, en
  `pos-backend/app/characterization_tests/test_cart_service.py` (Acceptance Scenario 3 de US1 /
  CA-6, SC-005, depende de T006)

- [X] T012 [US1] Agregar un test de atomicidad: forzar un fallo tras el chequeo de disponibilidad
  (p. ej. mockear `check_availability` para lanzar, o usar un método de pago inactivo) y verificar
  que el `Cart`/`CartItem[]` del participante siguen existiendo intactos (mismo `id`, mismas
  líneas) y que no se creó ningún `CustomerOrder`, en
  `pos-backend/app/characterization_tests/test_cart_service.py` (Acceptance Scenario 4 de US1 /
  CA-8, FR-004, depende de T004)

**Checkpoint**: en este punto, User Story 1 es completamente funcional y verificable de forma
independiente — el carrito se elimina físicamente al confirmar, el snapshot de descuento se
persiste y se expone, y una falla en la creación del pedido no toca el carrito.

---

## Phase 4: User Story 2 - Un segundo intento de confirmar nunca duplica el pedido (Priority: P1)

**Goal**: un segundo intento de confirmar (mala conexión, doble clic, dos pestañas) sobre un
carrito ya vaciado por un pedido anterior se rechaza con un mensaje explícito de "ya fue enviado",
sin crear un pedido adicional.

**Independent Test**: confirmar un carrito con éxito y, sin agregar nada nuevo, confirmar de nuevo
(o simular una segunda pestaña con el carrito viejo) — verificar que solo existe un pedido y que la
segunda respuesta indica explícitamente que ya fue enviado.

**Depende de**: Phase 3 (US1) — el mensaje de "ya fue enviado" solo tiene sentido una vez que el
carrito se elimina físicamente al confirmar (FR-007 lo describe explícitamente así).

### Implementation for User Story 2

- [X] T013 [US2] Reordenar las validaciones de `submit_cart` para que la rama de "carrito no
  existe" también pueda producir el `409` de "ya fue enviado" — esto exige tocar **`_load_open_cart`**
  (`pos-backend/app/api/v1/cart/service.py`, líneas ~171-178), que hoy lanza
  `HTTPException(404, "No hay carrito abierto para el comensal")` inmediatamente si no hay fila
  `'abierto'`, antes de que cualquier código posterior a su llamada (línea ~511) pueda ejecutarse.
  Introducir una variante que devuelva `None` en vez de lanzar (o envolver la llamada en
  `try/except HTTPException` capturando solo el 404), calcular `active_order` (la misma consulta
  `_NON_TERMINAL_ORDER_STATUSES` que ya existe) **antes** de decidir qué error devolver, y lanzar
  el nuevo `409` explícito `"Tu pedido ya fue enviado; revisa el estado en Mis pedidos."` cuando el
  carrito sea `None` o esté vacío **y** exista una orden no terminal previa — sin tocar el resto de
  validaciones (comanda manual, método de pago, comprobante, disponibilidad) ni
  `idx_active_order_per_participant` (garantía de última instancia, sin cambios), en
  `pos-backend/app/api/v1/cart/service.py` (líneas ~171-178 y ~511-534) (research.md Decisión 3,
  FR-007, FR-008, contracts/cart-submit.md, depende de T004). **Nota**: este es el escenario más
  frecuente en producción — tras cualquier confirmación exitosa (FR-003 ya borró la fila), un
  segundo intento cae exactamente en esta rama ("no existe"), no en la de "carrito vacío".

- [X] T014 [US2] Agregar un test: confirmar un carrito con éxito y, sin agregar nada nuevo, invocar
  `submit_cart` de nuevo para el mismo participante → `409` con el mensaje explícito de "ya fue
  enviado" (texto distinto al de carrito vacío), y el conteo de `CustomerOrder` del participante
  sigue en 1 (mismo camino de código que el escenario de "dos pestañas", ya que ambas apuntan al
  mismo `participant_id`), en `pos-backend/app/characterization_tests/test_cart_router.py`
  (Acceptance Scenarios 1-2 de US2 / CA-3, CA-4, FR-007, depende de T013)

- [X] T015 [US2] Agregar un test: un participante con carrito vacío (sin ítems) y sin ninguna orden
  no terminal previa → `submit_cart` sigue devolviendo el `409`/`404` genérico de "carrito vacío"/
  "no hay carrito abierto", distinguible del mensaje de "ya fue enviado", sin crear nada, en
  `pos-backend/app/characterization_tests/test_cart_router.py` (Acceptance Scenario 3 de US2 /
  CA-7, FR-009, depende de T013)

**Checkpoint**: en este punto, User Stories 1 y 2 funcionan juntas de forma independiente — el
vaciado nunca produce un pedido duplicado y el mensaje de error distingue "ya enviado" de "carrito
vacío".

---

## Phase 5: User Story 3 - Segunda ronda desde un carrito limpio (Priority: P2)

**Goal**: después de confirmar un pedido, el comensal puede agregar ítems nuevos y confirmarlos
como un segundo pedido independiente, sin arrastrar ninguna línea de la ronda ya confirmada.

**Independent Test**: confirmar un pedido y, en la misma sesión, agregar un ítem nuevo — verificar
que el carrito resultante contiene solo ese ítem nuevo, no los de la ronda ya confirmada, y que se
puede confirmar como un segundo pedido independiente.

**Depende de**: Phase 3 (US1) — sin el vaciado físico no existe un "carrito limpio" del que partir
(spec.md, "Why this priority" de US3).

### Implementation for User Story 3

- [X] T016 [US3] Extender el test renombrado en T009
  (`test_submit_cart_elimina_carrito_y_abre_uno_nuevo_tras_pedido`) para, tras confirmar el primer
  pedido, llamar `add_item` con un ítem nuevo y verificar que el carrito resultante contiene
  únicamente ese ítem — no arrastra ninguna línea del pedido ya confirmado, en
  `pos-backend/app/characterization_tests/test_cart_service.py` (Acceptance Scenario 1 de US3 /
  RF-08, depende de T009)

- [X] T017 [US3] Agregar un test: confirmar ese carrito de la segunda ronda crea un segundo
  `CustomerOrder` independiente del primero, con ambos coexistiendo con sus respectivas líneas —
  usando dos participantes (o avanzando el primer pedido a un estado terminal) para no chocar con
  el `409` de "orden activa" de FR-008, en
  `pos-backend/app/characterization_tests/test_cart_service.py` (Acceptance Scenario 2 de US3,
  User Story 3 Acceptance Scenario 2, depende de T004)

**Checkpoint**: la segunda ronda funciona sin arrastrar líneas y produce pedidos verdaderamente
independientes.

---

## Phase 6: User Story 4 - El vaciado de un comensal no afecta a los demás de su mesa (Priority: P2)

**Goal**: cuando un comensal confirma su pedido, los carritos de los demás comensales de la misma
mesa —sin confirmar todavía— siguen intactos, y ninguna operación puede vaciar o modificar el
carrito de otro participante, sesión de mesa o tenant.

**Independent Test**: con dos comensales en la misma mesa, cada uno con su carrito, confirmar el
pedido de uno y verificar que el carrito del otro no cambió ni de contenido ni de cantidad de
líneas.

**Depende de**: Phase 3 (US1) para tener un escenario real de vaciado que aislar, aunque el filtro
por `participant_id` en sí no cambia (spec.md, "Why this priority" de US4: no bloquea la entrega de
US1, pero se verifica sobre el comportamiento que US1 introduce).

### Implementation for User Story 4

- [X] T018 [US4] Agregar un test: dos participantes (`ana`, `beto`) en la misma `table_session`,
  cada uno con su propio `Cart` con ítems → `submit_cart(db, ana, ...)` → verificar que el `Cart`
  de `beto` sigue existiendo, con el mismo `id`, las mismas líneas y la misma cantidad, en
  `pos-backend/app/characterization_tests/test_cart_service.py` (Acceptance Scenario 1 de US4 /
  CA-5, FR-006, depende de T004)

- [X] T019 [US4] Re-ejecutar los tests existentes de aislamiento por `x-session-token` en
  `pos-backend/app/characterization_tests/test_cart_router.py` y confirmar que una operación de
  carrito con el token/participante de otra sesión sigue siendo rechazada sin vaciar ni modificar
  el carrito ajeno — sin cambio de código esperado, solo verificación de que el comportamiento
  existente sigue vigente (Acceptance Scenario 2 de US4 / RF-10)

**Checkpoint**: todas las historias de usuario son funcionales de forma independiente — el vaciado
de un comensal está estrictamente aislado a su propio carrito.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: verificación final de no-regresión y validación manual del frontend

- [X] T020 Ejecutar la suite completa de `pos-backend`
  (`python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v`) y confirmar que
  `test_consolidate_table_consolida_carritos_en_orden_existente`
  (`app/characterization_tests/test_orders_consolidation.py:269-315`) sigue en verde **sin ninguna
  modificación**, y que ningún otro test `"CONGELA comportamiento actual:"` quedó en rojo sin
  autorización (research.md Decisión 2/10, Principio III)

- [X] T021 [P] Validar manualmente en `pos-heladeria` (`ng serve`, guía completa en
  [quickstart.md](./quickstart.md) §Frontend): confirmar un pedido con y sin promoción activa
  (CA-1), recargar la página (CA-2), agregar un ítem nuevo sin recargar (US3), verificar en la
  pestaña de red que un reenvío con el carrito viejo responde `409` "ya fue enviado" (US2), y
  confirmar aislamiento entre dos comensales de la misma mesa (US4)

- [X] T022 [P] Verificar la migración de T003 en una base de datos de desarrollo: `alembic upgrade
  head` (confirma `discounted_unit_price`/`discounted_line_total` en `order_items`, `NUMERIC(12,2)`
  nullable, sin `DEFAULT`, y que una fila `order_items` preexistente queda en `NULL` en ambas, no en
  `0`), luego `alembic downgrade -1` y `alembic upgrade head` de nuevo para confirmar el rollback
  limpio (quickstart.md Paso 1, FR-015)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — puede empezar de inmediato
- **Foundational (Phase 2)**: vacía para este feature (ver nota en la Fase 2) — no bloquea nada por
  sí misma
- **User Story 1 (Phase 3)**: depende de Setup. Es la base técnica (borrado físico del carrito) de
  la que dependen US2, US3 y US4
- **User Story 2 (Phase 4)**: depende de que Phase 3 esté completa (el mensaje de "ya fue enviado"
  solo ocurre porque el carrito ya no existe, FR-007)
- **User Story 3 (Phase 5)**: depende de que Phase 3 esté completa (necesita el "carrito limpio" que
  produce el vaciado)
- **User Story 4 (Phase 6)**: depende de que Phase 3 esté completa (verifica el aislamiento del
  vaciado que US1 introduce)
- **Polish (Phase 7)**: depende de que todas las historias deseadas estén completas

### User Story Dependencies

- **User Story 1 (P1)**: sin dependencia de otras historias — es la base de todas las demás
- **User Story 2 (P1)**: depende de User Story 1 (mismo `submit_cart`, mismo archivo
  `pos-backend/app/api/v1/cart/service.py` — T013 se implementa después de T004-T006)
- **User Story 3 (P2)**: depende de User Story 1
- **User Story 4 (P2)**: depende de User Story 1

**Nota sobre archivos compartidos**: US1 y US2 modifican la misma función (`submit_cart`) en el
mismo archivo (`pos-backend/app/api/v1/cart/service.py`) — aunque son historias independientemente
verificables una vez implementadas, su implementación debe ser secuencial (US1 → US2), no paralela
por dos personas distintas sobre el mismo archivo.

### Within Each User Story

- Modelo/esquema antes de la lógica que los usa (T002/T003 antes de T006)
- Cambio de comportamiento antes de los tests que lo verifican (T004 antes de T009/T010/T012/T013)
- Historia completa antes de pasar a la siguiente en prioridad

### Parallel Opportunities

- T002 (modelo) y T003 (migración) en paralelo — archivos distintos, sin dependencia entre sí
- T007 (schema backend) y T008 (interfaz frontend) en paralelo — repos distintos
- T009 (test_cart_service.py) y T010 (test_cart_router.py) en paralelo — archivos distintos, ambos
  dependen solo de T004
- T021 (validación manual frontend) y T022 (verificación de migración) en paralelo — actividades
  independientes

---

## Parallel Example: User Story 1

```bash
# T002 y T003 en paralelo (modelo backend + migración):
Task: "Agregar columnas discounted_unit_price/discounted_line_total en pos-backend/app/models/order_item.py"
Task: "Crear migración Alembic de esas columnas en pos-backend/alembic/versions/{rev}_order_item_discount_snapshot.py"

# Tras T004, T009 y T010 en paralelo (tests CONGELA en archivos distintos):
Task: "Reescribir/renombrar test CONGELA en pos-backend/app/characterization_tests/test_cart_service.py"
Task: "Reescribir aserción del test CONGELA en pos-backend/app/characterization_tests/test_cart_router.py"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Completar Phase 1: Setup
2. Completar Phase 3: User Story 1 (T002-T012)
3. **DETENER y VALIDAR**: probar User Story 1 de forma independiente (carrito eliminado físicamente,
   snapshot de descuento correcto, atomicidad ante fallo)
4. Desplegar/demostrar si está listo — resuelve ya el problema de negocio central (SC-001, SC-004,
   SC-005)

### Incremental Delivery

1. Setup → Phase 3 (US1) → probar de forma independiente → Demo (MVP, resuelve la ambigüedad de "mi
   pedido salió")
2. Agregar Phase 4 (US2) → probar de forma independiente → Demo (cierra el riesgo de duplicados,
   SC-002/SC-003)
3. Agregar Phase 5 (US3) → probar de forma independiente → Demo (segunda ronda fluida)
4. Agregar Phase 6 (US4) → probar de forma independiente → Demo (aislamiento verificado, SC-006)
5. Phase 7: Polish — no-regresión completa + validación manual de frontend

---

## Notes

- [P] = archivos distintos, sin dependencias entre sí
- [Story] mapea cada tarea a su historia de usuario para trazabilidad
- FR-010/FR-011/FR-012 (comportamiento de frontend ya correcto hoy, sin cambio de código) no generan
  tareas de implementación — se verifican en T021 (research.md Decisión 8, plan.md §Project
  Structure)
- FR-014 (el snapshot de descuento no debe alimentar `orders/checkout.py`) no genera tarea propia —
  es una restricción negativa verificada por omisión: ningún task de este documento toca
  `orders/checkout.py`
- FR-001 (el pedido se construye del carrito del servidor, nunca del body de la petición) no genera
  tarea propia — es comportamiento **sin cambio** de `submit_cart`, documentado explícitamente en
  spec.md solo para dejar sentado que el vaciado (FR-003) depende de esa garantía
- FR-005 (rechazar la confirmación si el snapshot de descuento no puede resolverse) no genera tarea
  propia — cubierto por el `except Exception: db.rollback()` ya existente que envuelve toda la
  creación del pedido (research.md Decisión 5); no hay un modo de fallo identificado que amerite un
  test dedicado
- Confirmar que los tests fallan/reflejan el comportamiento viejo antes de cada cambio, cuando
  aplique (T009/T010 dependen de T004 estar hecho para que sus nuevas aserciones tengan sentido)
- Detenerse en cada checkpoint para validar la historia de forma independiente
