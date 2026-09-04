# Research: Rol Mesero con Acceso Restringido

**Spec**: [spec.md](./spec.md) | **Fecha**: 2026-09-04

Este documento resuelve las decisiones técnicas necesarias para implementar el
rol Mesero en los dos repos existentes: `pos-backend` (FastAPI + PostgreSQL,
schema-per-tenant) y `pos-heladeria` (Angular). No hay `NEEDS CLARIFICATION`
pendiente — las tres decisiones de negocio (cobro incluido en Terminal de
Mesas, bloqueo real del lado del servidor, y Admin/Cajero sin cambios) ya
quedaron resueltas en `spec.md` (sección Clarifications). Lo que sigue es
exclusivamente investigación técnica: cómo implementar esas decisiones sobre
el código real.

## D1 — Cómo se identifica y almacena el rol Mesero

**Decisión**: Agregar `"MESERO"` como tercer valor de:
1. `RoleName` (enum Pydantic) en `app/api/v1/users/schemas.py` — hoy solo
   tiene `ADMIN`/`CASHIER`. Este enum valida el cuerpo de
   `POST /invitations` y `PATCH /users/{id}/role`.
2. `ROLE_NAMES` (tupla) en `app/core/db.py` — usada por `_seed_shared_data()`
   para poblar `shared.roles` la primera vez que se inicializa una base de
   datos nueva.
3. Una **migración de Alembic nueva** que inserte la fila `MESERO` en
   `shared.roles` para las bases de datos que **ya existen en producción** —
   `_seed_shared_data()` solo corre en la inicialización de una base nueva,
   nunca en un deploy sobre una base ya inicializada, así que agregar el
   valor únicamente a `ROLE_NAMES` no alcanza para el ambiente real.

**Rationale**: `Role` (`app/core/models.py`) es una tabla real en el schema
`shared` (`shared.roles`), con `id`, `name: String(150)` (sin `UNIQUE` ni
`CHECK` a nivel de base de datos) y `active`. `User.role_id` y
`UserInvitation.role_id` son FK a esa tabla — el rol no es un enum de
PostgreSQL, así que no hace falta una migración de esquema, solo un `INSERT`
de datos. La migración debe ser idempotente (verificar por `name` antes de
insertar, igual que ya hace `_seed_shared_data()`), porque Alembic corre una
sola vez por ambiente pero el patrón de este proyecto para datos semilla
siempre valida existencia antes de insertar.

**Alternativas consideradas**:
- *Agregar el valor solo a `ROLE_NAMES`*: rechazada — no tiene efecto en
  ningún ambiente que ya pasó por la inicialización (todos los tenants reales
  hoy), dejando el rol inutilizable en producción sin que nada lo señale
  como roto.
- *Convertir `Role.name` en un enum real de PostgreSQL*: rechazada por fuera
  de alcance — es un cambio de modelo de datos no relacionado con esta
  funcionalidad (Principio V/VI de la constitución) y el mecanismo actual
  (tabla + FK) ya funciona para ADMIN/CASHIER sin ese cambio.

**Rollback**: la migración de bajada elimina la fila `MESERO` de
`shared.roles` **solo si ningún usuario la referencia** (mismo criterio de
seguridad que cualquier baja de una fila con FK entrante); si algún usuario
ya tiene ese rol asignado, la baja debe fallar explícitamente en vez de dejar
usuarios con un `role_id` huérfano.

## D2 — Mecanismo de autorización en el backend (bloqueo real, no solo UI)

**Decisión**: Un único punto de verificación centralizado, agregado dentro
(o inmediatamente después de) `get_current_user()` en
`app/core/dependencies.py`, que se activa **solo cuando
`user.role.name == "MESERO"`**: compara la ruta y el método de la solicitud
en curso contra una lista explícita de rutas permitidas (`contracts/`, ver
D3) y responde `403 Forbidden` si la combinación (método, plantilla de ruta)
no está en esa lista. Para Admin y Cajero, `get_current_user()` no cambia de
comportamiento (FR-008) — la verificación nueva es un `if` adicional que solo
entra en juego para el rol nuevo.

**Rationale**:
- `get_current_user()` ya es la dependencia base de prácticamente todos los
  endpoints de negocio (confirmado revisando cada router: `orders`,
  `table_sessions`, `sales`, `cash`, `products`, `categories`, `inventory`,
  `promotions`, `reports`, `users`, `tenant`, `invoices`, `unit_measures`,
  `uploads`, `business_hours`) — es el único lugar del código por el que
  pasa el 100% de las solicitudes autenticadas de un usuario del tenant.
  Centralizar ahí la verificación evita tener que tocar los ~35+ endpoints
  individuales dispersos en más de 15 archivos.
- `get_current_user()` ya recibe `req: Request` como parámetro opcional
  (agregado en spec 074 para el middleware de logging operativo) — es un
  punto de extensión ya existente y usado con el mismo propósito
  (inspeccionar la solicitud en curso sin cambiar la firma que consumen los
  demás endpoints).
- Es **default-deny**: cualquier endpoint nuevo que se agregue en el futuro
  queda bloqueado para Mesero automáticamente, a menos que alguien lo agregue
  explícitamente a la lista de permitidos. Esto es lo que exige FR-007 ("sin
  importar por qué vía llegue esa solicitud") de forma sostenible — una
  lista de bloqueos en vez de permisos requeriría que cada desarrollador
  recuerde bloquear cada endpoint nuevo que no sea para Mesero, lo cual falla
  en silencio si alguien lo olvida.
- Los endpoints ya protegidos por `require_tenant_admin` (que a su vez
  depende de `get_current_user()`) no requieren ningún cambio: Mesero ya
  queda excluido de ellos hoy porque no es rol `ADMIN`. La verificación nueva
  es una capa adicional, no un reemplazo.

**Alternativas consideradas**:
- *Agregar `Depends(require_roles(...))` a cada endpoint individualmente*:
  rechazada — son más de 35 endpoints repartidos en más de 15 archivos; el
  riesgo real es que alguien agregue un endpoint nuevo sin recordar
  protegerlo (falla abierta, no cerrada), justo lo opuesto de lo que exige
  FR-007.
- *Dependencia a nivel de `APIRouter(dependencies=[...])` por archivo*: útil
  y correcta para los routers que están **100% bloqueados** para Mesero
  (`catalog`, `products`, `categories`, `inventory`, `reports`, `users`,
  `business_hours`, `unit_measures`, `uploads`, `super_admin`, `plan`,
  `audit`, `invitations`) — pero **no alcanza** para los routers mixtos
  (`orders`, `sales`, `cash`, `promotions`, `tenant`), donde conviven
  endpoints permitidos y bloqueados en el mismo archivo. Se descarta como
  mecanismo único porque dejaría los routers mixtos sin cubrir; el mecanismo
  centralizado en D2 cubre ambos casos con una sola implementación.
- *Middleware ASGI que intercepta antes de resolver la ruta*: rechazada —
  Starlette resuelve la ruta coincidente (`request.scope["route"]`) durante
  el enrutamiento; hacerlo en la dependencia (después de resolver la ruta,
  antes de ejecutar el endpoint) es más simple y no requiere lidiar con el
  ciclo de vida de un middleware aparte.

## D3 — Qué rutas quedan permitidas para Mesero (el "alcance" en términos de endpoints reales)

**Decisión**: Ver [contracts/backend-endpoint-access.md](./contracts/backend-endpoint-access.md)
para la lista exhaustiva. Resumen de las tres fuentes que la componen:

1. **Todo `table_sessions/router.py`** (prefijo `/table-sessions`) — el
   propio docstring del archivo ya lo describe como "sesiones de mesa
   (staff)... se consulta, se cobra y se cierra"; no mezcla nada de
   configuración de mesas ni de otro módulo.
2. **La mayoría de `orders/router.py`** (prefijo `/orders`) — este archivo
   mezcla tres cosas distintas bajo el mismo prefijo: configuración de mesas
   (crear/editar mesa, token QR — ya protegidas por `require_tenant_admin`,
   sin cambios necesarios), las acciones de la Terminal de Mesas (tomar
   pedido, mover/fusionar mesas, cocina, pagos, cobro, cancelar, liberar
   mesa) y la consulta de Órdenes (listar/obtener). Las dos últimas quedan
   permitidas para Mesero; la primera ya está bloqueada por el rol Admin
   existente.
3. **Endpoints de solo lectura de otros módulos**, porque la pantalla
   Terminal de Mesas necesita datos de apoyo aunque esos módulos permanezcan
   bloqueados para Mesero como pantallas propias: catálogo de productos para
   armar un pedido (`GET /menu`, ya público/sin autenticación — ni siquiera
   pasa por `get_current_user`), promociones activas para el precio mostrado
   (`GET /promotions`), métodos de pago disponibles para el cobro
   (`GET /sales/payment-methods`), el turno de caja abierto para poder cobrar
   (`GET /cash/shifts/current`), la venta y la factura generadas al cobrar
   (`GET /sales/{sale_id}`, `GET /invoices`, `GET /invoices/{invoice_id}` —
   este último router es 100% de solo lectura, no tiene ningún endpoint de
   escritura), el nombre/logo del negocio para el recibo impreso
   (`GET /tenant`), y el ticket para la conexión en tiempo real que mantiene
   la pantalla actualizada (`POST /realtime/ticket`).

**Rationale**: cada uno de estos endpoints de apoyo se identificó revisando
qué llama realmente la pantalla Terminal de Mesas en el frontend (los
servicios inyectados en el store que la alimenta), no por suposición — así
se evita tanto dejar afuera algo que la pantalla necesita (rompería la
funcionalidad para Mesero) como dejar adentro de más (abriría acceso a datos
de un módulo bloqueado, como el listado completo de ventas o el historial de
turnos de caja, que si comparten el prefijo `get_current_user` pero **no**
los usa la Terminal de Mesas).

**Alternativas consideradas**:
- *Bloquear routers completos aunque tengan un endpoint de apoyo necesario
  (por ejemplo, todo `sales/router.py`)*: rechazada — rompería el cobro
  dentro de la Terminal de Mesas, que sí debe funcionar igual que hoy para
  Cajero (FR-004, decisión de negocio confirmada).
- *Permitir routers completos para simplificar la lista (por ejemplo, todo
  `sales/router.py` o todo `cash/router.py`)*: rechazada — expondría a
  Mesero funciones que no le corresponden y que la Terminal de Mesas no usa,
  como crear una venta de mostrador (`POST /sales`), listar todas las ventas
  (`GET /sales`), abrir/cerrar turno de caja o registrar movimientos de
  efectivo — contradice directamente FR-007/SC-002.

## D4 — Frontend: navegación, rutas y pantalla por defecto

**Decisión**: Extender el mecanismo ya existente, sin crear uno nuevo:
- `UserRole` (`src/app/core/interfaces/user.interface.ts`) gana el valor
  `MESERO = 'mesero'`.
- `NAV_ITEMS` (`src/app/core/config/navigation.config.ts`) agrega
  `UserRole.MESERO` al arreglo `roles` de los ítems "Terminal de mesas" y
  "Órdenes" — ningún otro ítem lo incluye.
- `dashboardRoutes` (`src/app/modules/dashboard/routes.ts`) agrega
  `UserRole.MESERO` al `roleGuard([...])` de las rutas `mesas-sesiones`,
  `mesas-sesiones/:tableId/orden-manual`, `orders` y `orders/:id` —
  exactamente las mismas cuatro rutas que hoy ya incluyen `UserRole.CASHIER`.
- `ROLE_DEFAULT_ROUTES` (`src/app/core/guards/role.guard.ts`) agrega
  `[UserRole.MESERO]: '/dashboard/mesas-sesiones'` — mismo criterio que ya
  usa Cajero con `/dashboard/caja` (pantalla principal del rol).

**Rationale**: `roleGuard(allowedRoles)` ya es genérico (recibe un arreglo de
roles, no está escrito para uno específico) y `NAV_ITEMS` ya filtra por un
arreglo `roles` por ítem — Cajero ya es, hoy, un precedente real de rol con
navegación restringida (ve exactamente Ventas, Órdenes, Terminal de mesas y
Caja, nada más). No hace falta ningún mecanismo nuevo en el frontend, solo
extender las listas existentes con el nuevo valor en los lugares correctos.
La redirección al denegar acceso ya es silenciosa (sin mensaje, ver
`role.guard.ts`) — mismo criterio para Mesero, sin necesidad de una decisión
nueva de UX.

**Alternativas consideradas**: ninguna — el mecanismo existente resuelve el
requisito sin ninguna carencia identificada.

## D5 — Frontend: asignar el rol desde el panel de administración

**Decisión**: Agregar la opción "Mesero" en los tres puntos donde hoy están
codificadas "Admin"/"Cajero" como opciones fijas:
- `RoleName` (`src/app/modules/users/interfaces/user-profile.interface.ts`):
  tipo `'ADMIN' | 'CASHIER' | 'MESERO'`.
- `<option value="MESERO">Mesero</option>` en el `<select>` de
  `invitation-form.component.ts` (invitar usuario nuevo) y
  `user-role-modal.component.ts` (cambiar rol de uno existente).
- `ROLE_LABELS`/mapa de clases de badge en `users-page.component.ts`:
  agregar `MESERO: 'Mesero'` y una clase de color propia (distinta de
  Admin/Cajero) para que el listado lo distinga visualmente.

**Rationale**: son las tres superficies que la spec (FR-001/FR-002)
identifica como "el mismo mecanismo ya disponible hoy para Admin y Cajero" —
no requieren ningún componente ni flujo nuevo, solo la tercera opción en
listas que ya existen.

## D6 — Fuera de alcance, confirmado

- `app/api/v1/cart/` (carrito del comensal vía QR): autenticación
  completamente distinta (token de sesión firmado, no JWT de staff) — nunca
  pasa por `get_current_user()`, así que el mecanismo de D2 no lo toca ni
  necesita tocarlo.
- `business_hours`, `catalog`, `products`: confirmado que la Terminal de
  Mesas no los consume (el selector de productos se alimenta de `GET /menu`,
  no de esos routers) — quedan bloqueados por completo para Mesero sin
  necesidad de ninguna excepción.
- Rol Super Admin: panel de plataforma aparte (`get_current_super_admin`,
  flujo de autenticación propio) — no interactúa con este mecanismo.

## Resumen de lo que hay que construir

| # | Cambio | Repo |
|---|---|---|
| 1 | `RoleName` enum + `ROLE_NAMES` + migración de datos para `shared.roles` | pos-backend |
| 2 | Verificación de alcance por rol dentro de `get_current_user()` (D2) | pos-backend |
| 3 | Lista de rutas permitidas para Mesero (D3 / contracts) | pos-backend |
| 4 | `UserRole.MESERO`, `NAV_ITEMS`, `roleGuard(...)` por ruta, `ROLE_DEFAULT_ROUTES` | pos-heladeria |
| 5 | Opción "Mesero" en formulario de invitación, modal de cambio de rol, y etiquetas/badge del listado de usuarios | pos-heladeria |
