# Contrato: `GET /invitations`

Lista las invitaciones **pendientes** del tenant, paginadas — mismo patrón `Page[T]` que
`GET /users`.

## Autenticación y alcance

- Requiere `ADMIN` del tenant (`require_tenant_admin`).
- Solo devuelve invitaciones del tenant del ADMIN autenticado (FR-013) — nunca de otro tenant, ni
  siquiera para el mismo correo (FR-014).

## Request

Query params (idénticos a `GET /users`):

| Param | Default | Reglas |
|---|---|---|
| `page` | `1` | `>= 1` |
| `size` | `20` | `1..100` |

## Respuesta

`200` con `Page[InvitationResponse]`:

```json
{
  "items": [
    { "id": "5f2c...", "email": "cajero1@acme.com", "role_name": "CASHIER", "sent_at": "2026-08-25T14:00:00Z" }
  ],
  "total": 1,
  "page": 1,
  "size": 20,
  "pages": 1
}
```

Solo incluye filas con `status='pending'` — una invitación consumida o cancelada desaparece de
este listado de inmediato (US3, Acceptance Scenario 2). El historial de invitaciones ya
consumidas/canceladas no se expone (no lo pide ninguna FR; fuera de alcance).

## Errores

| Código | Cuándo |
|---|---|
| `401` | No autenticado. |
| `403` | No es ADMIN del tenant. |
