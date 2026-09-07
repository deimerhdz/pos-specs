# Implementation Plan: Paginación y filtros en Órdenes y Mesas

**Branch**: `079-paginacion-ordenes-mesas` | **Date**: 2026-09-07 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/079-paginacion-ordenes-mesas/spec.md`

## Summary

Llevar la paginación en servidor —el patrón ya vigente en Ventas, Inventario y Auditoría (`app/core/pagination.py::paginate` + `Page[T]` en el backend; `injectPagedQuery` + `app-pagination-bar` en el frontend)— a dos pantallas que hoy descargan y pintan el conjunto completo: **"Órdenes"** (`/dashboard/orders`) y **"Mesas"** (`/dashboard/mesas`). La paginación es **opt-in**: se añaden parámetros `page`/`size` opcionales a `GET /orders` y a `GET /orders/tables`; sin esos parámetros la respuesta y el comportamiento son exactamente los de hoy, de modo que la Terminal de Mesas, el Dashboard, la hoja de impresión de QR y la resolución de etiqueta de mesa siguen recibiendo la lista completa sin cambios (FR-023–FR-025).

Además, en la pantalla "Órdenes": (a) el filtrado pasa a resolverse en el servidor junto con la paginación; (b) se retira el botón de acceso rápido "Bloqueadas" y se añaden **dos filtros independientes de selección única** —estado (6 opciones) y tipo de orden (4 opciones), combinables— (FR-008–FR-016); (c) cada fila muestra el **tipo de orden** (En mesa / Para llevar / Domicilio / Sin especificar), dato que la API ya expone (`OrderResponse.order_type`) pero la lista no pinta (FR-017–FR-018). El filtro de estado coincide con lo que la persona ve en la fila: "Pagada" = pedido con `Sale` emitida **o** `status='pagada'` crudo, en ambos casos si no está cancelado; "Cancelada" = `status='cancelada'`; el resto = ese `status` y aún sin `Sale` (FR-016) — el port fiel de `displayOrderStatus()` (`status==='cancelada'` gana; si no, `paid ? 'pagada' : status`), movido al servidor (ver data-model.md §3).

Sin cambios de modelo de datos, sin migración, sin dependencias nuevas (FR-026). El cambio de comportamiento deliberado de la pantalla "Órdenes" (retiro de los botones de acceso rápido de estado, filtrado server-side, listado segmentado) exige una entrada de decisión de negocio (`A-72`) en `specs/000-reconocimiento/registro-de-anomalias.md` **antes** de implementarse (Principio II) — este plan la deja identificada; `/speckit-tasks` la secuencia como prerrequisito.

## Technical Context

**Language/Version**: Python 3.12 (`pos-backend`, imagen `python:3.12-slim`, sin cambio) · TypeScript 5.x / Angular 20 standalone + signals (`pos-heladeria`, sin cambio)

**Primary Dependencies**: FastAPI 0.136.3, SQLAlchemy 2.0.50, Pydantic v2 (`pos-backend`) · `@tanstack/angular-query-experimental` ^5.101.4, `@angular/forms` (`pos-heladeria`). **Ninguna dependencia nueva** (Principio IX): `app/core/pagination.py`, `app/core/query/paged-query.ts` y `app/shared/pagination/pagination-bar.component.ts` ya existen y están en producción en otras pantallas.

**Storage**: PostgreSQL 16, schema-per-tenant. **Sin cambios de esquema, sin migración de Alembic.** Se leen columnas ya existentes (`customer_orders.status`, `customer_orders.order_type`, `customer_orders.created_at`, `customer_orders.id`, `dining_tables.number`) y la existencia de `sales.customer_order_id` (subconsulta, patrón ya usado por `order_has_sale`/`paid_order_ids`).

**Testing**: `unittest` (biblioteca estándar) vía `python -m unittest discover -s app/characterization_tests -p 'test_*.py'` en `pos-backend` (no hay `pytest`). Jasmine/Karma (`ng test`) en `pos-heladeria`.

**Target Platform**: Linux server (contenedor Docker existente de `pos-backend`) · SPA servida como estático + PWA (`pos-heladeria`).

**Project Type**: web — backend FastAPI (`pos-backend`) + frontend Angular (`pos-heladeria`), repositorios independientes de `pos-specs`.

**Performance Goals**: en "Órdenes", tiempo hasta la primera página < 2 s con ≥ 5.000 órdenes acumuladas (SC-001); en "Mesas", carga < 2 s con 500 mesas (SC-005). La pantalla nunca transfiere ni pinta más filas que el tamaño de página elegido (SC-002). El `COUNT(*)` envolvente de `paginate()` sobre el conjunto filtrado se resuelve con los índices ya presentes (`idx_customer_orders_order_type`, `ck_customer_order_status` no es índice pero el filtro por `status` es de baja cardinalidad; `dining_tables.number` es `unique`, por tanto indexado).

**Constraints**:
- Compatibilidad de contrato (FR-023, "Decisiones de compatibilidad"): `GET /orders` y `GET /orders/tables` **sin** `page`/`size` devuelven la **misma forma de respuesta** (array JSON, con `ETag`/`304` en el caso de órdenes) y el mismo comportamiento que hoy; solo con `page`/`size` presentes devuelven `Page[T]`.
- El parámetro `active_sessions_only` de `GET /orders` conserva su semántica (FR-024) y **no** se combina con paginación (solo lo usa la Terminal de Mesas, que no pagina).
- El listado de "Órdenes" no es en tiempo real hoy y sigue sin serlo (Assumptions): la respuesta paginada no necesita `ETag` (se recarga a mano).
- `TableService` (`pos-heladeria`) es un singleton `providedIn: 'root'` cuya signal `tables()` la comparten Terminal de Mesas, Dashboard, `order-detail` y la hoja de QR: la ruta paginada de "Mesas" **no** puede reemplazar `loadTables()`/`tables()`; se añade un carril de estado paginado separado que solo consume `tables-page.component.ts`.
- Órdenes históricas sin `order_type` (NULL): se muestran como "Sin especificar" y quedan **fuera** al filtrar por un tipo concreto (FR-018) — nunca se rellenan ni se modifican (Principio VII).

**Scale/Scope**:
- `pos-backend`: 2 endpoints tocados (`GET /orders`, `GET /orders/tables`), 1 función de servicio reescrita como constructora de `Select` (`orders/service.py::list_orders`), sin routers ni modelos nuevos.
- `pos-heladeria`: 2 pantallas rediseñadas (`orders-page.component.ts`, `tables-page.component.ts`), 1 servicio de transporte nuevo o ampliado para el carril paginado de órdenes, `TableService` ampliado con un carril paginado, 1 utilidad de etiqueta de tipo de orden. `app-pagination-bar` se reutiliza sin tocar.
- 3 historias de usuario priorizadas (P1 paginación de órdenes · P2 filtros + tipo visible · P3 paginación de mesas), entregables de forma independiente y en ese orden.

## Constitution Check

*GATE: Debe pasar antes de la Fase 0. Re-evaluado tras la Fase 1 (ver final de la sección).*

| Principio | Evaluación |
|---|---|
| I. Las Nuevas Funcionalidades Nacen de un Spec | ✅ Pass — `specs/079-paginacion-ordenes-mesas/spec.md` aprobado, con 8 aclaraciones registradas (sesión 2026-09-07) y checklist de calidad en verde. |
| II. El Comportamiento Existente Sigue Protegido | ⚠️ Pass **condicionado** — la pantalla "Órdenes" cambia de comportamiento de forma deliberada: (a) se retira el grupo de botones de acceso rápido de estado (el acceso rápido "Bloqueadas" no se reintroduce; los demás estados migran al filtro desplegable de US2, con el filtrado por estado temporalmente ausente si US1 se despliega antes que US2); (b) el filtrado pasa de cliente a servidor; (c) el listado deja de mostrar todas las órdenes y pasa a segmentarse; (d) se añade "Por confirmar" (`recibida`) como opción de filtro de estado, hoy inexistente. El spec (§"Impacto sobre el Sistema Existente") ya lo declara. Requiere una entrada **`A-72`** en `specs/000-reconocimiento/registro-de-anomalias.md` con quién/cuándo/qué cambia/por qué/funcionalidades afectadas, creada **antes** de implementar las Historias 1 y 2 — `/speckit-tasks` la ordena como T00x prerrequisito y los commits de esas fases la citan. La pantalla "Mesas" y los demás consumidores no cambian de comportamiento (FR-023–FR-025), por lo que solo "Órdenes" entra en este condicionamiento. |
| III. Los Characterization Tests Protegen el Comportamiento Heredado | ✅ Pass — no se modifica ningún test `"CONGELA comportamiento actual:"`. `test_orders_service.py::test_sin_el_parametro_el_resultado_no_cambia_respecto_a_hoy` y la familia `test_create_order_*` siguen intactos y en verde: `list_orders` conserva su firma y su salida cuando se la llama sin los parámetros nuevos. Se añaden tests nuevos junto a los existentes (backend: contrato sin/ con `page`, semántica de cada filtro, orden determinista, clamp de página; frontend: nuevos specs de ambas pantallas). |
| IV. Los Nuevos Specs Pueden Introducir Nuevo Comportamiento | ✅ Pass — el nuevo comportamiento (paginación, filtros server-side, tipo visible, retiro de los botones de acceso rápido de estado) está definido y acotado en el spec; el criterio de éxito es la conformidad con él más la ausencia de regresiones no autorizadas en los demás consumidores. |
| V. Nuevas Funcionalidades Antes que Refactorizaciones Oportunistas | ✅ Pass — cada cambio se asocia a un FR y a una historia. Reescribir `list_orders` como constructora de `Select` filtrable es lo mínimo que exige FR-006/FR-012 (paginar y filtrar en el servidor), no una generalización especulativa. No se toca el patrón de `ETag`, ni `paginate()`, ni `pagination-bar.component.ts`, ni los otros consumidores. |
| VI. Evolución Incremental | ✅ Pass — una sola clase de cambio (paginación/filtrado de listados), dividida en 3 historias verificables por separado; sin migraciones, sin cambio de arquitectura, sin tocar el ciclo de vida de la orden ni la semántica de "bloqueada" (FR-026). |
| VII. Compatibilidad con Datos Históricos | ✅ Pass — ninguna factura ni venta emitida se recalcula ni se re-representa. Las órdenes históricas sin `order_type` se leen como "Sin especificar" en presentación y filtrado; no hay `UPDATE` ni backfill. |
| VIII. Evolución del Modelo de Datos | N/A — sin entidades, campos, relaciones, valores por defecto ni migraciones nuevas. `data-model.md` describe formas de consulta y de respuesta (`Page[T]`, mapeo estado-mostrado→predicado), no cambios de esquema. |
| IX. Dependencias Nuevas Permitidas con Justificación | ✅ Pass (no aplica justificación) — se reutilizan `app/core/pagination.py`, `injectPagedQuery` y `app-pagination-bar`, ya presentes y en producción. Cero dependencias nuevas. |
| X. Verificación Obligatoria | ✅ Pass (planificado) — `research.md` §7 define la matriz de verificación: characterization del contrato sin parámetros (backend), tests de la envoltura `Page` y de la semántica de cada filtro, test del orden determinista página a página (SC-008), test del clamp de página fuera de rango (FR-005); specs de frontend de ambas pantallas; y corrida completa de las suites existentes de `pos-backend` y `pos-heladeria`. Más los criterios de aceptación del spec ejecutados vía `quickstart.md`. |
| XI. Decisiones de Negocio Frente a Decisiones Técnicas | ✅ Pass — las decisiones de negocio (opt-in, alcance de "módulo de mesas", 6 valores exactos del filtro de estado, retiro de los botones de acceso rápido de estado, tratamiento de "Pagada" (venta emitida o estado interno ya "pagada"), clamp a la última página válida, desempate determinista) ya están fijadas en el spec y sus aclaraciones; este plan solo resuelve el "cómo". `A-72` registra el cambio de comportamiento. |
| XII. Trazabilidad | ✅ Pass — Necesidad (pedido del negocio) → `spec.md` + Aclaraciones → este plan → `research.md`/`data-model.md`/`contracts/`/`quickstart.md` → `tasks.md` (`/speckit-tasks`) → tests. `A-72` cierra la cadena del cambio de comportamiento. |
| XIII. Todo en Español de Colombia | ✅ Pass — todos los artefactos de este feature y los textos visibles nuevos ("Por confirmar", "En mesa", "Para llevar", "Domicilio", "Sin especificar", "No hay órdenes con estos filtros") se redactan en español de Colombia. |

**Complexity Tracking**: sin violaciones que justificar — la sección no aplica.

**Re-chequeo post Fase 1**: tras generar `research.md`, `data-model.md`, `contracts/orders-list-api.md`, `contracts/tables-list-api.md` y `quickstart.md`, ninguna decisión de diseño introdujo una dependencia nueva, una migración ni una desviación de la tabla anterior. La única fila que sigue abierta es la II (entrada `A-72` como prerrequisito de implementación), que por diseño se resuelve en `/speckit-tasks` + `/speckit-implement`, no en `/speckit-plan`. El diseño elegido (unión `list[T] | Page[T]` en el `response_model`, ya con precedente en `sales/router.py`; carril de estado paginado separado del singleton `TableService`) confirma que no hace falta tocar ningún consumidor no listado en el spec.

## Project Structure

### Documentation (this feature)

```text
specs/079-paginacion-ordenes-mesas/
├── plan.md                    # Este archivo (/speckit-plan)
├── research.md                # Fase 0 (/speckit-plan)
├── data-model.md              # Fase 1 (/speckit-plan)
├── quickstart.md              # Fase 1 (/speckit-plan)
├── contracts/                 # Fase 1 (/speckit-plan)
│   ├── orders-list-api.md     #   GET /orders — page/size/status/order_type, unión list|Page, orden, filtros
│   └── tables-list-api.md     #   GET /orders/tables — page/size, unión list|Page, orden
├── checklists/
│   └── requirements.md        # (ya existente)
└── tasks.md                   # Fase 2 (/speckit-tasks — NO lo crea /speckit-plan)
```

### Source Code (repositorio `pos-backend`, independiente de `pos-specs`)

```text
pos-backend/
├── app/
│   ├── core/
│   │   └── pagination.py                     # SIN CAMBIOS — se reutiliza `Page[T]` y `paginate(db, stmt, page, size)`
│   ├── api/v1/orders/
│   │   ├── router.py                         # `list_orders` (GET /orders): + Query page/size/order_type;
│   │   │                                     #   response_model = list[OrderResponse] | Page[OrderResponse];
│   │   │                                     #   rama sin page/size = idéntica a hoy (json_or_304). Sin tocar
│   │   │                                     #   `active_sessions_only`.
│   │   │                                     # `list_tables` (GET /orders/tables): + Query page/size;
│   │   │                                     #   response_model = list[TableResponse] | Page[TableResponse].
│   │   ├── service.py                        # `list_orders(...)` conserva su firma actual (Terminal/Dashboard);
│   │   │                                     #   se añade `list_orders_query(status_mostrado, order_type) -> Select`
│   │   │                                     #   con orden `created_at DESC, id DESC` y la traducción
│   │   │                                     #   estado-mostrado → predicado (Pagada = EXISTS Sale, etc.).
│   │   └── schemas.py                        # SIN CAMBIOS — `OrderResponse.order_type` y `TableResponse` ya sirven.
│   └── characterization_tests/
│       ├── test_orders_service.py            # SIN MODIFICAR los tests CONGELA existentes
│       ├── test_orders_pagination.py         # NUEVO — contrato sin/con page, unión de forma, clamp (FR-005),
│       │                                     #   orden determinista página a página (SC-008)
│       └── test_orders_status_type_filters.py # NUEVO — semántica de los 6 estados y los 4 tipos, combinación,
│                                             #   "Sin especificar" excluido al filtrar por tipo (FR-016/FR-018)
```

### Source Code (repositorio `pos-heladeria`, independiente de `pos-specs`)

```text
pos-heladeria/src/app/
├── core/
│   ├── interfaces/page.interface.ts          # SIN CAMBIOS — `Page<T>`
│   └── query/paged-query.ts                  # SIN CAMBIOS — `injectPagedQuery`
├── shared/pagination/pagination-bar.component.ts   # SIN CAMBIOS — se reutiliza tal cual
├── modules/orders/
│   ├── order-status.util.ts                  # + mapa de tipo de orden (En mesa / Para llevar / Domicilio /
│   │                                         #   Sin especificar) y sus clases; el resto sin cambios
│   ├── services/orders-list.service.ts       # NUEVO — transporte paginado de "Órdenes" (patrón `SalesService`:
│   │                                         #   signals page/size/status/orderType + `injectPagedQuery`);
│   │                                         #   NO toca `DiningSessionService.listOrders()`
│   └── pages/orders-page.component.ts         # Rediseño: consume el servicio paginado, `app-pagination-bar`,
│   │                                         #   dos <select> (estado / tipo), columna de tipo por fila,
│   │                                         #   se retira el botón "Bloqueadas"
│   └── pages/orders-page.component.spec.ts    # Reescrito para el nuevo flujo (los asserts de `displayOrderStatus`
│                                             #   que sobrevivan se mantienen)
├── modules/tables/
│   ├── services/table.service.ts             # + carril paginado: signals `tablesPage/tablesSize` +
│   │                                         #   `injectPagedQuery` + `loadTablesPage(page,size)`; `loadTables()`,
│   │                                         #   `tables()` y las mutaciones (create/update/toggle/setStatus)
│   │                                         #   siguen intactas para Terminal/Dashboard/order-detail/hoja QR
│   └── pages/tables-page.component.ts         # Rediseño: consume el carril paginado + `app-pagination-bar`;
│   │                                         #   tras crear/editar/activar/cambiar estado, refresca la página
│   │                                         #   actual (o la última válida)
│   └── pages/tables-page.component.spec.ts    # NUEVO
└── modules/dashboard/pages/admin-dashboard.component.ts   # SIN CAMBIOS — sigue con `listOrders()` completo
```

**Structure Decision**: aplicación web de dos repositorios ya existentes. No se crean paquetes de alto nivel. En el backend, el cambio se concentra en `app/api/v1/orders/` (router + service) reutilizando `app/core/pagination.py`. En el frontend, se añade un servicio de transporte dedicado para el carril paginado de "Órdenes" (espejo de `SalesService`) y un carril paginado dentro del singleton `TableService` para "Mesas", de forma que ningún otro consumidor del listado completo se vea afectado (FR-023–FR-025). `app-pagination-bar` y `injectPagedQuery` se reutilizan sin modificarse.

## Complexity Tracking

*Sin violaciones del Constitution Check — sección no aplica.* (La fila II de la tabla es un prerrequisito de proceso —registrar `A-72` antes de implementar—, no una violación de diseño que exija una alternativa más simple.)
