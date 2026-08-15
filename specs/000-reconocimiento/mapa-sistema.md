# Mapa del sistema — POS Heladería

**Fecha de reconocimiento**: 2026-08-15
**Alcance**: `../pos-backend` (API) y `../pos-heladeria` (frontend), según define la
[Constitución](../../.specify/memory/constitution.md) del repositorio `pos-specs`.
**Método**: lectura directa de código y `grep -rn` sobre ambos repositorios. Toda afirmación
cita fichero y línea. Donde el código no basta para confirmar un comportamiento, se marca
explícitamente como `SUPOSICIÓN` con la pregunta que la resolvería (Principio II de la
Constitución). Este documento no propone correcciones — los hallazgos raros se listan en
"Zonas oscuras" para que el negocio decida (Principio III).

---

## 1. Resumen ejecutivo

El sistema son dos repositorios independientes que se comunican por HTTP/SSE:

- **`pos-backend`**: API en FastAPI, PostgreSQL 16 con **un schema por tenant** (multi-tenant
  real a nivel de base de datos), Redis para bus de eventos y bloqueo de tokens, Celery para
  correo asíncrono, APScheduler para tareas periódicas. Cubre catálogo, inventario, caja,
  ventas de mostrador, facturación interna y el flujo completo de pedidos de mesa por QR
  (carrito por comensal → consolidación → cobro).
- **`pos-heladeria`**: SPA en Angular 21 (standalone, sin NgModules), consumidor exclusivo de
  esa API. Sirve dos audiencias distintas bajo el mismo dominio: el personal del negocio
  (`/dashboard/**`, autenticado) y el comensal que escanea el QR de su mesa (`/menu/t/:token`,
  público, sin login).
- El tenant se resuelve por **subdominio** en el frontend
  (`pos-heladeria/src/app/core/tenant/tenant-resolver.ts:22-51`) y se propaga al backend por la
  cabecera `x-tenant-host` (uso confirmado en `pos-backend/app/core/db.py:135-150`) — excepto en
  el flujo QR del comensal, donde el tenant viaja firmado dentro del JWT de la mesa
  (`pos-backend/app/core/qr_context.py:39-50`, `pos-backend/app/core/qr_token.py`).

---

## 2. Backend — `pos-backend`

### 2.1 Arranque y plataforma (`app/main.py`, `app/core/`)

- **`app/main.py:100-122`**: registra 23 routers bajo el prefijo `/api/v1`, uno por cada
  subcarpeta de `app/api/v1/` (admin, auth, categories, unit_measures, products, users,
  super_admin, menu, orders, catalog, inventory, cash, sales, uploads, cart, table_sessions,
  invoices, health, tenant, reports, promotions, audit, realtime).
- **Middleware**: un único `CORSMiddleware` (`app/main.py:86-96`) restringido por
  `allow_origin_regex` a `*.localhost:4200` (dev) y `*.skeilopos.com` (prod), con
  `allow_credentials=True` y `expose_headers=["ETag","Retry-After"]`.
- **Ciclo de vida** (`app/main.py:45-81`, función `lifespan`): al arrancar —
  `initialize_database()` (`:50`), `token_blocklist.ping()` (`:51`), `event_bus.start()`
  (`:55`), `start_scheduler()` (`:59`). Al apagar, en orden inverso: scheduler → bus de eventos
  → Redis del blocklist (`:67-81`).
- **`initialize_database()` se llama dos veces** en el arranque: dentro de `lifespan` (`:50`) y
  otra vez fuera de él en `create_app()` (`:98`). No rompe nada porque la función corta temprano
  si ya hay una revisión de Alembic aplicada (`app/core/db.py:233-235`), pero es una llamada
  redundante sin comentario que la explique — ver Zonas oscuras.
- **`app/core/models.py` vs `app/models/*.py`** no se solapan: `app/core/models.py:19-20,43-127`
  define la base declarativa `Base` y solo las tres entidades del schema `shared`
  (`Tenant`, `Role`, `User`, multi-tenant transversal). `app/models/__init__.py` reexpone las
  ~30 entidades de negocio por-tenant, cada una en su propio fichero, todas heredando de ese
  mismo `Base` (ej. `app/models/product.py` → `from app.core.models import Base`). El import de
  `app.models` en `app/core/db.py:21` existe solo para forzar el registro de esas tablas en
  `Base.metadata` antes de crearlas.
- **Multi-tenant** (`app/core/db.py`): `with_db(tenant_schema)` (`:32-44`) abre una `Session`
  con `schema_translate_map` de SQLAlchemy, de forma que el mismo modelo ORM se traduce en
  tiempo de ejecución al schema Postgres real del tenant. `get_tenant()` (`:135-150`) resuelve
  el tenant por cabecera `x-tenant-host` para el personal autenticado;
  `resolve_tenant_by_id()` (`:152-160`) es el equivalente para el flujo QR, donde el tenant va
  en el JWT y no hay cabecera. La creación de un tenant nuevo (schema Postgres, tablas, usuario
  ADMIN inicial) vive en `app/core/db.py:47-101` (`tenant_create`), invocada únicamente desde
  `app/api/v1/admin/router.py:18-29`.
- **Bus de eventos** (`app/core/event_bus.py`, `app/core/events.py`): pub/sub sobre **Redis
  Streams**, un lector (`XREAD BLOCK`) por proceso y por tenant que reparte a colas
  `asyncio.Queue` en memoria (`app/core/event_bus.py:82-241`). `app/core/events.py:169-312`
  define el catálogo de eventos de negocio (`order_created`, `item_kitchen_changed`,
  `bill_changed`, `payment_completed`, `session_closed`, `table_status_changed`, etc.) con un
  cliente Redis síncrono dedicado (`:58-84`) para publicarse desde código no async. Publican
  eventos: `app/api/v1/table_sessions/service.py:18`, `app/api/v1/cart/service.py:15`,
  `app/api/v1/orders/router.py:8`, `app/core/scheduler.py:36`. El único consumidor es
  `app/api/v1/realtime/router.py:26` (endpoint SSE `GET /realtime/stream`).
- **Tareas periódicas** (`app/core/scheduler.py`, arrancadas por `start_scheduler()` en
  `app/main.py:59`):
  - `sweep_orphan_sessions()` (`:185-207`), cada `SESSION_SWEEP_INTERVAL_MINUTES` (15 min por
    defecto) — cierra mesas abandonadas sin pedir (30 min) o con consumo olvidado (tope duro de
    6 h), publicando `session_closed`/`table_status_changed`.
  - `expire_promotions()` (`:210-244`), a medianoche vía `CronTrigger(hour=0, minute=0)`
    (`:300-306`) — marca `finished` las promociones vencidas (informativo: la evaluación real ya
    filtra por fecha en cada uso).
  - Ambos jobs toman un lock distribuido en Redis (`:43-44`) para no duplicarse con varios
    workers uvicorn.
- **`app/celery_task.py:10-14`**: una única tarea Celery, `send_email_task`, usada solo por
  `app/api/v1/admin/router.py:10,36` para el correo de bienvenida al crear un tenant.
- **`app/scripts/sweep_sessions.py` y `expire_promotions.py`** son wrappers CLI de esas mismas
  funciones, para poder correrlas por cron externo sin depender de APScheduler dentro del
  proceso web.

### 2.2 Módulos de negocio (`app/api/v1/*`)

| Módulo | Criticidad |
|---|---|
| `cart` | Alta |
| `catalog` | Alta |
| `orders` | Alta |
| `table_sessions` | Alta |
| `inventory` | Alta |
| `cash` | Alta |
| `sales` | Alta |
| `invoices` | Alta |
| `promotions` | Alta |
| `menu` | Alta |
| `auth` | Alta |
| `admin` | Media |
| `tenant` | Media |
| `users` | Media |
| `products` | Media |
| `reports` | Media |
| `audit` | Media |
| `realtime` | Media |
| `categories` | Baja |
| `unit_measures` | Baja |
| `uploads` | Baja |
| `super_admin` | Baja |
| `health` | Baja (operacional, no de negocio) |

#### `auth`
- **Ficheros**: `app/api/v1/auth/routes.py`, `schemas.py`.
- **Responsabilidad de negocio**: autentica al personal del local (inicio de sesión, cambio de
  contraseña, refresco y cierre de sesión) para que puedan operar el POS.
- **Depende de**: `app/core/db.py` (`routes.py:10`), `app/core/dependencies.py` (`:13-14`),
  `app/core/redis.py` (`:16`), `app/core/utils.py` (`:12`).
- **Del que dependen**: ningún otro módulo de `app/api/v1` lo importa directamente; se apoya en
  él, indirectamente, todo endpoint protegido vía `app/core/dependencies.py` (mismo JWT).
- **Criticidad**: **alta** — sin login no hay acceso a caja, inventario ni pedidos.

#### `admin`
- **Ficheros**: `app/api/v1/admin/router.py`, `schema.py`.
- **Responsabilidad de negocio**: da de alta un negocio (tenant) nuevo en la plataforma, con su
  usuario administrador, y le envía el correo de bienvenida.
- **Depende de**: `app/core/db.tenant_create` (`router.py:6`), `app/core/mail.welcome_email_body`
  (`:8`), `app/celery_task.send_email_task` (`:10`).
- **Del que dependen**: solo `app/main.py:12`.
- **Criticidad**: **media** — crítico para las altas de clientes del SaaS, pero de uso muy poco
  frecuente y sin efecto en la operación diaria de un tenant ya existente.

#### `super_admin`
- **Ficheros**: `app/api/v1/super_admin/router.py`, `schemas.py`.
- **Responsabilidad de negocio**: da visibilidad global (listar todos los usuarios y todos los
  tenants) al operador de la plataforma, no al dueño de un negocio individual.
- **Depende de**: `app/api/v1/users/schemas.UserResponse` (`router.py:8`).
- **Del que dependen**: solo `app/main.py`.
- **Criticidad**: **baja** — solo lectura administrativa, sin efecto operativo en ningún tenant.

#### `tenant`
- **Ficheros**: `app/api/v1/tenant/router.py`, `schemas.py`.
- **Responsabilidad de negocio**: gestiona los datos propios del negocio (logo, mensaje de
  recibo, prefijo de numeración de factura).
- **Depende de**: `app/core/storage.py` (`router.py:15`).
- **Del que dependen**: solo `app/main.py`.
- **Criticidad**: **media** — afecta la imagen y la facturación, pero se edita rara vez.

#### `users`
- **Ficheros**: `app/api/v1/users/router.py`, `schemas.py`.
- **Responsabilidad de negocio**: administra al personal del negocio (crear cuentas, asignar rol
  ADMIN/CASHIER, activar/desactivar accesos).
- **Depende de**: `app/core/models.User, Role`.
- **Del que dependen**: `app/api/v1/super_admin/router.py:8` (reutiliza `UserResponse`).
- **Criticidad**: **media** — gestión de accesos, no mueve dinero directamente.

#### `categories`
- **Ficheros**: `app/api/v1/categories/router.py`, `schemas.py`.
- **Responsabilidad de negocio**: organiza el menú en categorías (ej. helados, bebidas,
  toppings).
- **Depende de**: `app/models/category.py`, `app/core/crud.py`.
- **Del que dependen**: nadie importa el módulo directamente; `products`, `menu` y `reports`
  consumen el **modelo** `Category`, no este router.
- **Criticidad**: **baja** — catálogo de apoyo, cambia poco.

#### `unit_measures`
- **Ficheros**: `app/api/v1/unit_measures/router.py`, `schemas.py`.
- **Responsabilidad de negocio**: define las unidades de medida del inventario (kg, g, l, ml…)
  para poder convertir cantidades entre presentaciones.
- **Depende de**: `app/models/unit_measure.py`.
- **Del que dependen**: nadie a nivel de módulo; `app/core/units.py` usa el modelo directamente.
- **Criticidad**: **baja** — configuración base, cambia casi nunca tras el alta inicial.

#### `products`
- **Ficheros**: `app/api/v1/products/router.py`, `service.py`, `schemas.py`.
- **Responsabilidad de negocio**: da de alta y mantiene los productos del menú (nombre,
  categoría, imagen), creándoles automáticamente una presentación vendible por defecto.
- **Depende de**: `app/api/v1/catalog/service.ensure_default_variant` (`service.py:19`),
  `app/core/storage.py` (`:16`).
- **Del que dependen**: solo `app/main.py`.
- **Criticidad**: **media** — sin productos no hay menú, pero el catálogo cambia con frecuencia
  moderada, no en cada venta.

#### `menu`
- **Ficheros**: `app/api/v1/menu/router.py`, `schemas.py`.
- **Responsabilidad de negocio**: sirve el menú público que ve el comensal al escanear el QR de
  su mesa, con disponibilidad real y precios con descuento ya aplicados.
- **Depende de**: `app/core/qr_context.open_qr_context` (`router.py:16`),
  `app/core/rate_limit` (`:17`), `app/api/v1/promotions/service.active_discount_promotions,
  best_line_discount` (`:26`).
- **Del que dependen**: solo `app/main.py`. Un comentario propio (`router.py:26,187-190`) deja
  constancia de que existió un endpoint anterior (`GET /menu/qr/{qr_token}`) eliminado por
  inseguro (tenant por cabecera falsificable, id de mesa plano), reemplazado por
  `/menu/qr-token/{token}`.
- **Criticidad**: **alta** — es la puerta de entrada del comensal; si falla, nadie puede pedir
  por QR.

#### `catalog`
- **Ficheros**: `app/api/v1/catalog/router.py`, `service.py`, `schemas.py`, `line_pricing.py`,
  `consumption_plan.py`.
- **Responsabilidad de negocio**: define qué presentaciones (tamaños) tiene cada producto, su
  precio, sus grupos de sabores/opciones, y qué insumos consume cada combinación — es el motor
  de precios y de "qué se descuenta de inventario" para toda venta.
- **Depende de**: `app/models/product.py, product_variant.py, recipe_item.py, option.py,
  option_group.py, variant_option_group.py, inventory_item.py`.
- **Del que dependen** (módulo muy central): `orders/service.py`, `orders/consumption.py`,
  `orders/consolidation.py`, `orders/kitchen.py`, `sales/service.py`, `sales/consumption.py`,
  `cart/service.py:31-36`, `products/service.py:19`.
- **Criticidad**: **alta** — determina el precio y el consumo de inventario de cada venta; un
  error aquí descuadra dinero e inventario a la vez.

#### `inventory`
- **Ficheros**: `app/api/v1/inventory/router.py`, `service.py`, `stock.py`, `schemas.py`.
- **Responsabilidad de negocio**: lleva el stock único de insumos (kardex), registra compras a
  proveedores y aplica ajustes manuales de existencias.
- **Depende de**: `app/core/exceptions.InsufficientStockError`,
  `app/core/inventory_reasons.py`.
- **Del que dependen**: `orders/consumption.py:18` (`lock_items, record_movement`),
  `sales/consumption.py:16` — todo consumo de inventario del sistema (mostrador y mesa) pasa por
  `inventory/stock.py`.
- **Criticidad**: **alta** — `record_movement` (`stock.py:39-89`) es el único punto de mutación
  de stock, con `SELECT...FOR UPDATE` para evitar condiciones de carrera; un fallo aquí es
  pérdida de mercancía real.

#### `cart`
- **Ficheros**: `app/api/v1/cart/router.py`, `service.py`, `schemas.py`.
- **Responsabilidad de negocio**: gestiona el carrito de cada comensal en la mesa (unirse vía
  QR, añadir/editar/quitar líneas, enviar el pedido a cocina), sin tocar inventario todavía.
- **Depende de**: `app/core/qr_context.py:18-19`, `app/api/v1/catalog/line_pricing.py:31-36`,
  `app/api/v1/table_sessions/service.try_release_if_empty` (`service.py:20`),
  `app/api/v1/promotions/service.py:38`.
- **Del que dependen**: `app/api/v1/table_sessions/service.py:317` importa
  `unique_display_label` de `cart.service`.
- **Criticidad**: **alta** — es la vía de entrada de los pedidos QR, el grueso del volumen de
  ventas del negocio.

#### `table_sessions`
- **Ficheros**: `app/api/v1/table_sessions/router.py`, `service.py`, `schemas.py`.
- **Responsabilidad de negocio**: gestiona la sesión completa de una mesa — sus comensales, la
  cuenta consolidada, el reparto entre comensales y el cobro/cierre final (unificado o
  dividido).
- **Depende de**: `app/api/v1/orders/checkout.py:27`, `app/api/v1/sales/builder.build_sale,
  ensure_open_shift` (`:28`), `app/api/v1/promotions/service.py:29`.
- **Del que dependen**: `app/core/qr_context.py:97` (`try_release_if_empty`),
  `app/core/scheduler.py:62,96` (barrido de sesiones huérfanas), `app/api/v1/cart/service.py:20`.
- **Criticidad**: **alta** — aquí se cobra efectivamente la mesa; implementa un lock optimista
  (`_load(..., lock=True)`, `service.py:38-55`) para no cobrar la misma mesa dos veces.

#### `orders`
- **Ficheros**: `router.py`, `service.py`, `checkout.py`, `consolidation.py`, `consumption.py`,
  `kitchen.py`, `tables_advanced.py`, `schemas.py`.
- **Responsabilidad de negocio**: es el corazón operativo de la mesa — confirma pedidos (momento
  en que se descuenta inventario), gestiona el flujo de cocina, cobra y cancela pedidos, y
  administra las mesas físicas (crear, mover, fusionar).
- **Depende de**: `app/api/v1/catalog/consumption_plan.py` (vía `consumption.py:19-23`),
  `app/api/v1/sales/builder.py` (vía `checkout.py:32`), `app/api/v1/promotions/service.py:38`.
- **Del que dependen**: `app/core/scheduler.py:93,96` (`checkout.TERMINAL,
  close_participants, close_table_sessions`), `app/api/v1/cart/service.py:37,447`
  (`checkout.cancel_order`), `app/api/v1/table_sessions/service.py:27` (módulo `checkout`
  completo).
- **Criticidad**: **alta** — `confirm_order` (`checkout.py:307-352`) es, según comentario propio
  del código (`:309`), "el único punto de descuento de los pedidos por QR"; `cancel_order`
  implementa la reversa asimétrica de inventario según el estado de cocina (`:357-461`).

#### `cash`
- **Ficheros**: `app/api/v1/cash/router.py`, `service.py`, `schemas.py`.
- **Responsabilidad de negocio**: administra los turnos de caja (apertura, cierre, arqueo) y los
  movimientos manuales de efectivo (ingresos/egresos/retiros).
- **Depende de**: `app/models/sale.py, payment.py` — el arqueo deriva lo vendido de las ventas y
  pagos reales, no de los movimientos manuales de caja (`service.py:4-7`).
- **Del que dependen**: `sales/builder.ensure_open_shift`, `orders/checkout.py` y
  `table_sessions/service.py` reutilizan `CashShift`/`ensure_open_shift` (definido en
  `sales/builder.py`, no en `cash`); el módulo `cash` en sí solo lo importa `app/main.py`.
- **Criticidad**: **alta** — el arqueo (`reconcile`, `service.py:76-183`) concilia dinero real
  contra lo vendido; contiene un campo marcado `DEPRECADO` (`:180-181`, alias `cash_sales`)
  mantenido solo por compatibilidad con el frontend.

#### `sales`
- **Ficheros**: `router.py`, `service.py`, `builder.py`, `consumption.py`, `schemas.py`.
- **Responsabilidad de negocio**: registra la venta de mostrador (sin pasar por mesa/QR) y
  provee `build_sale`, el constructor de venta compartido por los cuatro caminos de cobro del
  sistema.
- **Depende de**: `app/api/v1/catalog/line_pricing.py` (`service.py:23`),
  `app/api/v1/promotions/service.py:27`.
- **Del que dependen**: `app/api/v1/orders/checkout.py:32` (`build_sale, ensure_open_shift`),
  `app/api/v1/table_sessions/service.py:28`, `app/api/v1/invoices/service.py` (import local
  dentro de `builder.py:176`, para evitar ciclo).
- **Criticidad**: **alta** — `build_sale` (`builder.py:67-179`) es el único lugar donde nace una
  `Sale` pagada y su factura; un bug aquí afecta a la vez mostrador, cierre unificado, cierre
  dividido y `pay_order` legado de mesa.

#### `invoices`
- **Ficheros**: `app/api/v1/invoices/router.py` (solo lectura), `service.py`, `schemas.py`.
- **Responsabilidad de negocio**: emite y consulta las facturas internas, una por cada venta,
  con numeración consecutiva por prefijo.
- **Depende de**: `app/models/invoice.py, sale.py`.
- **Del que dependen**: `app/api/v1/sales/builder.py:176-178` (`issue_for_sale`, import local
  para evitar ciclo, explicado en el propio comentario `builder.py:174-176`).
- **Criticidad**: **alta** — un comentario propio (`service.py:1-9`) documenta que antes del
  diseño actual hubo "20 ventas reales, cero facturas" por depender de un botón manual; ahora se
  emite dentro de la misma transacción del cobro.

#### `promotions`
- **Ficheros**: `app/api/v1/promotions/router.py`, `service.py`, `schemas.py`.
- **Responsabilidad de negocio**: evalúa y administra descuentos (porcentaje, fijo, por
  cantidad) y combos, aplicándolos automáticamente al carrito, al menú público y en cada cobro.
- **Depende de**: `app/models/promotion.py`.
- **Del que dependen** (módulo muy extendido): `menu/router.py:26`, `cart/service.py:38`,
  `sales/service.py:27`, `orders/service.py`, `orders/checkout.py:38`,
  `orders/consolidation.py:29`, `table_sessions/service.py:29`.
- **Criticidad**: **alta** — toca el precio final de cada venta; su script de test
  (`app/scripts/test_promotions_rules.py`) es, según se confirma en 2.4, el **único** que corre
  en CI.

#### `uploads`
- **Ficheros**: `app/api/v1/uploads/router.py`, `schemas.py`.
- **Responsabilidad de negocio**: entrega URLs firmadas para que el panel de administración suba
  imágenes de producto directamente a Cloudflare R2.
- **Depende de**: `app/core/storage.py`.
- **Del que dependen**: solo `app/main.py`.
- **Criticidad**: **baja** — soporte visual, no afecta ventas ni inventario.

#### `reports`
- **Ficheros**: `app/api/v1/reports/router.py`, `service.py`, `schemas.py`.
- **Responsabilidad de negocio**: agrega ventas, inventario y rentabilidad ya cerrados para dar
  visibilidad al dueño del negocio (sin tablas propias, todo derivado en consulta).
- **Depende de**: `app/models/sale.py, recipe_item.py, variant_option_group.py,
  inventory_item.py`.
- **Del que dependen**: nadie (solo `app/main.py`).
- **Criticidad**: **media** — informativo, no transaccional; útil pero no bloquea la operación
  diaria.

#### `audit`
- **Ficheros**: `app/api/v1/audit/router.py` (solo lectura).
- **Responsabilidad de negocio**: expone la bitácora de auditoría (quién hizo qué) para revisión
  administrativa.
- **Depende de**: `app/models/audit_log.py`.
- **Del que dependen**: nadie importa el router; la **escritura** de auditoría vive aparte, en
  `app/core/audit.py:10-26` (`record_audit`), usada por `app/api/v1/promotions/router.py:12`
  (6 llamadas, líneas 57-135) y `app/api/v1/orders/checkout.py:437-450` (pérdidas de inventario
  al cancelar un pedido ya en cocina).
- **Criticidad**: **media** — trazabilidad legal/operativa; no bloquea ninguna operación de
  negocio si falla.

#### `realtime`
- **Ficheros**: `app/api/v1/realtime/router.py`, `auth.py`, `schemas.py`.
- **Responsabilidad de negocio**: mantiene la pantalla de cocina y el panel del cajero
  actualizados en vivo (SSE) sin depender de que el navegador refresque.
- **Depende de**: `app/core/event_bus.py`, `app/core/events.py:30`
  (`CH_STAFF, session_channel`), `app/core/rate_limit.py`.
- **Del que dependen**: nadie a nivel de módulo (solo `app/main.py`); es puramente consumidor.
- **Criticidad**: **media** — degrada de forma controlada: si el stream falla, el frontend cae a
  sondeo REST (mencionado explícitamente en `app/core/events.py:115` y en `rate_limit.py`), así
  que no llega a ser alta pese a tocar la experiencia de cocina.

#### `health`
- **Ficheros**: `app/api/v1/health/router.py` (7 líneas).
- **Responsabilidad de negocio**: informa si el servicio está vivo, para el healthcheck de
  Docker/orquestación.
- **Depende de**: nada.
- **Del que dependen**: `docker-compose.prod.yml:94` (`curl -fsS
  http://localhost:8000/api/v1/health`).
- **Criticidad**: **baja** para el negocio en sí, pero operacionalmente necesaria para el
  despliegue.

### 2.3 Modelos (`app/models/*.py`)

~30 entidades de negocio, una por fichero, todas registradas vía `app/models/__init__.py`.
Salvo el caso señalado en Zonas oscuras (`BusinessHours`), cada modelo tiene al menos un router
o service que lo usa; el detalle de "quién usa qué modelo" queda cubierto dentro de la
descripción de cada módulo de negocio en 2.2 (los modelos no forman una capa aparte con
consumidores propios distintos a los ya listados).

### 2.4 Alembic, Docker y CI/CD

- **`alembic/`**: 21 migraciones versionadas más una de fusión
  (`01a0d2359c2f_merge_promotions_and_variant_option_.py`) — evidencia de al menos dos ramas de
  migraciones reconciliadas. `app/scripts/tenant.py:9-20` (`for_each_tenant_schema`) es un
  decorador usado por 19 de las 21 migraciones para aplicar el cambio de schema a todos los
  tenants existentes.
- **`docker-compose.yml`** (dev): `postgres`, `redis`, `api`, `worker` (Celery).
  **`docker-compose.prod.yml`**: añade healthchecks, límites de log, red dedicada
  `skeilopos-network`, y monta `docker/api-entrypoint.sh` (corre `initialize_database()` +
  `alembic upgrade head` antes de levantar uvicorn) y `worker-entrypoint.sh` (trivial).
- **`.github/workflows/deploy.yml`**: pipeline de 3 jobs (`test` → `build` → `deploy`). El job
  `test` (`:14-22`) instala solo `sqlalchemy pydantic pydantic-settings fastapi` (no todo
  `requirements.txt`) y ejecuta una única cosa: `python -m app.scripts.test_promotions_rules`.
  Es el único de los 12 scripts `test_*.py` que corre en CI (ver Zonas oscuras, punto 2).

---

## 3. Frontend — `pos-heladeria`

Angular 21 standalone (sin NgModules), TypeScript estricto (`strict: true`, `strictTemplates`).
Bootstrap en `src/main.ts:5` (`bootstrapApplication(App, appConfig)`); `App`
(`src/app/app.ts`) es un shell mínimo con solo `<router-outlet>` que inyecta `AuthService` para
forzar su construcción temprana.

### 3.1 Rutas (`src/app/app.routes.ts`)

| Ruta | Guards | Carga |
|---|---|---|
| `/` | — | redirect a `login` |
| `/login` | `redirectIfAuthGuard` | `modules/auth/pages/login.component.ts` |
| `/change-password` | `authGuard`, `changePasswordPageGuard` | `modules/auth/pages/change-password.component.ts` |
| `/super-admin/**` | `authGuard`, `superAdminDomainGuard`, `passwordChangeGuard` | `modules/super-admin/routes.ts` (lazy) |
| `/dashboard/**` | `authGuard`, `tenantDomainGuard`, `passwordChangeGuard` | `modules/dashboard/routes.ts` (lazy) |
| `/menu/t/:token` | — (pública) | `modules/tables/pages/public-menu.component.ts` — menú del comensal vía QR |
| `/menu/qr/:token` | — (pública) | `modules/tables/pages/expired-qr.component.ts` — QR antiguos (UUID plano) que el backend ya no acepta |
| `**` | — | redirect a `login` |

Dentro de `dashboard/routes.ts:6-183`, todo cuelga de `DashboardLayoutComponent` y cada hija
lleva `roleGuard([...])`: `admin`, `caja`, `ajustes` (con hijas `informacion`, `metodos-pago`,
`unidades`, `grupos-opciones`), `ventas`, `categories`, `products`, `products/new`,
`products/:id`, `promotions`, `mesas/qr`, `mesas`, `mesas-sesiones`, `orders`, `orders/:id`,
`inventario`, `proveedores`, `users`, `reports`. Hay 7 redirects de compatibilidad por
reubicaciones históricas de navegación (`ajustes/mesas→mesas`, `ajustes/promociones→promotions`,
`metodos-pago→ajustes/metodos-pago`, `unit-measures→ajustes/unidades`,
`option-groups→ajustes/grupos-opciones`, `tables→mesas`, `cocina→mesas-sesiones`), más
`insumos→inventario`; todas verificadas contra un destino real, ninguna rota. La ruta `cocina`
lleva un comentario explícito (`routes.ts:73-75`) de que el tablero de cocina se deprecó: la
preparación se marca desde la misma terminal de mesas.

`super-admin/routes.ts:4-28`: `tenants`, `users`, sin guards propios de rol (el filtrado de rol
ya ocurre en `app.routes.ts` vía `superAdminDomainGuard`).

No se hallaron rutas huérfanas ni páginas sin ruta: cada `*.component.ts` de página aparece
referenciado en algún `routes.ts`.

### 3.2 Core (infraestructura transversal)

#### `core/auth/`
- **Ficheros**: `auth-api.service.ts`, `auth.models.ts`, `auth-token.interceptor.ts`,
  `token-storage.service.ts`, `jwt.util.ts`.
- **Responsabilidad**: transporta y mantiene el ciclo de vida de la sesión del personal (login,
  refresh, logout, cambio de contraseña) contra el backend.
- **Depende de**: `environments/environment.ts` (`auth-api.service.ts:4`);
  `core/tenant/tenant-context.service.ts` y `modules/tables/services/diner-token.store.ts`
  (`auth-token.interceptor.ts:5,8` — acoplamiento inverso: un fichero de `core` importa de un
  módulo de negocio para distinguir las rutas del comensal del resto; ver Zonas oscuras).
- **Endpoints consumidos**: `POST /auth/login`, `GET /auth/refresh-token`, `GET /auth/logout`,
  `POST /auth/change-password`.
- **Del que dependen**: `core/services/auth.service.ts`, `app.config.ts:9` (interceptor
  registrado global), todos los guards.
- **Criticidad**: **alta** — sin esto no hay sesión ni tenant en ninguna petición.

#### `core/guards/` y `core/tenant/guards/`
- **Responsabilidad**: es la barrera de autorización del lado cliente — exige sesión (`auth.guard.ts`),
  filtra por rol admin/cajero (`role.guard.ts`), fuerza el cambio de contraseña temporal
  (`password-change.guard.ts`) y exige coherencia entre el dominio visitado y el área
  (tenant vs super-admin, `tenant/guards/`).
- **Depende de**: `AuthService` y/o `TenantContextService`.
- **Criticidad**: **alta** — única barrera de autorización en el cliente (el backend es la
  barrera real, pero sin esto la UX queda inconsistente).

#### `core/tenant/`
- **Ficheros**: `tenant-resolver.ts`, `tenant-context.service.ts`, `tenant-info.service.ts`,
  `tenant.initializer.ts`, `tenant-context.model.ts`, `app-environment.interface.ts`.
- **Responsabilidad**: resuelve a qué negocio (tenant) pertenece la sesión actual, a partir del
  subdominio, antes de que la app renderice nada.
- **Cómo lo hace**: `tenant-resolver.ts:22-51` mapea `window.location.hostname` a un
  `TenantContext` (`SUPER_ADMIN` o `TENANT(slug)`) por reglas de subdominio, ejecutándose una
  sola vez en `provideAppInitializer` (`tenant.initializer.ts:12-20`, registrado en
  `app.config.ts:14`) antes del primer render — de ahí que `TenantContextService.context` lance
  si se lee antes de tiempo (`tenant-context.service.ts:16-19`). `TenantInfoService` es un
  servicio aparte que sí habla con el backend (`GET/PATCH /tenant`, `POST /uploads/presign`).
- **Depende de**: `environments/environment.ts` (`rootDomain`, `devRootHosts`,
  `reservedSlugs`).
- **Del que dependen**: `authTokenInterceptor` (cabecera `X-Tenant-Host`), todos los guards de
  dominio, `sidebar.component.ts`, `dashboard-layout.component.ts`, `login.component.ts`,
  `pos-terminal.store.ts` (mensaje de factura).
- **Criticidad**: **alta** — es la base del multi-tenant en el cliente.

#### `core/realtime/`
- **Responsabilidad**: mantiene al personal (y al comensal en el menú público) informados en
  vivo de pedidos y cambios de mesa sin recargar la página.
- **Mecanismo**: Server-Sent Events propio (`EventSource`), con reconexión reimplementada con
  backoff exponencial + jitter (`sse-client.ts:131-141`), porque `EventSource` no reintenta tras
  un error no-2xx y el ticket de staff es de un solo uso.
- **Endpoint**: `GET {apiBaseUrl}/realtime/stream` (`realtime.service.ts:54,95`), con `ticket`
  para el personal (obtenido de `POST /realtime/ticket`) o `token` de sesión para el comensal.
- **Del que dependen**: `PosTerminalStore` (`pos-terminal.store.ts:611`, `connectStaff()`) y
  `PublicMenuComponent` (`public-menu.component.ts:911`, `connectDiner(token)`).
- **Criticidad**: **alta** — sin esto el personal solo se entera de pedidos por sondeo lento.

#### `core/query/`
- **Responsabilidad**: capa de caché/reactividad para las listas paginadas del panel admin.
- `paged-query.ts:12-23` envuelve `injectQuery` de TanStack; `app.config.ts:20-36` configura
  `staleTime: 30s`, `gcTime: 5min`, `refetchOnWindowFocus: false`, `retry: 1`.
- **Del que dependen**: `CategoryService`, `ProductService`, `InventoryService`,
  `PromotionService`, `SalesService` (`ReportsService` usa `injectQuery` directo).
- **Criticidad**: **media** — mejora de arquitectura, no requisito funcional.

#### `core/printing/`
- **Responsabilidad**: guarda la preferencia de ancho de papel (48/58/80mm) por dispositivo. La
  lógica real de impresión (generar el HTML del ticket e imprimirlo por iframe oculto) vive en
  `modules/tables/services/receipt.util.ts:1-9`, no aquí — "printing" en `core/` es solo
  configuración.
- **Del que dependen**: `PosTerminalStore` (`pos-terminal.store.ts:185,1175`).
- **Criticidad**: **media** — degradación aceptable (valor por defecto 48mm) si falla.

#### `core/services/` (nota de arquitectura)
`menu.service.ts` y `unit-measure.service.ts` viven en `core/services/` en vez de en sus módulos
naturales — inconsistencia frente al resto de módulos, que siempre tienen su servicio HTTP en
`modules/x/services/`. Ver Zonas oscuras.

### 3.3 Módulos de negocio

#### `auth`
- **Responsabilidad de negocio**: permite a meseros/administradores iniciar sesión y, si tienen
  contraseña temporal, cambiarla antes de usar el sistema.
- **Depende de**: `core/services/auth.service.ts`, `core/tenant/tenant-context.service.ts`
  (`login.component.ts:6`), `shared/password-input/`.
- **Criticidad**: **alta** — puerta de entrada obligatoria a todo el sistema.

#### `dashboard`
- **Responsabilidad de negocio**: es el armazón visual (menú lateral, cabecera, cierre de
  sesión) del área operativa, y muestra al administrador un resumen del día (usuarios, productos
  activos, órdenes activas).
- **Depende de**: `core/services/auth.service.ts`, `core/config/navigation.config.ts`,
  `core/tenant/tenant-info.service.ts`, `products/services/product.service.ts`,
  `users/services/users.service.ts`, `sales/services/sales.service.ts`,
  `tables/services/{dining-session,table}.service.ts` (`admin-dashboard.component.ts:4-9`).
- **Del que dependen**: prácticamente todos los módulos operativos cuelgan de
  `DashboardLayoutComponent`, incluido `super-admin`, que reutiliza el mismo layout
  (`super-admin/routes.ts:2,7`).
- **Criticidad**: **alta** — contenedor de toda la operación diaria.

#### `cash-register`
- **Responsabilidad de negocio**: permite abrir/cerrar cajas físicas, registrar ingresos/egresos
  de efectivo, hacer arqueo de cierre de turno y ver el histórico.
- **Endpoints**: `GET/POST /cash/registers`, `POST /cash/shifts/open`, `GET
  /cash/shifts/current`, `POST /cash/shifts/{id}/movements`, `GET
  /cash/shifts/{id}/reconciliation`, `POST /cash/shifts/{id}/close`, `GET
  /cash/shifts/{id}/report`, `GET /cash/shifts`, `POST /cash/shifts/{id}/partial-count`.
- **Del que dependen**: `tables/services/pos-terminal.store.ts:180` (necesita `cash_shift_id`
  para poder cobrar).
- **Criticidad**: **alta** — sin turno de caja abierto no se puede cobrar ninguna mesa.

#### `categories`
- **Responsabilidad de negocio**: gestiona las categorías del menú (crear, editar,
  activar/desactivar).
- **Depende de**: `core/query/paged-query.ts`.
- **Endpoints**: `GET/POST/PATCH /categories`.
- **Del que dependen**: `products` (selector de categoría), `promotions` (alcance por
  categoría).
- **Criticidad**: **media** — catálogo base, cambia poco en operación diaria.

#### `products`
- **Responsabilidad de negocio**: administra el catálogo de productos y sus presentaciones
  (variantes), con precio, receta de insumos por presentación y grupos de opciones que ofrece
  cada una.
- **Depende de**: `core/services/menu.service.ts`, `option-groups/services/option-group.service.ts`
  (`product.service.ts:8,10`), Cloudflare R2 vía `POST /uploads/presign`.
- **Endpoints**: `/products`, `/products/{id}/variants`, `/variants/{id}`,
  `/variants/{id}/recipe`, `/variants/{id}/option-groups`, `/uploads/presign`.
- **Del que dependen**: `dashboard` (contador de productos activos), `promotions` (alcance por
  producto), `tables` vía `MenuService` (catálogo de venta).
- **Criticidad**: **alta** — sin catálogo no hay venta.

#### `option-groups`
- **Responsabilidad de negocio**: define los grupos de sabores/toppings/extras (con
  mínimos/máximos de selección) asignables a las presentaciones de un producto.
- **Depende de**: `inventory/services/inventory.service.ts` (una opción puede descontar un
  insumo), `core/services/unit-measure.service.ts`.
- **Endpoints**: `GET /option-groups`, `POST/PATCH /option-groups/{id}`, `POST
  /option-groups/{id}/options`, `PATCH /options/{id}`.
- **Del que dependen**: `products/services/product.service.ts:10` (variantes referencian
  grupos).
- **Criticidad**: **media-alta** — impacta directamente el ticket de venta (extras cobrables).

#### `inventory`
- **Responsabilidad de negocio**: controla el stock de insumos (kardex de movimientos), registra
  compras a proveedores (recepción parcial/total) y permite ajustes manuales, avisando de
  insumos bajo mínimo.
- **Depende de**: `suppliers/services/suppliers.service.ts`,
  `core/services/unit-measure.service.ts`.
- **Endpoints**: `/inventory/items`, `/inventory/items/{id}/adjust`,
  `/inventory/items/{id}/movements`, `/inventory/items/low-stock`, `/inventory/purchases`,
  `/inventory/purchases/order`, `/inventory/purchases/{id}/receive`.
- **Del que dependen**: `products` (receta consume insumos), `option-groups` (opciones pueden
  consumir insumo), `reports` (inventario/rentabilidad).
- **Criticidad**: **alta** — el backend descuenta stock automáticamente al confirmar pedidos; sin
  visibilidad de esto el negocio pierde control de costos.

#### `suppliers`
- **Responsabilidad de negocio**: mantiene el directorio de proveedores usado al registrar
  compras de insumos.
- **Endpoints**: `/inventory/suppliers`.
- **Del que dependen**: `inventory` (formulario de compras).
- **Criticidad**: **baja-media** — catálogo de apoyo.

#### `promotions`
- **Responsabilidad de negocio**: crea y administra descuentos (porcentaje, monto fijo, precio
  por cantidad) y combos, con vigencia por fecha/día/hora, y calcula el precio final que verá el
  cajero y el comensal.
- **Depende de**: `products/services/product.service.ts`, `categories/services/category.service.ts`
  (alcance), `core/query/paged-query.ts`.
- **Endpoints**: `GET/POST /promotions`, `PATCH /promotions/{id}`, `PATCH
  /promotions/{id}/shape`, `PATCH /promotions/{id}/status`, `POST /promotions/{id}/duplicate`,
  `DELETE /promotions/{id}`.
- **Del que dependen**: `tables/services/pos-terminal.store.ts` (combos vendibles, insignias de
  descuento), `tables/components/pending-orders-panel.component.ts`.
- **Criticidad**: **alta** — afecta directamente el monto cobrado; un bug aquí es un error de
  caja.

#### `sales`
- **Responsabilidad de negocio**: muestra el historial de ventas emitidas (con factura) y
  administra qué métodos de pago acepta el negocio, clasificados para el arqueo.
- **Endpoints**: `GET/POST /sales`, `GET /sales/{id}`, `/sales/payment-methods`.
- **Del que dependen**: `tables/services/pos-terminal.store.ts` (checkout final y reimpresión de
  factura), `dashboard` (métricas), `cash-register` indirectamente (el arqueo desglosa por
  método).
- **Criticidad**: **alta** — es el registro fiscal/contable de cada cobro.

#### `reports`
- **Responsabilidad de negocio**: presenta al administrador ventas por periodo,
  productos/categorías más vendidos, desempeño por cajero, valor de inventario y rentabilidad
  (margen).
- **Endpoints**: `/reports/sales`, `/reports/top-products`, `/reports/categories`,
  `/reports/cashiers`, `/reports/inventory`, `/reports/profitability`.
- **Del que dependen**: nadie más lo importa.
- **Criticidad**: **media** — informativo, no bloquea la operación si falla.

#### `tables` (el módulo más grande — terminal POS + flujo QR del comensal)
- **Ficheros clave**: `services/pos-terminal.store.ts` (~1225 líneas, store central de la
  terminal), `services/{table,dining-session,table-session,diner,diner-token.store,dining-cart,
  menu-lookup,payment-draft.util,receipt.util,table-qr.util}.ts`,
  `pages/{tables-page,table-sessions,table-qr-sheet,public-menu,expired-qr}.component.ts`.
- **Responsabilidad de negocio**: administra las mesas físicas y sus QR; es la terminal donde el
  personal toma pedidos, marca preparación, consolida la cuenta y cobra (con soporte de cuenta
  dividida entre comensales); y es también el backend del **menú público** que el cliente ve al
  escanear el QR, con su propio carrito, envío de pedido y seguimiento en vivo.
- **Depende de**: `core/services/menu.service.ts`, `core/realtime/*`,
  `core/printing/printer-settings.store.ts`, `core/tenant/tenant-info.service.ts`,
  `promotions`, `sales`, `cash-register`, `shared/feedback/*` (toast, confirm, sonido).
- **Endpoints**: `/orders`, `/orders/tables`, `/orders/tables/{id}/items`,
  `/orders/{id}/confirm|cancel|ready|move`, `/orders/items/{id}/kitchen|void`, `/orders/merge`,
  `/orders/group/{id}/bill`, `/table-sessions`,
  `/table-sessions/{id}/{bill,close,participants,assignments}`, `/menu/qr-token/{token}`,
  `/cart`, `/cart/items`, `/cart/submit`, `/cart/orders`, `/cart/orders/{id}/cancel`,
  `/cart/leave`, `/cart/sessions`, `/realtime/*`.
- **Del que dependen**: `dashboard`, `orders` (reutiliza `DiningSessionService`/`TableService`,
  no tiene servicio propio), `core/auth/auth-token.interceptor.ts` (importa `DinerTokenStore`
  desde aquí — dependencia inversa notable de `core` hacia un módulo).
- **Criticidad**: **alta** — es literalmente el negocio: toma de pedidos, cocina y cobro.

#### `orders`
- **Responsabilidad de negocio**: da al personal una vista de todas las comandas (con filtro por
  estado) y su detalle, con las reglas de qué acciones son válidas según el estado del pedido y
  de cada ítem en cocina.
- **Depende de**: `tables/services/{dining-session,table}.service.ts` — no tiene servicio HTTP
  propio, es una vista sobre los datos del módulo `tables`.
- **Del que dependen**: `dashboard` (`orderStatusClass/Label`),
  `tables/pages/public-menu.component.ts` (reglas `esperaConfirmacion`,
  `puedeCancelarComensal`).
- **Criticidad**: **media** — vista de consulta/gestión, redundante en parte con la terminal de
  mesas.

#### `users`
- **Responsabilidad de negocio**: permite al administrador dar de alta, cambiar de rol y
  activar/desactivar al personal (cajeros/administradores).
- **Endpoints**: `GET/POST /users`, `PATCH /users/{id}/role`, `PATCH /users/{id}/status`, `GET
  /users/{id}` (sin uso confirmado desde la UI, ver Zonas oscuras).
- **Del que dependen**: `dashboard` (contador total de usuarios).
- **Criticidad**: **media-alta** — control de acceso al sistema.

#### `settings`
- **Responsabilidad de negocio**: es el contenedor de pestañas de configuración del negocio
  (información básica, métodos de pago, unidades de medida, grupos de opciones); su pantalla
  propia "Información básica" gestiona nombre, logo, mensaje de factura, ancho de papel de
  impresión y cambio de la propia contraseña.
- **Depende de**: `core/tenant/tenant-info.service.ts`,
  `core/printing/printer-settings.store.ts`, `core/services/auth.service.ts`, y reutiliza
  páginas de `sales`, `unit-measures`, `option-groups` como pestañas hijas.
- **Criticidad**: **media**.

#### `unit-measures`
- **Ficheros**: `pages/unit-measures-page.component.ts`,
  `components/unit-measure-form.component.ts` — **sin carpeta `services/` propia**.
- **Responsabilidad de negocio**: administra el catálogo de unidades de medida (kg, litro,
  unidad, etc.) usado por insumos y compras.
- **Depende de**: `core/services/unit-measure.service.ts`.
- **Del que dependen**: `inventory`, `option-groups` (selector de insumo con su unidad).
- **Criticidad**: **baja** — catálogo auxiliar, cambia raramente.

#### `super-admin`
- **Responsabilidad de negocio**: es el panel de la plataforma (fuera de cualquier tenant) donde
  se crean negocios nuevos con su primer administrador, y se gestionan usuarios de plataforma.
- **Depende de**: reutiliza `dashboard/layout/dashboard-layout.component.ts`
  (`super-admin/routes.ts:2`).
- **Endpoints**: `GET /super-admin/tenants`, `POST /admin/tenants` (nota: base URL distinta a la
  del listado — `tenant.service.ts:11-13`), `/super-admin/users`.
- **Criticidad**: **alta** para el negocio SaaS (alta de clientes nuevos), pero fuera del día a
  día de cada heladería individual.

### 3.4 Shared

- **`shared/feedback/`**: `ToastService` (notificaciones efímeras), `ConfirmService`
  (confirmación modal basada en Promise), `SoundService` (campana WebAudio para avisar pedidos
  nuevos, silenciable, persistida en `localStorage`) — usados intensivamente por
  `PosTerminalStore` y páginas admin.
- **`shared/charts/`**: `bars-chart`, `ranked-bars-chart`, `share-bar-chart`, `stat-tile`,
  `chart-card`, `chart-theme` — usados por `reports-page.component.ts` y
  `admin-dashboard.component.ts`.
- **`shared/icon/icon.component.ts`**: sistema de iconos semánticos (usado por
  `sidebar.component.ts`).
- **`shared/pagination/pagination-bar.component.ts`**: paginador reutilizado por `categories`,
  `products`, `inventory`, `promotions`, `sales`.
- **`shared/password-input/`, `shared/searchable-select/`**: inputs reutilizables de
  formularios.
- **`shared/money.ts` / `money.pipe.ts`**: formateo de moneda (COP), reexportado también desde
  `receipt.util.ts`.
- **`shared/normalize-text.ts`**: normalización de texto (búsquedas sin tildes).
- **`shared/data/mock-data.ts`**: código muerto — ver Zonas oscuras.

### 3.5 Entornos

`src/environments/environment.ts` (producción) vs `environment.development.ts`: mismas claves,
sin drift de nombres. Diferencias esperadas: `production` (bool), `rootDomain`
(`skeilopos.com` vs `localhost`), `apiBaseUrl`
(`https://api.skeilopos.com/api/v1` vs `http://localhost:8000/api/v1`). `devRootHosts` incluye
`localhost`/`127.0.0.1` incluso en el fichero de producción (`environment.ts:6`) —
`SUPOSICIÓN`: parece intencional para poder probar el build de producción en local, pero no hay
comentario que lo confirme; pregunta abierta al negocio/equipo: *¿es deliberado permitir hosts de
desarrollo en el build de producción?*

---

## 4. Diagrama de dependencias (texto)

```
┌───────────────────────────────────────────────────────────────────────────┐
│                      pos-heladeria (Angular 21, SPA)                      │
│                                                                             │
│  app.routes.ts ──> dashboard/routes.ts ──> [módulos protegidos por rol]   │
│                └──> super-admin/routes.ts ─> [tenants, users]             │
│                └──> menu/t/:token (pública) ─> tables (menú comensal)     │
│                                                                             │
│  core/ (auth, tenant, guards, realtime, query, printing, services)        │
│    └── consumido por TODOS los módulos de negocio                         │
│    └── auth/auth-token.interceptor.ts ──(dependencia inversa)──> tables/  │
│                                                                             │
│  dashboard ──depende de──> products, users, sales, tables                 │
│  settings  ──depende de──> tenant-info(core), sales, unit-measures,       │
│                             option-groups                                 │
│  tables    ──depende de──> menu(core), realtime(core), promotions, sales, │
│                             cash-register, printing(core)                 │
│  orders    ──depende de──> tables (sin servicio propio)                   │
│  products  ──depende de──> menu(core), option-groups                     │
│  option-groups ─depende de─> inventory, unit-measures(core)               │
│  promotions ──depende de──> products, categories                          │
│  inventory ──depende de──> suppliers, unit-measures(core)                 │
│  super-admin ─reutiliza──> layout de dashboard                            │
└───────────────────────────────┬─────────────────────────────────────────┘
                                 │ HTTP (REST) + SSE
                                 │ prefijo /api/v1, cabecera x-tenant-host
                                 │ (o JWT de mesa en el flujo QR)
                                 ▼
┌───────────────────────────────────────────────────────────────────────────┐
│                     pos-backend (FastAPI, PostgreSQL 16)                  │
│                                                                             │
│  app/main.py ──registra 23 routers──> app/api/v1/*                        │
│                                                                             │
│  core/db.py (tenant/schema) ──usado por──> TODOS los módulos              │
│  core/qr_context.py + qr_token.py ─────────> menu, cart (flujo comensal)  │
│  core/event_bus.py + events.py ────────────> realtime (consume)           │
│                              └──publican──── table_sessions, cart,        │
│                                               orders, scheduler            │
│  core/scheduler.py ─────────────────────────> table_sessions, orders,     │
│                                                 promotions (expira)        │
│                                                                             │
│  catalog (motor de precio/consumo) ◄── products, cart, orders, sales      │
│  inventory (stock único) ◄── orders/consumption, sales/consumption        │
│  promotions ◄── menu, cart, sales, orders, table_sessions                 │
│  sales/builder.build_sale ◄── orders/checkout, table_sessions             │
│  invoices ◄── sales/builder (import local, evita ciclo)                   │
│  cash (turnos) ◄── sales/builder.ensure_open_shift (usado por orders,     │
│                      table_sessions)                                      │
│  cart ──> table_sessions.try_release_if_empty                             │
│  table_sessions ──> orders/checkout, sales/builder, promotions            │
│  audit (escritura: core/audit.py) ◄── promotions, orders/checkout         │
│                                                                             │
│  admin ──crea tenant──> core/db.tenant_create, celery_task.send_email     │
│  super_admin, users, categories, unit_measures, uploads, reports,         │
│  tenant, health ──módulos hoja, sin dependientes internos──               │
└───────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Tecnologías y versiones reales

### 5.1 Backend (`pos-backend/requirements.txt`, `docker-compose*.yml`)

| Tecnología | Versión | Evidencia |
|---|---|---|
| Python | 3.12 | `.github/workflows/deploy.yml:22` (`python-version: "3.12"`) |
| FastAPI | 0.136.3 | `requirements.txt` |
| Starlette | 1.2.0 | `requirements.txt` |
| Uvicorn | 0.48.0 | `requirements.txt` |
| SQLAlchemy | 2.0.50 | `requirements.txt` |
| Alembic | 1.18.4 | `requirements.txt` |
| Pydantic | 2.13.4 (+ `pydantic-settings` 2.14.1) | `requirements.txt` |
| PostgreSQL | 16 (dev, `docker-compose.yml:3`) / 16.10 (prod, `docker-compose.prod.yml:4`) | `docker-compose*.yml` |
| psycopg | 3.3.4 (+ `psycopg2-binary` 2.9.12 en paralelo) | `requirements.txt` |
| Redis (cliente) | 8.0.0 | `requirements.txt` |
| Redis (servidor) | 7.4-alpine (prod) / 7-alpine (dev) | `docker-compose*.yml` |
| Celery | 5.6.3 | `requirements.txt` |
| APScheduler | 3.11.3 | `requirements.txt` |
| PyJWT | 2.13.0 | `requirements.txt` |
| bcrypt | 5.0.0 | `requirements.txt` |
| boto3 | 1.43.48 (Cloudflare R2, S3-compatible) | `requirements.txt` |
| httpx | 0.28.1 (cliente HTTP, usado para envío real de correo) | `requirements.txt` |
| sentry-sdk | 2.61.0 | `requirements.txt` (sin uso confirmado en `app/`, ver Zonas oscuras) |

### 5.2 Frontend (`pos-heladeria/package.json`)

| Tecnología | Versión | Evidencia |
|---|---|---|
| Angular (core/cdk/cli/build) | ^21.1.0 / ^21.2.14 (cdk) / ^21.1.1 (cli, build) | `package.json` |
| TypeScript | ~5.9.2 | `package.json` |
| RxJS | ~7.8.0 | `package.json` |
| @tanstack/angular-query-experimental | ^5.101.4 | `package.json` |
| TailwindCSS | ^4.1.12 (vía `@tailwindcss/postcss`) | `package.json` |
| Chart.js / ng2-charts | ^4.5.1 / ^10.0.0 | `package.json` |
| qrcode | ^1.5.4 | `package.json` |
| Vitest | ^4.0.8 (test runner, `@angular/build:unit-test`) | `package.json` |
| jsdom | ^27.1.0 | `package.json` |
| Playwright | ^1.60.0 | `package.json` — sin carpeta `e2e/`, ver Zonas oscuras |
| npm | 10.9.2 (`packageManager`) | `package.json:8` |

### 5.3 Infraestructura y despliegue

- **Contenedores**: Docker Compose (dev y prod separados); imagen backend publicada en
  `ghcr.io/deimerhdz/pos-backend` (`.github/workflows/deploy.yml:9,53-59`).
- **CI/CD**: GitHub Actions, pipeline `test → build → deploy` solo para `pos-backend`
  (`.github/workflows/deploy.yml`); **no se encontró workflow de CI en `pos-heladeria`**
  (`SUPOSICIÓN`: no se halló ningún fichero bajo `.github/workflows/` en ese repo — confirmar con
  el negocio si el despliegue del frontend es manual o vive en otro sitio no versionado aquí).
- **Almacenamiento de imágenes**: Cloudflare R2 (S3-compatible, vía `boto3`), confirmado por
  `app/core/storage.py` y el flujo de `uploads`/`presign` consumido desde `products` en el
  frontend.
- **Dominio de producción**: `skeilopos.com` (backend en `api.skeilopos.com`,
  `pos-heladeria/src/environments/environment.ts`; CORS en `pos-backend/app/main.py:88`).

---

## 6. Zonas oscuras

Cada hallazgo se lista con su evidencia. Ninguno se corrige en este documento (Principio III).

1. **Modelo `BusinessHours` huérfano.** `pos-backend/app/models/business_hours.py:8-25` define
   la tabla `business_hours` (creada por
   `alembic/versions/a6b7c8d9e0f1_business_hours_audit.py:34-43`) y se registra en
   `app/models/__init__.py:44`. `grep -rn "business_hours\|BusinessHours"
   pos-backend/app --include=*.py` confirma que no existe ningún router ni service que la lea o
   escriba. La tabla existe en base de datos pero no hay API que la use.

2. **CI del backend solo cubre 1 de 12 scripts de test.** De los 12 scripts
   `pos-backend/app/scripts/test_*.py`, únicamente `test_promotions_rules.py` corre en
   `.github/workflows/deploy.yml:20-22` (confirmado leyendo el fichero). Los otros 11 —
   `test_cancel_inventory.py`, `test_event_bus.py`, `test_facturacion.py`, `test_qr_token.py`,
   `test_realtime_stream.py`, `test_receta_obligatoria.py`, `test_session_ttl.py`,
   `test_split_blindaje.py`, `test_table_release.py`, `test_table_sessions.py`,
   `test_variantes_duplicadas.py`, `test_variant_option_groups.py`— son scripts manuales
   ejecutables (`python -m app.scripts.X`), no suites `pytest`/`unittest` (`pytest` no aparece en
   `requirements.txt`). Cubren lógica crítica (reversa de inventario al cancelar, facturación
   automática, blindaje de cobro dividido) que no se verifica automáticamente en cada despliegue.

3. **Scripts de un solo uso, sin automatizar.**
   `pos-backend/app/scripts/variantes_sin_receta.py`, `opciones_con_consumo_fijo.py`,
   `opciones_fuera_de_grupo.py` son explícitamente de solo lectura y pensados para correrse una
   vez, a mano, antes de activar ciertos flags o desplegar ciertos cambios. No están en CI ni
   los referencia ningún otro módulo (confirmado por grep, cero referencias cruzadas).

4. **`.env` con la misma clave definida dos veces con valores distintos.** Verificado
   directamente: `pos-backend/.env:4` y `:12` definen `ENVIRONMENT=dev` dos veces;
   `pos-backend/.env:13` (`EMAIL_API_URL=http://localhost:8080`) y `:32`
   (`EMAIL_API_URL=https://app.skeilopos.com`) definen la misma variable con valores
   incompatibles (local vs producción) en el mismo fichero. `python-dotenv`/Pydantic-settings
   toma la última ocurrencia, así que el valor efectivo depende del orden del fichero — riesgo de
   que un reordenamiento futuro cambie en silencio a qué API de correo apunta el sistema.
   `.env.example` no tiene esta duplicación.

5. **Dependencias de correo declaradas y aparentemente no usadas.** `requirements.txt` incluye
   `resend`, `fastapi-mail`, `aiosmtplib`, `passlib`, `fastar` — ninguno se importa en ningún
   fichero bajo `app/` (confirmado por grep de cada nombre). El envío real de correo se hace por
   HTTP directo a un servicio externo vía `httpx` (`app/core/mail.py:19-38`). `sentry-sdk`
   tampoco tiene ningún `import sentry_sdk` bajo `app/` según el mismo método de grep —
   `SUPOSICIÓN`: podría inicializarse fuera del código versionado (variable de entorno + agente
   externo) o ser un resto sin retirar; pregunta abierta al negocio/equipo: *¿Sentry se activa
   por configuración externa al repo, o es una dependencia sin usar?*

6. **`initialize_database()` se llama dos veces en el arranque del backend**: una dentro de
   `lifespan` (`app/main.py:50`) y otra fuera de él en `create_app()` (`app/main.py:98`, antes de
   registrar los routers). No es un bug — la función corta temprano si ya hay una revisión de
   Alembic aplicada (`app/core/db.py:233-235`) — pero es una llamada redundante sin comentario
   que explique por qué está en ambos sitios.

7. **Alias marcado `DEPRECADO` sigue expuesto en la API de caja.**
   `pos-backend/app/api/v1/cash/service.py:180-181` (`"cash_sales": ventas_efectivo`) y su
   reflejo en `app/api/v1/cash/schemas.py:112` están comentados como "alias de ventas_efectivo
   por compatibilidad con el frontend" — deuda técnica documentada, no accidental, pero sigue en
   producción.

8. **Endpoint público inseguro, retirado y documentado pero no purgado del historial de
   decisiones.** `pos-backend/app/api/v1/menu/router.py:187-190` deja constancia de que `GET
   /menu/qr/{qr_token}` se eliminó por inseguro (tenant por cabecera falsificable, id de mesa
   plano) y fue reemplazado por `/menu/qr-token/{token}`. El frontend conserva la ruta
   `/menu/qr/:token` (`pos-heladeria/src/app/app.routes.ts:56-61`) apuntando ahora a
   `ExpiredQrComponent`, específicamente para explicar a quien escanee un QR físico antiguo por
   qué ya no funciona, en vez de mostrar un 404 mudo.

9. **`app/app.log` versionado en el repositorio backend, pese al `.gitignore` actual.**
   `pos-backend/.gitignore` tiene la regla `*.log`, pero `git ls-files` confirma que
   `app.log` sigue trackeado — la regla no aplica retroactivamente a ficheros ya versionados
   (fue añadido en el commit `6a07849`, antes o sin la regla vigente en ese momento). El fichero
   sigue en el árbol del repo y puede seguir recibiendo cambios si alguien hace `git add -f` o si
   ya estaba fuera del alcance de la regla al añadirse.

10. **`pos-heladeria/src/app/shared/data/mock-data.ts` — código muerto confirmado.** Exporta
    `MOCK_ORDERS`, `MOCK_PRODUCTS`, `MOCK_TABLES` (datos ficticios). `grep -rn "mock-data"
    pos-heladeria/src/app` no devuelve ninguna referencia fuera del propio fichero.

11. **`UsersService.getUser()` sin llamada real en la UI.**
    `pos-heladeria/src/app/modules/users/services/users.service.ts:123` — el propio comentario
    del código dice "No usado por la UI base". Confirmado con `grep -rn "\.getUser("
    pos-heladeria/src/app` → cero llamadas.

12. **`playwright` instalado sin carpeta de tests e2e.**
    `pos-heladeria/package.json:47` lo declara en `devDependencies`; no existe ningún
    directorio `e2e/` ni `playwright.config.*` en el repo. `SUPOSICIÓN`: puede ser preparación
    para un trabajo futuro no iniciado; pregunta abierta: *¿hay intención de retomar
    e2e con Playwright, o se puede retirar la dependencia?*

13. **Cobertura de tests desigual en el frontend.** 33 ficheros `*.spec.ts` sobre 162 ficheros
    fuente no-spec (~20%, ambos conteos verificados con `find`). Módulos **sin ningún test**:
    `auth`, `categories`, `option-groups`, `orders`, `settings`, `suppliers`, `unit-measures`,
    `users`. `dashboard` solo cubre `sidebar.component.spec.ts`. `super-admin` cubre servicios
    pero ningún componente. El núcleo mejor cubierto es `core/` (auth, guards, realtime, tenant)
    y partes críticas de `tables` (`pos-terminal.store`, `receipt.util`, `payment-draft.util`,
    `dining-cart`, `diner`, `dining-session`, `table`).

14. **Acoplamiento inverso `core → modules` en el frontend.**
    `pos-heladeria/src/app/core/auth/auth-token.interceptor.ts:8` importa `DinerTokenStore` desde
    `modules/tables/services/diner-token.store.ts`. Es una decisión que el propio código
    necesita para distinguir las rutas públicas del comensal del resto, pero rompe la
    convención de que `core/` no debería depender de `modules/`.

15. **Servicios de catálogo alojados fuera de su módulo natural.**
    `core/services/menu.service.ts` y `core/services/unit-measure.service.ts` viven en `core/`
    en vez de en `modules/products` (o `modules/tables`) y `modules/unit-measures`
    respectivamente — inconsistente con el resto de módulos. `modules/unit-measures/` ni
    siquiera tiene carpeta `services/` propia.

16. **`fileReplacements` de producción en `angular.json` es un no-op.**
    `pos-heladeria/angular.json:34-39` reemplaza `environment.ts` por sí mismo en la
    configuración `production` (`replace` y `with` apuntan al mismo fichero) — inofensivo pero
    confuso, probablemente resto de cuando existió un tercer fichero de entorno.

17. **Sin workflow de CI versionado para `pos-heladeria`.** No se encontró ningún fichero bajo
    `pos-heladeria/.github/workflows/` (a diferencia de `pos-backend`, que sí tiene
    `deploy.yml`). `SUPOSICIÓN`: el despliegue del frontend podría ser manual o gestionarse
    fuera de este repositorio; pregunta abierta al negocio: *¿cómo y con qué frecuencia se
    despliega `pos-heladeria` a producción?*

18. **No se hallaron adicionalmente**: `TODO`/`FIXME`/`HACK` reales en el frontend (las
    coincidencias de "TODO" son la palabra española "todo"), `console.log` de depuración
    olvidados, bloques de código comentado, botones deshabilitados permanentemente, ni datos
    hardcodeados sustituyendo llamadas API — en ninguno de los dos repositorios se encontró
    evidencia de estas categorías más allá de lo ya listado arriba.

---

## 7. Preguntas abiertas para el negocio

Recopiladas de las `SUPOSICIÓN` marcadas arriba:

1. ¿`sentry-sdk` (`pos-backend/requirements.txt`) se activa por configuración externa al
   repositorio, o es una dependencia instalada y nunca conectada? (Zona oscura 5)
2. ¿Es deliberado que `devRootHosts` incluya `localhost`/`127.0.0.1` en el fichero de entorno de
   **producción** del frontend (`environment.ts:6`)? (sección 3.5)
3. ¿Hay intención de retomar pruebas end-to-end con Playwright en `pos-heladeria`, o se puede
   retirar esa dependencia? (Zona oscura 12)
4. ¿Cómo y con qué frecuencia se despliega `pos-heladeria` a producción, dado que no hay
   workflow de CI versionado para ese repositorio? (Zona oscura 17)
