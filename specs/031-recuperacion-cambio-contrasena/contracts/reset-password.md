# Contrato: validar y consumir el enlace de restablecimiento (US1)

Router: `app/api/v1/auth/routes.py`. Sin autenticación — el token en sí es la credencial.

## `GET /auth/reset-password/validate` — NUEVO

Se llama al abrir el enlace, **sin efecto secundario** (no consume el token) — permite que la
pantalla decida qué mostrar antes de pedir la contraseña nueva (FR-007), y que abrir el mismo
enlace dos veces sin guardar no lo invalide por sí solo.

```text
Request (query param):
  ?token=<string>

Response 200:
  { "valid": true }

Response 400/404:
  { "valid": false, "reason": "expired" | "invalid" | "used" }
```

- `reason="expired"`: `now() >= expires_at` y no está usado/invalidado — pantalla de "enlace
  caducado" (Acceptance Scenario 5 de US1).
- `reason="used"`: `used_at IS NOT NULL` — pantalla de "enlace ya usado" (Acceptance Scenario 7).
- `reason="invalid"`: token inexistente, `invalidated_at IS NOT NULL`, o `email_snapshot !=
  user.email` (FR-012) — mismo trato que "usado" a nivel de pantalla (botón "Pedir un enlace
  nuevo"), solo cambia el texto.

## `POST /auth/reset-password` — NUEVO

```text
Request:
  ResetPasswordRequest:
    token: str (requerido)
    new_password: str (requerido, 8-12 caracteres — FR-019)

Response 200:
  { "message": "Contraseña actualizada correctamente." }
```

**Flujo** (dentro de una transacción, `SELECT ... WHERE token_hash=:hash WITH FOR UPDATE`,
research.md Decisión 5):
1. Re-validar el token con las mismas reglas de `GET .../validate` (nunca confiar en una
   validación previa del cliente) → si no es `vigente`, `400`/`404` con el mismo `reason` de
   arriba, sin aplicar ningún cambio.
2. `new_password` debe tener 8-12 caracteres (`422` si no, validado por el schema antes de llegar
   aquí) y ser distinta de `user.password_hash` actual (`verify_password(new_password,
   user.password_hash)` debe ser `False`) → `400` si es igual a la actual (FR-021).
3. Si todo pasa: `user.password_hash = generate_passwd_hash(new_password)`,
   `user.must_change_password = False` (FR-009, mismo criterio que `RN-AUTH-02`),
   `user.tokens_valid_after = now()` (cierra **todas** las sesiones de la cuenta, FR-009 —
   research.md Decisión 1). Marcar `used_at = now()` en la fila del token. `db.commit()`.
4. Despachar `send_email_task.delay(...)` con `password_changed_email_body(...)` (FR-022),
   envuelto en `try/except` (FR-028).
5. Responder `200` con el mensaje de éxito — el frontend redirige a `/login` (FR-009).

**Responses**:
| Código | Caso |
|---|---|
| `200` | Contraseña actualizada; enlace consumido. |
| `400` | Token inválido/caducado/usado (con `reason`, mismo formato que `.../validate`), o `new_password` igual a la actual (FR-021). |
| `422` | `new_password` fuera de 8-12 caracteres. |

**Doble confirmación** (Edge Case — doble clic, reintento de red, dos pestañas con el mismo
enlace): la segunda petición concurrente encuentra la fila ya bloqueada/`used_at` ya escrito por la
primera y responde `400 reason="used"` — nunca aplica un segundo cambio ni un error distinto
(FR-008).
