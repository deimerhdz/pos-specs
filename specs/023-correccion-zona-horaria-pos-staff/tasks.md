---

description: "Task list for: Corrección de zona horaria en el POS de staff (previsualización de promociones) (A-09)"
---

# Tasks: Corrección de zona horaria en el POS de staff (previsualización de promociones) (A-09)

**Input**: Design documents from `/specs/023-correccion-zona-horaria-pos-staff/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md),
[data-model.md](./data-model.md),
[contracts/promotions-list-endpoint.md](./contracts/promotions-list-endpoint.md),
[quickstart.md](./quickstart.md)

**Tests**: FR-008 exige explícitamente al menos un test de characterization por cada uno de los
cuatro puntos de invocación corregidos — los tests están incluidos abajo, no son opcionales en esta
spec.

**Alcance**: esta spec cruza dos repositorios sibling de `pos-specs` (Constitución §Alcance):
`../pos-backend` (header nuevo en `GET /promotions`) y `../pos-heladeria` (consumo del header,
corrección de los 4 puntos de invocación). Rutas de fichero relativas a la raíz de cada uno,
indicadas en cada tarea.

**Nota — FR-007 ya satisfecho**: la reapertura de la decisión de negocio de A-09 (de "mitigado
operativamente" a "corregir en modernización") ya quedó registrada en
`specs/000-reconocimiento/registro-de-anomalias.md` durante la fase `/speckit-specify` de esta
misma spec (nota "Actualización — reapertura de la decisión, 2026-08-18"), citando quién y cuándo.
No hay tarea de implementación pendiente para FR-007.

**Nota sobre paralelismo**: la pista backend (T002-T007, `../pos-backend`) y la pista frontend
foundational (T008-T011, `../pos-heladeria`) no comparten ningún fichero — pueden trabajarse en
paralelo por dos personas. Dentro de cada pista, las tareas son secuenciales (RED → fix → GREEN).
Dentro de US1/US2, cada historia toca sus propios dos sitios de `pos-terminal.store.ts`, pero todos
en el mismo fichero — ninguna tarea de edición se marca `[P]` entre sí.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Puede ejecutarse en paralelo (ficheros distintos, sin dependencias)
- **[Story]**: A qué historia de usuario pertenece (US1, US2, US3)

## Phase 1: Setup

**Purpose**: confirmar la línea base antes de tocar código, en ambos repos.

- [X] T001 Confirmar que no existe hoy `app/characterization_tests/test_promotions_router.py` en
      `../pos-backend` (`ls app/characterization_tests/test_promotions_router.py` → "No such file
      or directory"); confirmar que `promotion.service.ts` no tiene hoy `now()`/`ready`/
      `serverTimeOffsetMs` (`grep -n "now\|ready\|serverTimeOffsetMs"
      src/app/modules/promotions/services/promotion.service.ts` sin esos símbolos); y confirmar que
      `pos-terminal.store.spec.ts` no instancia `PosTerminalStore` hoy (solo prueba
      `deriveTableStatus`/`newPendingIds`) — quickstart.md Paso 1 y Paso 6, research.md Decisión 3.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: dar a ambos repos la capacidad de obtener/exponer una hora sincronizada con el
servidor, antes de que ninguna historia de usuario pueda corregir sus puntos de invocación.

**⚠️ CRITICAL**: ninguna tarea de US1/US2/US3 puede empezar hasta que esta fase esté completa —
son las dos historias P1 las que dependen de que `PromotionService.now()`/`ready()` existan.

### Pista backend (`../pos-backend`)

- [X] T002 [P] Escribir `app/characterization_tests/test_promotions_router.py`
      (`TestListPromotionsA09`, docstring citando A-09 y la reapertura en
      `registro-de-anomalias.md`), invocando `list_promotions` directamente como función Python
      (mismo patrón que `test_table_sessions_router.py` — `Depends(...)` nunca se resuelve así) con
      un `Response()` real y un doble mínimo de `User` — quickstart.md Paso 2, contrato
      [promotions-list-endpoint.md](./contracts/promotions-list-endpoint.md).
- [X] T003 Ejecutar `python3 -m unittest app.characterization_tests.test_promotions_router -v` y
      confirmar que T002 **falla** (`list_promotions` no acepta hoy `response` ni escribe ningún
      header) — quickstart.md Paso 2 (depende de T002).
- [X] T004 Aplicar la corrección en `list_promotions`
      (`app/api/v1/promotions/router.py:37`): agregar parámetro `response: Response`, importar
      `datetime`/`timezone` (stdlib, no existían en este fichero) y `Response` de `fastapi`, y
      escribir `response.headers["X-Server-Time"] = datetime.now(timezone.utc).isoformat()` antes
      del `return paginate(...)` — research.md Decisión 1, quickstart.md Paso 3 (depende de T003).
- [X] T005 Agregar `"X-Server-Time"` a `expose_headers` de `CORSMiddleware`
      (`app/main.py:95`, junto a `"ETag"`, `"Retry-After"`) — sin esto el header viaja en la
      respuesta pero el JS del navegador no puede leerlo en el origen cruzado
      `api.skeilopos.com` — research.md Decisión 1, quickstart.md Paso 3 (depende de T004).
- [X] T006 Re-ejecutar `python3 -m unittest app.characterization_tests.test_promotions_router -v`
      y confirmar que T002 pasa en verde — quickstart.md Paso 4 (depende de T005).
- [X] T007 Ejecutar `python3 -m unittest app.scripts.test_promotions_rules -v` y confirmar que
      sigue en verde sin cambios — este script (único en CI) ejercita
      `active_discount_promotions`/`local_now`/`best_line_discount` (spec 012, A-07 protegida), que
      esta tarea no toca — quickstart.md Paso 5 (depende de T006).

### Pista frontend (`../pos-heladeria`)

- [X] T008 [P] Escribir/ampliar `src/app/modules/promotions/services/promotion.service.spec.ts`
      (harness `TestBed` + `provideHttpClientTesting()` + `provideTanStackQuery()`, mismo patrón
      que `product.service.spec.ts`): un test que confirme `service.ready() === false` antes de
      cualquier respuesta, y otro que, tras simular `GET /promotions?status=active` con
      `req.flush(body, { headers: { 'X-Server-Time': ... } })` y `vi.setSystemTime(...)` fijado,
      confirme que `service.now()` usa el offset calculado, no `Date.now()` local — quickstart.md
      Paso 6 (independiente de T002-T007, distinto repositorio y ficheros).
- [X] T009 Ejecutar `npm test -- promotion.service.spec.ts` y confirmar que T008 **falla**
      (`PromotionService` no tiene hoy `now()`/`ready`) — quickstart.md Paso 6 (depende de T008).
- [X] T010 Aplicar la corrección en `src/app/modules/promotions/services/promotion.service.ts`:
      agregar `serverTimeOffsetMs = signal<number | null>(null)`, `ready = computed(() =>
      serverTimeOffsetMs() !== null)` y `now(): Date`; modificar el `queryFn` de `activeQuery`
      (línea 94) para pedir `observe: 'response'`, leer el header `X-Server-Time` de la respuesta,
      actualizar `serverTimeOffsetMs` como efecto lateral y devolver `res.body` (mismo tipo
      `Page<Promotion>` que antes, sin romper `activePromotions`/`activeQuery.data()`) —
      research.md Decisiones 1/2/4, quickstart.md Paso 7 (depende de T009).
- [X] T011 Re-ejecutar `npm test -- promotion.service.spec.ts` y confirmar que T008 pasa en verde
      (FR-001/FR-002 del lado cliente, FR-004) — quickstart.md Paso 7 (depende de T010).

**Checkpoint**: `GET /promotions` expone `X-Server-Time` y `PromotionService.now()`/`ready` existen
y funcionan de forma aislada — US1 y US2 ya pueden corregir sus puntos de invocación en
`pos-terminal.store.ts`.

---

## Phase 3: User Story 1 - El cajero ve la vigencia real de las promociones, sin importar el reloj del terminal (Priority: P1) 🎯 — anomalía A-09

**Goal**: `combos` (`pos-terminal.store.ts:248`) y `productDiscountBadges` (línea 262) evalúan
vigencia con `promotionService.now()`, no con el reloj del dispositivo (FR-001).

**Independent Test**: fijar el reloj del entorno de prueba a un instante en el que la hora UTC cae
dentro de la ventana de una promoción pero la hora de Bogotá no (o viceversa), simular la respuesta
de `PromotionService` (offset conocido) y verificar que `combos`/`productDiscountBadges` reflejan
la vigencia real en hora de Bogotá, no la del reloj del entorno de prueba.

### Implementation for User Story 1

- [X] T012 [US1] Escribir en `src/app/modules/tables/services/pos-terminal.store.spec.ts` un doble
      simple de `PromotionService` (`ready: () => boolean`, `now: () => Date`, sin `TestBed`
      completo del store — research.md Decisión 3) y dos tests: (a) con `ready` en `false`, el
      cálculo equivalente a `combos`/`productDiscountBadges` no debe considerar ninguna promoción
      vigente; (b) con `ready` en `true` y `now()` del doble fijado dentro de la ventana de una
      promoción mientras el reloj del sistema de pruebas está fijado fuera de ella, el resultado
      debe ser "vigente" — solo posible si se usó el doble, no el reloj real — quickstart.md Paso 8.
- [X] T013 [US1] Ejecutar `npm test -- pos-terminal.store.spec.ts` y confirmar que T012 **falla**
      contra el código actual (los sitios siguen usando `new Date()`) — quickstart.md Paso 8
      (depende de T012, T011).
- [X] T014 [US1] Aplicar la corrección en `combos` (`pos-terminal.store.ts:248`): agregar guarda
      `if (!this.promotionService.ready()) return [];` antes del filtro, y reemplazar
      `const now = new Date();` por `const now = this.promotionService.now();` — research.md
      Decisión 4, quickstart.md Paso 9 (depende de T013).
- [X] T015 [US1] Aplicar la corrección en `productDiscountBadges` (`pos-terminal.store.ts:262`):
      agregar guarda `if (!this.promotionService.ready()) return new Map();` antes del bucle, y
      reemplazar `const now = new Date();` por `const now = this.promotionService.now();` — mismo
      patrón que T014 (depende de T013).
- [X] T016 [US1] Re-ejecutar `npm test -- pos-terminal.store.spec.ts` y confirmar que T012 pasa en
      verde (FR-001, CA1) (depende de T014, T015).

**Checkpoint**: los combos y las insignias de descuento del POS de staff ya reflejan la vigencia
real de las promociones, sin importar el reloj del terminal — verificable de forma aislada con
T012/T016, sin tocar todavía el carrito.

---

## Phase 4: User Story 2 - El precio del carrito que ve el cajero coincide con lo que el sistema cobra al confirmar (Priority: P1) — anomalía A-09

**Goal**: `cartView` (`pos-terminal.store.ts:386`) y `orderSubtotal` (línea 1190) calculan
`discountedUnitPrice` con `promotionService.now()`, coincidiendo con lo que el backend cobraría para
el mismo instante (FR-002).

**Independent Test**: fijar el reloj igual que en Historia 1, simular `PromotionService` con offset
conocido, y comparar el `discountedUnitPrice` resultante contra el monto que `promotions.evaluate`
del backend produciría para el mismo instante.

### Implementation for User Story 2

- [X] T017 [US2] Ampliar `src/app/modules/tables/services/pos-terminal.store.spec.ts` (mismo doble
      de `PromotionService` de T012) con dos tests equivalentes para el cálculo que hace
      `cartView`/`orderSubtotal` sobre `discountedUnitPrice`: (a) sin `ready`, sin descuento de
      previsualización; (b) con `ready` y `now()` del doble dentro de ventana pese al reloj real
      fuera de ella, el precio refleja el descuento — quickstart.md Paso 8.
- [X] T018 [US2] Ejecutar `npm test -- pos-terminal.store.spec.ts` y confirmar que T017 **falla**
      contra el código actual — quickstart.md Paso 8 (depende de T017, T016).
- [X] T019 [US2] Aplicar la corrección en `cartView` (`pos-terminal.store.ts:386`): agregar guarda
      `if (!this.promotionService.ready()) { /* calcular sin descuento de previsualización */ }`, y
      reemplazar `const now = new Date();` por `const now = this.promotionService.now();` —
      research.md Decisión 4, quickstart.md Paso 9 (depende de T018).
- [X] T020 [US2] Aplicar la corrección en `orderSubtotal` (`pos-terminal.store.ts:1190`): mismo
      patrón que T019 (depende de T018).
- [X] T021 [US2] Re-ejecutar `npm test -- pos-terminal.store.spec.ts` y confirmar que T017 pasa en
      verde (FR-002, CA2) (depende de T019, T020).

**Checkpoint**: US1 + US2 juntas entregan la corrección completa — combos, insignias y precio del
carrito del POS de staff ya no divergen del cobro real ante ninguna ventana horaria (FR-003).

---

## Phase 5: User Story 3 - La corrección no toca el motor de promociones ni introduce bloqueos si el servidor no responde (Priority: P1)

**Goal**: confirmar que (a) el desempate A-10 y el motor de promociones A-07 no cambiaron, y (b) el
POS de staff nunca bloquea el carrito por falta de hora sincronizada — degrada explícito
(FR-003/FR-004/FR-005/FR-006).

**Independent Test**: comparar el resultado de `bestProductDiscount` antes/después con los mismos
datos (debe ser idéntico); simular un terminal que arranca sin respuesta del backend y confirmar que
el carrito se muestra sin bloquearse.

### Implementation for User Story 3

- [X] T022 [US3] Ejecutar `npm test -- promotion-pricing.util.spec.ts` y confirmar que sigue en
      verde sin cambios — `isPromoActiveNow`/`inTimeWindow`/`bestProductDiscount`/
      `discountedUnitPrice` (funciones puras) no se tocaron (FR-005, A-10 intacta) (depende de
      T014, T015, T019, T020).
- [X] T023 [US3] Ejecutar `git diff src/app/modules/promotions/services/promotion-pricing.util.ts`
      en `../pos-heladeria` y confirmar salida vacía — verificación directa de que ningún cambio
      tocó este fichero (Principio III) (depende de T014, T015, T019, T020).
- [X] T024 [US3] Confirmar, con los dobles de T012/T017 (`ready` en `false`, simulando un terminal
      que aún no recibió ninguna respuesta de `GET /promotions?status=active`), que `combos`
      devuelve `[]`, `productDiscountBadges` devuelve un `Map` vacío y `cartView`/`orderSubtotal` no
      lanzan ninguna excepción ni dejan de renderizar — el carrito se muestra completo, solo sin
      insignias de promoción (FR-003/FR-004, Historia 3 Escenario 2) (depende de T016, T021).
- [X] T025 [US3] Ejecutar `python3 -m unittest app.scripts.test_promotions_rules -v` en
      `../pos-backend` una segunda vez (tras T004/T005) y confirmar que sigue en verde — ningún
      pedido, factura ni promoción ya aplicada se recalcula (FR-006, Historia 3 Escenario 3, sin
      cambio retroactivo posible porque `list_promotions`/`pos-terminal.store.ts` son de solo
      lectura) (depende de T007).

**Checkpoint**: las tres historias quedan cubiertas — la corrección es completa (US1+US2), y
confirmada como sin efectos fuera de su alcance (US3).

---

## Phase 6: Polish & Cross-Cutting Concerns

- [X] T026 Ejecutar `python3 -m unittest discover -s app/characterization_tests -p "test_*.py"`
      en `../pos-backend` (la suite completa, no solo `test_promotions_router.py`) y confirmar cero
      regresiones fuera del alcance de esta spec (Principio II) (depende de T006, T007, T025).
- [X] T027 [P] Ejecutar `npm test` completo en `../pos-heladeria` (todas las suites, no solo las
      tocadas) y confirmar cero regresiones (Principio II) (depende de T011, T016, T021, T022).
- [X] T028 Recorrer [quickstart.md](./quickstart.md) de punta a punta (Pasos 1-10) contra ambos
      repos con el fix aplicado y confirmar SC-001 a SC-006: SC-001 (T016), SC-002 (T021), SC-003
      (T022), SC-004 (T024), SC-005 (T025), SC-006 (T002+T008+T012+T017, cuatro scripts de
      characterization citando A-09) (depende de T026, T027).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sin dependencias — arranca de inmediato en ambos repos
- **Foundational (Phase 2)**: depende de Setup; dos pistas internas independientes
  (T002-T007 backend, T008-T011 frontend) — BLOQUEA todas las historias de usuario
- **US1 (Phase 3)**: depende de Foundational completo (necesita `promotionService.now()`/`ready()`
  reales); toca `pos-terminal.store.ts:248,262`
- **US2 (Phase 4)**: depende de Foundational completo; toca `pos-terminal.store.ts:386,1190` —
  independiente de US1 a nivel de código (líneas distintas del mismo fichero, sin superposición),
  secuenciada después para mantener el mismo orden RED→GREEN por historia
- **US3 (Phase 5)**: depende de T014, T015 (US1) y T019, T020 (US2) — verifica el resultado
  conjunto, no introduce cambio propio
- **Polish (Phase 6)**: depende de que US1+US2+US3 estén completas en ambos repos

### Notas de secuencia

A diferencia de la spec 022 (un solo repo), aquí **dos repos cambian en paralelo dentro de
Foundational** (T002-T007 backend, T008-T011 frontend) antes de que ninguna historia de usuario
pueda empezar — ambas pistas son prerequisito de US1/US2, no alternativas. Dentro de
`pos-terminal.store.ts`, US1 y US2 tocan líneas distintas del mismo fichero (248/262 vs. 386/1190):
técnicamente paralelizables por dos personas, pero ninguna tarea de edición se marca `[P]` porque
comparten fichero y revisor.

### Parallel Opportunities

T002 (backend) y T008 (frontend) pueden arrancar en paralelo — repos y ficheros distintos, ninguna
depende de que la otra pista esté completa. Dentro de cada pista, el resto es estrictamente
secuencial (RED → fix → GREEN). En Polish, T026 (backend) y T027 (frontend) son independientes
entre sí.

---

## Parallel Example: Foundational

```bash
# Estas dos pistas son independientes entre sí (repos y ficheros distintos):
cd ../pos-backend && python3 -m unittest app.characterization_tests.test_promotions_router -v   # T002-T007
cd ../pos-heladeria && npm test -- promotion.service.spec.ts                                     # T008-T011
```

---

## Implementation Strategy

### Orden recomendado

1. Phase 1 (Setup): confirmar la línea base en ambos repos (T001).
2. Phase 2 (Foundational): header backend (T002-T007) y `now()`/`ready` frontend (T008-T011), en
   paralelo si hay dos personas — al terminar, ambos repos saben calcular una hora sincronizada.
3. Phase 3 (US1): T012-T013 (RED, documenta el defecto en combos/insignias), T014-T015 (fix),
   T016 (GREEN) — al terminar, el catálogo del POS ya queda corregido.
4. Phase 4 (US2): T017-T018 (RED, documenta el defecto en el carrito), T019-T020 (fix), T021
   (GREEN) — al terminar, el precio del carrito ya queda corregido.
5. Phase 5 (US3): T022-T025 — confirma por ejecución y por revisión que no hay efectos fuera de
   alcance.
6. Phase 6 (Polish): corridas completas en ambos repos + validación de quickstart.md.

### Incremental Delivery

US1 y US2 son técnicamente desplegables por separado una vez completado Foundational (líneas
distintas del mismo fichero, sin dependencia funcional entre sí) — si se necesitara priorizar, US1
(combos/insignias, el catálogo completo del POS) podría entregarse primero, dejando el carrito
(US2) para un segundo despliegue. El PR natural de esta spec, sin embargo, entrega
Foundational+US1+US2+US3 juntas dado que Foundational ya es prerequisito de ambas y el tamaño total
es pequeño (1 header + ~10 líneas de servicio + 4 sustituciones de una línea). El desglose por
historia existe para trazabilidad de requisito → tarea → test (FR-001→US1, FR-002→US2,
FR-003/FR-004/FR-005/FR-006→US3), no para sugerir necesariamente PRs separados.
