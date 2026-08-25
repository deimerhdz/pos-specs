# Contrato: `POST /auth/login` (MODIFICADO — consumo de invitación)

El contrato externo de `POST /auth/login` (spec 001, `RN-AUTH-03/04/05`) **no cambia**: mismo
request (`email`, `password`), mismo header opcional `x-tenant-host`, misma forma de respuesta
(`access_token`, `refresh_token`, `user`). Lo que cambia es qué pasa **antes** de rechazar con `401`
cuando no existe un `User` con ese correo en el tenant.

## Comportamiento nuevo (FR-007)

Cuando la búsqueda normal de `User` no encuentra ninguna fila **y** el request llegó con un
`x-tenant-host` que resolvió a un tenant real:

1. Se busca una `UserInvitation` con `status='pending'`, `tenant_id` de ese tenant, y el correo
   normalizado igual al enviado — bloqueada con `WITH FOR UPDATE` (research.md Decisión 7).
2. Si no hay ninguna, o si `verify_password(body.password, invitation.password_hash)` es `False`:
   sigue el mismo `401 "Invalid credentials"` de siempre — indistinguible desde afuera de un
   usuario inexistente o una contraseña equivocada (no revela si había una invitación).
3. Si coincide:
   - Se crea el `User` (valores en [data-model.md](../data-model.md#user-shareduser--sin-cambios-de-esquema)),
     con `must_change_password=True`.
   - Se marca la invitación `status='consumed'`, `consumed_at=now()`.
   - Se hace `commit()`.
   - El login **continúa exactamente igual** que con cualquier otro usuario recién leído: mismo
     `user_data`, mismos `access_token`/`refresh_token`, mismo `200`.

El resto de `login()` (resolución de tenant por host, verificación `active==True`, forma del JWT,
manejo de errores de conexión) no se modifica.

## Ejemplo — primer ingreso exitoso

**Request**:
```json
POST /auth/login
x-tenant-host: acme.skeilopos.com

{ "email": "cajero1@acme.com", "password": "Tg7!kP9qXvWz" }
```

**Respuesta** (`200`, idéntica en forma a un login normal):
```json
{
  "message": "Login successful",
  "access_token": "...",
  "refresh_token": "...",
  "user": {
    "email": "cajero1@acme.com",
    "uid": "5f2c...",
    "tenant_id": 12,
    "is_super_admin": false,
    "role": "CASHIER",
    "must_change_password": true
  }
}
```

`must_change_password: true` dispara el flujo ya existente (spec 001/031) que obliga a fijar una
contraseña propia antes de continuar — sin cambios en ese flujo.

## Casos que siguen en `401`, sin crear ninguna cuenta

- Invitación inexistente para ese correo+tenant.
- Invitación `cancelled` o `consumed` (ya usada, o cancelada antes o durante este intento —
  research.md Decisión 7/9).
- Contraseña que no coincide con `invitation.password_hash` vigente (p. ej. la contraseña original
  de una invitación ya reenviada — Acceptance Scenario 4 de US2).
- Sin `x-tenant-host` resuelto a un tenant (login de super admin): nunca se busca invitación.
