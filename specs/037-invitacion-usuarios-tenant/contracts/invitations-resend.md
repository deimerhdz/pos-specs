# Contrato: `POST /invitations/{invitation_id}/resend`

Reenvía una invitación pendiente: genera una contraseña temporal nueva, invalida de inmediato la
anterior, actualiza la fecha de envío y despacha un nuevo correo (FR-010).

## Autenticación y alcance

- Requiere `ADMIN` del tenant; la invitación debe pertenecer a ese tenant (`404` si no, mismo
  criterio que `GET /users/{user_id}` con un id de otro tenant).

## Request

Sin cuerpo. `invitation_id` en la ruta.

## Respuestas

| Código | Cuándo | Cuerpo |
|---|---|---|
| `200` | Reenviado con éxito. | `InvitationResponse` con `sent_at` actualizado. |
| `401` | No autenticado. | — |
| `403` | No es ADMIN del tenant. | — |
| `404` | La invitación no existe en el tenant del ADMIN. | `{"detail": "Invitación no encontrada"}` |
| `409` | La invitación ya no está `pending` (fue consumida o cancelada, incluida una carrera contra un login concurrente — research.md Decisión 9). | `{"detail": "La invitación ya no está pendiente"}` |
| `502` | El envío del correo falló — la contraseña **anterior** sigue vigente sin cambios (research.md Decisión 10, FR-012). | `{"detail": "No se pudo enviar el correo de invitación. Intenta de nuevo."}` |

## Efectos secundarios

1. Genera una contraseña temporal nueva (`generate_random_password`) y su hash.
2. Envía el correo de forma **síncrona**, con los mismos tres datos de la invitación original
   (enlace, correo, contraseña temporal nueva).
3. Solo si el envío tiene éxito: `UPDATE` de `password_hash` y `sent_at` sobre la misma fila —
   `id`, `status` y `email` no cambian.
4. La contraseña anterior deja de servir de inmediato (Acceptance Scenario 2 de US4): cualquier
   intento de login posterior con ella ya no coincide con `password_hash`.
