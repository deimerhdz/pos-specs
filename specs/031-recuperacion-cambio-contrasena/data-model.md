# Data Model: Recuperación y Cambio de Contraseña (Personal)

Todo lo nuevo o modificado vive en el schema `shared` (`app/core/models.py`), igual que
`Tenant`/`Role`/`User` hoy — no en el schema `tenant` per-tenant. Las decisiones de diseño detrás
de cada elección están en [research.md](./research.md); este documento se limita a columnas,
restricciones y transiciones.

## User (`shared.users`) — MODIFICADA

Entidad ya existente (`app/core/models.py`). Columna nueva:

| Columna | Tipo | Nulable | Notas |
|---|---|---|---|
| `tokens_valid_after` | `DateTime` | Sí | `NULL` por defecto (ninguna cuenta existente se ve afectada hasta su primer cambio de contraseña bajo esta spec). Se fija a `now()` en cada cambio de contraseña exitoso (Flujo A o B, FR-009/FR-017). Cualquier JWT con `iat` anterior a este valor deja de aceptarse (research.md Decisión 1). |

Columnas sin cambio: `id`, `name`, `email`, `password_hash`, `phone`, `active`, `must_change_password`,
`role_id`, `tenant_id`.

**Regla de validación nueva**: en `get_current_user`, `get_authenticated_user`
(`app/core/dependencies.py`) y en el handler de `GET /auth/refresh-token`
(`app/api/v1/auth/routes.py`), después de la relectura por id/tenant y del chequeo `active==True`
ya existentes (`RN-AUTH-07`, protegido A-23, sin cambios): si `user.tokens_valid_after is not None`
y `token_data["iat"] < user.tokens_valid_after.timestamp()`, responder `401` con un detalle
distinguible (p. ej. `"Session revoked due to password change"`) — no el mismo texto que
"revocado por logout" ni "inactivo", para que el frontend pueda distinguir el caso si algún día
quiere mostrarlo.

## PasswordResetToken (`password_reset_tokens`) — NUEVA

| Columna | Tipo | Nulable | Notas |
|---|---|---|---|
| `id` | UUID (PK) | No | `UUIDPrimaryKeyMixin`, igual que el resto de entidades de `shared`. |
| `user_id` | UUID (FK → `shared.users.id`) | No | Indexado. Cuenta a la que pertenece el enlace. |
| `token_hash` | `String(64)` | No | SHA-256 hex del token crudo enviado por correo. Único, indexado. El valor crudo nunca se persiste (research.md Decisión 2). |
| `email_snapshot` | `String(255)` | No | Correo de la cuenta al momento de emitir el enlace. Se compara contra `user.email` al validar — un cambio de correo posterior invalida el enlace en vivo (FR-012, research.md Decisión 4). |
| `issued_at` | `DateTime` | No | `server_default now()`. Momento de emisión — la vigencia de 30 min se cuenta desde aquí, no desde que se abre el correo (FR-011). |
| `expires_at` | `DateTime` | No | `issued_at + PASSWORD_RESET_TOKEN_EXPIRY_MINUTES` (default 30), calculado en la aplicación al insertar. |
| `used_at` | `DateTime` | Sí | `NULL` hasta que el guardado exitoso lo consume (FR-008). |
| `invalidated_at` | `DateTime` | Sí | `NULL` hasta que una solicitud posterior de la misma cuenta lo supera (FR-005). |

**Restricciones** (`__table_args__`, `{"schema": "shared"}`):
- Índice único sobre `token_hash`.
- Índice sobre `user_id` (para la invalidación masiva de FR-005 y el listado por cuenta).

**Estado derivado** (no persistido como columna — se deriva al leer, igual que "pendiente de pago"
en spec 024):

| Estado | Condición |
|---|---|
| `vigente` | `used_at IS NULL AND invalidated_at IS NULL AND now() < expires_at AND email_snapshot == user.email` |
| `usado` | `used_at IS NOT NULL` |
| `invalidado` | `invalidated_at IS NOT NULL`, o (`email_snapshot != user.email` en el momento de validar) |
| `caducado` | ninguna de las anteriores y `now() >= expires_at` |

`vigente`/`usado`/`invalidado` corresponden exactamente al Key Entity "Solicitud de
restablecimiento (enlace)" de `spec.md`. `caducado` es un sub-caso de "invalidado" a efectos de
FR-007 (mismo trato: enlace no usable), separado aquí solo porque el copy de pantalla distingue
"enlace caducado" de "enlace inválido" (Acceptance Scenarios 5 y 7 de US1).

## Transiciones de estado de `PasswordResetToken`

```text
                    ┌──────────┐
  (POST /auth/    ► │ vigente  │
   forgot-password) └────┬─────┘
                         │
        ┌────────────────┼──────────────────┐
        │                │                  │
  se abre y se       pasan 30 min      se pide un enlace
  guarda con éxito   sin abrirse       nuevo de la misma
  (FR-008)           (FR-011)          cuenta (FR-005), o
        │                │             la cuenta cambia de
        ▼                ▼             correo (FR-012)
   ┌─────────┐      ┌───────────┐            │
   │  usado  │      │ caducado  │             ▼
   └─────────┘      └───────────┘      ┌─────────────┐
   (terminal)      (terminal, mismo    │ invalidado  │
                     trato que          └─────────────┘
                     "inválido" en           (terminal)
                     pantalla, FR-007)
```

- Una cuenta puede tener **N** filas de `PasswordResetToken` a lo largo del tiempo, pero a lo sumo
  **una** `vigente` en cualquier instante: pedir un enlace nuevo invalida de inmediato cualquier
  enlace anterior no usado y no caducado de esa cuenta (FR-005) — no hace falta un índice único
  parcial (a diferencia de `OrderPaymentAttempt` en spec 024) porque la invalidación es explícita
  en el momento de crear la fila nueva (`UPDATE ... SET invalidated_at = now() WHERE user_id = :uid
  AND used_at IS NULL AND invalidated_at IS NULL`), no una condición de carrera entre dos
  solicitudes independientes del usuario.
- Ninguna transición vuelve a `vigente` — `usado`, `caducado` e `invalidado` son terminales.

## Reglas de validación (resumen por historia de usuario)

| Regla | Dónde se aplica | Historia |
|---|---|---|
| Límite de 3 solicitudes / 15 min por correo ingresado, ventana deslizante, dentro del tenant | `enforce_sliding_window` sobre Redis, antes de tocar la BD (`POST /auth/forgot-password`) | US1 |
| Mensaje genérico idéntico exista o no la cuenta | Servicio — mismo código de respuesta y mensaje en ambos casos, salvo bloqueo por límite | US1 |
| Cuenta debe existir, estar activa, y en el tenant de `x-tenant-host` | `SELECT ... WHERE email=:email AND tenant_id=:tenant_id AND active=true` (mismo criterio que login, `RN-AUTH-04`/`RN-AUTH-05`) | US1 |
| Vigencia de 30 min contada desde la emisión | `expires_at = issued_at + 30min`, comparado contra `now()` al validar, nunca contra la apertura del correo | US1 |
| Enlace nuevo invalida el anterior | `UPDATE` de superseding antes de insertar la fila nueva | US1 |
| Enlace inválido/caducado/usado → mismo mensaje de error + botón "Pedir un enlace nuevo" | `GET /auth/reset-password/validate` | US1 |
| Doble consumo del mismo enlace no aplica un segundo cambio | `WITH FOR UPDATE` sobre la fila del token en `POST /auth/reset-password` | US1 |
| `must_change_password → False` también en el Flujo A | `POST /auth/reset-password`, mismo criterio que `RN-AUTH-02` | US1 |
| Cierre de sesiones tras éxito (todas en A, todas-menos-origen en B) | `user.tokens_valid_after = now()` + relectura en `get_current_user`/`get_authenticated_user`/refresh (research.md Decisión 1) | US1, US2 |
| Contraseña actual correcta antes de cambiar (Flujo B) | `verify_password(current_password, user.password_hash)`, sin cambios (`RN-AUTH-01`) | US2 |
| Contraseña 8-12 caracteres, sin exigir combinaciones | `Field(min_length=8, max_length=12)` en `ChangePasswordRequest`/`ResetPasswordRequest` | US1, US2 |
| Nueva contraseña ≠ actual | `verify_password(new_password, user.password_hash)` debe ser `False` | US1, US2 |
| Cambio de contraseña no toca otros datos del perfil | El endpoint solo escribe `password_hash`/`must_change_password`/`tokens_valid_after` — ningún otro campo de `User` | US2 |
| Correo de aviso tras cualquier cambio exitoso | `send_email_task.delay(...)` con `password_changed_email_body`, envuelto en `try/except` (FR-028) | US3 |
| Falla de envío no bloquea ni expone el error al usuario | `try/except` alrededor de cada `send_email_task.delay(...)`, solo logueado | US1, US2, US3 |
