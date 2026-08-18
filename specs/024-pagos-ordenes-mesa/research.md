# Research: Pagos de Órdenes en Mesa (Skeilopos)

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — las 4 clarificaciones de
negocio ya se resolvieron en `spec.md` (sesión 2026-08-18) y el resto de incógnitas era puramente
técnico, resuelto leyendo directamente `pos-backend`/`pos-heladeria`. Este documento registra las
decisiones de diseño y las alternativas descartadas.

## Decisión 1 — "Pendiente de pago" es un estado derivado, no una columna nueva

- **Decisión**: no se agrega ningún valor nuevo al `CHECK` de `CustomerOrder.status`
  (`recibida|abierta|bloqueada|pagada|cancelada`, `app/models/customer_order.py:82`). Una orden está
  "pendiente de pago" (Key Entity `Orden` del spec) exactamente cuando `status == "recibida"` — que
  es, además, la única condición bajo la cual `confirm_order` acepta operar hoy
  (`app/api/v1/orders/checkout.py`). Al agregarle a `confirm_order` la precondición "existe un
  `OrderPaymentAttempt` con `status='confirmado'`" (FR-017), la orden deja de poder salir de
  `recibida` sin pago confirmado — "pendiente de pago" queda sinónimo de `recibida` sin necesidad de
  persistirlo aparte.
- **Rationale**: `confirm_order` es, por diseño de la spec 008 (invariante A-25), el único punto que
  hace avanzar `recibida → abierta` — no existe un endpoint de transición de estado genérico. Agregar
  un estado literal nuevo (`"pendiente_pago"`) obligaría a tocar el `CHECK` constraint protegido y a
  decidir dónde encaja frente a `recibida` en la máquina de estados documentada en el docstring de
  `CustomerOrder`, sin aportar ninguna capacidad que la derivación no dé ya.
- **Alternatives considered**: agregar `payment_status` como columna nueva en `CustomerOrder`
  (`pendiente|confirmado|rechazado`) — descartado: duplicaría información que ya vive, de forma más
  rica (historial completo, no solo el último estado), en `order_payment_attempts`; mantenerlas
  sincronizadas es una fuente de bugs que la derivación evita por construcción (FR-016 exige
  conservar el historial completo, no solo el último intento).

## Decisión 2 — Extender `PaymentMethod` existente, no crear una tabla nueva

- **Decisión**: `app/models/payment.py::PaymentMethod` (ya tenant-scoped, ya con `name`, `is_cash`,
  `type`, `active`) gana una columna nueva `payment_info: JSONB | None` — diccionario libre
  (`{"cuenta": "...", "titular": "...", "telefono": "..."}`) sin esquema fijo por campo.
- **Rationale**: `PaymentMethod` ya modela exactamente "efectivo o transferencia, con nombre y
  estado activo/inactivo" (FR-001/FR-004) — es el mismo concepto que pide el spec, solo le falta el
  dato de pago que el comensal necesita ver (FR-002/FR-011). Un diccionario JSONB evita fijar de
  antemano qué campos tiene cada método (el spec dice explícitamente "cuenta, teléfono, código, u
  otro identificador **según el método**" — el conjunto de campos varía por método, no hay una forma
  única).
- **Alternatives considered**: columnas fijas (`account_number`, `phone`, `code`) — descartado, no
  cubre "cualquier otro que use" (User Story 1) sin agregar columnas cada vez que aparezca un método
  con datos distintos; tabla separada `payment_method_details` (clave/valor por fila) — descartado
  por sobre-ingeniería frente a JSONB nativo de Postgres para un dato que solo se lee/escribe entero,
  nunca se filtra por campo individual en una consulta.

## Decisión 3 — El comprobante es una columna del intento de pago, no una tabla propia

- **Decisión**: `OrderPaymentAttempt` (tabla nueva `order_payment_attempts`) tiene una columna
  `receipt_file_url: str | None`, en vez de una tabla `payment_attempt_receipts` separada.
- **Rationale**: el spec fija la relación como 1:1 y de un solo archivo ("cada comprobante
  corresponde a un único archivo... pertenece siempre a un intento de pago específico", Clarification
  4 + Key Entity `Comprobante`) — no hay necesidad de reemplazar o versionar el archivo dentro de un
  mismo intento (un nuevo comprobante siempre implica un intento nuevo, FR-015/FR-015a). Una columna
  nullable en la misma fila representa esa relación 1:1 sin el costo de un `JOIN` extra en cada
  lectura del historial de pagos (FR-016).
- **Alternatives considered**: tabla separada `payment_attempt_receipts` — descartada para esta
  spec por no aportar nada mientras la relación sea 1:1-un-archivo; si una futura spec permite
  múltiples archivos por comprobante (explícitamente fuera de alcance aquí), esa sería la corrección
  natural, no algo que anticipar ahora sin necesidad (Constitución, Principio V).

## Decisión 4 — `order_payment_attempts`: tabla nueva, un solo intento "pendiente" por orden garantizado por índice único parcial

- **Decisión**: tabla nueva en el esquema `tenant`, mismo patrón que `business_hours.py`/
  `payment.py` (`UUIDPrimaryKeyMixin`, `{"schema": "tenant"}`). FR-015a ("no permitir un segundo
  intento mientras uno sigue pendiente") se garantiza con un índice único parcial de Postgres:
  `CREATE UNIQUE INDEX ... ON order_payment_attempts (order_id) WHERE status = 'pendiente'` — no solo
  con una validación de aplicación.
- **Rationale**: el propio `cart/service.py` ya usa este patrón (comentario en
  `service.py:61-63`: `idx_active_session_per_table` garantiza una sola `TableSession` activa por
  mesa) — es un precedente ya establecido en el repo para "a lo sumo una fila activa por padre", más
  robusto que una verificación de aplicación ante dos peticiones casi simultáneas (el mismo riesgo de
  condición de carrera que ya motiva `with_for_update` en `confirm_order`).
- **Alternatives considered**: verificación solo a nivel de servicio (`SELECT ... WHERE status =
  'pendiente'` antes de insertar) — descartada como único mecanismo: sin el índice, dos peticiones
  casi simultáneas del mismo comensal (doble tap en el botón de pago) podrían crear dos intentos
  `pendiente` a la vez, violando FR-015a exactamente en el escenario que motivó la clarificación.

## Decisión 5 — `confirm_order` es el único punto de enforcement de FR-017, no un endpoint nuevo

- **Decisión**: FR-017 (una orden solo avanza a comanda con pago confirmado) se implementa **dentro**
  de `checkout.confirm_order` (`app/api/v1/orders/checkout.py`), agregando una verificación antes del
  `select ... with_for_update` existente: si no hay ningún `OrderPaymentAttempt` con
  `status='confirmado'` para la orden, `409 Conflict`. No se crea un endpoint `/orders/{id}/gate` ni
  un middleware de precondición genérico.
- **Rationale**: invariante A-25 (spec 008, characterization) — el repo eliminó deliberadamente un
  `PATCH /status` genérico tras un bug real; cada transición tiene su propio endpoint dedicado.
  Agregar la condición nueva a la única función que ya hace `recibida → abierta` respeta ese
  invariante en vez de crear una segunda vía paralela para la misma transición.
- **Alternatives considered**: gate en el router (`@router.post("/{order_id}/confirm")`) en vez de en
  el servicio — descartado: el router de `orders` no tiene hoy lógica de negocio, solo delega a
  `checkout.py` (ver `router.py:170-178`); moverla ahí rompería esa separación sin necesidad.

## Decisión 6 — Auth del comensal para el presign de comprobantes: extender el patrón de sesión, no reutilizar `require_tenant_admin`

- **Decisión**: el presign para subir un comprobante no reutiliza `POST /uploads/presign` (gateado
  hoy por `require_tenant_admin` + `User`, `app/api/v1/uploads/router.py:35`). Se agrega un endpoint
  nuevo dentro del router de `cart` (`POST /cart/payment-attempts/{id}/receipt/presign`), gateado por
  `Depends(get_session_context)` — el mismo dependency que ya usan todos los endpoints del comensal
  (`add_item`, `submit_cart`, etc.) — que llama directamente a las primitivas de
  `app/core/storage.py` (`generate_presigned_put_url`, `build_object_key`) ya usadas por el endpoint
  de admin, sin pasar por `require_tenant_admin`.
- **Rationale**: el comensal no es un `User` (no tiene login, invariante de spec 007) — no puede
  satisfacer `require_tenant_admin` ni ninguna variante de él. El tenant/mesa/participante del
  comensal siempre debe resolverse desde el `x-session-token` firmado, nunca de un parámetro de
  body/query (invariante A-24) — igual que el resto de `cart/router.py`. Reutilizar las primitivas de
  `storage.py` (no el endpoint) evita duplicar la lógica de `build_object_key`/`public_url_for` sin
  violar el aislamiento de autenticación entre comensal y staff.
- **Alternatives considered**: relajar `require_tenant_admin` en el endpoint existente para aceptar
  también un `SessionContext` — descartado: mezclaría dos modelos de autenticación completamente
  distintos (`User` con JWT de staff vs. `SessionContext` con JWT de comensal) en un mismo endpoint,
  complicando su lectura y su superficie de autorización sin necesidad.
- **Cambio adicional necesario**: `PresignRequest.folder` (`app/api/v1/uploads/schemas.py`) es un
  `Literal["products", "logo"]` — el endpoint nuevo del comensal construye la key con
  `build_object_key(tenant.schema, "comprobantes", extension)` directamente (no reutiliza
  `PresignRequest`/`PresignResponse`, que son específicas del endpoint de admin), así que no hace
  falta ampliar ese `Literal` ni tocar el endpoint de admin en absoluto.

## Decisión 7 — "Cajero" se resuelve como cualquier usuario staff autenticado, no un rol nuevo

- **Decisión**: los endpoints de aprobar/rechazar comprobante y confirmar efectivo se gatean con
  `Depends(get_current_user)` — el mismo dependency que ya usa `confirm_order` y `close_session` —
  no con un rol `"CAJERO"` nuevo.
- **Rationale**: `app/core/models.py` no tiene hoy ningún rol distinto de `ADMIN`/`SUPER_ADMIN`
  verificado en código; todo lo demás (confirmar pedido, cobrar, mover ítems de cocina) solo exige
  "algún usuario staff autenticado". `spec.md` describe "Cajero" como el rol de negocio que hace esta
  acción, pero ningún FR exige una restricción de autorización más fina que "no es el comensal, y sí
  es staff" — introducir un rol nuevo sería una decisión de negocio no pedida por el spec
  (Constitución, Principio XI: no confundir una decisión técnica de implementación con una de
  negocio no tomada). Si el negocio decide más adelante que un cajero es distinguible de otro perfil
  de staff (ej. mesero), esa restricción de RBAC es una spec propia.
- **Alternatives considered**: crear `require_role("CAJERO")` ahora, anticipando esa necesidad —
  descartado por Principio V (no diseñar para requisitos hipotéticos no pedidos por el spec actual).

## Decisión 8 — "Orden finalizada" (User Story 6) = `status IN ('pagada', 'cancelada')`

- **Decisión**: FR-005/FR-006 ("una orden activa a la vez" / "puede crear una nueva una vez
  finalizada") se implementan comprobando, en `submit_cart`
  (`app/api/v1/cart/service.py:471`), que el comensal no tenga ya una `CustomerOrder` con
  `status NOT IN ('pagada', 'cancelada')` antes de crear la nueva.
- **Rationale**: `spec.md` deja "el evento que marca una orden como finalizada" explícitamente fuera
  de alcance, asumiendo que ya es reconocible por el sistema existente. La characterization spec 017
  confirma que `pagada`/`cancelada` son los dos únicos estados terminales de `CustomerOrder`
  (`mark_order_ready` "bloquea solo los estados terminales de pago (pagada/cancelada), no
  bloqueada" — spec 017 líneas 463-466) — es la definición de "finalizada" que el propio código ya
  usa en otro punto, no una interpretación nueva de esta spec.
- **Alternatives considered**: exigir además `status != 'bloqueada'` como no-finalizada (ya cubierto,
  `bloqueada` no es terminal) — sin cambio; ninguna alternativa razonable compite con reusar la
  misma noción de "terminal" que ya usa `mark_order_ready`.

## Decisión 9 — Bloqueo de doble confirmación de un intento (FR-018): `with_for_update` + columna `version`

- **Decisión**: aprobar/rechazar/confirmar-efectivo sobre un `OrderPaymentAttempt` se hace con
  `SELECT ... WHERE id = :id AND status = 'pendiente' WITH FOR UPDATE`, igual que `confirm_order` ya
  bloquea la fila de `CustomerOrder`; si la fila no aparece (porque ya no está `pendiente`), la
  segunda petición recibe `409 Conflict` sin efecto. Se agrega además una columna `version` (mismo
  patrón que `CustomerOrder.version`, `app/models/customer_order.py:53`) para consistencia con el
  resto del modelo, aunque el `with_for_update` sobre `status='pendiente'` es lo que efectivamente
  garantiza SC-007/FR-018.
- **Rationale**: es exactamente el mismo patrón de bloqueo pesimista que ya usa `confirm_order` para
  el mismo tipo de problema (dos cajeros casi simultáneos) — reutilizarlo evita introducir un segundo
  mecanismo de concurrencia (ej. optimismo puro con reintento) para un caso que el repo ya resuelve
  de una forma.
- **Alternatives considered**: solo lock optimista (`version`) sin `with_for_update` — descartado,
  exigiría que el cliente reintente tras un conflicto de versión, más complejo que el patrón
  pesimista ya establecido para esta misma clase de problema en `confirm_order`.

## Decisión 10 — "Al menos un método de pago activo" (FR-003): validación de servicio dentro de una transacción, no un `CHECK` de fila

- **Decisión**: al desactivar un `PaymentMethod` (`PATCH /sales/payment-methods/{id}`, endpoint
  nuevo), el servicio cuenta, dentro de la misma transacción, cuántos métodos del tenant siguen
  `active=true` excluyendo el que se está desactivando; si el resultado es 0, `409 Conflict`.
- **Rationale**: un `CHECK` de fila no puede expresar una condición sobre el conjunto de filas de la
  tabla (Postgres no soporta `CHECK` agregados entre filas) — la única forma correcta es una
  verificación a nivel de aplicación dentro de la transacción que hace el `UPDATE`.
- **Alternatives considered**: trigger de base de datos (`BEFORE UPDATE` que cuente filas activas) —
  descartado por añadir lógica de negocio a la capa de base de datos, fuera del patrón que sigue el
  resto del repo (toda la validación de negocio vive en `service.py`, no en triggers/constraints más
  allá de invariantes estructurales simples).

## Migraciones — estrategia de rollback (Principio VIII)

- `order_payment_attempts` (tabla nueva): rollback = `op.drop_table("order_payment_attempts",
  schema=tenant_schema)` por cada schema de tenant, vía `@for_each_tenant_schema` — sin datos
  preexistentes que preservar (tabla nueva, ningún dato histórico depende de ella).
- `payment_methods.payment_info` (columna nueva, nullable): rollback = `op.drop_column`. Al ser
  nullable con default `NULL`, agregarla no requiere backfill y no cambia el significado de ninguna
  fila existente — los métodos ya creados (ej. "Efectivo") simplemente quedan con `payment_info =
  NULL` hasta que un admin los edite.
