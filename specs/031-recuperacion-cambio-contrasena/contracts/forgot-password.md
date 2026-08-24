# Contrato: solicitar enlace de restablecimiento (US1)

Router: `app/api/v1/auth/routes.py`. Sin autenticación. Requiere `x-tenant-host` (mismo criterio
que `POST /auth/login`, `RN-AUTH-05`).

## `POST /auth/forgot-password` — NUEVO

```text
Request:
  ForgotPasswordRequest:
    email: EmailStr (requerido)

Response 200 (caso general — bloqueado o no por el límite, ver abajo):
  { "message": "..." }
```

**Flujo**:
1. Resolver tenant por `x-tenant-host` (400 si falta el header, mismo criterio que `get_tenant`).
2. `enforce_sliding_window` sobre `rl:pwreset:{tenant_id}:{email_normalizado}` (research.md
   Decisión 3). Si excede 3 en 15 min → `429`, **sin** tocar la base de datos ni enviar correo:
   ```text
   Response 429:
     { "detail": "Has pedido demasiados enlaces. Vuelve a intentarlo en unos minutos." }
   ```
3. Si no está bloqueado: buscar `User` con `email=:email AND tenant_id=:tenant.id AND active=true`.
   - **Si existe**: invalidar cualquier `PasswordResetToken` `vigente` de esa cuenta
     (`invalidated_at = now()`), crear una fila nueva (`token_hash`, `email_snapshot=user.email`,
     `issued_at=now()`, `expires_at=issued_at+30min`), despachar
     `send_email_task.delay(...)` con el enlace (`password_reset_email_body`, research.md Decisión
     8), en menos de 2 minutos (SC-002).
   - **Si no existe, o existe pero `active=false`, o existe en otro tenant**: no se crea fila, no
     se envía correo — mismo trato exacto (FR-004).
4. En ambos casos del paso 3 (exista o no la cuenta), responder **el mismo** `200`:
   ```text
   Response 200:
     { "message": "Si existe una cuenta con ese correo, te enviamos un enlace para restablecer tu
       contraseña. Revisa tu bandeja de entrada y la carpeta de spam." }
   ```

**Responses**:
| Código | Caso |
|---|---|
| `200` | Solicitud aceptada (mensaje genérico, exista o no la cuenta) — FR-003. |
| `400` | Falta `x-tenant-host` o no resuelve a ningún tenant. |
| `422` | `email` no tiene forma de correo válida. |
| `429` | Límite de 3/15min superado para ese correo en ese tenant (FR-010). |

**No expone**: en ningún caso (incluida la respuesta `429`) el código revela si la cuenta existe —
el límite se cuenta por correo ingresado, exista o no cuenta detrás (SC-003, Clarification de
`spec.md`).
