# Contrato: invitaciones con nombre completo

**Spec**: FR-011, FR-016…FR-019, FR-021, FR-024 | **Reglas del nombre**: [full-name-rules.md](./full-name-rules.md) | Anomalía: A-100

Todos los endpoints exigen ADMIN del negocio (sin cambio). Solo se describe lo que cambia.

## `POST /api/v1/invitations`

**Cuerpo (`InvitationCreate`)**

| Campo | Tipo | Cambio |
|---|---|---|
| `email` | `EmailStr` | sin cambio |
| `role` | `ADMIN \| CASHIER \| MESERO` | sin cambio |
| `name` | `string` | **nuevo, obligatorio**. Se valida y se guarda recortado y en NFC. |

`name` se declara con `default=None` y `validate_default=True` a propósito (research D7): **omitir la clave** y mandar vacío producen el mismo error. En la documentación OpenAPI el campo se describe como obligatorio.

**Respuestas**

- `201` — `InvitationResponse` (abajo). Orden de comprobaciones: **primero** la validación del cuerpo (incluido el nombre) y **después** las existentes (límite del plan, correo repetido, rol). Por tanto un nombre inválido responde 422 y **no** crea la invitación, **no** envía correo y **no** consume cupo del plan (FR-016).
- `422` — nombre ausente, vacío, solo espacios, fuera de 2–100 o con formato inválido:
  ```json
  {"detail":[{"type":"value_error","loc":["body","name"],"msg":"Value error, El nombre es obligatorio","input":"   ","ctx":{"error":{}}}]}
  ```
  El `msg` es uno de los tres textos de [full-name-rules.md](./full-name-rules.md).
- `403`/`404`/`409`/`502` — sin cambio.

**Efectos**: `UserInvitation.name` = nombre normalizado; el correo se arma con ese nombre (abajo). Dos invitaciones con el mismo nombre y correos distintos se aceptan (FR-014 / Historia 4.8).

## `InvitationResponse` (alta, lista y reenvío)

| Campo | Tipo | Cambio |
|---|---|---|
| `id`, `email`, `role_name`, `sent_at` | — | sin cambio |
| `name` | `string \| null` | **nuevo**. `null` en invitaciones anteriores a esta spec. |

## `GET /api/v1/invitations`

Sin cambio de parámetros ni de paginación; cada elemento incluye `name`. La pantalla muestra el nombre junto al correo si existe, o solo el correo si es `null` (FR-021).

## `POST /api/v1/invitations/{id}/resend`

Sin cambio de contrato. El correo se reenvía **con el nombre guardado** (FR-018/FR-019); si la invitación es anterior (`name` nulo), con saludo genérico. El nombre no se puede cambiar al reenviar.

## `POST /api/v1/auth/login` — aceptación de la invitación

Sin cambio de contrato. Al consumirse una invitación pendiente: `User.name = invitation.name or invitation.email`. Una invitación anterior (sin nombre) se acepta exactamente como hoy (FR-024, SC-007).

## Correo de invitación (`invitation_email_body`)

| Caso | Primer párrafo del cuerpo |
|---|---|
| con nombre | `Hola, <nombre-escapado>:` |
| sin nombre (legacy) | `Hola:` |

- El nombre se inserta con `html.escape(nombre, quote=True)`: `O'Brien` → `O&#x27;Brien` (se ve `O'Brien`). Con la lista blanca de D5 es la única diferencia posible, pero el escape se mantiene como segunda capa (FR-017/FR-019).
- Asunto (`Bienvenido a <negocio>`), tabla de datos de acceso, aviso de contraseña temporal y el resto **no cambian**.
- La firma pasa a `invitation_email_body(tenant_name, login_url, email, password, name=None)`; los llamadores sin `name` siguen funcionando.

## Frontend

- Formulario "Invitar usuario": campo **Nombre completo** (primero, antes de correo y rol), validado con `fullNameValidator`; muestra el texto del primer error al tocar o enviar; bloquea el envío mientras sea inválido; manda `name` ya recortado.
- Error de servidor 422: `extractError` lee `detail[0].msg` cuando `detail` es un arreglo y retira el prefijo `Value error, `.
- **Limpieza (FR-023)**: al **Cancelar**, tras un **envío exitoso** y al **abrir** el formulario: `form.reset()` de todos los campos y `invitationsService.error.set(null)`.
- Lista de pendientes: nombre (si hay) como título y correo debajo; con `truncate`/`min-w-0` para que 100 caracteres no rompan la fila.
- Página "Usuarios": título = `displayName`, avatar = `initialOf` (mayúscula). Para usuarios sin nombre propio (vacío o igual al correo) se ve el correo y su inicial (FR-022).
