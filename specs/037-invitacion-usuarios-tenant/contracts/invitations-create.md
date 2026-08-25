# Contrato: `POST /invitations`

Crea una invitación pendiente. Reemplaza a `POST /users` (eliminado, ver
[users-create-removed.md](./users-create-removed.md)) como único mecanismo para dar de alta un
usuario interno del tenant.

## Autenticación y alcance

- Requiere `Authorization` (access token) de un `User` con rol `ADMIN` del tenant
  (`require_tenant_admin`, ya existente).
- Requiere header `x-tenant-host` (`get_tenant`, ya existente) — la invitación se crea en el tenant
  del ADMIN autenticado, nunca en otro (FR-013).

## Request

```json
{
  "email": "cajero1@acme.com",
  "role": "CASHIER"
}
```

| Campo | Tipo | Reglas |
|---|---|---|
| `email` | string (email válido) | Se normaliza (recorte + minúsculas) antes de cualquier comparación o escritura (FR-016). |
| `role` | `"ADMIN"` \| `"CASHIER"` | Mismo enum `RoleName` que `UserRoleUpdate` ya usa. |

Sin campo de contraseña — no existe en este contrato (FR-001, FR-006).

## Respuestas

| Código | Cuándo | Cuerpo |
|---|---|---|
| `201` | Invitación creada y correo despachado con éxito. | `InvitationResponse` (ver abajo). |
| `401` | No autenticado o token inválido. | — |
| `403` | El usuario autenticado no es ADMIN del tenant. | — |
| `403` | El plan del tenant venció (`ensure_plan_not_expired`, FR-018) o alcanzó su límite de usuarios contando invitaciones `pending` (FR-005/006/007 de spec 033, extendido — research.md Decisión 5). | `{"detail": "..."}`, mismo texto que `enforce_plan_limit` ya usa hoy para otros recursos. |
| `409` | Ya existe un `User` (activo o inactivo) con ese correo en el tenant (FR-015). | `{"detail": "Ya existe un usuario con ese correo en el tenant"}` |
| `409` | Ya existe una invitación `pending` con ese correo en el tenant — incluye el caso de dos ADMIN invitando casi simultáneamente (índice único parcial, research.md Decisión 3). | `{"detail": "Ya existe una invitación pendiente para ese correo"}` |
| `404` | El rol indicado no existe (no debería ocurrir con los roles seed, mismo trato que `create_user` legado). | `{"detail": "Role '...' not found"}` |
| `422` | Datos de entrada inválidos (`email` mal formado, `role` fuera del enum). | Validación estándar de Pydantic. |
| `502` | El envío del correo falló (`send_email()` lanzó `RuntimeError`) — la invitación **no** queda persistida (`rollback()`), FR-012. | `{"detail": "No se pudo enviar el correo de invitación. Intenta de nuevo."}` |

### `InvitationResponse`

```json
{
  "id": "5f2c...",
  "email": "cajero1@acme.com",
  "role_name": "CASHIER",
  "sent_at": "2026-08-25T14:00:00Z"
}
```

Nunca incluye `password_hash` ni ningún derivado de la contraseña temporal (FR-006).

## Efectos secundarios

1. `INSERT` en `UserInvitation` (`status='pending'`).
2. Correo enviado de forma **síncrona** (no Celery) al correo invitado, con el enlace de login del
   tenant, el correo como identificador y la contraseña temporal (mismos tres datos que
   `welcome_email_body`, FR-005).
3. Ningún `INSERT`/`UPDATE` sobre `User`.
