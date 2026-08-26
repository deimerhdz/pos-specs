# Contrato: efecto secundario sobre los 4 endpoints y el job que liberan una mesa

Ninguno de los endpoints involucrados cambia de **forma** — mismo método, misma ruta, mismo body,
mismo `response_model`, mismos códigos de estado ya existentes. Lo que cambia es un **efecto
secundario** interno, ya presente hoy (la mesa se libera), al que esta spec le agrega una
consecuencia (los carritos de la sesión cerrada dejan de existir). Ningún payload de request o
response gana ni pierde un campo — no hay nada nuevo que `pos-heladeria` deba leer, y por eso este
repo no requiere ningún cambio en el frontend.

## 1. `POST /cart/leave` (US1)

`app/api/v1/cart/router.py:109-130`, `service.leave_session` → `try_release_if_empty`
(`table_sessions/service.py:88-130`).

```text
Body: ninguno (auth vía header x-session-token)                                  # SIN CAMBIO
→ 204 No Content                                                                  # SIN CAMBIO

Antes: si era el último comensal activo y no queda nada por cobrar, la mesa pasa a 'libre'; los
       carritos de los participantes de esa sesión (el propio y los de quienes ya se habían ido)
       quedan en su status actual ('abandonado' o 'confirmado') para siempre.

Ahora: mismo disparador, misma condición de liberación (SIN CAMBIO, FR-005). Además, en la misma
       transacción, TODOS los `Cart` de los participantes de la sesión que se cerró se ELIMINAN
       físicamente (FR-001/FR-002), sin importar su status. Si el comensal que se va NO era el
       último, o queda algo por cobrar, la mesa sigue 'ocupada' y no se borra nada — comportamiento
       idéntico al de hoy (FR-003).
```

Mismo endpoint cubre, indirectamente, `cancel_my_order` (`POST /cart/orders/{id}/cancel` →
`try_release_if_empty` también) — mismo efecto, mismo disparador compartido.

## 2. `POST /table-sessions/{table_session_id}/close` (US2)

`app/api/v1/table_sessions/router.py:121-136`, `service.close_session`
(`table_sessions/service.py:~193-260`).

```text
Body: CloseSessionIn { billing_mode, cash_shift_id, ... }                        # SIN CAMBIO
→ 200 CloseSessionResponse { table_session, sale_ids }                            # SIN CAMBIO

Antes: cobra, marca los pedidos 'pagada', cierra a los comensales (carritos → 'abandonado'), libera
       la mesa. Los carritos, incluidos los huérfanos de comensales ya cerrados antes de este
       cobro, sobreviven para siempre.

Ahora: mismo cobro, misma validación (`_assert_closable`, SIN CAMBIO). Al liberar la mesa, se
       eliminan físicamente todos los `Cart` de los participantes de la(s) sesión(es) cerradas en
       esta operación (FR-001), en la misma transacción que el cobro (FR-002) — si el cobro falla y
       hace rollback, ningún carrito queda tocado.
```

## 3. `POST /table-sessions/{table_session_id}/release` (US2)

`app/api/v1/table_sessions/router.py:142-155`, `service.release_paid_session`
(`table_sessions/service.py:~335-398`).

```text
Body: ninguno                                                                     # SIN CAMBIO
→ 200 ReleaseSessionResponse { dining_table_id, status }                          # SIN CAMBIO

Antes: rechaza con 409 si queda algo por cobrar; si no, cierra la sesión y libera la mesa. Carritos
       huérfanos de la sesión sobreviven.

Ahora: misma validación 409 (SIN CAMBIO). Al liberar, se eliminan los `Cart` de los participantes
       de la sesión cerrada, misma transacción (FR-001/FR-002).
```

## 4. `POST /orders/tables/{table_id}/release` ("Liberar Mesa", US2)

`app/api/v1/orders/checkout.py:697-736`, `checkout.release_table`.

```text
Body: ninguno                                                                     # SIN CAMBIO
→ 200 DiningTable                                                                 # SIN CAMBIO
→ 409 si hay órdenes no terminales de la mesa                                     # SIN CAMBIO

Antes: 409 con el detalle de las órdenes bloqueantes si hay alguna no terminal; si no, libera la
       mesa y cierra en cascada sus sesiones 'active' (carritos → 'abandonado', sin borrar).

Ahora: misma regla de bloqueo por órdenes activas (SIN CAMBIO, FR-005). Al liberar, se eliminan los
       `Cart` de los participantes de las sesiones cerradas por esta llamada — incluye el caso de
       "dos `TableSession` activas sobre la misma mesa cerrándose en la misma operación" (edge case
       de spec.md): se borran los carritos de participantes de AMBAS.
```

## 5. Barrido automático (`_sweep_schema`, sin endpoint HTTP — job periódico)

`app/core/scheduler.py:88-165`, invocado desde el `lifespan` de `app/main.py` y, a demanda, desde
`python -m app.scripts.sweep_sessions`.

```text
Sin request/response — job interno, sin contrato HTTP.

Antes: dos ramas según RN-SCHED-01..04 (spec 010):
  (a) sesión vencida CON algo por cobrar → cierra solo a los comensales (`close_participants`),
      la mesa NO se libera, carritos → 'abandonado', sin borrar.
  (b) sesión vencida SIN nada por cobrar → cierra la sesión completa (`close_table_sessions`); si
      además no queda ningún `CustomerOrder` huérfano de la misma mesa física, la mesa pasa a
      'libre'; si sí queda uno (RN-SCHED-04), la sesión se cierra pero la mesa sigue 'ocupada'.
      En ambos casos, los carritos sobreviven.

Ahora: mismas dos ramas, mismas condiciones (SIN CAMBIO, FR-005). En la rama (a), sigue sin
       borrarse ningún carrito (US3, escenario 1) — `delete_orphan_carts` ni se invoca, porque esa
       rama nunca llama `close_table_sessions`. En la rama (b), `delete_orphan_carts` se invoca
       SOLO si la mesa efectivamente queda 'libre' (US2, escenario 3); si queda el `CustomerOrder`
       huérfano y la mesa sigue 'ocupada' (RN-SCHED-04), ningún carrito se toca (US3, escenario 2).
```

## Lo que NO cambia en ningún contrato

- Ningún schema Pydantic (`CloseSessionResponse`, `ReleaseSessionResponse`, `DiningTable` response,
  `CartResponse`) gana, pierde o renombra un campo.
- Ningún código de estado HTTP nuevo ni existente cambia de condición de disparo.
- `pos-heladeria` no requiere ningún cambio: no consume ningún carrito por su `id` después de que
  su mesa se libera (research.md Decisión 4) — la próxima vez que ese participante (si volviera a
  operar, lo cual no debería ocurrir tras cerrar su sesión) necesite un carrito, `_get_or_create_
  open_cart` le abre uno nuevo, exactamente igual que si el anterior siguiera existiendo con
  status `'abandonado'`/`'confirmado'`.
