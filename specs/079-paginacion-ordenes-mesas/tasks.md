---
description: "Task list para spec 079 — paginación y filtros en Órdenes y Mesas"
---

# Tasks: Paginación y filtros en Órdenes y Mesas

> **Estado de implementación** (`/speckit-implement`, 2026-09-07): **27/32 tareas hechas**. Código de
> las 3 historias completo en `../pos-backend` (rama `079-paginacion-ordenes-mesas`) y `../pos-heladeria`
> (rama `079-paginacion-ordenes-mesas`), con `A-72` registrada en `../pos-specs`.
> Suites verdes sin regresiones: **backend 755/755 OK** (baseline 727, +28 nuevos; `test_orders_service.py`
> sin tocar); **frontend 788 pasan / 18 fallan** (baseline 764/18 — los mismos 18 fallos preexistentes en
> 6 archivos ajenos: `app.spec`, `auth.service`, `menu.service`, `tenant.service`, `pos-checkout-panel`,
> `transfer-details-step`; **0 regresiones**, +24 nuevos). Build de producción del frontend: OK.
> **Pendientes (5)**: T012, T020, T027 (recorridos de `quickstart.md` por historia) y T032 (recorrido
> completo) — requieren stack corriendo + navegador + tenant sembrado; T031 quedó **parcial** (contrato
> compatible cubierto por tests nuevos + diff revisado; faltan `curl` y verificación visual).
> Una desviación registrada en T024 (carril paginado de `TableService` con signals + HTTP manual en vez
> de `injectPagedQuery`, por Principio II).

**Input**: Design documents de `/specs/079-paginacion-ordenes-mesas/`

**Prerequisites**: [plan.md](./plan.md), [spec.md](./spec.md), [research.md](./research.md), [data-model.md](./data-model.md), [contracts/orders-list-api.md](./contracts/orders-list-api.md), [contracts/tables-list-api.md](./contracts/tables-list-api.md), [quickstart.md](./quickstart.md)

**Tests**: incluidos — **no son opcionales aquí**. El plan (Constitution Check, Principio X — "Verificación Obligatoria"; [research.md §7](./research.md)) enumera la matriz completa de verificación: contrato sin parámetros (backend), envoltura `Page`, semántica de cada filtro, orden determinista página a página, clamp de página fuera de rango, guardia de trabajo acotado por petición (T005 f), specs de ambas pantallas de frontend, y corrida completa de las dos suites. Mismo criterio que [spec 078](../078-fix-terminal-mesas-responsive/tasks.md).

**Verificación diferida y manual**: los umbrales de reloj SC-001 (< 2 s, ≥ 5.000 órdenes), SC-003 (≤ 3 interacciones) y SC-005 (< 2 s, 500 mesas) **no** tienen guardia automatizada; se verifican en el recorrido de `quickstart.md` (T012/T020/T027/T032), que requiere navegador y datos sembrados. Responsable: propietario del proyecto, antes del despliegue de cada historia. El riesgo de regresión de rendimiento entre despliegues queda cubierto solo por T005 (f) (trabajo acotado por petición), no por un benchmark.

**Organización**: por historia de usuario de `spec.md` (US1 → US2 → US3, en orden de prioridad P1 → P2 → P3). **Dos repositorios**, independientes de `pos-specs`:

- `../pos-backend` — FastAPI. Se tocan `app/api/v1/orders/router.py` (2 endpoints) y `app/api/v1/orders/service.py` (1 constructora de `Select` nueva + 1 helper de clamp). Cero routers, modelos, migraciones o dependencias nuevas.
- `../pos-heladeria` — Angular. Se rediseñan `orders-page.component.ts` y `tables-page.component.ts`; 1 servicio de transporte nuevo (`orders-list.service.ts`); `table.service.ts` gana un carril paginado aditivo; `order-status.util.ts` gana un mapa de tipo de orden. `app-pagination-bar`, `injectPagedQuery` y `Page<T>` se reutilizan **sin tocar**.

El único artefacto fuera de esos dos repos es la entrada **`A-72`** en `../pos-specs/specs/000-reconocimiento/registro-de-anomalias.md` (T003), bloqueante **solo** para US1 y US2 (pantalla "Órdenes"). US3 (pantalla "Mesas") **no** depende de ella ([research.md §8](./research.md)).

Las tres piezas son 1:1 con las tres historias y verificables por separado:

1. **US1** — `GET /orders` gana `page`/`size` opcionales (unión `list | Page`); la pantalla "Órdenes" pagina en servidor y retira los botones de acceso rápido de estado (el filtrado en cliente desaparece con ellos).
2. **US2** — `GET /orders` gana `order_type` y amplía la semántica de `status` a "estado que ve la persona"; la pantalla añade dos `<select>` (estado / tipo) y muestra el tipo por fila.
3. **US3** — `GET /orders/tables` gana `page`/`size` opcionales; la pantalla "Mesas" pagina en servidor por un carril separado del singleton `TableService`.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: puede ejecutarse en paralelo (archivo/repo distinto, sin dependencia de una tarea sin terminar)
- **[Story]**: US1 a US3 — solo en fases de historia de usuario
- Cada tarea lleva la ruta con prefijo de repo (`pos-backend/…`, `pos-heladeria/…`, `pos-specs/…`)

---

## Phase 1: Setup

**Purpose**: línea base verde en ambos repos antes de tocar código, y la autorización de proceso (Principio II) para las historias que cambian comportamiento.

- [X] T001 [P] Registrar la línea base de tests de **`pos-backend`**: correr
      `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` desde `pos-backend/` y
      anotar el conteo verde/rojo, con atención a los tres grupos que este feature **NO** debe tocar
      en `pos-backend/app/characterization_tests/test_orders_service.py` —
      `TestService` (familia `test_create_order_*`), `TestOrderHasSale`,
      `TestListOrdersActiveSessionsOnly` — para distinguir después cualquier regresión de esta spec
      de un fallo preexistente (Principio X)
- [X] T002 [P] Registrar la línea base de tests de **`pos-heladeria`**: correr
      `npx ng test --watch=false` desde `pos-heladeria/` y anotar el conteo verde/rojo, con atención a
      `pos-heladeria/src/app/modules/tables/services/table.service.spec.ts`,
      `pos-heladeria/src/app/modules/orders/pages/orders-page.component.spec.ts` y
      `pos-heladeria/src/app/modules/orders/order-status.util.spec.ts`
- [X] T003 [P] Registrar la anomalía **`A-72`** en
      `pos-specs/specs/000-reconocimiento/registro-de-anomalias.md`, como entrada nueva **justo después
      de `A-71`** (antes de la sección `## Nota sobre una entrada de memoria-historica.md deliberadamente
      excluida`, línea ~2123), con el **mismo formato que `A-70`/`A-71`** (bloques **Qué cambia**,
      **Por qué cambia**, **Quién tomó la decisión y cuándo**, **Funcionalidades afectadas**,
      **Clasificación**, **Tratamiento acordado**) y el contenido del borrador de
      [research.md §8](./research.md): (1) retiro del **grupo completo de botones de acceso rápido de
      estado** (Todas / Abiertas / Bloqueadas / Pagadas / Canceladas) — los estados migran al filtro
      desplegable de US2 y el acceso rápido "Bloqueadas" no se reintroduce (FR-014); **si US1 se
      despliega antes que US2, la pantalla "Órdenes" queda sin filtro de estado en esa ventana**;
      (2) filtrado de "Órdenes" server-side; (3) listado segmentado en páginas; (4) nueva opción de
      filtro de estado "Por confirmar" (`recibida`). **Quién/cuándo**: propietario del proyecto,
      2026-09-07, en `spec.md` + las 8 aclaraciones de esa fecha. **Funcionalidades afectadas**: solo
      la pantalla "Órdenes" de `pos-heladeria` (incluida la ausencia temporal del filtrado por estado
      entre el despliegue de US1 y el de US2); `GET /orders` sigue compatible sin parámetros (FR-023).
      **Tratamiento**: no retroactivo — sin esquema, sin `UPDATE`; revertir los commits de frontend de
      US1+US2 restaura el comportamiento previo. Añadir además la fila `A-72` a la `## Tabla resumen`
      del mismo documento (línea ~2137). **Bloquea US1 y US2**; los commits de esas fases citan `A-72`.
      **US3 NO depende de esta tarea.** Es barata y va en el Setup, así que no retrasa el MVP

**Checkpoint**: ambas suites con conteo base registrado; `A-72` existe y cierra la cadena de trazabilidad (Necesidad → spec → plan → research → tasks → `A-72`).

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: la única pieza compartida por más de una historia — la regla de "página fuera de rango"
([data-model.md §2](./data-model.md), FR-005) — que aplican por igual la rama paginada de `GET /orders`
(US1) y la de `GET /orders/tables` (US3). `app/core/pagination.py` **no se toca** (plan.md, Structure
Decision): el clamp lo aporta este feature en la capa de servicio de `orders/`.

- [X] T004 En `pos-backend/app/api/v1/orders/service.py`, añadir el helper
      `clamp_page(db: Session, stmt: Select, page: int, size: int) -> int`: cuenta el `total` del
      conjunto **ya filtrado** (mismo `select(func.count()).select_from(stmt.order_by(None).subquery())`
      que usa `paginate`), calcula `pages = ceil(total / size)` (`0` si `total == 0`) y devuelve
      `1` si `total == 0`, si no `min(max(page, 1), pages)` ([data-model.md §2](./data-model.md), regla 3).
      El router llamará `paginate(db, stmt, clamp_page(...), size)`; los dos `COUNT(*)` (clamp +
      `paginate`) recaen sobre índices ya presentes (`sales.customer_order_id` está indexado; los
      filtros de `status`/`order_type` son de baja cardinalidad), así que su coste no crece de forma
      apreciable con el total, y T005 (f) fija que el nº de sentencias es constante
      ([research.md §3](./research.md)). No cambia la firma ni la salida de `list_orders()`.
      **Bloquea T007 (US1) y T022 (US3).**

**Checkpoint**: helper de clamp disponible y probado en contexto por T005 (US1) y T021 (US3).

---

## Phase 3: User Story 1 - Navegar las órdenes por páginas (Priority: P1) 🎯 MVP

**Goal**: `GET /orders` acepta `page`/`size` opcionales y, cuando llegan, devuelve un
`Page[OrderResponse]` (`items`/`total`/`page`/`size`/`pages`) con orden determinista
`created_at DESC, id DESC`; **sin** esos parámetros la respuesta y el comportamiento (array + `ETag` +
`304`, `active_sessions_only`) son idénticos a hoy. La pantalla "Órdenes" consume la ruta paginada por
un servicio de transporte nuevo y monta `app-pagination-bar`; ningún otro consumidor cambia.

**Independent Test**: con ≥ 130 órdenes, abrir `/dashboard/orders` sin filtros y verificar que se
muestran solo `size` filas (20 por defecto), que "Anterior/Siguiente" recorren todo el conjunto sin
huecos ni repeticiones, que "Página X de Y" y el total son correctos, que la respuesta de red nunca
trae más de `size` órdenes, y que el conjunto/orden página a página coincide con el listado actual
([quickstart.md §Historia 1](./quickstart.md)). No depende de US2 ni de US3.

### Tests for User Story 1

- [X] T005 [P] [US1] Crear `pos-backend/app/characterization_tests/test_orders_pagination.py`:
      (a) `GET /orders` **sin** `page`/`size` → cuerpo `list[OrderResponse]`, mismo `ETag`, y `304` con
      `If-None-Match`; `active_sessions_only=True` sigue filtrando como hoy (contrato compatible,
      FR-023/FR-024); `GET /orders?page=1&active_sessions_only=true` → `Page[OrderResponse]` con la
      bandera **ignorada** (sin `400`, [research.md §2](./research.md)). (b) `GET /orders?page=1&size=20` → `Page[OrderResponse]` con las 5 claves y
      `len(items) ≤ size`; solo `page` → `size` toma `20`; solo `size` → `page` toma `1`
      ([contracts/orders-list-api.md](./contracts/orders-list-api.md)). (c) **orden determinista**
      (SC-008): con N órdenes, algunas con `created_at` idéntico, concatenar todas las páginas ==
      `ORDER BY created_at DESC, id DESC` del conjunto completo — sin repetidos ni huecos.
      (d) **clamp** (FR-005): `page` > `pages` → devuelve la última página con resultados; conjunto
      vacío → `page=1`, `items=[]`, `total=0`, `pages=0`; nunca `404`. (e) `total`/`pages` del conjunto
      completo sin filtrar (FR-007). (f) **guardia de trabajo acotado (SC-001/SC-002)**: con un
      listener de `before_cursor_execute`, contar las sentencias SQL de un `GET /orders?page=1&size=20`
      y verificar que el número es **constante** (no crece con el total de órdenes ni con `size`) —
      atrapa un N+1 en la asignación en bloque de `paid` / `staff_user_name` sobre los `items`.
      Deben **fallar** antes de T006/T007
- [X] T006 [US1] En `pos-backend/app/api/v1/orders/service.py`, añadir
      `list_orders_query(status_mostrado: str | None = None, order_type: str | None = None) -> Select`:
      `select(CustomerOrder).options(<mismos selectinload que list_orders>)
      .order_by(CustomerOrder.created_at.desc(), CustomerOrder.id.desc())`. En US1 los dos parámetros
      son **no-ops** (los predicados se implementan en T014, US2) — solo se fija la firma y el orden.
      `list_orders()` (Terminal/Dashboard) **no se toca**: conserva firma y salida
      (plan.md, Constitution Check, Principio III)
- [X] T007 [US1] En `pos-backend/app/api/v1/orders/router.py::list_orders`: añadir
      `page: int | None = Query(None, ge=1)` y `size: int | None = Query(None, ge=1, le=100)`;
      `response_model = list[OrderResponse] | Page[OrderResponse]` (precedente:
      `sales/router.py::list_payment_methods`). Ramificar: si `page is None and size is None` →
      **camino de hoy sin cambios** (`service.list_orders(...)` + `paid`/`staff_user_name` en bloque +
      `json_or_304`). Si no → `stmt = service.list_orders_query(status_mostrado=status_filter,
      order_type=None)`; `pg = service.clamp_page(db, stmt, page or 1, size or 20)`;
      `result = paginate(db, stmt, pg, size or 20)`; asignar `paid` y `staff_user_name` **solo sobre
      `result["items"]`** (mismo patrón en bloque, `service.paid_order_ids` / `service.staff_user_names`);
      devolver `result` **sin `ETag`**; `active_sessions_only` se ignora en esta rama
      ([contracts/orders-list-api.md](./contracts/orders-list-api.md)). Hace pasar T005. Depende de
      T004, T006

### Implementation for User Story 1 (frontend)

- [X] T008 [P] [US1] Crear
      `pos-heladeria/src/app/modules/orders/services/orders-list.service.spec.ts`: el primer `list()`
      dispara `GET {apiBaseUrl}/orders?page=1&size=20`; la respuesta `Page<DiningOrder>` se mapea a
      `orders()` / `total()` / `totalPages()`; `list(2)` → `page=2` en el siguiente `queryKey`;
      `list(1, 50)` → `size=50`. Debe **fallar** antes de T009
- [X] T009 [US1] Crear `pos-heladeria/src/app/modules/orders/services/orders-list.service.ts`,
      calcado de `pos-heladeria/src/app/modules/sales/services/sales.service.ts`:
      `@Injectable({ providedIn: 'root' })`, `baseUrl = ${environment.apiBaseUrl}/orders`; signals
      `page = signal(1)`, `size = signal(20)`, `wantsPage = signal(false)` (US2 añade `status` y
      `orderType`); `injectPagedQuery<DiningOrder>` con
      `queryKey: () => ['orders','page',{ page: this.page(), size: this.size() }]`,
      `queryFn` → `fetchOrdersPage(...)` (`HttpParams` con `page`/`size`, `GET Page<DiningOrder>`),
      `enabled: () => this.wantsPage()`; `computed` `orders`, `total`, `totalPages`, `loading`
      (`isFetching`), `error` (con un `extractError` como el de `SalesService`); setter síncrono
      `list(page = this.page(), size = this.size())`. **No** importa ni toca
      `DiningSessionService.listOrders()` (lo usan Terminal y Dashboard, FR-024). Hace pasar T008
- [X] T010 [US1] Rediseñar `pos-heladeria/src/app/modules/orders/pages/orders-page.component.ts` para
      la paginación: inyectar `OrdersListService` en vez de leer `DiningSessionService`; `ngOnInit`
      llama `this.svc.list()` y `this.tableService.loadTables()` (la etiqueta de mesa sigue
      resolviéndose contra `tableService.tables()` completo, FR-025); eliminar el `computed`
      `visibleOrders` de orden/segmentación en cliente y pintar `svc.orders()` directo; montar
      `<app-pagination-bar [page]="svc.page()" [size]="svc.size()" [total]="svc.total()"
      [totalPages]="svc.totalPages()" [loading]="svc.loading()"
      (pageChange)="svc.list($event, svc.size())" (sizeChange)="svc.list(1, $event)" />`; el botón
      "Actualizar" llama `svc.list(svc.page(), svc.size())`; estado vacío "No hay órdenes"
      (el texto con filtros llega en US2). Importar `PaginationBarComponent`. **Retirar en esta misma
      tarea** el grupo de botones de acceso rápido `filters` / `setFilter` / `activeFilter` (incluido
      el botón "Bloqueadas", FR-014) y toda referencia de plantilla a `visibleOrders()` /
      `activeFilter()` — sin esos botones el filtrado en cliente ya no existe y la pantalla no puede
      quedar con controles muertos entre US1 y US2 (Principio II). Los dos `<select>` de estado/tipo y
      la columna de tipo llegan en US2 (T018). **Commit cita `A-72`** (retiro del botón, autorizado
      por T003). Depende de T003, T009
- [X] T011 [US1] Reescribir
      `pos-heladeria/src/app/modules/orders/pages/orders-page.component.spec.ts` para el flujo
      paginado: el `beforeEach` espera `GET {API}/orders?page=1&size=20` y le hace `flush` de un
      `Page` de prueba; la lista pinta exactamente `size` filas; `(pageChange)`/`(sizeChange)` de
      `app-pagination-bar` disparan un nuevo `GET` con los parámetros correctos; la pantalla nunca
      pide `GET /orders` sin `page`/`size`. Conservar los asserts de `displayOrderStatus` que sigan
      teniendo sentido (badge "Pagada" para `paid`); **añadir** asserts de que (i) la pantalla ya no
      renderiza ningún botón de acceso rápido de estado ("Todas" / "Abiertas" / "Bloqueadas" / …) y
      (ii) `app-pagination-bar` muestra "Página X de Y" y el total del `Page` recibido (FR-002).
      Depende de T010
- [ ] T012 [US1] Ejecutar [quickstart.md §Historia 1](./quickstart.md) (pasos 1–8) contra
      `pos-backend` + `pos-heladeria` reales con el tenant sembrado (≥ 130 órdenes; para el paso 7,
      ≥ 5.000). Verificar SC-001 (< 2 s), SC-002 (nunca más de `size` filas en red) y SC-008 (conjunto
      página a página == listado actual).
      → **Pendiente** — recorrido con navegador + datos sembrados (no ejecutable en esta sesión;
      ver T032)

**Checkpoint**: la pantalla "Órdenes" pagina en servidor, ya no muestra los botones de acceso rápido
de estado (retirados junto con el filtrado en cliente — no quedan controles muertos, Principio II) y
los tres consumidores del listado completo (`list_orders()` de Terminal y Dashboard, `json_or_304`
con `ETag`) siguen intactos. MVP entregable.

---

## Phase 4: User Story 2 - Filtrar las órdenes por estado y por tipo, con el tipo visible (Priority: P2)

**Goal**: la rama paginada de `GET /orders` acepta `order_type` (4 valores) y amplía `status` a los 6
valores de "estado que ve la persona" (data-model.md §3: "Pagada" = venta emitida o `status='pagada'` y no cancelada, etc.), combinables
con `AND`. La pantalla "Órdenes" —ya sin los botones de acceso rápido, retirados en US1/T010— añade
dos `<select>` (estado / tipo) de selección única que vuelven a la página 1 al cambiar, y pinta el
tipo de orden en cada fila ("Sin especificar" cuando `order_type` es NULL).

**Independent Test**: con órdenes de varios estados y tipos, aplicar cada filtro por separado y
combinados y comprobar que el listado, el total y las páginas reflejan exactamente el subconjunto;
que un pedido `abierta` con venta emitida cae bajo "Pagada" y no bajo "Abierta"; que la orden sin
tipo desaparece al filtrar por un tipo concreto y reaparece en "Todos" con la etiqueta
"Sin especificar"; y que el botón "Bloqueadas" ya no existe pero "Bloqueada" sí está en el filtro de
estado ([quickstart.md §Historia 2](./quickstart.md)).

**⚠️ Depende de US1** (mismos archivos: `list_orders_query`, `list_orders` del router,
`orders-list.service.ts`, `orders-page.component.ts`) **y de `A-72` (T003)** — el filtrado
server-side y la ampliación de la semántica de `status` son el cambio de comportamiento que `A-72`
autoriza en esta fase (el retiro de los botones de acceso rápido ya ocurrió en US1/T010, también
citando `A-72`); los commits de esta fase citan `A-72`.

### Tests for User Story 2

- [X] T013 [P] [US2] Crear
      `pos-backend/app/characterization_tests/test_orders_status_type_filters.py`: cada uno de los 6
      valores de `status` (`recibida`, `abierta`, `bloqueada`, `pagada`, `cancelada`, y ausente =
      "Todos") devuelve exactamente el subconjunto de [data-model.md §3](./data-model.md) —
      incluido el caso **`status='abierta'` con `Sale` emitida → aparece bajo `pagada`, no bajo
      `abierta`** (FR-016), **`pagada` excluye las canceladas con venta previa** e **incluye una orden
      `status='pagada'` sin `Sale`** (coincide con el badge que ve la persona, FR-016); cada uno de los 4
      valores de `order_type`, con **`order_type IS NULL` excluido al filtrar por un tipo concreto e
      incluido en "Todos"** (FR-018); `status` + `order_type` juntos → intersección (FR-011); valor
      de `status`/`order_type` fuera del enum → `422`. Deben **fallar** antes de T014/T015
- [X] T014 [US2] En `pos-backend/app/api/v1/orders/service.py::list_orders_query`, implementar los
      predicados (antes no-ops): definir
      `tiene_venta = exists(select(Sale.id).where(Sale.customer_order_id == CustomerOrder.id))`
      (mismo patrón que `order_has_sale`/`paid_order_ids`, sin tocarlas) y traducir `status_mostrado`
      según la tabla de [data-model.md §3](./data-model.md)
      (`recibida`/`abierta`/`bloqueada` → `status == X AND NOT tiene_venta`;
      `pagada` → `status != 'cancelada' AND (tiene_venta OR status == 'pagada')`;
      `cancelada` → `status == 'cancelada'`;
      ausente → sin predicado). `order_type` → `CustomerOrder.order_type == order_type` cuando no es
      `None` (la lógica ternaria de SQL ya excluye los NULL, FR-018). Hace pasar la parte de servicio
      de T013. Depende de T006
- [X] T015 [US2] En `pos-backend/app/api/v1/orders/router.py::list_orders`: añadir
      `order_type: OrderType | None = Query(None)` (enum ya en
      `app/api/v1/orders/schemas.py::OrderType`); en la **rama paginada**, validar `status_filter`
      contra el enum cerrado de 6 valores (`Query(pattern=...)` o `Enum` dedicado → `422` fuera de
      rango) y pasarlo como `status_mostrado`, y pasar `order_type` a
      `service.list_orders_query(...)`. La **rama compatible** conserva el filtro por `status` crudo
      (nota de compatibilidad de [contracts/orders-list-api.md](./contracts/orders-list-api.md)).
      Hace pasar T013. Depende de T007, T014. **Commit cita `A-72`**

### Implementation for User Story 2 (frontend)

- [X] T016 [P] [US2] En `pos-heladeria/src/app/modules/orders/order-status.util.ts`, añadir un mapa
      `ORDER_TYPE` análogo a `ORDER_STATUS` con `orderTypeLabel(t)` / `orderTypeClass(t)`:
      `DINE_IN` → "En mesa", `TAKEAWAY` → "Para llevar", `DELIVERY` → "Domicilio",
      `null`/`undefined`/`''` → "Sin especificar" ([data-model.md §5](./data-model.md)). No cambia
      `DiningOrder` (el campo `order_type?: string | null` ya existe). Actualizar
      `pos-heladeria/src/app/modules/orders/order-status.util.spec.ts` con los 4 casos + el nulo
- [X] T017 [US2] En `pos-heladeria/src/app/modules/orders/services/orders-list.service.ts`, añadir
      signals `status = signal<'' | 'recibida' | 'abierta' | 'bloqueada' | 'pagada' | 'cancelada'>('')`
      y `orderType = signal<'' | 'DINE_IN' | 'TAKEAWAY' | 'DELIVERY'>('')`; incluirlas en el
      `queryKey` y en los `HttpParams` de `fetchOrdersPage` (solo si no vacías); añadir setters
      `setStatus(v)` / `setOrderType(v)` que hacen `this.list(1)` (vuelta a página 1, FR-004 — igual
      que `SalesService.setStatus`). Extender T008 con estos casos. Depende de T009
- [X] T018 [US2] En `pos-heladeria/src/app/modules/orders/pages/orders-page.component.ts`
      (el grupo de botones de acceso rápido `filters` / `setFilter` / `activeFilter` ya se retiró en
      T010): añadir dos `<select>` de selección única —estado (Todos, Por confirmar, Abierta,
      Bloqueada, Pagada, Cancelada) y tipo (Todos, En mesa, Para llevar, Domicilio)— enlazados a
      `svc.setStatus(...)` / `svc.setOrderType(...)` con `svc.status()` / `svc.orderType()` como
      valor; pintar en cada fila la etiqueta de tipo con `orderTypeLabel(order.order_type)` (FR-017);
      estado vacío "No hay órdenes con estos filtros" (Edge Cases). Depende de T010, T016, T017
- [X] T019 [US2] Extender `pos-heladeria/src/app/modules/orders/pages/orders-page.component.spec.ts`:
      cambiar el `<select>` de estado o de tipo dispara un `GET /orders` con `status=` / `order_type=`
      y `page=1` (FR-004); cada fila muestra la etiqueta de tipo; `order_type == null` → "Sin
      especificar"; **no** existe ningún botón "Bloqueadas"; con 0 resultados se ve "No hay órdenes
      con estos filtros" y el pager muestra 0/0. Depende de T011, T018
- [ ] T020 [US2] Ejecutar [quickstart.md §Historia 2](./quickstart.md) (pasos 1–12) contra el stack
      real: cada filtro por separado y combinados, el caso "Pagada" con `status` interno `abierta`,
      la orden `order_type = NULL`, la combinación sin resultados, y SC-003 (aislar un subconjunto en
      ≤ 3 interacciones) / SC-004 (tipo visible en el 100 % de las filas).
      → **Pendiente** — recorrido con navegador + datos sembrados (ver T032)

**Checkpoint**: la pantalla "Órdenes" filtra por estado y tipo en servidor mediante los dos `<select>`,
muestra el tipo por fila y no tiene ningún botón de acceso rápido de estado (retirado en US1) — todo
cubierto por `A-72`. US1 + US2 funcionan juntas y por separado.

---

## Phase 5: User Story 3 - Navegar las mesas por páginas (Priority: P3)

**Goal**: `GET /orders/tables` acepta `page`/`size` opcionales y devuelve `Page[TableResponse]`
(orden `number ASC`, sin desempate — `number` es `UNIQUE`) cuando llegan; sin ellos, array idéntico a
hoy. La pantalla "Mesas" consume un carril paginado **añadido dentro** del singleton `TableService`
(no reemplaza `loadTables()` / `tables()` / mutaciones, que siguen sirviendo a Terminal, Dashboard,
`order-detail` y la hoja de QR — FR-025) y monta `app-pagination-bar`; tras crear/editar/activar/
cambiar estado de una mesa, refresca la página actual (o la última válida).

**Independent Test**: con ≥ 64 mesas, abrir `/dashboard/mesas` y verificar que se muestran solo `size`
mesas por número ascendente, que la navegación recorre todas, que la respuesta de red trae ≤ `size`
mesas, y que crear / editar / activar / desactivar / cambiar estado / ver QR sigue funcionando y deja
al usuario en una página válida ([quickstart.md §Historia 3](./quickstart.md)). **Independiente de US1
y US2; NO requiere `A-72`.**

### Tests for User Story 3

- [X] T021 [P] [US3] Añadir a `pos-backend/app/characterization_tests/test_orders_pagination.py` una
      clase para mesas: `GET /orders/tables` **sin** `page`/`size` → cuerpo `list[TableResponse]`,
      `ORDER BY number`, forma idéntica a hoy (FR-023/FR-025); `GET /orders/tables?page=1&size=20` →
      `Page[TableResponse]` con las 5 claves, orden `number ASC`, `len(items) ≤ size`; `total`/`pages`
      sobre el conjunto completo de mesas del tenant; **clamp** (`page` fuera de rango → última página;
      conjunto vacío → `page=1`, `items=[]`). Debe **fallar** (rama paginada) antes de T022
- [X] T022 [US3] En `pos-backend/app/api/v1/orders/router.py::list_tables`: añadir
      `page: int | None = Query(None, ge=1)` y `size: int | None = Query(None, ge=1, le=100)`;
      `response_model = list[TableResponse] | Page[TableResponse]`. Si `page is None and size is None`
      → hoy sin cambios (`select(DiningTable).order_by(DiningTable.number)`). Si no →
      `stmt = select(DiningTable).order_by(DiningTable.number.asc())`;
      `pg = service.clamp_page(db, stmt, page or 1, size or 20)`;
      `return paginate(db, stmt, pg, size or 20)`
      ([contracts/tables-list-api.md](./contracts/tables-list-api.md)). Hace pasar T021. Depende de T004

### Implementation for User Story 3 (frontend)

- [X] T023 [P] [US3] En `pos-heladeria/src/app/modules/tables/services/table.service.spec.ts`:
      confirmar que los tests existentes de `loadTables()` / `tables()` / mutaciones **siguen pasando
      sin cambios** (el carril paginado es aditivo); añadir: `loadTablesPage(1, 20)` dispara
      `GET {apiBaseUrl}/orders/tables?page=1&size=20` y mapea `Page<Table>` a `pagedTables()` /
      `tablesTotal()` / `tablesTotalPages()`; `loadTablesPage(2, 20)` → `page=2`. Debe **fallar**
      antes de T024
- [X] T024 [US3] En `pos-heladeria/src/app/modules/tables/services/table.service.ts`, añadir el
      carril paginado **sin tocar** `loadTables()`, `tables()`, `createTable`, `updateTable`,
      `toggleActive`, `setStatus`, `getQrToken`: signals `tablesPage = signal(1)`,
      `tablesSize = signal(20)`, `wantsTablesPage = signal(false)`; `injectPagedQuery<Table>` con
      `queryKey: () => ['tables','page',{ page: this.tablesPage(), size: this.tablesSize() }]`,
      `queryFn` (`HttpParams` `page`/`size`, `GET Page<Table>` sobre `this.baseUrl`),
      `enabled: () => this.wantsTablesPage()`; `computed` `pagedTables`, `tablesTotal`,
      `tablesTotalPages`, `tablesLoading`; método `loadTablesPage(page, size)` y
      `refreshTablesPage()` (re-`invalidate`/re-fetch de la página actual) para llamar tras una
      mutación. Depende de T023
      → **Desviación implementada** (`/speckit-implement`, 2026-09-07): el carril se resolvió con
      **signals + HTTP manual** (`firstValueFrom`, mismo estilo que el `loadTables()` existente) en
      vez de `injectPagedQuery`. Motivo (Principio II): `injectPagedQuery` en el inicializador de
      campo del singleton `TableService` introduce una dependencia dura de construcción de
      `QueryClient` que rompía specs existentes de 8 consumidores que no la proveen
      (`order-detail.component.spec.ts` confirmado en rojo). El carril manual es aditivo de verdad —
      cero blast radius, todos los specs previos de `table.service`/`order-detail`/`pos-terminal`…
      siguen verdes sin tocarlos (T029). API pública idéntica a la planeada (`tablesPage`,
      `tablesSize`, `pagedTables`, `tablesTotal`, `tablesTotalPages`, `tablesLoading`,
      `loadTablesPage`, `refreshTablesPage`); se pierde solo el `placeholderData` de TanStack, que
      "Mesas" no necesitaba (su `loadTables()` tampoco lo tiene). `OrdersListService` (US1) **sí**
      usa `injectPagedQuery` — servicio nuevo, consumidores bajo control.
- [X] T025 [US3] Rediseñar `pos-heladeria/src/app/modules/tables/pages/tables-page.component.ts`:
      `ngOnInit` → `this.tableService.loadTablesPage(1, 20)` en vez de `loadTables()`; el `@for` itera
      `tableService.pagedTables()`; montar `<app-pagination-bar [page]="tableService.tablesPage()"
      [size]="tableService.tablesSize()" [total]="tableService.tablesTotal()"
      [totalPages]="tableService.tablesTotalPages()" [loading]="tableService.tablesLoading()"
      (pageChange)="tableService.loadTablesPage($event, tableService.tablesSize())"
      (sizeChange)="tableService.loadTablesPage(1, $event)" />`; en `onSaved` / `onToggle` /
      `changeStatus` (y tras cerrar el form de creación) llamar `tableService.refreshTablesPage()`
      para que la lista quede en una página válida y coherente (FR-022). Importar
      `PaginationBarComponent`. No se añaden filtros a "Mesas" (Assumptions). Depende de T024
- [X] T026 [P] [US3] Crear
      `pos-heladeria/src/app/modules/tables/pages/tables-page.component.spec.ts`: la pantalla pide
      `GET /orders/tables?page=1&size=20` al montar y pinta ≤ `size` filas; `(pageChange)` dispara el
      `GET` de la página siguiente; tras `createTable` / `setStatus` (mock del `TableService`) se
      llama `refreshTablesPage()` y la vista sigue mostrando una página válida. Depende de T025
- [ ] T027 [US3] Ejecutar [quickstart.md §Historia 3](./quickstart.md) (pasos 1–7) contra el stack
      real (≥ 64 mesas; para el paso 7, 500): navegación completa, todas las operaciones de gestión,
      SC-005 (< 2 s con 500 mesas) y SC-008 (conjunto/orden == listado actual).
      → **Pendiente** — recorrido con navegador + datos sembrados (ver T032)

**Checkpoint**: la pantalla "Mesas" pagina en servidor por un carril propio; Terminal, Dashboard,
`order-detail` y la hoja de QR siguen recibiendo `tables()` completo. Las tres historias funcionan de
forma independiente.

---

## Phase 6: Polish & Cross-Cutting Concerns (No regresión — FR-023 a FR-026, SC-006/SC-007)

- [X] T028 [P] Correr `python -m unittest discover -s app/characterization_tests -p 'test_*.py'`
      completo en `pos-backend` y confirmar **0 fallos nuevos** frente a la línea base de T001, con
      `test_orders_service.py` (`TestService`, `TestOrderHasSale`, `TestListOrdersActiveSessionsOnly`)
      **sin modificar** y en verde
- [X] T029 [P] Correr `npx ng test --watch=false` completo en `pos-heladeria` y confirmar **0 fallos
      nuevos** frente a la línea base de T002; `table.service.spec.ts` conserva sus tests previos
      sin cambios y en verde (carril paginado aditivo)
- [X] T030 [P] Verificación de no-intrusión: confirmar que el diff **no** toca
      `pos-backend/app/core/pagination.py`, `pos-heladeria/src/app/core/query/paged-query.ts`,
      `pos-heladeria/src/app/core/interfaces/page.interface.ts`,
      `pos-heladeria/src/app/shared/pagination/pagination-bar.component.ts`,
      `pos-heladeria/src/app/modules/tables/services/dining-session.service.ts` ni
      `pos-heladeria/src/app/modules/dashboard/pages/admin-dashboard.component.ts`; y que **no** hay
      ninguna migración de Alembic nueva (`grep` en `pos-backend/alembic/versions/` por fecha)
- [~] T031 Ejecutar [quickstart.md §"No-regresión"](./quickstart.md) (pasos 1–8): Terminal de Mesas
      y Dashboard muestran todas las órdenes/mesas como hoy; "Pagos por confirmar" sin cambios; la
      hoja de QR y la resolución de etiqueta de mesa en el detalle de orden funcionan;
      `curl "$API/api/v1/orders"` sin parámetros → array + `ETag`, y `304` con `If-None-Match`;
      `curl "$API/api/v1/orders/tables"` sin parámetros → array ordenado por `number`. Dejar
      constancia en el commit.
      → **Parcial hecho** (`/speckit-implement`, 2026-09-07): la parte de contrato compatible está
      cubierta por characterization tests nuevos —`test_orders_pagination.py::TestOrdersListModoCompatible`
      (`test_sin_page_ni_size_devuelve_array_con_etag_y_304`, `test_active_sessions_only_sigue_filtrando_como_hoy`,
      `test_status_crudo_se_conserva_en_modo_compatible`) y `TestTablesListPaginado::test_sin_page_ni_size_devuelve_array_ordenado_por_number`—;
      diff revisado (T030 verde: no toca `pagination.py` ni ningún consumidor completo; sin migración
      Alembic; `test_orders_service.py` sin modificar). **Falta**: las llamadas `curl` contra un
      backend levantado y la parte visual (Terminal, Dashboard, QR, detalle de orden) — requieren
      stack corriendo + navegador (ver T032).
- [ ] T032 Ejecutar el recorrido manual completo de [quickstart.md](./quickstart.md) (Historias 1–3
      + no-regresión) contra `pos-backend` + `pos-heladeria` reales con el tenant sembrado
      (Prerrequisitos de `quickstart.md`), confirmando SC-001 a SC-008. **Pendiente** — requiere
      navegador y datos sembrados; no ejecutable sin entorno visual. Responsable: propietario del
      proyecto, antes del despliegue de cada historia (incluye los umbrales de reloj SC-001/SC-003/
      SC-005, sin guardia automatizada).

---

## Dependencies & Execution Order

### Phase Dependencies

- **Fase 1 (Setup)**: sin dependencias. **T003 (`A-72`) bloquea US1 y US2**, no US3.
- **Fase 2 (Foundational)**: T004 depende solo del repo `pos-backend`. **Bloquea T007 (US1) y
  T022 (US3).**
- **Fase 3 (US1)**: depende de la línea base (T001/T002), de T003 (`A-72`) y de T004. Sin dependencia
  de US2 ni US3.
- **Fase 4 (US2)**: **depende de US1** (mismos archivos: `list_orders_query`, `list_orders` del
  router, `orders-list.service.ts`, `orders-page.component.ts`) **y de T003 (`A-72`)**.
- **Fase 5 (US3)**: depende de la línea base y de T004. **Independiente de US1/US2 y de `A-72`**
  (archivos distintos: `list_tables` del router, `table.service.ts`, `tables-page.component.ts`).
- **Fase 6 (Polish)**: depende de todas las historias en alcance que se vayan a entregar.

### User Story Dependencies

- **US1 (P1)** — MVP. Setup (incl. `A-72` / T003, que ahora también cubre el retiro de los botones
  de acceso rápido en T010) + T004. Independiente de US2/US3.
- **US2 (P2)** — **depende de US1** (extiende sus archivos) y de `A-72`.
- **US3 (P3)** — independiente en código; solo necesita T004. Puede adelantarse en paralelo a US1
  con otro desarrollador.

### Within Each User Story

- Los tests preceden a la implementación y deben **fallar antes**: T005 → T006/T007;
  T008 → T009; T010 → T011; T013 → T014/T015; T008(ext) → T017; T018 → T019;
  T021 → T022; T023 → T024; T025 → T026.
- Backend: `list_orders_query` (T006) y `clamp_page` (T004) antes del router (T007).
- Frontend: el servicio de transporte (T009 / T024) antes del componente (T010 / T025).
- La tarea de recorrido de `quickstart.md` cierra cada fase de historia.

---

## Parallel Opportunities

- **Fase 1**: T001 (`pos-backend`), T002 (`pos-heladeria`) y T003 (`pos-specs`) en paralelo — repos
  distintos.
- **US1 y US3 en paralelo** con dos desarrolladores tras T004: US1 toca
  `orders/router.py::list_orders` + `orders/service.py::list_orders_query` +
  `orders-list.service.ts` + `orders-page.component.ts`; US3 toca
  `orders/router.py::list_tables` + `table.service.ts` + `tables-page.component.ts`. El único archivo
  compartido es `orders/router.py` (funciones distintas — coordinar el merge) y `orders/service.py`
  (US1 añade `list_orders_query`, T004 añadió `clamp_page` — sin solape).
- **US2 va después de US1** (mismos archivos).
- Dentro de cada historia, los tests marcados `[P]` (T005, T008, T013, T016, T021, T023, T026) corren
  en paralelo entre sí y con las tareas de backend de otra historia.

### Ejemplo — reparto por desarrollador

```
Dev A: US1 → US2      (la pantalla "Órdenes": paginación server-side, luego filtros + tipo visible)
Dev B: US3            (la pantalla "Mesas": carril paginado en TableService, tras T004)
```

### Ejemplo — tests en paralelo, US1

```bash
Task T005: "pos-backend/…/test_orders_pagination.py — contrato sin/con page, orden determinista, clamp"
Task T008: "pos-heladeria/…/orders-list.service.spec.ts — primer list() → GET ?page=1&size=20"
# Luego T006/T007 (backend) y T009 (servicio) hacen pasar sus tests.
```

---

## Implementation Strategy

### MVP (solo US1)

1. Fase 1 (Setup) — líneas base + `A-72` (T003, barata, desbloquea también US2).
2. Fase 2 (Foundational) — `clamp_page` (T004).
3. Fase 3 (US1) — `page`/`size` opt-in en `GET /orders` + servicio de transporte + `app-pagination-bar`.
4. **DETENERSE Y VALIDAR**: [quickstart.md §Historia 1](./quickstart.md) — la pantalla pagina en
   servidor, los consumidores completos intactos (SC-001, SC-002, SC-006, SC-008).
5. Desplegar — resuelve el problema central (la pantalla que más crece del sistema).

### Entrega incremental

1. Setup + Foundational → base lista.
2. + US1 → "Órdenes" pagina en servidor y se retiran los botones de acceso rápido de estado (MVP,
   requiere `A-72`). Validar SC-001/SC-002/SC-008. Desplegar.
3. + US2 → filtros de estado y tipo en servidor (dos `<select>`), tipo visible (requiere `A-72`).
   Validar SC-003/SC-004/SC-007. Desplegar.
4. + US3 → "Mesas" pagina en servidor por carril propio. Validar SC-005/SC-006/SC-008. Desplegar.
5. Fase 6 (Polish) sobre lo entregado — no regresión (FR-023–FR-026, SC-006).

US3 puede adelantarse en paralelo a US1; US2 siempre después de US1.

---

## Notes

- `[P]` = archivos/repos distintos, sin dependencia de una tarea sin terminar.
- La etiqueta `[Story]` mapea cada tarea a su historia para trazabilidad (Principio XII).
- Un commit por tarea (o grupo lógico), citando el `FR`/contrato; los commits de **US1 y US2** citan
  además `A-72`.
- Verificar que los tests fallan antes de implementar donde el test precede a la implementación
  (Principio III/X).
- Detenerse en cada checkpoint para validar la historia de forma aislada.
- Evitar: cambiar `app/core/pagination.py`, `paged-query.ts`, `page.interface.ts` o
  `pagination-bar.component.ts` (se reutilizan tal cual, plan.md); tocar `list_orders()`,
  `DiningSessionService.listOrders()` o el `admin-dashboard` (FR-024); añadir `ETag` a la rama
  paginada (research.md §2); rellenar o modificar `order_type` de órdenes históricas (Principio VII,
  FR-018); convertir `TableService.loadTables()` / `tables()` en paginado (research.md §6); emitir
  `400` si `active_sessions_only` llega con `page` (se ignora, research.md §2); añadir búsqueda por
  texto o filtros nuevos a "Mesas" (Assumptions).
