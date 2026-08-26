# Quickstart: validar Vaciado del Carrito del Participante al Crear el Pedido

Guía de ejecución para comprobar que la implementación cumple `spec.md`. No repite firmas ni
columnas ya detalladas en [data-model.md](./data-model.md) y `contracts/` — solo enlaza a ellas.

**Prerequisitos**: entorno virtual de `pos-backend` (Python 3.14), ejecutado desde la raíz de
`../pos-backend` (sibling de este repo `pos-specs`).

```bash
cd ../pos-backend
source env/bin/activate
```

No hace falta PostgreSQL: los characterization tests usan SQLite en memoria
(`app/characterization_tests/cart_fixtures.py::new_session`).

## Paso 0 — Confirmar la línea base antes de tocar código

```bash
python -m unittest app.characterization_tests.test_cart_service -v
python -m unittest app.characterization_tests.test_cart_router -v
python -m unittest app.characterization_tests.test_orders_consolidation -v
```

**Resultado esperado**: todo en verde, incluidos los 2 tests `CONGELA` que esta spec va a
modificar (`test_submit_cart_confirma_pedido_y_abre_carrito_nuevo`,
`test_submit_cart_endpoint_evento_tras_commit`) y el tercero que **no** se toca
(`test_consolidate_table_consolida_carritos_en_orden_existente`, research.md Decisión 2).

## Paso 1 — Migración (antes de cualquier test que dependa de las columnas nuevas)

```bash
alembic upgrade head
```

Verificar en una sesión `psql`/`sqlite` local de desarrollo (o leyendo la migración) que
`order_items` ganó `discounted_unit_price`/`discounted_line_total`, ambas `NUMERIC(12,2)` nullable,
sin `DEFAULT` — y que una fila `order_items` preexistente (creada antes de la migración) tiene
`NULL` en ambas, no `0` (FR-015). Rollback de prueba:

```bash
alembic downgrade -1   # confirma que ambas columnas desaparecen sin error
alembic upgrade head
```

## US1 — El carrito desaparece en cuanto el pedido se confirma

Ficheros modificados: `app/characterization_tests/test_cart_service.py`,
`app/characterization_tests/test_cart_router.py` (research.md Decisión 9 para los 2 tests
`CONGELA` reescritos).

1. Carrito con 2 ítems, uno con una promoción de descuento activa sobre su línea → `submit_cart` →
   `201`; verificar en la respuesta: `items[]` incluye el snapshot de descuento
   (`discounted_unit_price`/`discounted_line_total` no nulos en la línea con promoción, nulos en la
   otra) — igual a lo que `serialize_cart`/`GET /cart` mostraba para ese mismo carrito justo antes
   de confirmar (Acceptance Scenario 3 de US1 / CA-6, SC-005,
   [contracts/cart-orders.md](./contracts/cart-orders.md)).
2. Sobre el mismo escenario: `db.get(Cart, old_cart.id) is None` inmediatamente después del
   `submit_cart` exitoso — la fila y sus `CartItem`/`CartItemOption` ya no existen (SC-001,
   `test_submit_cart_elimina_carrito_y_abre_uno_nuevo_tras_pedido`, renombrado de
   `test_submit_cart_confirma_pedido_y_abre_carrito_nuevo`, research.md Decisión 9).
3. Tras el paso 2, llamar `get_cart(db, participant.id)` (equivalente a recargar la página) →
   `CartResponse.items == []`, `status == 'abierto'`, con un `id` de carrito **distinto** al
   eliminado (Acceptance Scenario 1-2 de US1 / CA-1, CA-2,
   [contracts/cart-get.md](./contracts/cart-get.md)).
4. Endpoint completo (`test_submit_cart_endpoint_evento_tras_commit`, aserción reescrita): el spy de
   `events.order_created` confirma, en el momento en que se dispara, que ya no existe ninguna fila
   de `Cart` para ese participante — el evento se sigue publicando estrictamente después del
   `commit()` (garantía sin cambio, research.md Decisión 9).
5. Fallo forzado tras el chequeo de disponibilidad (p. ej. mockear `check_availability` para lanzar,
   o método de pago inactivo) → `submit_cart` lanza la excepción esperada; verificar que el
   `Cart`/`CartItem[]` del participante **siguen existiendo intactos** (mismo `id`, mismas líneas) y
   que no se creó ningún `CustomerOrder` — la transacción completa se revirtió (Acceptance Scenario
   4 de US1 / CA-8, FR-004).

```bash
python -m unittest app.characterization_tests.test_cart_service -v
python -m unittest app.characterization_tests.test_cart_router -v
```

## US2 — Un segundo intento de confirmar nunca duplica el pedido

Fichero: `app/characterization_tests/test_cart_router.py` (o `test_cart_service.py`, según dónde
viva la aserción más directa — ambos ejercitan `submit_cart`).

1. Confirmar un carrito con éxito y, sin agregar nada nuevo, invocar `submit_cart` de nuevo para el
   mismo participante → el carrito ya no existe (paso US1.2) **y** el participante tiene un pedido
   `recibida` (no terminal) → `409` con el mensaje explícito de "ya fue enviado" (NO el genérico de
   carrito vacío) — verificar el texto exacto, distinto del de carrito vacío (Acceptance Scenario 1
   de US2 / CA-3, FR-007, [contracts/cart-submit.md](./contracts/cart-submit.md)). Ningún
   `CustomerOrder` nuevo se creó — contar filas de `CustomerOrder` para el participante, sigue en 1.
2. Simular "dos pestañas": tras confirmar (paso 1), llamar `add_item` sobre el carrito nuevo que
   `_get_or_create_open_cart` abrió automáticamente **no** aplica aquí (ese es un carrito distinto,
   ver US3) — el escenario de "pestaña vieja" es exactamente el paso 1: la pestaña vieja sigue
   apuntando al mismo `participant`, así que su segundo intento de `submit_cart` cae en el mismo
   camino de "ya fue enviado" (Acceptance Scenario 2 de US2 / CA-4).
3. Carrito realmente vacío (participante sin ítems y sin ningún pedido no terminal previo) →
   `submit_cart` → `409` `"El carrito está vacío"` (o `404` si nunca existió fila de carrito) —
   mensaje **distinto** al de "ya fue enviado", sin crear nada (Acceptance Scenario 3 de US2 / CA-7,
   FR-009).

```bash
python -m unittest app.characterization_tests.test_cart_router -v
```

## US3 — Segunda ronda desde un carrito limpio

Sin fichero de test nuevo dedicado — se verifica como extensión del test renombrado en US1.

1. Tras confirmar el primer pedido (US1.2), agregar un ítem nuevo (`add_item`) → el carrito
   resultante contiene únicamente ese ítem — no arrastra ninguna línea del pedido ya confirmado
   (Acceptance Scenario 1 de US3 / RF-08). Esto ya lo cubre
   `test_submit_cart_elimina_carrito_y_abre_uno_nuevo_tras_pedido` (bloque final, sin cambios de
   lógica respecto al test original — `_get_or_create_open_cart` no cambió).
2. Confirmar ese segundo carrito → segundo `CustomerOrder` independiente, con el primero y el
   segundo coexistiendo (ambos con sus propias líneas) — verificar contando `CustomerOrder` para el
   participante tras ambas confirmaciones (Acceptance Scenario 2 de US3).
   *Nota*: solo es posible confirmar el segundo si el primero ya salió de
   `_NON_TERMINAL_ORDER_STATUSES` (p. ej. staff lo movió a `'pagada'`/`'cancelada'`), o si el
   escenario de prueba usa dos participantes — de lo contrario FR-008 lo bloquearía con el `409` de
   "orden activa", que es exactamente el comportamiento correcto y no una falla de este paso.

## US4 — El vaciado de un comensal no afecta a los demás de su mesa

Fichero: `app/characterization_tests/test_cart_router.py` o `test_cart_service.py` (test nuevo).

1. Dos participantes (`ana`, `beto`) en la misma `table_session`, cada uno con su propio `Cart` con
   ítems → `submit_cart(db, ana, ...)` → verificar: el `Cart` de `beto` sigue existiendo, con el
   mismo `id`, las mismas líneas y la misma cantidad — sin ningún cambio (Acceptance Scenario 1 de
   US4 / CA-5, FR-005, FR-006).
2. Intentar cualquier operación de carrito de `beto` usando el `participant_id`/token de `ana` (o
   viceversa) → rechazada (404/403 según el mecanismo existente de `SessionContext`), sin vaciar ni
   modificar el carrito ajeno — comportamiento ya cubierto por el aislamiento existente de
   `x-session-token`, sin cambios de esta spec; se verifica que sigue vigente (Acceptance Scenario 2
   de US4 / RF-10).

## Verificación de no regresión — el tercer test `CONGELA` y el resto de la suite

```bash
python -m unittest app.characterization_tests.test_orders_consolidation -v
python -m unittest discover -s app/characterization_tests -p 'test_*.py' -v
```

**Resultado esperado**:
- `test_consolidate_table_consolida_carritos_en_orden_existente` sigue en verde **sin ninguna
  modificación** — sus aserciones `cart_ana.status == "confirmado"` / `cart_beto.status ==
  "confirmado"` (líneas 308-309) siguen siendo ciertas porque `consolidate_table` no se tocó
  (research.md Decisión 2).
- `orders/checkout.py` (`close_participants`) y `core/qr_context.py` (`close_participant`) siguen
  operando sobre carritos `'abierto'` exactamente igual — sus tests, si existen, no deberían
  requerir ningún cambio (research.md Decisión 10).
- Todo el resto de la suite de `pos-backend` sigue en verde — ningún otro test `"CONGELA
  comportamiento actual:"` queda en rojo sin una decisión que lo autorice (Principio III).

## Verificación final — SC-001 a SC-006

| Criterio | Cómo se verifica |
|---|---|
| SC-001 (100% de pedidos confirmados dejan el carrito en cero líneas) | US1 pasos 2-3 |
| SC-002 (100% de reintentos sobre un carrito ya vaciado se rechazan sin duplicar, con mensaje de "ya fue enviado") | US2 paso 1 |
| SC-003 (0% de confirmaciones dobles casi simultáneas producen más de un pedido) | US2 paso 1 + garantía de `idx_active_order_per_participant` (sin cambio, spec 025) |
| SC-004 (100% de confirmaciones fallidas dejan el carrito intacto) | US1 paso 5 |
| SC-005 (100% de pedidos con promoción activa muestran el mismo precio con descuento que el carrito) | US1 paso 1 |
| SC-006 (0% de confirmaciones de un comensal afectan el carrito de otro) | US4 paso 1 |

## Frontend — validación manual (no automatizada en esta guía)

`pos-heladeria` no gana ningún `.spec.ts` nuevo (research.md Decisión 8: solo cambia un tipo
TypeScript, sin componente/plantilla tocada) — se valida con `ng serve` + navegación manual sobre
el menú público QR.

```bash
cd ../pos-heladeria
npm start   # o el script equivalente de dev del repo
```

1. Como comensal, agregar ítems al carrito (con y sin una promoción activa) y confirmar el pedido
   con un método de pago válido → la pantalla muestra la confirmación (`confirmation-step`), sin
   error ni pantalla en blanco (CA-1).
2. Recargar la página (F5) inmediatamente después → el carrito aparece vacío; el pedido recién
   enviado sigue apareciendo en la sección "Mis pedidos" (`section() === 'pedidos'`,
   `public-menu.component.ts`) con su estado correcto (CA-2).
3. Sin recargar, agregar un ítem nuevo → el carrito muestra únicamente ese ítem, no los de la ronda
   ya confirmada (US3).
4. Intentar confirmar de nuevo desde una pestaña que quedó con el carrito viejo en memoria (o
   simular reenviando la misma petición) → el guard de hidratación
   (`checkout-hydration.guard.ts`) ya redirige a la pantalla de confirmación cuando detecta una
   orden no terminal; verificar además, a nivel de red (herramientas de desarrollador), que si la
   petición `POST /cart/submit` llega a ejecutarse, el backend responde `409` con el mensaje de "ya
   fue enviado" (no el genérico de carrito vacío) — confirma que la protección no depende solo del
   guard del frontend (US2).
5. Con dos comensales en la misma mesa (dos pestañas/dispositivos distintos, cada uno con su propio
   `x-session-token`), confirmar el pedido de uno → el carrito del otro, visible en su propia
   pantalla, no cambia (US4).
