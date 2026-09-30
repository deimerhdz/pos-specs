# Contrato — Cierre de sesión de mesa y pantalla de gracias

**Historia**: 4 · **FR**: 014–019, 015a–c, 017a · **Decisiones**: research D8–D12, D9b · **Decisión de negocio**: A-95

## 1. Evento SSE `session.closed` (existente; se completa su emisión)

Canales: `staff` **y** `session:{table_session_id}` (ya así). Payload:

```json
{ "table_session_id": "<uuid>", "dining_table_id": "<uuid>", "reason": "paid|swept|released|empty" }
```
`reason` gana el valor `empty` (cierre automático de una sesión sin órdenes). El cliente lo ignora: cualquier `reason` produce la misma pantalla.

### Matriz de emisión (después del `commit`, nunca antes)

| Camino que cierra la sesión | Punto de código | Hoy | Tras la spec |
|---|---|---|---|
| Cobro completo / cierre con `billing_mode` | `table_sessions/service.py::close_session` | ✅ `paid` | ✅ igual |
| Sesión ya pagada, liberar | `table_sessions/service.py::release_paid_session` | ✅ `paid` | ✅ igual |
| Barrido del scheduler (sin pedidos por cobrar) | `core/scheduler.py::_sweep_schema` | ✅ `swept` | ✅ igual |
| **Botón "Liberar mesa"** (cajero o mesero) | `orders/router.py::release_table` | ❌ | ✅ `released` |
| **Cierre automático de sesión vacía** | `try_release_if_empty` ← `cart/service.py` (×2) y `core/qr_context.py::_abandon_expired` | ❌ | ✅ `empty` |

Helper nuevo `notify_sessions_closed(tenant_id, sessions, reason)` (best-effort, no lanza si Redis cae, igual que `events.publish`). `try_release_if_empty` pasa de `bool` a `list[TableSession]` (vacía ⇒ no cerró; truthiness preservada).

Nota (research D11): el barrido con pedidos por cobrar **solo cierra comensales** (no la sesión) y no emite; el cliente lo detecta por el 401 del sondeo (≤ 10 s con la pestaña visible, o al reusar la sesión). Por eso SC-005 (< 5 s) aplica a los caminos con evento y este camino tiene su propio tiempo; no se cambia el barrido (Principio V).

## 2. Rechazo de acciones con sesión cerrada (existente) + marca de estado

`open_session_context` ya responde **401** a cualquier acción del comensal si su participante no está `open`, o si su `table_session_id` ya no es la sesión activa de la mesa. Se mantiene sin excepciones por cuentas pendientes (FR-016) y **sin cambiar cuerpo ni mensajes**. Se añade una cabecera:

```
HTTP/1.1 401 Unauthorized
X-Session-State: closed          ← SOLO en los dos casos de cierre
{"detail": "Sesión no activa"}  ← cuerpo sin cambios
```

| 401 | Cabecera |
|---|---|
| `Sesión no activa` (participante no `open`) | `X-Session-State: closed` |
| `La mesa ya no tiene esta sesión abierta…` (sesión cerrada o distinta) | `X-Session-State: closed` |
| `Sesión expirada.` (firma/`exp`) · `…por inactividad…` · `…duración máxima…` | *(ausente)* |

`CORSMiddleware.expose_headers` gana `"X-Session-State"` (`app/main.py`).

## 3. Cliente (`pos-heladeria`)

- `DinerService.call()`: en un 401 lee `X-Session-State`; lanza `DinerSessionExpiredError` con `closed: boolean`.
- `DinerTokenStore.endAccess(tableToken)` (**único** borrado, compartido por "Salir" y cierre de mesa; FR-015a): quita el token de sesión, elimina de `localStorage` y `sessionStorage` toda clave `pos.diner.*` (progreso de checkout incluido), expira cualquier cookie `pos.diner*` y escribe **solo** la marca `pos.diner.exited_token` = token público de la URL de la mesa, en `sessionStorage` (FR-015c). Sin datos de sesión ni de comensal en la marca.
- `public-menu.component.ts`: `type MenuView = 'loading'|'error'|'name'|'menu'|'exited'|'closed'`.

| Vista | Cuándo | Contenido |
|---|---|---|
| **`closed`** (gracias) | en el momento: evento `session.closed`; 401 `closed` durante el uso (sondeo, `resync`/`reconnected`, acción, envío, pago); al pulsar "Salir" | "¡Gracias por tu visita! Esperamos verte pronto." con logo y nombre del negocio; sin botones, enlaces, historial ni recibo (FR-018) |
| **`exited`** (acceso denegado) | al cargar la URL con la marca presente (F5, "Atrás", "Adelante"); al cargar con un token anterior rechazado (401 `closed` en `ngOnInit`, `?s=` o `localStorage`) | "Por favor, escanea nuevamente el código QR de la mesa para ingresar al menú"; sin campo de nombre, menú ni botón; **no** crea sesión (FR-015b). Reemplaza "Acceso finalizado, vuelve a escanear…" |
| `name` | sin marca y sin token; o 401 **sin** la cabecera (vencimiento por inactividad/duración/firma) | sin cambio |

- Entrar a `closed` (por evento o 401 en uso): `stopPolling`, `disconnectRealtime`, `endAccess(token de la URL)`, `cart.clear()`, `cart.clearDiner()`, `myOrders.set([])`, `orderError.set(null)`, y cierre de overlays (selector, edición, pago, cajón).
- Orden de `ngOnInit` (sin cambio de estructura): resolver menú → ¿marca? ⇒ `exited` → ¿token? si no, `name` → `cart.load()/refreshOrders()`; en el `catch`, `closed` ⇒ `endAccess` + `exited` (antes: `closed`/gracias).
- Un pago por transferencia en curso no bloquea la pantalla (FR-019): el comprobante y el intento pendiente ya están en el servidor y siguen visibles para el cajero ("Pagos por confirmar").
- Reingreso: solo una **pestaña nueva** o un **escaneo nuevo** (sin marca en esa pestaña) ⇒ `name` ⇒ sesión nueva. `GET /menu/qr-token/{token}` no exige sesión y no cambia (es la entrada de ese escaneo en una mesa libre). Reabrir la mesa en el POS **no** revive el token anterior (FR-017; apunta a otra `table_session`).
- Almacenamiento bloqueado: se purga en memoria y se muestra gracias; sin marca, la recarga pide nombre (limitación aceptada, igual que hoy); el backend sigue rechazando el token anterior.

## 4. Pruebas de contrato

- Backend: `release_table` y cada camino de la matriz publican exactamente un `session.closed` por sesión cerrada tras el commit (y ninguno si hay `409` por órdenes sin cerrar); 401 con cabecera solo en los dos casos; token de la sesión anterior sigue en 401 tras reabrir/ocupar la mesa.
- Backend (FR-017a): un token de sesión anterior recibe 401 con `X-Session-State: closed` con la mesa **libre**, **cerrada** y **reocupada** por otra sesión; `GET /menu/qr-token/{token}` sigue resolviendo sin sesión.
- Frontend: vista `closed` por evento y por 401 en uso; tras el cierre y tras "Salir" el estado del navegador es **idéntico** (sin `pos.diner.session_token`, `checkout_progress.*`, datos de comensal ni cookies; solo `pos.diner.exited_token` en `sessionStorage`; SC-011); recarga con la marca ⇒ `exited` y **cero** llamadas a `openSession`; 401 `closed` en la carga inicial ⇒ `exited`; sin cabecera ⇒ `name`; pestaña sin marca ⇒ `name`; `sessionStorage` bloqueado ⇒ gracias sin excepción; texto "Acceso finalizado…" ya no existe.
