# Research: Auditoría del ciclo de vida de una orden

Todas las incógnitas técnicas de la Fase 0. No queda ningún `NEEDS CLARIFICATION` — las decisiones de negocio (Sentry como único destino, retención, hash, alcance) ya están resueltas en `spec.md` § Clarifications; lo que sigue son las decisiones técnicas necesarias para implementarlas.

## 1. Mecanismo de envío a Sentry

**Decision**: usar la API de Sentry Logs del SDK (`sentry_sdk.logger.info(...)`, con atributos estructurados: `order_id`, `tenant_id`, `actor_type`, `actor_id`, `actor_role`, `event_type`), no `sentry_sdk.capture_message` ni breadcrumbs.

**Rationale**: Sentry Logs ya está habilitado en el cliente existente (`sentry_sdk.init(..., enable_logs=True)`, `app/main.py:98`) — es exactamente el mecanismo que el pedido original de este feature nombró explícitamente ("Sentry Logs tiene retención de 7 a 30 días según plan"). Es distinto del canal de errores (`capture_exception`, ya usado por `error_middleware.py` para el módulo de super-admin), lo que permite cumplir FR-007 (los eventos de auditoría deben ser una categoría identificable, sin mezclarse con el registro operativo de errores existente) sin construir nada nuevo del lado del cliente Sentry.

**Verificado contra el código real** (no solo contra la documentación): se descargó el wheel de `sentry-sdk==2.61.0` — la versión pinneada en `requirements.txt` — desde PyPI y se inspeccionó `sentry_sdk/logger.py` y `sentry_sdk/utils.py::format_attribute` directamente. Confirmado: `sentry_sdk.logger.info(template, **kwargs)` existe y acepta `attributes=` como se describe aquí; pero `format_attribute` solo preserva como atributo estructurado un valor `bool`/`int`/`float`/`str` (o lista homogénea de esos) — cualquier objeto/dict anidado como valor de un atributo se degrada a un `repr()` de Python, no a un campo filtrable. Por eso los atributos listados arriba son deliberadamente **planos** (`actor_type`/`actor_id`/`actor_role`, no un objeto `actor` anidado) — ver `data-model.md` y `contracts/order-audit-log-event.md`, que deben mantenerse alineados con esta decisión (un desalineamiento aquí fue detectado y corregido por `/speckit-analyze` el 2026-09-03).

**Alternatives considered**:
- `sentry_sdk.capture_message(..., level="info")`: descartado — viaja por el mismo canal que los eventos de error, dificultando distinguir auditoría de logging operativo (FR-007), y no está pensado para un volumen alto de eventos informativos rutinarios.
- `sentry_sdk.add_breadcrumb(...)`: descartado — un breadcrumb solo se envía adjunto a un evento de error posterior; si la transición de negocio nunca produce un error (el caso normal), el breadcrumb nunca llega a Sentry, incumpliendo FR-001 (el evento debe registrarse siempre, no solo cuando algo falla).

## 2. Gate por entorno (dev/test vs. prod)

**Decision**: reutilizar el mismo guard que ya usa `error_middleware.py` (`if settings.ENVIRONMENT != "prod": return` antes de tocar `sentry_sdk`).

**Rationale**: fuera de `prod`, `sentry_sdk.init()` nunca se llama (`app/main.py:96-98`), así que cualquier llamada al SDK sin cliente activo ya es un no-op silencioso — el guard solo lo deja explícito, igual que en el único precedente existente en el proyecto. Mantiene el mismo patrón de gating en todo el código que toca `sentry_sdk`, evitando una segunda convención.

**Alternatives considered**: inicializar un cliente Sentry independiente en dev/test para poder verificar el envío real end-to-end — descartado: es justamente el tipo de infraestructura operativa que el spec decidió no construir para el módulo de órdenes (spec.md § Mapeo del flujo actual, punto 4). Los tests de este feature verifican el payload en el punto de salida (mockeado), no una entrega real a Sentry — ver § 6.

## 3. Transformación no reversible de datos sensibles (FR-005, FR-012)

**Decision**: HMAC-SHA256 con una clave dedicada nueva (variable de entorno `AUDIT_HASH_SECRET`), no un hash simple sin clave y no el `JWT_SECRET` ya existente.

**Rationale**: un hash sin clave (`hashlib.sha256(valor)`, patrón ya usado en `app/api/v1/auth/routes.py` para el hash de tokens de sesión aleatorios de alta entropía) es apropiado para tokens, pero el nombre de un comensal es un valor de baja entropía y adivinable (nombres comunes, catálogo acotado) — sin clave, cualquiera con acceso a Sentry podría precomputar un diccionario de hashes de nombres frecuentes y revertir la ofuscación por fuerza bruta, incumpliendo el espíritu de FR-005. Una clave dedicada (HMAC) cierra esa vía. No se reutiliza `JWT_SECRET` porque pertenece a un dominio de seguridad distinto (autenticación de sesión); rotar el secreto de auditoría no debería forzar invalidar sesiones activas, ni viceversa. La propiedad de FR-012 (mismo valor de entrada → mismo hash, siempre) se cumple igual con HMAC, siempre que la clave no cambie.

**Alternatives considered**:
- Hash sin clave (`sha256` simple): descartado por lo expuesto arriba (ataque de diccionario sobre valores de baja entropía).
- Cifrado reversible: descartado — el spec exige explícitamente "no reversible" (FR-005); un cifrado reversible sería, en la práctica, seguir enviando el dato original a un proveedor externo.
- Reutilizar `JWT_SECRET` como clave HMAC: descartado por mezclar dominios de seguridad distintos.

## 4. Puntos de integración (dónde se emite cada uno de los 8 tipos de evento)

**Decision**: un único punto de entrada, `record_order_audit_event(...)`, en un módulo nuevo `app/core/order_audit.py`, invocado desde cada función de servicio **después** de que su `commit` correspondiente se completó con éxito (nunca antes — FR-010):

| Evento (`event_type`) | Función donde se emite | Cubre |
|---|---|---|
| `order.created` | `app/api/v1/cart/service.py::submit_cart` | Creación vía QR (orden nace `recibida`) |
| `order.created` | `app/api/v1/orders/service.py::create_order` | Creación manual por staff (con o sin `hold_for_payment`) |
| `order.confirmed` | `app/api/v1/orders/checkout.py::confirm_order`, `confirm_cash_payment_attempt`, `approve_payment_attempt`, `checkout_and_send` (los 4 disparadores de la transición `recibida → abierta`/`pagada`), vía un helper común `_record_order_confirmed(order_id, user, trigger)` | Confirmación manual (`confirm_order`) y automática (efecto colateral de confirmar un pago, incluido `checkout_and_send` — ver adenda abajo). **Corrección de implementación**: la decisión original de emitir *dentro* de `_confirm_order_impl` no es viable — esa función no tiene frontera transaccional propia (el `commit` lo hace cada llamador), así que emitir ahí publicaría el evento antes del `commit`, violando FR-010. Cada llamador invoca `_record_order_confirmed` después de su propio `commit`. |
| `order.payment_attempt.created` | `app/api/v1/cart/service.py::create_payment_attempt`, y también `submit_cart` | Registro de un intento de pago. `submit_cart` también lo emite porque el primer `OrderPaymentAttempt` del flujo QR (el que lleva el comprobante) nace ahí mismo (spec 025) — `create_payment_attempt` solo cubre los reintentos posteriores a un rechazo. |
| `order.payment.cash_confirmed` | `app/api/v1/orders/checkout.py::confirm_cash_payment_attempt` | Confirmación de pago en efectivo |
| `order.payment.transfer_approved` | `app/api/v1/orders/checkout.py::approve_payment_attempt` | Aprobación de comprobante de transferencia |
| `order.payment.transfer_rejected` | `app/api/v1/orders/checkout.py::reject_payment_attempt` | Rechazo de comprobante de transferencia |
| `order.cancelled` | `app/api/v1/orders/checkout.py::cancel_order` | Cancelación — cubre ambas rutas de entrada (staff vía `orders/router.py`, comensal vía `cart/service.py::cancel_my_order`, que delega en la misma función) |
| `order.payment.checkout_and_send` | `app/api/v1/orders/checkout.py::checkout_and_send` | **Adenda post-implementación (FR-014)**: cobro y envío a cocina en un solo paso de una comanda `hold_for_payment` — descubierto durante `/speckit-implement`, no estaba en el mapeo original. `checkout_and_send` no pasa por `_confirm_order_impl` ni por un `OrderPaymentAttempt`: construye la `Sale` directamente con `data.payments` (una o más líneas de pago, potencialmente de métodos distintos) y llama a `_deduct_and_open` — el mismo núcleo de descuento que usa `_confirm_order_impl`, así que semánticamente también es una confirmación. Por eso emite **dos** eventos tras su único `commit`: `order.confirmed` (vía `_record_order_confirmed(..., trigger="automatic_payment")`) y `order.payment.checkout_and_send` con el resumen de los pagos (ver `contracts/order-audit-log-event.md`). |

**Rationale**: emitir justo después del `commit` exitoso de cada función garantiza FR-010 sin necesitar envolver la sesión de base de datos en la lógica de auditoría. Un punto de entrada único (en vez de uno por endpoint HTTP) evita duplicar la construcción del payload entre rutas que ya comparten lógica de negocio hoy (p. ej. confirmación manual vs. automática, ambas dentro de `_confirm_order_impl`).

**Alternatives considered**: emitir desde la capa de routers (HTTP) en vez de la capa de servicio — descartado porque la confirmación automática (dentro de la confirmación de pago) nunca pasa por el router de `confirm_order`; emitir ahí perdería ese caso, exactamente el edge case que el spec señala explícitamente.

## 5. Actor y tenant en cada punto de emisión

**Decision**: pasar el actor y el tenant de forma explícita como parámetros al helper, en vez de inferirlos dentro de él. El actor se modela con un tipo propio (ver `data-model.md`), construido a partir de los mismos objetos `User`/`SessionParticipant` que cada función de servicio ya recibe hoy como parámetro. El tenant se pasa como `Tenant.id`, ya disponible en cada capa (router → servicio) por el mecanismo de resolución existente (`get_tenant`/`resolve_tenant_by_id`, sin cambios).

Para el evento `order.confirmed` cuando ocurre como efecto automático de un pago: el actor de ese evento se marca como `sistema` (no como el cajero), pero comparte el mismo `order_id`/`tenant_id` que el evento de pago que lo disparó, de forma que ambos se puedan correlacionar en Sentry filtrando por `order_id` (ver Edge Case del spec sobre esta distinción).

**Rationale**: sigue el patrón ya usado en el resto del módulo de órdenes (soft-reference por `user_id`/`participant_id`, nunca una FK) y cumple FR-004 (tenant explícito, sin inferencia posterior) sin introducir un mecanismo de contexto nuevo.

**Alternatives considered**: propagar tenant/actor de forma implícita vía `contextvars` a nivel de request — descartado porque el spec exige explícitamente no depender de inferencia posterior (FR-004), y porque pasar el valor explícito es más simple que introducir un mecanismo de contexto nuevo para un helper que ya se llama desde un lugar con esos datos a la mano.

## 6. Verificación (Principio X)

**Decision**: tests con `unittest`, en un módulo nuevo `app/characterization_tests/test_order_audit_log.py`, junto a la suite existente. Dos niveles:
- **Unitarios**: del helper `record_order_audit_event` y de la función de hash HMAC, mockeando el punto de salida a Sentry (`sentry_sdk.logger.info`) para verificar el payload exacto — nunca el valor original de nombre/comprobante, siempre su hash; siempre `order_id`/`tenant_id`/actor presentes.
- **De integración**: uno por cada uno de los 7 puntos de la tabla en § 4, verificando que la función de servicio invoca el helper con los parámetros correctos tras completar la transición, y que **no** lo invoca si la transición falla (cubre FR-010) ni si la transición de negocio se revierte.

**Rationale**: mantiene el mismo mecanismo de test ya usado en el proyecto (`unittest`, sin `pytest`), y prueba el contrato (FR-005, FR-010, FR-012) sin depender de un servicio externo real — igual que el resto de la suite no depende de un Sentry real hoy.

**Alternatives considered**: probar contra un proyecto Sentry real de pruebas (sandbox) — descartado por introducir una dependencia de red/infraestructura a la suite de tests, inconsistente con cómo se prueba hoy el único otro uso de `sentry_sdk` en el proyecto (siempre mockeado o cortado por el gate de entorno).

---

# Extensión: Logging operativo general (FR-015–FR-021)

Investigación de la Fase 0 para la extensión aprobada en Clarifications (tercera ronda) — con el feature de auditoría de orden (§1-6 arriba) ya implementado y en producción.

## 7. Mecanismo: middleware nuevo, separado de `RequestIdMiddleware`

**Decision**: introducir una clase de middleware nueva (p. ej. `OperationalLogMiddleware`) en `app/core/error_middleware.py`, junto al `RequestIdMiddleware` existente pero **sin modificarlo**. Se monta en `app/main.py` de forma global (sin `path_prefix` positivo), y internamente hace `if request.url.path.startswith(SUPER_ADMIN_ERROR_PREFIX) or request.method not in {"POST","PUT","PATCH","DELETE"}: return await call_next(request)` antes de cualquier otra cosa — el resto de la lógica (medir duración, armar atributos, emitir el log) solo corre para las peticiones que sí están en alcance.

**Rationale**: `RequestIdMiddleware` hoy resuelve un problema distinto (envolver errores no manejados en el formato de respuesta HTTP estandarizado de super-admin, vía `register_error_handlers`/`DomainError`) y **no registra nada en el camino feliz** — ni siquiera dentro de super-admin hay hoy un log de éxito. FR-015 pide lo contrario: registrar tanto éxito como error, y solo para peticiones mutativas, en todo el backend salvo super-admin. Generalizar `RequestIdMiddleware` para que además haga esto mezclaría dos responsabilidades (envelope de errores vs. trazabilidad operativa) en una clase pensada para la primera. Una clase nueva, sin tocar la existente, cumple igual el "cambio mecánico y pequeño" que anticipaba el docstring del archivo, sin arriesgar el comportamiento ya probado de super-admin (Principio II).

**Alternatives considered**:
- Generalizar `RequestIdMiddleware` para aceptar una lista de prefijos y una bandera "loguear también el éxito" — descartado: mezclaría el contrato de envelope de errores (pensado para HTTP 4xx/5xx con forma de respuesta específica) con el de trazabilidad genérica (que no toca la respuesta en absoluto), complicando ambos casos de uso para ahorrarse una clase.
- Loguear desde cada router/endpoint individualmente (como se hizo con los 8 eventos de orden) — descartado: son decenas de endpoints mutativos en ~20 routers; un middleware es exactamente el mecanismo que evita repetir esa integración en cada uno (y es coherente con FR-019: la etiqueta se deriva automáticamente, no se cura a mano por endpoint).

## 8. Resolución de actor/tenant: side-effect en las 3 dependencias compartidas, no en el middleware

**Decision**: `app/core/db.py::get_tenant`, `app/core/dependencies.py::get_current_user` y `app/core/qr_context.py::get_session_context` — las tres dependencias FastAPI que ya resuelven tenant/actor para prácticamente toda ruta de negocio — se modifican para, como efecto colateral, estampar `request.state.tenant_id` / `request.state.actor_id` / `request.state.actor_type` (`get_tenant` ya recibe `Request`; `get_current_user` y `get_session_context` necesitan agregar un parámetro `req: Request` — cambio no disruptivo, FastAPI lo inyecta solo). El middleware nuevo (§7), al ejecutarse *después* de `call_next()`, simplemente lee lo que ya quedó en `request.state` — no repite ninguna lógica de autenticación.

**Rationale**: es el punto donde tenant/actor ya se resuelven hoy, una sola vez por petición, con acceso directo al objeto `Request` (al menos `get_tenant`) o casi (`get_current_user`/`get_session_context` solo necesitan que se les inyecte). Evita duplicar la decodificación de JWT/token de sesión dentro del middleware — que además correría *antes* de que FastAPI resuelva las dependencias de la ruta, por lo que un middleware por sí solo no podría conocer el actor sin repetir esa lógica.

**Alternatives considered**:
- El middleware decodifica el JWT/token de sesión por su cuenta (duplicando la lógica de `get_valid_token_data`/`open_session_context`) — descartado: duplica lógica de autenticación en dos lugares que pueden divergir con el tiempo (p. ej. si cambia el algoritmo o la validación de expiración en un lugar y no en el otro).
- `contextvars` en vez de `request.state` — descartado por la misma razón que en research.md §5: ya existe un mecanismo (`request.state`, con el mismo ciclo de vida que la petición) sin necesidad de introducir uno nuevo.

## 9. Nivel de severidad del log según el status de la respuesta

**Decision**: `status < 400` → `sentry_sdk.logger.info(...)`; `400 ≤ status < 500` → `sentry_sdk.logger.warning(...)`; `status >= 500` → `sentry_sdk.logger.error(...)`. Coincide con la petición original del usuario ("loggear log de información y errores").

**Rationale**: permite filtrar en Sentry por severidad sin depender de inspeccionar el atributo `status` a mano, y es el mapeo estándar entre código HTTP y nivel de log.

**Alternatives considered**: todo a nivel `info`, dejando que el atributo `status` distinga éxito de error — descartado por ser menos útil para el caso de uso principal (FR-015/User Story 4: depurar fallos en producción), que se beneficia de poder filtrar directamente por severidad.

## 10. La "ruta" registrada es el patrón, no la URL resuelta

**Decision**: el atributo `route` de cada entrada es el patrón de ruta registrado en FastAPI (p. ej. `/orders/{order_id}/cancel`), leído de `request.scope["route"].path` **después** de que `call_next()` retorna (el enrutamiento ya ocurrió) — no la URL con los valores reales de los parámetros de path.

**Rationale**: agrupar/filtrar en Sentry por ruta solo funciona si todas las peticiones al mismo endpoint comparten el mismo valor de atributo; con la URL resuelta (con UUIDs reales), cada peticion tendría un valor distinto, rompiendo cualquier agregación — precisamente lo que FR-019 (etiqueta derivada de método+ruta) necesita para ser útil.

**Alternatives considered**: `request.url.path` (la URL tal cual llegó, con los IDs reales) — descartado por lo anterior; sigue estando disponible como parte de otros atributos si hiciera falta el detalle exacto de una petición puntual, pero no es la clave de agrupación.

## 11. Verificación de la extensión (Principio X)

**Decision**: un módulo de test nuevo, `app/characterization_tests/test_operational_log.py`, con: (a) tests directos del middleware (mutativo vs. lectura, dentro/fuera de super-admin, éxito vs. distintos rangos de status, nunca incluye el cuerpo de la petición/respuesta, nunca decae la petición real si el logging falla); (b) un test de humo que confirma que las 3 dependencias modificadas (`get_tenant`, `get_current_user`, `get_session_context`) siguen devolviendo exactamente lo mismo que antes para quien las usa (Principio II — el side-effect en `request.state` no debe cambiar su valor de retorno ni su firma para los llamadores existentes); (c) un test que confirma que un evento de auditoría de orden (de los 8 ya existentes) incluye el `request_id` de su petición (FR-021). Más la corrida completa de la suite existente, dado que este middleware se monta sobre casi todas las rutas — es el cambio de mayor radio de impacto potencial de todo este spec.

**Rationale**: sigue el mismo patrón `unittest` ya usado en todo el proyecto y en el resto de este spec; el énfasis en "nunca cambia lo que ya devolvían las 3 dependencias" es la salvaguarda concreta contra romper accidentalmente autenticación/resolución de tenant en el resto del sistema al tocar código tan ampliamente compartido.

**Alternatives considered**: ninguna — dado el radio de impacto, no se consideró razonable verificar esto con menos que characterization tests explícitos sobre las 3 dependencias modificadas.
