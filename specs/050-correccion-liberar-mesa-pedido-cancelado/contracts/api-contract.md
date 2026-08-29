# Contrato de API: `POST /table-sessions/{table_session_id}/release`

No hay cambio de firma ni de forma de request/response — este documento fija el contrato de
**comportamiento** (qué combinaciones de estado responden 200 vs 409) para que `/speckit-tasks`
pueda derivar tareas y tests verificables sin releer `research.md`/backend completo.

## Endpoint

`POST /api/v1/table-sessions/{table_session_id}/release` (`release_paid_session`,
`pos-backend/app/api/v1/table_sessions/service.py:337-370`; router:
`table_sessions/router.py:143-157`)

Sin cambios en el schema de request (sin body) ni de response (`ReleaseSessionResponse`).

## Tabla de comportamiento (antes → después)

| Escenario | Antes | Después |
|---|---|---|
| Sesión con un único pedido `'cancelada'`, con un ítem `'pendiente'` o `'en_preparacion'` | 409 "Hay ítems sin terminar en cocina..." (bug) | 200 — libera la mesa |
| Sesión con un único pedido `'cancelada'`, con todos sus ítems `'listo'` o `'anulado'` | 200 (ya funcionaba) | 200 (sin cambio) |
| Sesión con un pedido `'pagada'` cuya cocina sigue en curso (`'pendiente'`/`'en_preparacion'`) | 409 (correcto) | 409 (sin cambio — comportamiento protegido) |
| Sesión con un pedido `'cancelada'` (cocina sin terminar) **y** otro pedido `'abierta'` billable | 409 "Todavía hay algo por cobrar..." (por `has_billable_orders`, ya correcto) | 409, mismo motivo (sin cambio) |
| Sesión sin ningún pedido, o todos `'pagada'`/`'cancelada'` con cocina ya terminada | 200 (ya funcionaba) | 200 (sin cambio) |

## `ToastService` (contrato interno, sin API HTTP)

| Miembro | Contrato nuevo |
|---|---|
| `push(kind, text, ms)` (privado) | Si `this.toasts()` ya contiene una entrada con el mismo `kind` y `text`, no agrega ninguna — retorna sin efecto. En caso contrario, agrega como hoy. |
| `success`/`error`/`info` (públicos) | Sin cambio de firma ni de comportamiento observable más allá de heredar la deduplicación de `push`. |
