# Contrato eliminado: `POST /users`

FR-004 exige eliminar por completo — interfaz y backend — el mecanismo que permite a un ADMIN crear
un usuario proporcionando directamente su contraseña. Este documento registra qué se elimina, para
trazabilidad (Principio XII), no describe un contrato vigente.

## Lo que existía antes de esta spec

`POST /users` (`app/api/v1/users/router.py::create_user`), con body `UserCreate`:

```json
{
  "name": "Cajero 1",
  "email": "cajero1@acme.com",
  "password": "secret123",
  "phone": "3001234567",
  "role": "CASHIER"
}
```

Creaba un `User` directamente, con la contraseña en texto plano provista por el ADMIN, hasheada con
`generate_passwd_hash`.

## Lo que reemplaza este flujo

`POST /invitations` ([invitations-create.md](./invitations-create.md)) — mismo rol, tenant y
alcance de autorización (`require_tenant_admin`), pero sin campo de contraseña ni de nombre/teléfono
en el momento de la invitación (ver [data-model.md](../data-model.md), Decisión 8 de research.md
sobre por qué `name` deja de capturarse).

## Qué se elimina exactamente

- El handler `create_user()` en `app/api/v1/users/router.py`.
- El schema `UserCreate` en `app/api/v1/users/schemas.py`.
- El uso de `generate_passwd_hash` en ese router (si queda sin otro caller tras la eliminación).
- En `pos-heladeria`: `UserFormComponent` (formulario con nombre/correo/teléfono/contraseña/rol) y
  el método `UsersService.createUser()`/`TenantUserCreatePayload` que lo invocaban — reemplazados
  por un formulario de invitación nuevo (ver plan.md, Project Structure).

## Qué NO se elimina

- `GET /users`, `GET /users/{user_id}`, `PATCH /users/{user_id}/role`,
  `PATCH /users/{user_id}/status` — CRUD sin relación con la creación por contraseña, sin cambios.
- No existe ningún characterization test (`"CONGELA comportamiento actual:"`) sobre
  `app/api/v1/users` hoy (confirmado) — no hay ningún test protegido que esta eliminación deba
  actualizar o justificar ante el Principio III.
