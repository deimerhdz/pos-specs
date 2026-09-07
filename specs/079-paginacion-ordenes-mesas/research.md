# Research: Paginación y filtros en Órdenes y Mesas

Fase 0. El spec no dejó ningún `NEEDS CLARIFICATION` (8 aclaraciones resueltas con el negocio el 2026-09-07). Lo que sigue son las decisiones **técnicas** para implementar ese contrato reutilizando el patrón de paginación ya vigente en el sistema.

---

## 1. Forma del contrato: unión `list[T] | Page[T]` en el mismo endpoint

**Decision**: `GET /orders` y `GET /orders/tables` conservan su ruta y su verbo. Se les añaden parámetros de query **opcionales** `page` y `size`. El `response_model` pasa a ser una unión:

- `GET /orders` → `response_model=list[OrderResponse] | Page[OrderResponse]`
- `GET /orders/tables` → `response_model=list[TableResponse] | Page[TableResponse]`

Sin `page` **ni** `size`: la función devuelve exactamente lo de hoy (array; en órdenes, además, vía `json_or_304` con su `ETag`). Con `page` o `size` presente: devuelve un `Page[T]` construido con `app/core/pagination.py::paginate(db, stmt, page, size)`.

**Rationale**:
- "Decisiones de compatibilidad" del spec exige que *sin los parámetros de paginación, la respuesta y el comportamiento sean los actuales*. Un endpoint nuevo (`/orders/paged`) duplicaría la construcción de la consulta y dejaría dos rutas que mantener; cambiar el endpoint para devolver **siempre** `Page[T]` rompería el contrato de la Terminal de Mesas, el Dashboard y la hoja de QR (FR-023–FR-025).
- La unión en `response_model` ya tiene **precedente en el mismo codebase**: `app/api/v1/sales/router.py::list_payment_methods` declara `response_model=list[PaymentMethodResponse] | list[PaymentMethodCheckoutOption]` y ramifica por un query param. FastAPI valida la salida contra el primer miembro de la unión que encaje.
- El default de `page`/`size` es **ausente** (`None`), no `1`/`20`: es la única forma de distinguir "no quiero paginar" de "quiero la página 1". Cuando llega solo uno de los dos, el que falta toma su default de presentación (`page=1`, `size=20`).

**Alternatives considered**:
- *Endpoint nuevo `/orders/paged` + `/orders/tables/paged`*: descartado — dos rutas por listado, misma consulta construida dos veces, y el frontend tendría que elegir ruta según pantalla en vez de según parámetros. No reutiliza el patrón vigente (Ventas/Inventario paginan sobre su ruta única).
- *`GET /orders` devuelve siempre `Page[T]` y los consumidores completos piden `size=100000`*: descartado — rompe el contrato de 3 consumidores en producción, obliga a un `size` "mágico" y a un `COUNT(*)` inútil en cada sondeo de la Terminal.
- *Header `Prefer: pagination` en vez de query params*: descartado — invisible en logs y en el `queryKey` de TanStack, y sin precedente en el proyecto.

---

## 2. `GET /orders`: `active_sessions_only` y `ETag` conviven con la paginación sin mezclarse

**Decision**: la rama paginada y la rama `active_sessions_only=True` son **excluyentes por uso**, no por validación dura. El router:

1. Si `page is None and size is None` → camino de hoy: `service.list_orders(db, status_filter, active_sessions_only)` + `json_or_304(...)`. Idéntico byte a byte.
2. Si `page` o `size` presente → `paginate(db, service.list_orders_query(status_mostrado=..., order_type=...), page, size)`. **Sin** `ETag` (la pantalla "Órdenes" se recarga a mano; no se sondea). `active_sessions_only` se ignora en esta rama (solo lo manda la Terminal, que nunca pagina).

**Rationale**:
- El `ETag`/`304` de `json_or_404` existe porque la Terminal **sondea** `GET /orders` (ver docstring de `app/core/http_cache.py`). La pantalla "Órdenes" no sondea (Assumptions del spec: "se refresca al recargar la pantalla"), así que la respuesta paginada no gana nada con `ETag` y añadirlo obligaría a hashear el cuerpo de cada página.
- `active_sessions_only` filtra pedidos ya pagados de sesiones cerradas para la Terminal (spec 029 hotfix). La pantalla "Órdenes" **sí** quiere ver esos pedidos (histórico completo), por lo que combinar ambos no tiene caso de uso. No se emite un 400 si por error llegan juntos: se prioriza la paginación y se ignora la bandera, para no introducir una validación que ningún cliente legítimo puede disparar.

**Alternatives considered**:
- *Mantener `ETag` también en la rama paginada*: descartado — coste sin beneficio (no hay sondeo), y complica `json_or_304` para aceptar un `Page[T]`.
- *400 si `active_sessions_only` + `page`*: descartado — validación defensiva contra un caso que no existe; añade una rama de error que hay que testear y documentar.

---

## 3. Orden determinista y estable página a página (FR-003, SC-008)

**Decision**: el `Select` de órdenes ordena por **`customer_orders.created_at DESC, customer_orders.id DESC`**. El de mesas, por **`dining_tables.number ASC`** (ya es el orden actual, `DiningTable.number` es `UNIQUE` → sin empates posibles).

**Rationale**:
- `created_at` es `DateTime` con `server_default=func.now()` (resolución de microsegundos en Postgres). Los empates exactos son raros pero posibles (inserciones en la misma transacción / carga masiva). Sin un segundo criterio, `OFFSET/LIMIT` puede repetir u omitir una fila entre páginas.
- `customer_orders.id` es un `uuid4` aleatorio (`UUIDPrimaryKeyMixin`, `default=uuid.uuid4`), **no** una secuencia incremental. Como desempate cumple lo que SC-008 realmente pide —que "la paginación en servidor sea determinista y verificable página a página"—: para un conjunto fijo, `ORDER BY created_at DESC, id DESC` produce siempre la misma secuencia total, así que recorrer las páginas cubre el conjunto completo sin huecos ni repeticiones. La aclaración corregida del spec (sesión 2026-09-07) ya lo dice explícitamente: el identificador interno es "estable pero no cronológico", y dentro de un empate de milisegundo el orden entre esas pocas filas es arbitrario pero determinista; esas filas caen juntas de todos modos.
- El orden principal (`created_at DESC`) y el conjunto sin filtros son **idénticos a los de hoy**: `service.list_orders` ya hace `.order_by(CustomerOrder.created_at.desc())`. Solo se añade el desempate.

**Alternatives considered**:
- *Añadir una columna `sequence BIGSERIAL` a `customer_orders`*: descartado — es un cambio de esquema + migración (Principio VIII) para un problema que el desempate por `id` ya resuelve; el spec dice explícitamente "sin cambios de modelo de datos".
- *Desempatar solo por `id`* (sin `created_at` como primario): descartado — rompería el orden cronológico que la pantalla tiene hoy y que FR-003 exige conservar.
- *Keyset / cursor pagination* en vez de `OFFSET/LIMIT`: descartado — no es el patrón vigente (`paginate()` usa `OFFSET/LIMIT`), y el spec pide reutilizar el patrón existente, no introducir uno nuevo. El coste de `OFFSET` crece con la profundidad de página navegada (páginas × `size`), no con el total de órdenes; para profundidades realistas (`size ≤ 100`, primeras decenas de páginas) es barato. Los dos `COUNT(*)` del camino paginado (clamp + `paginate`) se resuelven sobre índices ya presentes, así que tampoco crecen de forma apreciable con el total (SC-001).

---

## 4. Traducción "estado que ve la persona" → predicado SQL (FR-008, FR-016)

**Decision**: el filtro de estado de "Órdenes" recibe uno de 6 valores y `list_orders_query` lo traduce así (misma lógica que `displayOrderStatus()` en el frontend hoy, movida al servidor):

| Valor del filtro | Etiqueta visible | Predicado sobre `customer_orders` |
|---|---|---|
| `todos` (ausente) | Todos | — sin filtro de estado — |
| `recibida` | Por confirmar | `status = 'recibida'` **AND NOT** `EXISTS (SELECT 1 FROM sales WHERE customer_order_id = customer_orders.id)` |
| `abierta` | Abierta | `status = 'abierta'` **AND NOT EXISTS** (venta) |
| `bloqueada` | Bloqueada | `status = 'bloqueada'` **AND NOT EXISTS** (venta) |
| `pagada` | Pagada | `status <> 'cancelada'` **AND** (`EXISTS (venta)` **OR** `status = 'pagada'`) |
| `cancelada` | Cancelada | `status = 'cancelada'` |

**Rationale**:
- FR-016 exige que el filtro coincida con lo que la fila muestra. `displayOrderStatus(order)` en `order-status.util.ts` ya define esa verdad: `if (status === 'cancelada') return 'cancelada'; return paid ? 'pagada' : status`. El port fiel a SQL es: `cancelada` gana siempre; una orden se ve "Pagada" si tiene `Sale` (`paid`) **o** si su `status` crudo ya es `'pagada'` (camino de sesión de mesa / datos legado) y no está cancelada; el resto conserva su `status` crudo solo si aún no tiene venta. La subconsulta `EXISTS` sobre `sales.customer_order_id` usa el mismo patrón ya probado en `order_has_sale`/`paid_order_ids` (`orders/service.py`).
- Los estados no terminales excluyen `EXISTS (venta)` para no mostrar bajo "Abierta"/"Bloqueada" un pedido que la persona ve como "Pagada".
- "Pagada" es `status <> 'cancelada' AND (EXISTS (venta) OR status = 'pagada')`: el `<> 'cancelada'` cubre el caso límite del spec (un pedido cancelado nunca se lista como pagado aunque tuviera venta previa); el `OR status = 'pagada'` evita que una orden con `status='pagada'` crudo sin `Sale` (datos legado) desaparezca de todos los filtros específicos pese a verse "Pagada" en la fila.
- Los 6 valores son el conjunto **completo y cerrado** (aclaración del spec): el parámetro se valida con un `Enum`/`pattern` de exactamente esos valores; cualquier otro → 422.

**Alternatives considered**:
- *Filtrar por `status` crudo* (sin la capa "estado mostrado"): descartado — un pedido `status='abierta'` con venta emitida aparecería bajo "Abierta" y no bajo "Pagada", contradiciendo lo que la fila muestra (FR-016, Edge Case explícito del spec).
- *Calcular `paid` fila a fila en Python tras paginar*: descartado — el filtro y el `COUNT(*)` tienen que resolverse en el servidor **antes** del `OFFSET/LIMIT` (FR-007: total y páginas del conjunto ya filtrado). Filtrar después de paginar daría páginas de tamaño irregular y un total equivocado.

---

## 5. Filtro por tipo de orden y órdenes sin tipo (FR-009, FR-018)

**Decision**: parámetro `order_type` opcional con 4 valores: ausente = "Todos"; `DINE_IN` = En mesa; `TAKEAWAY` = Para llevar; `DELIVERY` = Domicilio. Predicado: `customer_orders.order_type = :order_type`. Como la columna es *nullable*, una orden con `order_type IS NULL` **nunca** satisface `= 'DINE_IN'` (ni ningún otro valor) en SQL de tres valores → queda excluida automáticamente al filtrar por un tipo concreto (FR-018), y aparece cuando el filtro está en "Todos".

**Rationale**: coincide con `OrderType` (`app/api/v1/orders/schemas.py`) y con `_COMBINACIONES_CANAL_TIPO_ORDEN`. No hace falta un `COALESCE` ni un centinela: la semántica NULL de SQL ya da el comportamiento que pide FR-018. La etiqueta "Sin especificar" es puramente de presentación (frontend), sobre `order_type == null`.

**Alternatives considered**:
- *Backfill de `order_type` en órdenes históricas* (p. ej. a `DINE_IN` si tienen mesa): descartado — Principio VII (no aplicar reglas nuevas a operaciones ya finalizadas) y el spec lo prohíbe expresamente ("no se rellenan ni se modifican").
- *Un cuarto valor `UNSPECIFIED` en el filtro*: descartado — el spec fija 4 opciones (Todos, En mesa, Para llevar, Domicilio); "Sin especificar" es un estado de fila, no una opción de filtro.

---

## 6. Frontend: dos carriles distintos según el dueño del estado

**Decision**:

- **"Órdenes"** — servicio de transporte **nuevo y dedicado**, `orders-list.service.ts`, calcado de `SalesService`: signals de entrada (`page`, `size`, `status`, `orderType`), un `injectPagedQuery<DiningOrder>` cuyo `queryKey` las incluye, y `computed` derivados (`orders`, `total`, `totalPages`, `loading`, `error`). `DiningSessionService.listOrders()` **no se toca** (lo usan Terminal y Dashboard).
- **"Mesas"** — carril paginado **dentro** del singleton `TableService`: signals `tablesPage`/`tablesSize`, un `injectPagedQuery<Table>` separado, y `loadTablesPage(page, size)`. `loadTables()`, la signal `tables()` y las mutaciones (`createTable`/`updateTable`/`toggleActive`/`setStatus`) siguen exactamente como están: Terminal de Mesas, Dashboard, `order-detail` y `table-qr-sheet` las consumen sin enterarse del carril nuevo.

Ambas pantallas montan `<app-pagination-bar>` (sin cambios) y sus `(pageChange)`/`(sizeChange)` llaman al setter correspondiente; cambiar un filtro o el tamaño llama al setter con `page = 1` (FR-004), igual que `SalesService.setStatus()` hace hoy.

**Rationale**:
- `SalesService` e `InventoryService` ya demuestran el patrón exacto para una pantalla que **posee** su estado de paginación: `injectPagedQuery` + `placeholderData: prev` (mantiene las filas anteriores mientras carga, sin parpadeo) + `queryKey` reactivo (volver a una página/filtro ya visto sirve de caché). "Órdenes" encaja aquí sin inventar nada.
- `TableService` es `providedIn: 'root'` y su `tables()` es **estado compartido** por 4 consumidores. Un carril separado (signals + query propios) evita que paginar "Mesas" recorte la lista que la Terminal necesita completa. Es el mismo principio que en el backend (opt-in), aplicado al estado del cliente.

**Alternatives considered**:
- *Reutilizar `DiningSessionService.listOrders()` y paginar/filtrar en el cliente*: descartado — FR-006 exige paginación en servidor; el cliente no debe descargar el conjunto completo.
- *Convertir `TableService.loadTables()` en paginado y que Terminal/Dashboard pidan `size` grande*: descartado — mismo problema que en el backend (§1), trasladado al cliente; rompería `pos-terminal.store.ts` y `admin-dashboard`.
- *Un `TablesListService` nuevo aparte del singleton*: viable, pero duplica el transporte (`baseUrl`, `extractError`, tipos `Table`) que ya vive en `TableService`; el carril interno reutiliza todo eso y deja las mutaciones (que refrescan la lista) en un solo sitio.

---

## 7. Verificación (Principio X)

**Decision**: matriz de tests, sin modificar ningún `"CONGELA comportamiento actual:"` existente.

**Backend (`unittest`, `pos-backend`)**:
- `test_orders_pagination.py`:
  - `GET /orders` **sin** `page`/`size` → misma forma (array), mismo cuerpo, mismo `ETag` que hoy; `active_sessions_only=True` sigue funcionando.
  - `GET /orders?page=1&size=20` → `Page[OrderResponse]` con `items/total/page/size/pages`; `len(items) ≤ size`.
  - Orden determinista: con N órdenes (algunas con `created_at` idéntico), concatenar todas las páginas == `ORDER BY created_at DESC, id DESC` del conjunto completo, sin repetidos ni huecos (SC-008).
  - Página fuera de rango (`page` > `pages`) → devuelve la última página con resultados; conjunto vacío → `page=1`, `items=[]`, `total=0`, `pages=0` (FR-005).
  - `total`/`pages` corresponden al conjunto **ya filtrado** (FR-007).
- `test_orders_status_type_filters.py`:
  - cada uno de los 6 valores de estado devuelve exactamente el subconjunto esperado, incluido el caso "pedido `abierta` con `Sale`" → aparece bajo `pagada`, no bajo `abierta` (FR-016).
  - cada uno de los 4 valores de tipo; `order_type IS NULL` excluido al filtrar por un tipo, incluido en "Todos" (FR-018).
  - estado + tipo combinados → intersección (FR-011).
  - valor de estado/tipo inválido → 422.
- `GET /orders/tables?page=&size=` → `Page[TableResponse]`, orden `number ASC`; sin parámetros → array idéntico a hoy.
- Corrida completa de `app/characterization_tests/` en verde.

**Frontend (Jasmine/Karma, `pos-heladeria`)**:
- `orders-page.component.spec.ts` (reescrito): pinta solo `size` filas; cambiar `<select>` de estado o de tipo vuelve a página 1; cada fila muestra la etiqueta de tipo; `order_type == null` → "Sin especificar"; ya no existe el botón "Bloqueadas"; estado vacío "No hay órdenes con estos filtros".
- `tables-page.component.spec.ts` (nuevo): pinta una página; navegación recorre todas las mesas; tras `createTable`/`setStatus` la vista queda en una página válida.
- `table.service.spec.ts`: los tests existentes de `loadTables()`/`tables()` siguen pasando sin cambios (el carril paginado es aditivo).
- Corrida completa de `ng test` en verde.

**Aceptación**: los escenarios de las 3 historias y SC-001–SC-008 se ejecutan siguiendo `quickstart.md`.

**Rationale**: mantiene el mecanismo de test de cada repo (`unittest` / Jasmine) y prueba los tres contratos nuevos (compatibilidad sin parámetros, semántica de filtros, determinismo de páginas) sin tocar la red real ni los tests congelados.

**Alternatives considered**: ninguna — dado que el riesgo principal es una regresión en los consumidores del listado completo, el test de "sin parámetros = idéntico a hoy" es innegociable.

---

## 8. Decisión de negocio `A-72` (Principio II) — prerrequisito de implementación

**Decision**: antes de implementar las Historias 1 y 2, `/speckit-tasks` incluye una tarea T00x que crea la entrada **`A-72 — [DECISIÓN DE NEGOCIO — spec 079]`** en `specs/000-reconocimiento/registro-de-anomalias.md`, con el formato de A-71: **qué cambia** (retiro del grupo completo de botones de acceso rápido de estado —reemplazado por el filtro desplegable de US2, con el filtrado por estado ausente entre el despliegue de US1 y el de US2, y sin reintroducir el acceso rápido "Bloqueadas"—; filtrado de "Órdenes" server-side; listado segmentado en páginas; nueva opción de filtro "Por confirmar"), **por qué** (rendimiento de la pantalla que más crece; peticiones explícitas del negocio), **quién/cuándo** (propietario del proyecto, 2026-09-07, en `spec.md` + aclaraciones), **funcionalidades afectadas** (solo la pantalla "Órdenes" de `pos-heladeria`; el backend `GET /orders` sigue compatible sin parámetros), **clasificación** (DECISIÓN DE NEGOCIO), **tratamiento** (no retroactivo: sin esquema, sin `UPDATE`; revertir los commits de frontend restaura el comportamiento previo). La Historia 3 (paginación de "Mesas") **no** depende de `A-72`: no cambia comportamiento observable más allá de segmentar en páginas un listado cuyo orden y conjunto se conservan.

**Rationale**: el spec y la constitución (Principio II, "Flujo de Trabajo") exigen que la decisión de negocio esté registrada **antes** de implementar el cambio de comportamiento, con la cadena de trazabilidad cerrada. Este documento la deja especificada; crearla es trabajo de `/speckit-tasks` + `/speckit-implement`, no de `/speckit-plan`.

**Alternatives considered**: *no registrar `A-72` y apoyarse solo en el spec*: descartado — el Principio II pide explícitamente el registro en `registro-de-anomalias.md` para cambios de comportamiento que son decisión de negocio, y la revisión de cumplimiento lo verifica.
