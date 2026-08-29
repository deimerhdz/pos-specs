# Data Model: Corrección — "Liberar Mesa" bloqueada por un pedido ya cancelado

Esta spec **no agrega ni modifica entidades** (ni de backend ni de frontend) — es una corrección de
qué subconjunto de datos ya existentes se usa en dos validaciones ya existentes. Lo que sigue
documenta ese contrato, que es lo que realmente cambia.

## `CustomerOrder.status` — qué función excluye qué, antes y después

| Función | Excluye `'cancelada'` | Excluye `'pagada'` | Uso |
|---|---|---|---|
| `_billable_orders` (`table_sessions/service.py:141-157`) | Sí (ya hoy) | Sí (ya hoy) | Qué pedidos hay que cobrar (`close_session`, `compute_bill`) |
| `has_billable_orders` (`service.py:65-77`) | Sí (ya hoy) | Sí (ya hoy) | ¿Queda algo por cobrar? (`release_paid_session`, barrido automático) |
| `_assert_closable` (`service.py:217-241`) | N/A — recibe la lista ya filtrada por quien la llama, no filtra ella misma | N/A | ¿Hay cocina en curso o pedidos sin confirmar en la lista recibida? |
| Query de `release_paid_session` (`service.py:366-369`) | **No, hoy** → **Sí, tras esta spec** | No (intencional — necesita seguir viendo cocina en curso sobre pedidos `'pagada'`) | Arma la lista que se le pasa a `_assert_closable` antes de liberar |

La única celda que cambia es la última: `release_paid_session` pasa a excluir `'cancelada'` de su
query, igualándose al criterio que `_billable_orders`/`has_billable_orders` ya aplican, sin tocar
su intención deliberada de seguir incluyendo `'pagada'` (documentada en su propio docstring).

## `Toast` (`pos-heladeria/src/app/shared/feedback/toast.service.ts`)

Entidad de UI ya existente, sin cambios de forma (`{id, kind, text}`). Lo que cambia es la regla de
inserción en `push()`:

| Antes | Después |
|---|---|
| Cada llamada agrega una entrada nueva a `toasts` (signal), sin mirar las existentes | Antes de agregar, si ya existe una entrada con el mismo `kind` y el mismo `text`, no se agrega una copia |

No hay persistencia (`toasts` vive solo en memoria del cliente, por pestaña) — sin migración ni
estrategia de rollback que documentar (Principio VIII, N/A).
