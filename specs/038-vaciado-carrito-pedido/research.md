# Research: Vaciado del Carrito del Participante al Crear el Pedido (Menú QR)

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — las 4 clarificaciones de
negocio ya se resolvieron en `spec.md` (sesión 2026-08-26) y el resto de incógnitas era puramente
técnico, resuelto leyendo directamente `pos-backend`/`pos-heladeria`. Este documento registra las
decisiones de diseño, las alternativas descartadas y un hallazgo que corrige una Assumption del
spec.

## Decisión 1 — Borrado físico vía `db.delete(cart)`, sin migración de esquema

- **Decisión**: en `submit_cart` (`app/api/v1/cart/service.py:605`), reemplazar
  `cart.status = "confirmado"` por `db.delete(cart)`, dentro del mismo bloque `try` que ya crea la
  `CustomerOrder`/`OrderItem`s/`OrderPaymentAttempt` y hace `commit()` (línea 606). Sin cambio de
  esquema: `Cart.items` ya declara `cascade="all, delete-orphan"` (`app/models/cart.py:31-33`), y
  `cart_items.cart_id` ya tiene `ondelete="CASCADE"` (`app/models/cart_item.py:19`), igual que
  `cart_item_options.cart_item_id` (`app/models/cart_item.py:58`). `db.delete(cart)` sobre el
  objeto ya cargado con `selectinload(Cart.items).selectinload(CartItem.options)`
  (`_load_open_cart`, línea 174) es suficiente: SQLAlchemy emite los `DELETE` de `cart_item_options`
  → `cart_items` → `carts` en ese orden dentro de la misma transacción.
- **Rationale**: FR-003 exige borrado físico, no archivado; FR-004 exige que ocurra en la misma
  transacción que la creación del pedido. Las cascadas ya declaradas (puestas ahí originalmente
  para el caso de "abandonar sesión"/limpieza, no para este flujo) hacen que la operación sea un
  cambio de una línea de código, sin ninguna migración: no se agrega, quita ni modifica ninguna
  columna de `Cart`/`CartItem`/`CartItemOption`.
- **Alternatives considered**: `DELETE` en bloque vía SQL crudo (`db.execute(delete(CartItem)...)`
  seguido de `delete(Cart)...`) — descartado, más código para el mismo resultado que la cascada ya
  garantiza, y pierde la ventaja de que el ORM ya tiene el objeto cargado.

## Decisión 2 — El `CheckConstraint` de `Cart.status` no se toca; corrección de una Assumption del spec

- **Hallazgo**: `spec.md` (Assumptions) afirma que el estado `'confirmado'` "queda sin ningún
  camino que lo produzca... era el único código que lo asignaba". Verificado contra el código: esto
  es **inexacto**. `app/api/v1/orders/consolidation.py::consolidate_table` (línea 156,
  `cart.status = "confirmado"`) es la vía del mesero para consolidar los carritos abiertos de una
  mesa en una orden de mostrador/mesero (`channel='waiter'`) — explícitamente **fuera de alcance**
  de esta spec ("El carrito del terminal de staff", `spec.md` §Fuera de Alcance). Ese código no se
  toca, y sigue asignando `'confirmado'` después de esta spec exactamente igual que hoy. Existe
  además un **tercer** test `"CONGELA comportamiento actual:"` que protege justo eso —
  `test_consolidate_table_consolida_carritos_en_orden_existente`
  (`app/characterization_tests/test_orders_consolidation.py:269-315`), con aserciones explícitas
  `self.assertEqual(cart_ana.status, "confirmado")` / `self.assertEqual(cart_beto.status,
  "confirmado")` en las líneas 308-309 — no citado por `spec.md` (que solo identificó los 2 tests
  de `submit_cart`) porque protege un código distinto que esta spec no modifica.
- **Decisión**: no tocar el `CheckConstraint("status IN ('abierto', 'confirmado', 'abandonado')",
  name="ck_cart_status")` de `Cart` (`app/models/cart.py:36-38`). Ningún `ALTER TABLE` ni migración
  de datos sobre `carts`.
- **Rationale**: la conclusión práctica de la Assumption del spec ("no hace falta requisito
  funcional para esto, se resuelve en planeación") sigue siendo correcta — solo su premisa estaba
  incompleta. Con el hallazgo de arriba, la decisión es todavía más clara que si `'confirmado'`
  fuera un valor verdaderamente muerto: sigue siendo un valor **legítimamente vivo**, producido
  activamente por un flujo distinto (mesero) que esta spec no autoriza a tocar (Principio V). Quitar
  `'confirmado'` del constraint rompería `consolidate_table` de inmediato. Dejarlo intacto también
  preserva, sin ningún esfuerzo adicional, cualquier fila `'confirmado'` que ya exista en producción
  por el propio `submit_cart` de antes de esta spec (Principio VII) — ninguna migración las toca ni
  las purga.
- **Verificación de no regresión**: `test_consolidate_table_consolida_carritos_en_orden_existente`
  se ejecuta sin ninguna modificación de código de `consolidation.py`/`checkout.py`/`qr_context.py`
  — los tres módulos, verificados en la Decisión 10 de este documento, solo leen `Cart` con
  `status == 'abierto'`, nunca `'confirmado'`, así que el borrado físico introducido por esta spec
  (que solo aplica al camino de `submit_cart`) no interfiere con ellos.
- **Alternatives considered**: migrar el constraint para retirar `'confirmado'` — descartado,
  rompería `consolidate_table` (código activo, fuera de alcance) y violaría el Principio VII sobre
  cualquier fila preexistente con ese valor.

## Decisión 3 — Reordenar las validaciones de `submit_cart` para el mensaje de "ya fue enviado" (FR-007/FR-009)

- **Decisión**: mover el chequeo de "orden activa no terminal" (hoy en `submit_cart`, líneas
  522-532, ejecutado *después* de comprobar que el carrito tiene ítems) a **antes** de decidir qué
  error devolver cuando el carrito está vacío o no existe. Pseudocódigo del nuevo orden:

  ```python
  cart = db.execute(select(Cart)...).scalar_one_or_none()   # ya no HTTPException inmediata
  active_order = db.execute(                                 # misma query que ya existe hoy,
      select(CustomerOrder.id).where(                        # solo movida más arriba
          CustomerOrder.participant_id == participant.id,
          CustomerOrder.status.in_(_NON_TERMINAL_ORDER_STATUSES),
      )
  ).scalar_one_or_none()

  if cart is None or not cart.items:
      if active_order is not None:
          raise HTTPException(409, "Tu pedido ya fue enviado; revisa el estado en Mis pedidos.")
      if cart is None:
          raise HTTPException(404, "No hay carrito abierto para el comensal")   # sin cambio
      raise HTTPException(409, "El carrito está vacío")                        # sin cambio

  if active_order is not None:      # carrito CON ítems + orden activa: FR-008, mensaje sin cambio
      raise HTTPException(409, "Ya tienes una orden activa; espera a que finalice antes de enviar otra.")
  ```

  El resto de `submit_cart` (chequeo de comanda manual activa, método de pago, disponibilidad,
  creación del pedido) sigue exactamente igual, sin reordenar nada más.
- **Rationale**: FR-007 exige el mensaje explícito exactamente en la combinación "carrito
  vacío/inexistente **y** orden no terminal previa" — la única forma de distinguir ese caso del
  "carrito vacío sin más" (FR-009) es conocer si existe una orden activa *antes* de decidir el
  mensaje de error, no después (que es cuando el código de hoy la consulta, y solo para el caso de
  carrito con ítems). La query de `active_order` es la misma ya existente (mismo predicado
  `_NON_TERMINAL_ORDER_STATUSES`) — no se duplica, se reutiliza su resultado en ambas ramas.
- **Alternatives considered**: consultar `active_order` dos veces (una para cada rama) —
  descartado, una sola consulta computada una vez y reutilizada es más simple y evita una
  ventana de inconsistencia entre las dos lecturas.

## Decisión 4 — Snapshot de descuento: extraer el cómputo de `serialize_cart` a una función compartida

- **Decisión**: extraer el bloque de cómputo de descuento por línea que hoy vive inline en
  `serialize_cart` (`app/api/v1/cart/service.py:238-259`) a una función privada, p. ej.
  `_line_discount(promos, catalog, product_variant_id, combo_id, quantity, unit_price) ->
  tuple[Decimal | None, Decimal | None]` (retorna `discounted_unit_price`, `discounted_line_total`,
  `None`/`None` si no aplica ningún descuento o la línea es de combo). `serialize_cart` pasa a
  llamarla para cada `CartItem` (mismo resultado que hoy, sin cambio observable en `CartResponse`);
  `submit_cart` la llama para cada `CartItem` al construir el `OrderItem` correspondiente, con
  `promos = promotions.active_discount_promotions(db, now)` calculado una sola vez antes del bucle
  (igual patrón que `serialize_cart`) y el mismo `catalog` (`{variant_id: (product_id,
  category_id)}`, hoy construido inline en `serialize_cart:224-232`, también extraído a un helper
  reusado por ambas funciones).
- **Rationale**: FR-002/FR-013/CA-6/SC-005 exigen que el pedido recién confirmado muestre
  exactamente el mismo precio con descuento que el carrito mostraba justo antes de confirmar — la
  única forma de garantizar eso sin duplicar (y arriesgar divergencia futura) la lógica de
  `active_discount_promotions` + `best_line_discount` + exclusión de líneas de combo + redondeo
  (`Decimal("0.01")`, `ROUND_HALF_UP`) es que ambas funciones llamen exactamente al mismo código.
  Esta extracción es una consecuencia directa de la FR, no una refactorización oportunista
  (Principio V) — sin ella, la única alternativa sería copiar y pegar el bloque completo dentro de
  `submit_cart`, con el riesgo de que una corrección futura a `serialize_cart` no se replique ahí.
- **Alternatives considered**: duplicar el bloque de descuento dentro de `submit_cart` — descartado,
  viola la razón de ser de SC-005 (dos implementaciones del mismo cálculo divergen con el tiempo);
  llamar a `serialize_cart` directamente desde dentro de `submit_cart` para obtener los precios —
  descartado, `serialize_cart` construye un `CartResponse` completo (incluye IDs, opciones, y hace
  sus propias queries), un acoplamiento más pesado e indirecto que llamar al cómputo puntual por
  línea que de verdad se necesita.

## Decisión 5 — FR-005 (fallo al resolver el descuento) ya está cubierto por el `try/except` existente

- **Decisión**: no se agrega ningún manejo de excepción nuevo para el cómputo de descuento. La
  llamada a `_line_discount(...)` para cada `OrderItem` ocurre dentro del mismo bloque `try` que ya
  envuelve toda la creación del pedido (`submit_cart`, líneas 570-623) y que ya termina en
  `except Exception: db.rollback(); logger.exception(...); raise` (línea 620-623).
- **Rationale**: `promotions.active_discount_promotions`/`best_line_discount` son cómputo puro
  sobre datos ya cargados en memoria (sin I/O propio más allá de la consulta inicial de promociones
  vigentes, ya hecha antes del bucle) — no tienen hoy ningún modo de fallo documentado más allá de
  una excepción de programación genuina. FR-005 pide que, *si* algo así ocurriera, la confirmación
  se rechace sin crear el pedido ni borrar el carrito — que es exactamente lo que el `except
  Exception` ya garantiza para cualquier excepción no capturada explícitamente antes: `rollback()`
  deshace tanto la creación del pedido como el `db.delete(cart)` (ambos parte de la misma
  transacción), y la excepción se re-lanza sin haber hecho `commit()`.
- **Alternatives considered**: envolver específicamente el cómputo de descuento en un
  `try/except` propio con un `HTTPException` dedicado — descartado, no hay ningún caso de fallo
  identificado que amerite un mensaje distinto al genérico de error inesperado que ya produce el
  `except Exception` existente; añadirlo sería manejar un escenario hipotético sin caso de uso real
  (contradice la guía de no agregar manejo de errores para escenarios que no pueden ocurrir).

## Decisión 6 — El evento `events.order_created` no se actualiza para reflejar el descuento

- **Decisión**: el cálculo de `total` en `app/api/v1/cart/router.py:165`
  (`sum((i.unit_price * i.quantity for i in order.items), start=Decimal(0))`, usado solo como
  payload del evento de tiempo real `events.order_created`, consumido por dashboards/notificaciones
  de staff) **no se modifica** para restar el descuento.
- **Rationale**: FR-013/CA-6/SC-005 hablan explícitamente de "la respuesta del pedido que ve el
  comensal" — el `OrderResponse`/`OrderItemResponse` que el comensal recibe de
  `POST /cart/submit`/`GET /cart/orders`. El evento de tiempo real es un mecanismo interno para
  notificar a cocina/caja que un pedido nuevo llegó (con un total aproximado en su payload, no un
  contrato que el comensal consuma) — ninguna FR/CA/SC de esta spec lo menciona. Tocarlo sería
  alcance no pedido (Principio V).
- **Alternatives considered**: ninguna — no hay ninguna FR que lo requiera; se documenta aquí solo
  para que quede explícito que la omisión es deliberada, no un descuido.

## Decisión 7 — Migración: 2 columnas nullable en `order_items`, patrón `@for_each_tenant_schema`

- **Decisión**: una migración Alembic nueva, `down_revision='205f518df786'` (head actual, spec 037),
  siguiendo exactamente el patrón de `a63ddb1f0d97_combos_de_promociones_dias_del_mes_.py` (última
  migración que agrega columnas a `cart_items`/`order_items`):

  ```python
  @for_each_tenant_schema
  def upgrade(schema: str) -> None:
      if not _has_table(schema, "order_items"):
          return
      op.add_column("order_items", sa.Column(
          "discounted_unit_price", sa.Numeric(12, 2), nullable=True
      ), schema=schema)
      op.add_column("order_items", sa.Column(
          "discounted_line_total", sa.Numeric(12, 2), nullable=True
      ), schema=schema)

  @for_each_tenant_schema
  def downgrade(schema: str) -> None:
      if not _has_table(schema, "order_items"):
          return
      op.drop_column("order_items", "discounted_line_total", schema=schema)
      op.drop_column("order_items", "discounted_unit_price", schema=schema)
  ```

  Sin `server_default`: las filas existentes de `order_items` quedan con `NULL` en ambas columnas
  (FR-015), sin ningún `UPDATE` de backfill.
- **Rationale**: `Cart`/`CustomerOrder`/`CartItem`/`OrderItem` viven en el schema `tenant`
  (confirmado: `{"schema": "tenant"}` en cada modelo + `Base.metadata = MetaData(schema="tenant",
  ...)` en `app/core/models.py`) — a diferencia de `UserInvitation`/`Tenant`/`User` (schema
  `shared`, spec 037), cualquier migración sobre estas tablas debe recorrer cada schema de tenant
  vía `@for_each_tenant_schema`, exactamente como ya hace `a63ddb1f0d97` para el mismo par de
  tablas (`cart_items`, `order_items`).
- **Alternatives considered**: una sola columna `discount_amount` en vez de dos precios ya
  descontados — descartado por consistencia: `CartItemResponse` (el contrato que el comensal ya ve
  antes de confirmar) usa `discounted_unit_price`/`discounted_line_total`, no un monto de
  descuento aparte; reusar exactamente esos dos nombres/semántica evita introducir una
  representación nueva del mismo concepto (Principio V) y hace que FR-013 ("igual a lo que el
  carrito mostraba") sea una comparación campo a campo, no una traducción.

## Decisión 8 — Frontend: solo tipo, sin cambio de UI

- **Decisión**: `DiningOrderItem` (`pos-heladeria/src/app/modules/tables/interfaces/dining.interface.ts:70-99`)
  gana `discounted_unit_price?: string | null` y `discounted_line_total?: string | null`, mismos
  nombres/forma que `CartItemResponse` en `diner.interface.ts:75-76`. **Ningún componente ni
  plantilla se modifica.**
- **Rationale**: se verificó explícitamente dónde el comensal ve sus pedidos ya enviados —
  `public-menu.component.ts`, sección `section() === 'pedidos'` (líneas 201-303) — y la pantalla de
  confirmación inmediatamente posterior a enviar — `confirmation-step.component.ts` (94 líneas
  completas). **Ninguna de las dos muestra hoy ningún precio** de línea ni de pedido: la primera
  solo renderiza cantidad, nombre de variante, opciones, estado de cocina y estado de pago; la
  segunda solo el estado del pedido y el estado del intento de pago. Ninguna Acceptance Scenario ni
  criterio de éxito de `spec.md` pide agregar un elemento de precio visible en esas pantallas —
  "la respuesta del pedido que ve el comensal" (FR-013) se satisface con el campo disponible en el
  contrato HTTP que el frontend ya consume (`DinerService.submitCart`/`myOrders`,
  `diner.service.ts:203-214`), no con un requisito de renderizado. Agregar una vista de precio ahí
  sería una funcionalidad de UI enteramente nueva, no pedida por ningún FR/CA/SC — exactamente el
  tipo de alcance no solicitado que el Principio V prohíbe mezclar con esta spec.
- **Alternatives considered**: renderizar el precio con descuento en la sección "Mis pedidos" —
  descartado por lo anterior; queda como posible funcionalidad futura con su propio spec si el
  negocio lo pide.

## Decisión 9 — Reescritura exacta de los 2 tests `CONGELA` citados por `spec.md`

Sigue la convención ya usada por spec 020 (`quickstart.md` Paso 3: el nombre del test pasa a
reflejar el comportamiento corregido, se mantiene el prefijo `CONGELA`, se cita la spec que
autoriza el cambio en el docstring).

- **`test_submit_cart_confirma_pedido_y_abre_carrito_nuevo`**
  (`test_cart_service.py:484-522`) — se renombra a
  `test_submit_cart_elimina_carrito_y_abre_uno_nuevo_tras_pedido`. El bloque
  ```python
  db.refresh(old_cart)
  self.assertEqual(old_cart.status, "confirmado")
  ```
  se reemplaza por una verificación de ausencia (`db.refresh()` sobre una fila ya borrada lanza
  `sqlalchemy.orm.exc.ObjectDeletedError`, una señal indirecta y frágil para lo que el test
  realmente quiere afirmar):
  ```python
  self.assertIsNone(db.get(Cart, old_cart.id))
  ```
  El resto del test (creación del pedido, `current_payment_attempt`, y el bloque final que verifica
  que `_get_or_create_open_cart` abre un carrito nuevo distinto en la siguiente operación) no
  cambia — sigue siendo cierto sin modificación: `new_cart.id != old_cart.id` ya era la aserción
  correcta, y ahora es cierta por una razón más fuerte (`old_cart.id` ya no existe en absoluto, no
  solo que su `status` cambió).
- **`test_submit_cart_endpoint_evento_tras_commit`** (`test_cart_router.py:216-252`) — mismo
  nombre (la garantía que protege — el evento se publica después del commit — no cambia). Dentro de
  `_spy`, el bloque
  ```python
  seen["cart_confirmado"] = db.execute(
      select(Cart.status)
      .where(Cart.participant_id == participant.id, Cart.status == "confirmado")
  ).scalar_one_or_none()
  ```
  se reemplaza por una verificación de ausencia total de fila para ese comensal:
  ```python
  seen["cart_exists"] = db.execute(
      select(Cart.id).where(Cart.participant_id == participant.id)
  ).scalar_one_or_none()
  ```
  y la aserción `self.assertEqual(seen["cart_confirmado"], "confirmado")` pasa a
  `self.assertIsNone(seen["cart_exists"])`. El resto del test (orden `recibida`, el evento se
  dispara exactamente una vez, `result.status == "recibida"`) no cambia.

## Decisión 10 — Verificación de aislamiento: ningún otro consumidor de `Cart`/`CartItem` se ve afectado

Grep de `Cart`/`CartItem` fuera de `app/api/v1/cart/` en `pos-backend/app/` (excluyendo tests y
`app/scripts/test_table_sessions.py`, un script de desarrollo fuera de la app FastAPI):

| Consumidor | Qué hace con `Cart` | ¿Toca `status='confirmado'`? |
|---|---|---|
| `orders/consolidation.py::consolidate_table` | Lee `status='abierto'` (línea 116); **escribe** `status='confirmado'` (línea 156) | Sí, escribe — pero es la vía del mesero, fuera de alcance (Decisión 2) |
| `orders/checkout.py::close_participants` | Lee `status='abierto'` (línea 657), las pasa a `'abandonado'` | No |
| `core/qr_context.py::close_participant` | Lee `status='abierto'` (línea 88), las pasa a `'abandonado'` | No |

Ninguno de los tres consulta ni depende de que una fila `'confirmado'` exista tras convertirse en
pedido — todos filtran explícitamente por `status == 'abierto'` para su propio propósito (agrupar
carritos pendientes de consolidar, o abandonar carritos de comensales que se van). El borrado físico
que esta spec introduce en `submit_cart` es invisible para los tres: antes de esta spec, ellos ya
ignoraban cualquier carrito no `'abierto'`; después, ese carrito simplemente ya no existe en vez de
existir con otro estado — mismo resultado observable para estas tres funciones.

## Migraciones — estrategia de rollback (Principio VIII)

- `order_items.discounted_unit_price` / `order_items.discounted_line_total` (columnas nuevas,
  nullable, sin default, schema `tenant` vía `@for_each_tenant_schema`): rollback =
  `op.drop_column` × 2 (Decisión 7). Sin pérdida de datos: son puramente aditivas, ninguna fila
  existente se modifica al agregarlas ni al quitarlas.
- `Cart`/`CartItem`/`CartItemOption`: **sin migración** — el borrado físico es comportamiento de
  aplicación (`db.delete(cart)`), no un cambio de esquema; no hay nada que revertir a nivel de base
  de datos más allá de revertir el commit de código (Decisión 1).
- `Cart.status` `CheckConstraint`: **sin cambios**, ver Decisión 2.
