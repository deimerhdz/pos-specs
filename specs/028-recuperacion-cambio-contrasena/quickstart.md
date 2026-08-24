# Quickstart: validar Recuperación y Cambio de Contraseña (Personal)

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite firmas ni
columnas ya detalladas en [data-model.md](./data-model.md) y `contracts/` — solo enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` (Python 3.14), ejecutado desde la raíz de
`../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL, Redis ni el microservicio de correo real: `auth_fixtures.py` crea
SQLite en memoria con las tablas de `shared` **y** `tenant` (research.md Decisión 10);
`send_email_task.delay(...)` y `enforce_sliding_window(...)` se mockean con `unittest.mock.patch`,
mismo patrón que `test_products_service.py` (spec 021) mockea `delete_object`.

## Paso 0 — Confirmar la línea base antes de tocar código

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: todo en verde. Ningún test existente se modifica como parte de esta spec —
`auth` no tenía ninguno.

## US1 — Recuperar el acceso cuando se olvida la contraseña

Ficheros nuevos: `app/characterization_tests/auth_fixtures.py`,
`test_auth_forgot_password.py`, `test_auth_reset_password.py`.

1. Crear un tenant + un usuario activo (fixture nueva). `POST /auth/forgot-password` con su email,
   header `x-tenant-host` del tenant → `200` con el mensaje genérico
   ([contracts/forgot-password.md](./contracts/forgot-password.md)); mock de `send_email_task`
   recibe una llamada con un link que contiene el token crudo (Acceptance Scenario 3).
2. Mismo request con un email que no pertenece a ninguna cuenta del tenant → `200`, **mismo**
   mensaje, mock de `send_email_task` **sin** llamadas (Acceptance Scenario 4, SC-003).
3. Repetir con una cuenta `active=False` → mismo trato que el paso 2 (Edge Case, FR-004).
4. Repetir con un email que sí existe pero en **otro** tenant (`x-tenant-host` distinto) → mismo
   trato que el paso 2 (Edge Case, FR-004).
5. Emitir un enlace a las `14:00:00` (mockear `datetime.now`) → `GET .../validate` a las
   `14:29:59` → `{"valid": true}`; a las `14:30:01` → `{"valid": false, "reason": "expired"}`
   (Acceptance Scenario 5, FR-007/FR-011).
6. `POST /auth/reset-password` con ese token válido y `new_password` de 8-12 caracteres, repetida
   dos veces (simulando doble clic) → la primera `200`; la segunda `400 reason="used"`, sin aplicar
   un segundo cambio (Acceptance Scenario 6-7, FR-008). Verificar: `user.tokens_valid_after` quedó
   fijado, `user.must_change_password == False`, login con la contraseña anterior falla y con la
   nueva funciona.
7. Pedir un enlace a las `14:00` y otro a las `14:10` para la misma cuenta → intentar `GET
   .../validate` a las `14:15` sobre el token de las `14:00` → `{"valid": false, "reason":
   "invalid"}` (invalidado por el de las 14:10) (Acceptance Scenario 8, FR-005).
8. Pedir enlaces a las `10:00`, `10:02`, `10:05` (mock de reloj) → 4ª a las `10:07` → `429`
   (`enforce_sliding_window` bloquea, sin `send_email_task`); 5ª a las `10:15:01` → `200` normal,
   porque la de las `10:00` ya salió de la ventana de 15 min (Acceptance Scenario 9, FR-010,
   research.md Decisión 3 — este es el caso que distingue una ventana deslizante genuina de una
   ventana fija: repetir con una distribución de solicitudes que una ventana fija fallaría, p. ej.
   `10:00, 10:01, 10:02` seguido de `10:14` bloqueado en vez de permitido).

```bash
python -m unittest app.characterization_tests.test_auth_forgot_password -v
python -m unittest app.characterization_tests.test_auth_reset_password -v
```

## US2 — Cambiar la contraseña voluntariamente desde Ajustes de cuenta

Fichero nuevo: `app/characterization_tests/test_auth_change_password.py`.

1. Usuario autenticado, `current_password` correcta, `new_password` válida (8-12) y distinta de la
   actual → `POST /auth/change-password` → `200`; `user.tokens_valid_after` fijado
   ([contracts/change-password.md](./contracts/change-password.md)).
2. `current_password` incorrecta → `400`, `user.password_hash` sin cambios — verificable
   reintentando login con la contraseña anterior (Acceptance Scenario 3 de US2, `RN-AUTH-01`
   intacto).
3. `new_password` igual a la actual → `400` (FR-021, regla nueva de esta spec — spec 001 no la
   tenía).
4. `new_password` de 13 caracteres → `422` (antes válida con hasta 128, FR-019, cambio de
   comportamiento explícito #1).
5. Verificar que el endpoint no modifica `name`/`email`/`phone`/`role_id` de `User` — solo
   `password_hash`/`must_change_password`/`tokens_valid_after` (Acceptance Scenario 4 de US2,
   SC-007).

```bash
python -m unittest app.characterization_tests.test_auth_change_password -v
```

## US1 + US2 — Cierre de sesiones (FR-009/FR-017/SC-005)

Fichero nuevo: `app/characterization_tests/test_auth_session_revocation.py`.

1. **Sin regresión de A-23**: cuenta con `active=False`, `tokens_valid_after=None` → `GET
   /auth/refresh-token` con su refresh token → `401 "User not found or inactive"`, **idéntico** al
   comportamiento hoy documentado en spec 001 (RN-AUTH-07, protegida) — este test debe existir
   **antes** de tocar `dependencies.py`/`routes.py`, para confirmar que sigue en verde después.
2. Login → tomar `iat` del access emitido → fijar `user.tokens_valid_after = now() + 1s` (simula un
   cambio de contraseña) → llamar `get_current_user`/`get_authenticated_user` con ese mismo token →
   `401` con el detalle nuevo, distinto de `"Token has been revoked"` (logout) y de `"User not
   found or inactive"` (A-23).
3. Mismo escenario sobre `GET /auth/refresh-token` con el refresh emitido en el mismo login → `401`
   igual que el paso 2 (todas las sesiones cerradas, no solo el access).
4. Emular Flujo B completo: login → `POST /auth/change-password` exitoso → login de nuevo con la
   contraseña nueva (mismo patrón que ya usa `AuthService.changePassword()` hoy) → el token del
   **segundo** login (emitido después de `tokens_valid_after`) pasa `get_current_user` sin
   problema, mientras que el token del **primer** login (antes del cambio) ya no sirve —
   verificable con el mismo `GET /auth/refresh-token` (SC-005: "verificable recargando una sesión
   abierta en otro navegador").

```bash
python -m unittest app.characterization_tests.test_auth_session_revocation -v
```

## US3 — Correo de aviso de seguridad

Cubierto dentro de `test_auth_reset_password.py` (Flujo A) y `test_auth_change_password.py`
(Flujo B) — no tiene fichero propio porque no agrega ninguna pantalla ni endpoint (spec.md, "Why
this priority" de US3).

1. Tras el `POST /auth/reset-password` exitoso del paso 6 de US1: el mock de `send_email_task`
   registra **dos** llamadas — el correo del enlace (al pedirlo) y el correo de aviso de cambio (al
   guardarlo), con asuntos distintos (FR-022, Acceptance Scenario 1 de US3).
2. Tras el `POST /auth/change-password` exitoso del paso 1 de US2: el mock de `send_email_task`
   registra **una** llamada, el correo de aviso (Acceptance Scenario 2 de US3).
3. Mockear `send_email_task.delay` para que lance una excepción → el endpoint (cualquiera de los
   dos flujos) sigue respondiendo `200`/su mensaje de éxito normal, sin propagar el error (FR-028).

## Verificación de no regresión — spec 001

```bash
python -m unittest app.characterization_tests.test_auth_session_revocation -v   # paso 1 (A-23)
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: `POST /auth/login` (RN-AUTH-03/04/05, sin tests dedicados hoy pero sin
ningún cambio de código en esta spec), `GET /auth/logout` (RN-AUTH-08, sin cambios) y
`RN-AUTH-07`/A-23 (paso 1 de arriba) siguen comportándose exactamente igual — esta spec no modifica
`login()` ni `revoke_token()`.

## Verificación final — SC-001 a SC-008

| Criterio | Cómo se verifica |
|---|---|
| SC-001 (recuperación en <5 min sin soporte) | US1 completo, pasos 1 y 6 — un único ciclo pedir→abrir→guardar→login |
| SC-002 (correo en <2 min) | US1 paso 1 — fuera del alcance del test automatizado más allá de confirmar el despacho async (Celery); verificación real en producción/staging |
| SC-003 (0% revela existencia) | US1 pasos 2-4 (mismo mensaje) + paso 8 (mismo trato del límite exista o no cuenta) |
| SC-004 (100% enlaces caducados/usados muestran error) | US1 pasos 5-7 |
| SC-005 (cierre de sesiones correcto por flujo) | Bloque "US1 + US2 — Cierre de sesiones" completo |
| SC-006 (100% de cambios exitosos generan aviso) | Bloque US3, pasos 1-2 |
| SC-007 (Ajustes de cuenta no toca el resto del perfil) | US2 paso 5 |
| SC-008 (soporte deja de recibir solicitudes manuales) | No verificable en esta guía — criterio de negocio post-lanzamiento |

## Frontend — validación manual (no automatizada en esta guía)

`pos-heladeria` usa Vitest para specs de servicios (`auth.service.spec.ts`/
`auth-api.service.spec.ts`, extendidos con los métodos nuevos), pero `LoginComponent`,
`ChangePasswordComponent` y los dos componentes nuevos (`ForgotPasswordComponent`,
`ResetPasswordComponent`, `AccountSettingsComponent`) no tienen `.spec.ts` propio hoy (mismo
criterio que ya aplica el repo a `LoginComponent`/`ChangePasswordComponent`) — se validan con
`ng serve` + navegación manual:

1. Las 4 pantallas de esta spec (login, solicitud de enlace, definición de contraseña nueva,
   Ajustes de cuenta) en 375px de ancho (DevTools, modo responsive) → sin scroll horizontal ni
   elementos cortados; en login, el enlace "Restablecer contraseña" visible sin scroll (FR-001,
   FR-024).
2. Flujo A completo en el navegador: pedir enlace → abrir el correo (o copiar el link del log del
   microservicio de email en dev) → definir contraseña nueva → redirigido a login → entrar con la
   contraseña nueva.
3. Con una sesión ya iniciada en el navegador, abrir un enlace de reset en la misma pestaña → la
   sesión existente se limpia antes de mostrar el formulario (FR-006) — verificar que
   `localStorage` pierde `pos.access_token`/`pos.refresh_token` al cargar la pantalla.
4. Flujo B completo: dropdown de usuario → "Cambiar contraseña" → Ajustes de cuenta → cambiar con
   éxito → la sesión sigue activa sin pedir login de nuevo (verificar en la pestaña) mientras una
   segunda pestaña con sesión abierta pierde acceso al recargar (SC-005).
5. En la pantalla de definir contraseña nueva (Flujo A) y en Ajustes de cuenta (Flujo B), escribir
   "Nueva contraseña" y "Confirmar nueva contraseña" con valores distintos → el sistema lo señala
   antes de enviar nada, sin llamar al backend (FR-020).
