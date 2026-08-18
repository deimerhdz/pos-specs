---

description: "Task list for: Corrección de zona horaria en vigencia de promociones del menú y carrito QR (A-08)"
---

# Tasks: Corrección de zona horaria en vigencia de promociones del menú y carrito QR (A-08)

**Input**: Design documents from `/specs/022-correccion-zona-horaria-menu-carrito/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md),
[contracts/menu-endpoint.md](./contracts/menu-endpoint.md),
[contracts/cart-endpoint.md](./contracts/cart-endpoint.md), [quickstart.md](./quickstart.md)

**Tests**: FR-006 exige explícitamente al menos un test de characterization por cada uno de los dos
puntos de invocación corregidos — los tests están incluidos abajo, no son opcionales en esta spec.

**Alcance**: todo el trabajo de código vive en el repositorio sibling `../pos-backend` (Constitución
§Alcance). Rutas de fichero relativas a la raíz de `pos-backend`.

**Nota sobre paralelismo**: esta spec toca **cuatro ficheros** — dos de producción
(`app/api/v1/menu/router.py`, una línea; `app/api/v1/cart/service.py`, una línea) y dos de test
(`app/characterization_tests/cart_fixtures.py`, generalización de `frozen_now`;
`app/characterization_tests/test_menu_router.py`, nuevo) más el fichero de test existente
`app/characterization_tests/test_cart_service.py` (modificado). Ninguna tarea que edite el mismo
fichero se marca `[P]`; solo hay paralelismo real entre suites de test independientes en Polish.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3)

## Phase 1: Setup

**Purpose**: confirmar la línea base antes de tocar código.

- [X] T001 Confirmar que `app/characterization_tests/` no contiene ningún fichero
      `test_menu_*.py` (`ls app/characterization_tests/ | grep menu` sin salida) — quickstart.md
      nota previa al Paso 3. Confirmar además que
      `test_open_session_y_serialize_cart_a08_zona_horaria_no_aplicada`
      (`app/characterization_tests/test_cart_service.py:137-160`) pasa en verde hoy **congelando
      el defecto** de A-08 del lado `cart` (quickstart.md Paso 1).

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: infraestructura compartida que la Historia 1 (menú) necesita antes de poder escribir
su test; retrocompatible para que la Historia 2 (carrito) no se vea afectada.

- [X] T002 Generalizar `frozen_now` en `app/characterization_tests/cart_fixtures.py` para aceptar
      un parámetro `module: str` (nombre completo del módulo cuyo `datetime` se parchea), con valor
      por defecto `"app.api.v1.cart.service"` — mismo comportamiento de hoy para los tests
      existentes de `cart`, sin tocarlos — research.md Decisión 2, quickstart.md Paso 2.
- [X] T003 Crear `app/characterization_tests/test_menu_router.py` con docstring citando A-08
      (`registro-de-anomalias.md`, contraste con A-07), imports (`unittest`, `datetime`/`time`/
      `timezone`, `Decimal`, `app.characterization_tests.cart_fixtures as fx`,
      `app.api.v1.menu.router._build_menu`) y la clase `TestBuildMenuA08(unittest.TestCase)` vacía
      — research.md Decisión 3 (reutiliza el harness de `cart_fixtures.py`, sin `menu_fixtures.py`
      nuevo) (depende de T002).

**Checkpoint**: el fichero de test de `menu` existe y `frozen_now` acepta el módulo a parchear —
se pasa a Phase 3.

---

## Phase 3: User Story 1 - El menú público muestra la vigencia real de una promoción (Priority: P1) 🎯 — anomalía A-08

**Goal**: `_build_menu` (`menu/router.py:82`) evalúa la vigencia de promociones con un `datetime`
aware, coincidiendo con la hora local real del tenant (FR-001).

**Independent Test**: llamar `_build_menu` con el reloj fijado a un instante en el que la hora UTC
cae dentro de la ventana de una promoción pero la hora de Bogotá no, y verificar que la promoción
aparece como NO vigente.

### Implementation for User Story 1

- [X] T004 [US1] Escribir `test_a08_fuera_de_ventana_en_hora_local_no_descuenta` en
      `test_menu_router.py`: crea categoría/producto/variante y una promoción con ventana
      20:00-21:00, fija el reloj a las 20:00 UTC (15:00 Bogotá) vía
      `fx.frozen_now(instant, module="app.api.v1.menu.router")`, llama `_build_menu(db)` y afirma
      `discounted_price is None` en la variante resultante — quickstart.md Paso 3, Historia 1
      escenario 1 (depende de T003).
- [X] T005 [US1] Ejecutar
      `python3 -m unittest app.characterization_tests.test_menu_router -v` y confirmar que T004
      **falla** contra el código actual — el fallo mismo es la evidencia con datos reales del
      defecto A-08 en `menu` (`discounted_price` no es `None`). Ningún cambio de código en este
      paso (depende de T004).
- [X] T006 [US1] Aplicar la corrección en `_build_menu` (`app/api/v1/menu/router.py:82`): cambiar
      `now = datetime.now(timezone.utc).replace(tzinfo=None)` por
      `now = datetime.now(timezone.utc)` — research.md Decisión 1, quickstart.md Paso 4 (depende
      de T005).
- [X] T007 [US1] Re-ejecutar
      `python3 -m unittest app.characterization_tests.test_menu_router -v` y confirmar que T004
      pasa en verde (FR-001, CA1) (depende de T006).
- [X] T008 [US1] Añadir `test_a08_dentro_de_ventana_en_hora_local_si_descuenta` en
      `test_menu_router.py`: misma promoción y variante, reloj fijado a la 01:00 UTC del día
      siguiente (20:00 Bogotá, dentro de ventana), afirma que `discounted_price` refleja el
      descuento — Historia 1 escenario 2, sin regresión frente al caso correcto de hoy (CA3)
      (depende de T006).

**Checkpoint**: el menú público ya muestra la vigencia real de las promociones — verificable de
forma aislada con T004/T007/T008, sin tocar todavía el carrito.

---

## Phase 4: User Story 2 - El carrito del comensal muestra el mismo estado de vigencia que el menú y el cobro (Priority: P1)

**Goal**: `serialize_cart` (`cart/service.py:205`) evalúa la vigencia de promociones con el mismo
criterio corregido que el menú (FR-002), sin modificar la función compartida `_now()`.

**Independent Test**: llamar `serialize_cart` (vía `add_item`) con el reloj fijado igual que en
Historia 1 y verificar que `discounted_total` no aplica el descuento fuera de la ventana real.

### Implementation for User Story 2

- [X] T009 [US2] Modificar el test `CONGELA` existente
      `test_open_session_y_serialize_cart_a08_zona_horaria_no_aplicada`
      (`app/characterization_tests/test_cart_service.py:137-160`) para verificar que
      `resp.discounted_total is None` fuera de la ventana real, en vez de aceptar el descuento en
      silencio (Historia 2, escenario 1). Renombrar a
      `test_serialize_cart_a08_zona_horaria_aplicada_tras_la_correccion`, citando la entrada A-08
      de `registro-de-anomalias.md` en el docstring — exigido por el Principio II de la
      Constitución y por FR-006. **Recordatorio: el Principio II exige además citar A-08 en el
      propio mensaje de commit que incluya este cambio.**
- [X] T010 [US2] Ejecutar
      `python3 -m unittest app.characterization_tests.test_cart_service -v` y confirmar que T009
      **falla** contra el código actual (`resp.discounted_total` no es `None`) — evidencia con
      datos reales del defecto A-08 en `cart` (depende de T009).
- [X] T011 [US2] Aplicar la corrección en `serialize_cart` (`app/api/v1/cart/service.py:205`):
      cambiar `now = _now()` por `now = datetime.now(timezone.utc)` — SIN modificar `_now()`
      (líneas 52-53) ni su uso en `open_session:107` — research.md Decisión 1, quickstart.md
      Paso 4 (depende de T010).
- [X] T012 [US2] Re-ejecutar
      `python3 -m unittest app.characterization_tests.test_cart_service -v` y confirmar que T009
      pasa en verde (FR-002, CA2) (depende de T011).
- [X] T013 [US2] Añadir `test_serialize_cart_dentro_de_ventana_en_hora_local_si_descuenta` en
      `test_cart_service.py`: mismo escenario, reloj fijado dentro de la ventana real (20:00
      Bogotá), afirma que `discounted_total` refleja el descuento — Historia 2 escenario 2, sin
      regresión frente al caso correcto de hoy (depende de T011).

**Checkpoint**: US1 + US2 juntas entregan la corrección completa — menú y carrito ya no divergen
del cobro real ante ninguna ventana horaria (FR-003).

---

## Phase 5: User Story 3 - La corrección no toca nada fuera de la evaluación de promociones (Priority: P1)

**Goal**: confirmar que la corrección no altera el TTL de sesión del comensal (`expires_at`, vía
`_now()`) ni el motor de promociones protegido por A-07 (FR-004/FR-005).

**Independent Test**: comparar, antes y después de la corrección, el instante exacto en que expira
una sesión de comensal con el reloj fijado — debe ser idéntico.

### Implementation for User Story 3

- [X] T014 [US3] Ejecutar
      `python3 -m unittest app.characterization_tests.test_table_sessions_service -v` y
      `python3 -m unittest app.characterization_tests.test_cart_router -v` y confirmar que ningún
      test que dependa de `expires_at`/`open_session` cambia de resultado tras T011 — quickstart.md
      Paso 7 (depende de T011).
- [X] T015 [US3] Revisar `app/api/v1/cart/service.py` completo y confirmar que `_now()` (líneas
      52-53) y su uso en `open_session:107` (`expires_at=_now() + timedelta(...)`) quedan
      exactamente iguales a como estaban antes de T011 — **Verificación**:
      `git diff app/api/v1/cart/service.py` no debe mostrar ninguna línea distinta a la 205 (FR-004)
      (depende de T011).
- [X] T016 [US3] Ejecutar `python3 -m unittest app.scripts.test_promotions_rules -v` y confirmar
      que sigue en verde sin cambios — este script (único que corre en CI) no importa `cart` ni
      `menu` (verificado por sus imports), confirmando que `active_discount_promotions`/
      `local_now`/`best_line_discount` (spec 012, A-07 protegida) no se tocaron — quickstart.md
      Paso 8 (depende de T006, T011).

**Checkpoint**: las tres historias quedan cubiertas — la corrección es completa (US1+US2), y
confirmada como sin efectos fuera de su alcance (US3).

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T017 Ejecutar
      `python3 -m unittest app.characterization_tests.test_menu_router -v` y
      `python3 -m unittest app.characterization_tests.test_cart_service -v` completos y confirmar
      que todos los tests (T004, T008, T009, T013, más el resto de `test_cart_service.py` sin
      cambios) pasan sin ninguna regresión (Principio II).
- [X] T018 [P] Ejecutar `python3 -m unittest app.characterization_tests.test_cart_router -v` y
      confirmar que sigue en verde sin cambios —
      `app/characterization_tests/test_cart_router.py`.
- [X] T019 [P] Ejecutar
      `python3 -m unittest app.characterization_tests.test_table_sessions_service -v` y confirmar
      que sigue en verde sin cambios —
      `app/characterization_tests/test_table_sessions_service.py`.
- [X] T020 Ejecutar `python3 -m unittest discover -s app/characterization_tests -p "test_*.py"`
      (la suite completa de `pos-backend`, no solo los módulos tocados) y confirmar cero
      regresiones fuera del alcance de esta spec (Principio II) (depende de T017-T019).
- [X] T021 Recorrer [quickstart.md](./quickstart.md) de punta a punta (Pasos 1-8) contra el
      `pos-backend` con el fix aplicado y confirmar SC-001 a SC-005: SC-001 (T007), SC-002 (T012),
      SC-003 (T014-T015), SC-004 (T015-T016, sin recálculo retroactivo), SC-005 (T004+T009, dos
      scripts de characterization citando A-08) (depende de T020).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — arranca de inmediato
- **Foundational (Phase 2)**: depende de Setup; genera la infraestructura de test que US1 necesita
- **US1 (Phase 3)**: depende de Foundational; toca producción en `menu/router.py` (T006)
- **US2 (Phase 4)**: independiente de US1 a nivel de código (ficheros distintos), pero secuenciada
  después para mantener el mismo orden RED→GREEN por historia; toca producción en
  `cart/service.py` (T011)
- **US3 (Phase 5)**: depende de T006 (US1) y T011 (US2) — revisa el código ya cambiado en ambos, no
  introduce cambio propio
- **Polish (Phase 6)**: depende de que US1+US2+US3 estén completas

### Notas de secuencia

A diferencia de las specs 020/021 (una sola función de producción), aquí **dos historias modifican
producción de forma independiente**: US1 toca `menu/router.py` (T006), US2 toca `cart/service.py`
(T011) — ninguna depende del cambio de la otra, así que podrían implementarse en cualquier orden o
en paralelo si dos personas trabajan la spec; se listan secuencialmente aquí porque comparten el
mismo repositorio de test (`cart_fixtures.py`, T002) y el mismo revisor. US3 sí depende de ambas,
porque su verificación cubre el resultado conjunto.

### Parallel Opportunities

Ninguna entre T002/T003 (mismo objetivo: preparar el harness de `menu`) ni entre tareas que editan
el mismo fichero de test (T004/T008 en `test_menu_router.py`; T009/T013 en `test_cart_service.py`).
En Polish, T018 y T019 corren suites sobre ficheros distintos entre sí y respecto a T017 — pueden
lanzarse en paralelo.

---

## Parallel Example: Polish

```bash
# Estas dos corridas de suite son independientes entre sí y de T017 (ficheros de test distintos):
python3 -m unittest app.characterization_tests.test_cart_router -v             # T018
python3 -m unittest app.characterization_tests.test_table_sessions_service -v  # T019
```

---

## Implementation Strategy

### Orden recomendado

1. Phase 1 (Setup): confirmar la línea base (T001).
2. Phase 2 (Foundational): preparar el harness de test compartido (T002-T003).
3. Phase 3 (US1): T004-T005 (RED, documenta el defecto en menú), T006 (fix), T007-T008 (GREEN +
   no regresión) — al terminar esta fase, el menú público ya queda corregido.
4. Phase 4 (US2): T009-T010 (RED, documenta el defecto en carrito), T011 (fix), T012-T013 (GREEN +
   no regresión) — al terminar esta fase, el carrito ya queda corregido.
5. Phase 5 (US3): T014-T016 — confirma por ejecución y por revisión que no hay efectos fuera de
   alcance.
6. Phase 6 (Polish): corridas completas + validación de quickstart.md.

### Incremental Delivery

US1 y US2 son técnicamente desplegables por separado (ficheros de producción distintos, sin
dependencia entre sí) — si se necesitara priorizar, US1 (menú, mayor exposición según `spec.md`
§Why this priority) podría entregarse primero. El PR natural de esta spec, sin embargo, entrega
US1+US2+US3 juntas dado su tamaño total (2 líneas de producción + 1 línea de infraestructura de
test generalizada + 1 fichero de test nuevo + 1 test existente modificado). El desglose por
historia existe para trazabilidad de requisito → tarea → test (FR-001→US1, FR-002→US2,
FR-004/FR-005→US3), no para sugerir necesariamente dos PRs.
