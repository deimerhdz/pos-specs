# Contrato: `POST /invitations/{invitation_id}/cancel`

Cancela una invitación pendiente: su contraseña temporal deja de servir de inmediato y desaparece
del listado de pendientes (FR-011).

## Autenticación y alcance

- Requiere `ADMIN` del tenant; la invitación debe pertenecer a ese tenant.

## Request

Sin cuerpo. `invitation_id` en la ruta.

## Respuestas

| Código | Cuándo | Cuerpo |
|---|---|---|
| `200` | Cancelada con éxito. | `InvitationResponse` (con el estado ya cambiado — el frontend simplemente la retira de la lista de pendientes tras este `200`, mismo criterio que `toggleActive` hoy recarga la lista de usuarios). |
| `401` | No autenticado. | — |
| `403` | No es ADMIN del tenant. | — |
| `404` | La invitación no existe en el tenant del ADMIN. | `{"detail": "Invitación no encontrada"}` |
| `409` | La invitación ya no está `pending` (ya fue consumida o cancelada antes). | `{"detail": "La invitación ya no está pendiente"}` |

## Efectos secundarios

1. `UPDATE user_invitations SET status='cancelled', cancelled_at=now() WHERE id=... AND
   status='pending'` (con `WITH FOR UPDATE` previo, research.md Decisión 9).
2. No se envía ningún correo.
3. Un login concurrente que ya adquirió el lock de la fila antes de este `UPDATE` termina de
   consumirla con normalidad (la cancelación llega tarde y ve `409`, "ya no está pendiente") — si en
   cambio la cancelación adquiere el lock primero, el login que llega después ve la invitación como
   inexistente/no-`pending` y falla con `401 "Invalid credentials"` (research.md Decisión 7, Edge
   Case "la cancelación gana").
