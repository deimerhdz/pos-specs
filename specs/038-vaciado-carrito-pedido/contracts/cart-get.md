# Contrato: `GET /cart` (US1, US3, US4)

Endpoint ya existente (`app/api/v1/cart/router.py`, `service.get_cart` → `serialize_cart`). **Sin
ningún cambio de forma ni de código** — este contrato documenta explícitamente el efecto
*observable* que el borrado físico de `submit_cart` tiene sobre él, porque es la vía por la que el
frontend comprueba, tras recargar la página, que el carrito quedó vacío (US1, Acceptance Scenario
2 / CA-2).

## Comportamiento (sin cambios de código, efecto distinto por la Decisión 1 de research.md)

```text
GET /cart
Auth: x-session-token (SessionContext)

→ 200 CartResponse { id, participant_id, status: 'abierto', total, discounted_total, items: [] }
```

`get_cart` llama `_get_or_create_open_cart`, que busca una fila `Cart` con
`status == 'abierto'` para el participante y, si no la encuentra, crea una nueva vacía
(`app/api/v1/cart/service.py:182-197`, sin cambios en esta spec).

- **Antes de esta spec**: tras `submit_cart`, la fila anterior queda `status='confirmado'` (no
  `'abierto'`) — `_get_or_create_open_cart` tampoco la encuentra (filtra por `'abierto'`) y crea una
  fila `'abierto'` nueva. La fila `'confirmado'` queda huérfana en la base de datos, sin que
  `GET /cart` la exponga nunca.
- **Después de esta spec**: tras `submit_cart`, no queda ninguna fila del carrito anterior (fue
  eliminada). `_get_or_create_open_cart` tampoco encuentra una `'abierto'` (por la misma razón de
  siempre: no existe) y crea una fila `'abierto'` nueva — **mismo resultado observable** para quien
  llama a `GET /cart`: un carrito vacío, recién creado. La diferencia es interna (no queda fila
  huérfana en la base de datos), no visible en esta respuesta.

**Por qué este contrato no necesita cambiar**: `GET /cart` nunca leyó ni expuso el estado
`'confirmado'` — su comportamiento ya era "solo me importa si hay una fila `'abierto'`". El borrado
físico introducido por esta spec es indistinguible para este endpoint. Esto es exactamente lo que
garantiza US1 (Acceptance Scenario 2): recargar la página tras confirmar muestra un carrito vacío,
antes y después de esta spec — lo que cambia es que ahora es un carrito vacío *sin* una fila
huérfana detrás, no uno vacío con una fila `'confirmado'` sin usar.

## Lo que NO cambia

- Forma de `CartResponse`/`CartItemResponse` — sin campos nuevos ni removidos.
- El comportamiento de "segunda ronda" (US3): agregar un ítem tras confirmar sigue creando, vía
  `_get_or_create_open_cart`/`add_item`, un carrito `'abierto'` nuevo con solo esa línea — sin
  ningún cambio de código, ver [data-model.md](../data-model.md) §Ciclo de vida.
- El aislamiento entre comensales de la misma mesa (US4): `GET /cart` ya filtraba estrictamente por
  `participant_id` — nada en esta spec toca ese filtro.
