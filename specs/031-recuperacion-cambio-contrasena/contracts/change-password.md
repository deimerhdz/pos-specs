# Contrato: cambio de contraseña autenticado (US2) — MODIFICADO

Router: `app/api/v1/auth/routes.py`. Auth: `Depends(get_authenticated_user)` (sin cambio — ya así
hoy, spec 001).

## `POST /auth/change-password` — MODIFICADO

Ya existe (`routes.py:92-108`, `RN-AUTH-01`/`RN-AUTH-02` de spec 001, sin reabrirse). Cambia el
schema de entrada y gana efectos secundarios nuevos tras el éxito.

```text
Request (ChangePasswordRequest):
  current_password: str (requerido — sin cambio)
  new_password: str (CAMBIA de 6-128 a 8-12 caracteres — FR-019, spec.md "Cambio de
                      comportamiento explícito #1")

Response 200 (sin cambio de forma):
  { "message": "Password changed successfully" }
```

**Flujo** (pasos 1-2 sin cambio; 3-5 nuevos):
1. `verify_password(current_password, user.password_hash)` — si falla, `400 "Current password is
   incorrect"`, sin tocar nada (`RN-AUTH-01`, sin cambios).
2. `new_password` debe tener 8-12 caracteres (validado por el schema, `422` si no — FR-019).
3. **NUEVO** — `new_password` debe ser distinta de la actual: `verify_password(new_password,
   user.password_hash)` debe ser `False` → `400` si es igual (FR-021).
4. `user.password_hash = generate_passwd_hash(new_password)`, `user.must_change_password = False`
   (sin cambio, `RN-AUTH-02`). **NUEVO**: `user.tokens_valid_after = now()` — cierra todas las
   sesiones de la cuenta **excepto** la que originó el cambio (FR-017, research.md Decisión 1: la
   sesión de origen sobrevive porque el frontend vuelve a loguearse con la contraseña nueva
   inmediatamente después, obteniendo tokens acuñados después del corte — no porque el backend
   excluya un jti). `db.commit()`.
5. **NUEVO** — despachar `send_email_task.delay(...)` con `password_changed_email_body(...)`
   (FR-022), envuelto en `try/except` (FR-028).

**Responses**:
| Código | Caso |
|---|---|
| `200` | Cambiada. |
| `400` | `current_password` incorrecta, o `new_password` igual a la actual (FR-021). |
| `401` | Sin autenticar / token inválido o expirado (sin cambio). |
| `422` | `new_password` fuera de 8-12 caracteres (antes 6-128). |

## Efecto lateral cruzado: nuevo caso `401` en todo endpoint protegido

No es un endpoint nuevo — es un caso adicional que gana `get_current_user`/`get_authenticated_user`
(`app/core/dependencies.py`) y `GET /auth/refresh-token`, aplicable a **cualquier** endpoint
protegido del backend, no solo `auth`: un JWT emitido antes de `user.tokens_valid_after` responde
`401` (detalle distinguible de "revocado por logout" y de "inactivo") en vez del recurso pedido.
Ver [data-model.md](./../data-model.md#user-sharedusers--modificada) y research.md Decisión 1. No
altera el caso ya protegido de `RN-AUTH-07`/A-23 (cuenta desactivada) — se evalúa **después** de
ese chequeo, nunca en su lugar.
