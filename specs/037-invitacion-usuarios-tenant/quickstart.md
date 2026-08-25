# Quickstart: validar Alta de usuarios internos por invitación

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite firmas ni
columnas ya detalladas en [data-model.md](./data-model.md) y `contracts/` — solo enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` (Python 3.14), ejecutado desde la raíz de
`../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL, Redis ni el microservicio de correo real: `auth_fixtures.py` (extendida,
ver research.md, sección Testing) crea SQLite en memoria con las tablas de `shared` y `tenant`;
`send_email()` se mockea con `unittest.mock.patch("app.core.utils.send_email")`, tanto para simular
éxito como el `RuntimeError` de un fallo de envío.

## Paso 0 — Confirmar la línea base antes de tocar código

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**: todo en verde. `app/api/v1/users` no tenía ningún test — no hay ninguno que
deba seguir pasando después de eliminar `create_user`/`UserCreate` (research.md Decisión 11).

## US1 — El ADMIN invita a alguien por correo y rol

Ficheros nuevos: `app/characterization_tests/test_invitations_create.py`.

1. ADMIN de un tenant con plan sin límite de usuarios agotado → `POST /invitations` con un correo
   sin invitación previa ni cuenta, rol `CASHIER` → `201`, `UserInvitation` creada con
   `status='pending'`; **ningún** `User` nuevo (Acceptance Scenario 2,
   [contracts/invitations-create.md](./contracts/invitations-create.md)). El mock de `send_email`
   recibe una llamada con el correo, el enlace de login del tenant y una contraseña de 12
   caracteres — verificar que esa contraseña **no** aparece en el cuerpo de la respuesta `201`
   (Acceptance Scenario 3, FR-006).
2. Repetir el mismo correo, mismo tenant → `409` "ya existe una invitación pendiente" (índice único
   parcial, research.md Decisión 3) — simular la carrera insertando ambas filas casi
   simultáneamente (dos llamadas al endpoint sin esperar el `commit` de la primera) y verificar que
   solo una fila `pending` sobrevive (Edge Case, FR-015).
3. Invitar un correo que ya es un `User` **activo** del tenant → `409` (Acceptance Scenario 5). Con
   un `User` **inactivo** (`active=False`) del mismo correo → mismo `409`, mismo mensaje (edge case
   de la Clarification 1, FR-015) — no se libera el correo por desactivar la cuenta.
4. `CASHIER` autenticado intenta `POST /invitations` → `403` (`require_tenant_admin`, ya existente).
5. Tenant con `plan_vence_en` en el pasado → `POST /invitations` → `403`,
   `ensure_plan_not_expired` (FR-018, research.md Decisión 6) — sin llamar a `send_email`.
6. Tenant en el límite de su plan de "usuarios" **solo por invitaciones pendientes** (sin ningún
   `User` real todavía) → `POST /invitations` de una invitación más → `403` (research.md Decisión 5
   — este es el caso que distingue el conteo extendido de contar solo `User`).
7. Mockear `send_email` para que lance `RuntimeError` → `POST /invitations` → `502`; verificar que
   **no** quedó ninguna fila de `UserInvitation` persistida para ese correo (`rollback()`, FR-012).

```bash
python -m unittest app.characterization_tests.test_invitations_create -v
```

## US2 — La persona invitada activa su cuenta en su primer ingreso

Fichero nuevo: `app/characterization_tests/test_auth_login_invitation_consumption.py`.

1. Con la invitación `pending` del paso 1 de US1, llamar `login()` con ese correo y la contraseña
   temporal capturada del mock de `send_email` → `200`; verificar en la base: se creó un `User` con
   `must_change_password=True`, `role_id` igual al de la invitación, y la `UserInvitation` quedó
   `status='consumed'` con `consumed_at` fijado — todo en una sola llamada (Acceptance Scenario 1,
   [contracts/auth-login.md](./contracts/auth-login.md)).
2. Repetir el login con la misma contraseña temporal → `401 "Invalid credentials"` (la invitación ya
   no está `pending`) — no se crea un segundo `User` ni se duplica nada (Acceptance Scenario 4 de
   US2, edge case de doble consumo).
3. Verificar que el `User` recién creado, al intentar cualquier endpoint protegido antes de cambiar
   su contraseña, sigue el comportamiento ya existente de `must_change_password` (spec 001/031, sin
   tocar) — reutilizar el mismo test que spec 031 ya tiene para ese flag, solo con un usuario
   originado por invitación en vez de por `tenant_create`.
4. Reenviar una invitación (ver US4 abajo) y luego intentar login con la contraseña **original**
   (la de antes del reenvío) → `401` (Acceptance Scenario 4 de US2, FR-010).
5. Login concurrente simulado: dos llamadas a `login()` con la misma invitación `pending` y la
   misma contraseña, una tras otra sin dejar completar el `commit` de la primera antes de disparar
   la segunda (mismo patrón que el paso 2 de US1 para simular condición de carrera) → solo un
   `User` queda creado, la segunda ve la invitación ya `consumed` → `401` (research.md Decisión 7).

```bash
python -m unittest app.characterization_tests.test_auth_login_invitation_consumption -v
```

## US3 — El ADMIN distingue pendientes de activos en el listado

Fichero nuevo: `app/characterization_tests/test_invitations_list.py`.

1. Tenant con 2 `User` activos y 2 `UserInvitation` `pending` → `GET /users` devuelve los 2 activos
   (sin cambios); `GET /invitations` devuelve las 2 pendientes con `email`/`role_name`/`sent_at`
   (Acceptance Scenario 1, [contracts/invitations-list.md](./contracts/invitations-list.md)).
2. Consumir una de las dos invitaciones (login, como en US2) → `GET /invitations` ya no la incluye;
   `GET /users` ahora sí incluye ese correo como usuario activo (Acceptance Scenario 2, SC-004).
3. Invitación de **otro** tenant → nunca aparece en `GET /invitations` del tenant del ADMIN
   (FR-013).

```bash
python -m unittest app.characterization_tests.test_invitations_list -v
```

## US4 — El ADMIN reenvía o cancela una invitación pendiente

Fichero nuevo: `app/characterization_tests/test_invitations_resend_cancel.py`.

1. `POST /invitations/{id}/resend` sobre una `pending` → `200`; `password_hash` y `sent_at`
   cambiaron; `id`/`email`/`status` no ([contracts/invitations-resend.md](./contracts/invitations-resend.md)).
   Login con la contraseña **nueva** → `200` (consume con normalidad, US2). Login con la contraseña
   **anterior** → `401` (Acceptance Scenario 1-2 de US4, FR-010).
2. Mockear `send_email` para fallar en un reenvío → `502`; login con la contraseña **anterior**
   (la de antes de este intento de reenvío) sigue funcionando — no se perdió (research.md
   Decisión 10, FR-012).
3. `POST /invitations/{id}/cancel` sobre una `pending` → `200`, `status='cancelled'`; `GET
   /invitations` ya no la lista; login con su contraseña temporal → `401` (Acceptance Scenario 3 de
   US4, [contracts/invitations-cancel.md](./contracts/invitations-cancel.md)).
4. Cancelar una invitación ya `consumed` (llamar cancel después de un login exitoso) → `409` "ya no
   está pendiente" (research.md Decisión 9).
5. Simular el edge case de concurrencia: iniciar el login (hasta justo antes del `commit`) y, en
   paralelo, cancelar la misma invitación → si la cancelación se compromete primero, el login que
   sigue falla con `401`; si el login se compromete primero, la cancelación posterior ve `409` (Edge
   Case "la cancelación gana", research.md Decisión 7).

```bash
python -m unittest app.characterization_tests.test_invitations_resend_cancel -v
```

## Verificación de no regresión — `POST /users` eliminado y plan de spec 033 intacto

```bash
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**:
- `POST /users` ya no existe (`404`/`405` según cómo FastAPI enrute la ausencia) — verificar con un
  test puntual que golpea esa ruta y confirma que no hay handler ([users-create-removed.md](./contracts/users-create-removed.md)).
- `GET /users`, `PATCH /users/{id}/role`, `PATCH /users/{id}/status` siguen exactamente igual (sin
  tests dedicados hoy, pero sin ningún cambio de código en esos handlers).
- Los tests de plan (spec 033, `enforce_plan_limit` para `mesas`/`cajas`/`productos`/`métodos de
  pago`) siguen en verde — la extensión de conteo (research.md Decisión 5) solo toca la rama
  `"usuarios"` de `RESOURCE_CONFIG`.
- `test_auth_session_revocation.py`, `test_auth_forgot_password.py`, `test_auth_reset_password.py`,
  `test_auth_change_password.py` (spec 031) siguen en verde — esta spec solo agrega una rama nueva
  dentro de `login()` cuando `user is None`, sin tocar el resto de la función.

## Verificación final — SC-001 a SC-006

| Criterio | Cómo se verifica |
|---|---|
| SC-001 (invitar solo con correo+rol, sin inventar contraseña) | US1 paso 1 — el request de `POST /invitations` no tiene campo de contraseña |
| SC-002 (100% de cuentas nuevas se originan de una invitación consumida) | US2 paso 1 — el único camino de creación de `User` fuera de `tenant_create()` es el consumo de invitación; `POST /users` ya no existe |
| SC-003 (primer ingreso en un único intento) | US2 paso 1 completo (login → cuenta creada → `must_change_password=True`) |
| SC-004 (ADMIN distingue pendientes de activas de un vistazo) | US3 completo |
| SC-005 (contraseña anterior deja de servir desde el siguiente intento) | US2 paso 4, US4 pasos 1 y 3 |
| SC-006 (invitar un correo ya usado nunca envía correo) | US1 pasos 2-3 — el mock de `send_email` no recibe llamadas en ninguno de los dos casos |

## Frontend — validación manual (no automatizada en esta guía)

`pos-heladeria` no tiene `.spec.ts` propio para `UsersPageComponent`/`UserFormComponent` hoy — se
valida con `ng serve` + navegación manual, mismo criterio que specs 001/031 ya aplican a este
módulo.

1. Como ADMIN, abrir "Usuarios" → el botón "Agregar usuario" abre un formulario con **exactamente**
   dos controles (correo, rol) — sin campo de contraseña, nombre ni teléfono (FR-001).
2. Enviar una invitación válida → aparece de inmediato en la sección "Invitaciones pendientes" del
   listado, con correo/rol/fecha de envío, visualmente distinta de la sección de usuarios activos
   (FR-009).
3. Como CASHIER, entrar a la URL de "Usuarios" directamente → redirigido/bloqueado por el guard de
   ruta existente (`roleGuard([UserRole.ADMIN])`) — nunca ve el botón ni el listado
   (research.md Decisión 14).
4. Copiar la contraseña temporal del correo (o del log del microservicio de email en dev) → cerrar
   sesión → login con ese correo y esa contraseña en la pantalla de login del tenant → el sistema
   fuerza el cambio de contraseña (pantalla ya existente, sin cambios) → tras fijar una contraseña
   propia, refrescar "Usuarios" como ADMIN → ese correo ya aparece como activo, no como pendiente.
5. Reenviar una invitación pendiente → el correo anterior deja de servir (repetir el paso 4 con la
   contraseña vieja → error de login); la fecha de envío mostrada en el listado se actualiza.
6. Cancelar una invitación pendiente → desaparece de la sección de pendientes; su contraseña
   temporal ya no permite iniciar sesión.
