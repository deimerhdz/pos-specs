# Research: Recuperación y Cambio de Contraseña (Personal)

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — las 4 clarificaciones de
negocio ya se resolvieron en `spec.md` (sesión 2026-08-24) y el resto de incógnitas era puramente
técnico, resuelto leyendo directamente `pos-backend`/`pos-heladeria`. Este documento registra las
decisiones de diseño y las alternativas descartadas.

## Decisión 1 — Cierre de sesiones sin tabla de sesiones: columna `tokens_valid_after` + `iat` ya presente en el JWT

- **Decisión**: se agrega `User.tokens_valid_after: DateTime | None` (nullable, `shared.users`). Al
  completar con éxito cualquiera de los dos flujos, se fija `tokens_valid_after = now()`. Los tres
  puntos que ya releen el usuario contra base de datos en cada petición —
  `get_current_user`/`get_authenticated_user` (`app/core/dependencies.py`) y el handler de
  `GET /auth/refresh-token` (`app/api/v1/auth/routes.py:111-150`), los tres exigiendo ya
  `active==True`, `RN-AUTH-07`/A-23 — ganan una comparación adicional:
  `token_data["iat"] < user.tokens_valid_after → 401`. `iat` ya viaja en todo JWT emitido
  (`create_access_token`, `app/core/utils.py:44`: `payload["iat"] = now`), así que no hace falta
  tocar la forma del token.
- **Flujo A** (no autenticado): no hay ninguna sesión que preservar — `tokens_valid_after = now()`
  sin excepción alguna cierra "todas las sesiones activas" (FR-009) de una sola vez.
- **Flujo B** (autenticado, "la sesión de origen sigue activa", FR-017): en vez de "exentar" el jti
  de la sesión actual (que exigiría enseñar al backend a distinguir qué jti pertenece a esa
  sesión, sin tenerlo disponible para el refresh token asociado), el frontend reutiliza el patrón
  que **ya existe hoy** para el cambio de contraseña forzado: `AuthService.changePassword()`
  (`pos-heladeria/src/app/core/services/auth.service.ts`) vuelve a llamar `login()` con la
  contraseña nueva inmediatamente después de un `POST /auth/change-password` exitoso, para obtener
  tokens frescos con `must_change_password: false`. Esos tokens nuevos se acuñan **después** de
  `tokens_valid_after`, así que pasan la comparación sin necesitar ninguna excepción — la "sesión"
  percibida por el usuario (misma pestaña, sin interrupción visible) sigue activa exactamente como
  pide FR-017/SC-005, sin que el backend tenga que preservar el jti original.
- **Rationale**: el sistema no tiene tabla de sesiones ni lista positiva de jtis vigentes (solo una
  blocklist negativa de revocados, `app/core/redis.py`) — no hay forma de enumerar "todas las demás
  sesiones" para revocarlas una a una. Cerrar por corte de tiempo (`iat < cutover`) es el mecanismo
  mínimo que logra "todas las sesiones emitidas antes de este instante dejan de servir" sin
  rediseñar el esquema de tokens, y reutiliza una relectura de BD que **ya ocurre en cada petición
  protegida** (cero queries nuevas).
- **Alternatives considered**:
  - Claim `sid`/`family_id` compartido entre access y refresh de un mismo login, más un jti exento
    persistido — descartado: exige modificar la forma del JWT que emite `login()` (toca
    `RN-AUTH-06`) y sigue sin resolver cómo excusar el refresh token cuando `change-password` solo
    conoce el jti del access presentado.
  - Endpoint nuevo de "cerrar todas las sesiones" separado, invocado por el cliente tras el cambio
    — descartado: el corte ya ocurre como efecto secundario del cambio exitoso: introducir un
    segundo endpoint solo movería la misma responsabilidad sin necesidad (Principio V).

## Decisión 2 — Token de un solo uso: tabla nueva con hash, no `itsdangerous`

- **Decisión**: tabla nueva `password_reset_tokens` (schema `shared`, FK a `shared.users.id`). El
  token que viaja en el enlace es un valor aleatorio opaco (`secrets.token_urlsafe(32)`, mismo
  rigor criptográfico que `RN-AUTH-10`/`generate_random_password`); solo su hash SHA-256 se
  persiste (`token_hash`, único, indexado) — igual que una contraseña, el valor crudo nunca vive en
  la base de datos.
- **Rationale**: `app/core/utils.py:75-93` ya define `URLSafeTimedSerializer`
  (`create_url_safe_token`/`decode_url_safe_token`, salt `"email-configuration"`, de `itsdangerous`,
  ya en `requirements.txt`) **sin ningún caller hoy** — parece un remanente de boilerplate. Es
  tentador reusarlo, pero un token firmado y sin estado no puede satisfacer por sí solo tres reglas
  del spec: invalidar el enlace anterior al pedir uno nuevo (FR-005), marcarlo consumido tras un uso
  exitoso (FR-008) sin volver a aceptarlo, e invalidarlo si la cuenta cambia de correo (FR-012).
  Lograr eso con `itsdangerous` exigiría **igualmente** un registro de estado en Redis o SQL (para
  "usado"/"invalidado") — la misma complejidad que una tabla, pero repartida en dos mecanismos en
  vez de uno. Una tabla da directamente los tres estados que el propio Key Entity del spec describe
  ("vigente, usado, o invalidado"), es auditable, y sigue el patrón ya usado por `OrderPaymentAttempt`
  (spec 024) para modelar un ciclo de vida con estados.
- **Alternatives considered**: guardar el token en texto plano (más simple de depurar) — descartado
  por el mismo criterio que ya aplica el proyecto a las contraseñas (nunca texto plano en BD, aunque
  el uso sea de un solo uso y corta vida); token firmado (`itsdangerous`) + set en Redis para
  invalidación — descartado por la razón de arriba (dos mecanismos por el precio de uno).

## Decisión 3 — Límite de 3 solicitudes / 15 min: ventana deslizante genuina sobre Redis (ZSET), no se reutiliza `enforce()`

- **Decisión**: función nueva en `app/core/rate_limit.py` (p. ej. `enforce_sliding_window(key, limit,
  window_seconds)`) sobre el mismo cliente Redis ya importado (`token_blocklist`), usando un
  **sorted set** por clave: `ZADD key now now_id`, `ZREMRANGEBYSCORE key -inf (now-window)` para
  podar entradas fuera de ventana antes de contar, `ZCARD key` para el conteo, `EXPIRE key window`
  para limpieza. Clave: `f"rl:pwreset:{tenant_bucket}:{email_normalizado}"` (email en minúsculas y
  sin espacios extremos; `tenant_bucket` es el `id` del tenant resuelto por `x-tenant-host`, o un
  bucket fijo si no hay tenant).
- **Rationale**: `enforce()`/`_hit()` (`app/core/rate_limit.py:34-41`) implementan **ventana fija**
  (`INCR`+`EXPIRE` solo en el primer hit) — coincide por coincidencia con el ejemplo del spec
  (Acceptance Scenario 9: la solicitud de las 10:00 sale de ventana justo a las 10:15:00,
  exactamente el mismo instante en que expiraría una ventana fija iniciada en ese primer hit), pero
  no generaliza: dos ráfagas de 3 pegadas al límite de una ventana fija permitirían 6 solicitudes en
  segundos alrededor del corte, algo que FR-010 (**"ventana deslizante"**, término explícito) no
  autoriza. Un ZSET con poda por score sí sostiene la propiedad "como máximo N en cualquier ventana
  de W segundos" sin importar el patrón de llegada.
  Además, `enforce()` está atado a las claves fijas `(IP, mesa)` de `settings.RATE_LIMIT_*`
  (genéricas para el módulo QR) — el eje de esta spec es el **correo ingresado**, no IP ni
  dispositivo (Assumptions del spec), así que hacía falta una función con clave/ventana/límite
  parametrizables de todos modos.
- **Fail-open**: igual que `enforce()`, si Redis no responde no se bloquea la solicitud (se loguea y
  se deja pasar) — mismo criterio ya documentado en el docstring de `rate_limit.py`.
- **Config nueva** (`app/core/config.py`, mismo patrón que `RATE_LIMIT_*`):
  `PASSWORD_RESET_MAX_REQUESTS` (default `3`), `PASSWORD_RESET_WINDOW_SECONDS` (default `900`),
  `PASSWORD_RESET_TOKEN_EXPIRY_MINUTES` (default `30`).
- **Alternatives considered**: generalizar `enforce()` para aceptar `window`/`limit` explícitos
  mantenimiento la semántica de ventana fija — descartado, no cierra la brecha de "ventana
  deslizante" que pide FR-010 textualmente.

## Decisión 4 — FR-012 (invalidación por cambio de correo): snapshot de email en la fila del token, no un hook en un endpoint de cambio de correo

- **Decisión**: `password_reset_tokens.email_snapshot` guarda el correo de la cuenta en el instante
  de emisión. Al validar un token (`GET /auth/reset-password/validate` o el `POST` final), se
  compara `email_snapshot == user.email` en vivo; si no coinciden, se trata como enlace inválido.
- **Rationale**: se verificó `app/api/v1/users/router.py` (líneas 17-204) y **no existe hoy ningún
  endpoint que modifique `User.email`** (solo `PATCH` de rol y de `active`) — FR-012 es defensivo,
  protege contra una funcionalidad de "cambiar mi correo" que no existe todavía. Comparar contra un
  snapshot en el momento de validar cubre la regla sin necesitar enganchar lógica de invalidación en
  un endpoint que hoy no existe; si en el futuro se agrega "cambiar correo", bastará con que ese
  endpoint no haga nada especial — el snapshot ya desactiva cualquier token pendiente por sí solo.
- **Alternatives considered**: ninguna razonable — enganchar la invalidación a un endpoint
  inexistente sería diseñar para un requisito que no está en el alcance de ninguna spec vigente
  (Principio V).

## Decisión 5 — Doble consumo del enlace (FR-008): `SELECT ... WITH FOR UPDATE` sobre la fila del token

- **Decisión**: `POST /auth/reset-password` bloquea la fila de `password_reset_tokens` por
  `token_hash` con `WITH FOR UPDATE` antes de verificar `used_at IS NULL`; si ya está usada (o
  bloqueada por una petición concurrente que está a punto de marcarla), la segunda petición ve el
  estado "usado" sin aplicar un segundo cambio.
- **Rationale**: exactamente el mismo patrón pesimista que ya usa `confirm_order`
  (`app/api/v1/orders/checkout.py`) y que la spec 024 adoptó para `OrderPaymentAttempt` (su
  Decisión 9) frente al mismo tipo de problema — doble clic, reintento de red, o dos pestañas
  (Edge Cases de esta spec). Reutilizar el patrón evita introducir un segundo mecanismo de
  concurrencia para una clase de problema que el repo ya resuelve de una forma.
- **Alternatives considered**: verificación optimista (`UPDATE ... WHERE used_at IS NULL RETURNING
  ...`, sin `SELECT FOR UPDATE` previo) — funcionalmente equivalente en Postgres (el `UPDATE` ya
  toma el lock de fila); se documenta como implementación aceptable equivalente, la spec no exige
  una u otra, solo el resultado (FR-008).

## Decisión 6 — "Ajustes de cuenta" (US2) es una página nueva, no una pestaña de `SettingsPageComponent`

- **Decisión**: se agrega una página nueva en `pos-heladeria`
  (`src/app/modules/account/pages/account-settings.component.ts`, ruta
  `dashboard/mi-cuenta` o similar), accesible a **cualquier** usuario autenticado del dashboard
  (cajero o admin) — sin el `roleGuard([UserRole.ADMIN])` que hoy protege `SettingsPageComponent`.
  El dropdown del usuario en `HeaderComponent`
  (`pos-heladeria/src/app/modules/dashboard/layout/header.component.ts`) gana la opción "Cambiar
  contraseña" apuntando a esa ruta nueva.
- **Rationale**: `SettingsPageComponent` (`pos-heladeria/src/app/modules/settings/pages/`) es
  configuración del **negocio/tenant** (info básica, métodos de pago, unidades de medida, grupos de
  opciones), gateada a `ADMIN` en `dashboard/routes.ts` — un cajero no tiene acceso hoy y no debería
  ganarlo solo para poder cambiar su propia contraseña. "Ajustes de cuenta" en el spec es un
  concepto distinto (datos de la cuenta personal, no del negocio) — mezclarlo en el mismo componente
  forzaría relajar un guard de autorización existente sin que ninguna FR de esta spec lo pida.
- **Alternatives considered**: agregar una pestaña a `SettingsPageComponent` y relajar su
  `roleGuard` — descartado, cambiaría el control de acceso de las otras pestañas (negocio) como
  efecto colateral no pedido por esta spec (Principio V).

## Decisión 7 — URL del enlace de correo: mismo patrón que `login_url` en `admin/router.py`

- **Decisión**: `f"https://{tenant.host}.skeilopos.com/reset-password?token={raw_token}"` en
  producción, `f"http://{tenant.host}.localhost:4200/reset-password?token={raw_token}"` en
  desarrollo, condicionado por `settings.ENVIRONMENT` — idéntico al `if/else` ya usado para
  `login_url` en `app/api/v1/admin/router.py:31-34` al enviar el correo de bienvenida.
- **Rationale**: es el único lugar del backend que hoy construye una URL absoluta hacia el frontend
  a partir de un tenant; reusar el mismo criterio evita inventar una segunda convención de URL
  (Principio V/IX: no se agrega configuración nueva de "frontend base URL" cuando ya existe un
  patrón funcionando).

## Decisión 8 — Correo de aviso (FR-022) y plantillas: se extiende `mail.py`, se despacha vía Celery igual que el correo de bienvenida

- **Decisión**: dos funciones nuevas en `app/core/mail.py`, mismo estilo que `welcome_email_body`
  (HTML inline, sin motor de plantillas): `password_reset_email_body(reset_url, expiry_minutes)`
  (enlace del Flujo A) y `password_changed_email_body(when, email)` (aviso tras cualquier cambio
  exitoso, FR-022). Ambas se despachan con `send_email_task.delay(recipients=[...], subject=...,
  body=...)`, envuelto en `try/except` que solo loguea (`logger.warning(..., exc_info=True)`) sin
  romper la respuesta — exactamente el patrón de `app/api/v1/admin/router.py:35-44`.
- **Rationale**: satisface FR-028 (el usuario ve igual el mensaje de éxito aunque el envío falle)
  reusando la infraestructura async ya probada (Celery + Redis como broker,
  `app/celery_task.py::send_email_task`), sin añadir una dependencia ni un mecanismo de colas nuevo.
- **Alternatives considered**: enviar el correo de forma síncrona dentro del request (`send_email()`
  directo) — descartado, bloquearía la respuesta hasta 10s (timeout de `httpx.post`,
  `app/core/mail.py:23`) y violaría el espíritu de FR-028 (un proveedor caído no debe poder demorar
  ni fallar visiblemente la respuesta al usuario).

## Decisión 9 — Longitud 8-12 y "distinta de la actual" (FR-019/FR-021): se modifica `ChangePasswordRequest`, se crea `ResetPasswordRequest` con el mismo rango

- **Decisión**: `ChangePasswordRequest.new_password` pasa de `Field(min_length=6, max_length=128)` a
  `Field(min_length=8, max_length=12)` — el cambio de comportamiento explícito #1 que la propia
  `spec.md` ya autoriza y cita (`RN-AUTH-01`/`RN-AUTH-09` de spec 001). `ResetPasswordRequest`
  (Flujo A, nueva) usa el mismo rango. FR-021 ("no igual a la actual") se verifica con
  `verify_password(new_password, user.password_hash)` — si `True`, rechazar; reutiliza la misma
  primitiva que ya valida `current_password`, sin lógica nueva de hashing.
- **Rationale**: ninguna alternativa razonable — son los valores que la propia spec fija
  numéricamente (FR-019) y la primitiva de verificación ya existe.

## Decisión 10 — Testing de backend: fixtures nuevas que también materializan el schema `shared`, llamado directo a las funciones de router (sin `TestClient`)

- **Decisión**: se agrega `app/characterization_tests/auth_fixtures.py` con su propio
  `schema_translate_map` que colapsa **tanto** `tenant` como `shared` a `None` sobre SQLite en
  memoria, y crea las tablas de `Tenant`/`Role`/`User`/`PasswordResetToken` (hoy la fixture genérica,
  `fixtures.py:63-84`, solo materializa tablas del schema `tenant` — ninguna de `auth` vive ahí). Los
  tests llaman las funciones de `app/api/v1/auth/routes.py` **directamente** (pasando `body`, `req`
  o `user`/`db` ya resueltos a mano), igual que el resto de `characterization_tests` llama funciones
  de servicio en vez de pasar por el stack ASGI — no hay precedente de `TestClient` en el repo y no
  se introduce ahora.
- **Rationale**: es la extensión mínima del patrón ya establecido (mismo truco de
  `schema_translate_map`, mismo estilo de test) que hacía falta porque `auth` nunca tuvo tests
  (confirmado: no existe `test_auth_*.py`, y spec 001 SC-004 ya señalaba esta ausencia). No se
  introduce pytest ni `TestClient` porque ninguna FR de esta spec lo exige y el resto del repo no lo
  usa (Principio V: no cambiar convenciones de testing como efecto colateral).
- **Nota sobre A-23 (protegida)**: el nuevo chequeo de `tokens_valid_after` en el handler de
  `refresh-token` se agrega **después** de la relectura por `id` + `active==True` que exige
  `RN-AUTH-07`, nunca en su lugar — los tests nuevos verifican explícitamente que una cuenta
  desactivada sigue devolviendo `401 "User not found or inactive"` exactamente como hoy (sin tocar
  ese caso), y que el caso nuevo (revocado por cambio de contraseña) es un `401` **distinto**
  (motivo, no código) que solo aparece cuando `tokens_valid_after` está fijado.

## Migraciones — estrategia de rollback (Principio VIII)

Todo lo nuevo vive en el schema `shared` (mismo patrón que `tenants.logo_url`,
`alembic/versions/c2d3e4f5a6b7_tenant_logo_url.py`: una sola migración, sin
`for_each_tenant_schema`, porque `User`/`Tenant`/`Role` no son per-tenant):

- `users.tokens_valid_after` (columna nueva, nullable, sin default): rollback = `op.drop_column`.
  `NULL` para toda cuenta existente hasta su primer cambio de contraseña bajo esta spec — no
  requiere backfill ni cambia el significado de ninguna fila existente.
- `password_reset_tokens` (tabla nueva, schema `shared`, FK a `shared.users.id`): rollback =
  `op.drop_table`. Sin datos preexistentes que preservar.
