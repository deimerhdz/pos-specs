# Research: Revisión y Pago Antes de Enviar el Pedido (Skeilopos)

No quedó ningún `NEEDS CLARIFICATION` en el Technical Context del plan — las 3 clarificaciones de
negocio ya se resolvieron en `spec.md` (sesión 2026-08-18) y el resto era puramente técnico,
resuelto leyendo directamente el `pos-backend` tal como lo dejó spec 024. Este documento registra
las decisiones de diseño y las alternativas descartadas.

## Decisión 1 — Fusionar "crear la orden" y "crear el primer intento de pago" en `POST /cart/submit`

- **Decisión**: `submit_cart` (`app/api/v1/cart/service.py:490`) gana los parámetros
  `payment_method_id` y `receipt_file_url: str | None`. Crea la `CustomerOrder`, sus `OrderItem` y
  el primer `OrderPaymentAttempt` en la misma transacción — un solo `db.commit()`, no dos
  llamadas HTTP separadas como hoy (`POST /cart/submit` seguido de `POST
  /cart/orders/{id}/payment-attempts`).
- **Rationale**: es el requisito central de `spec.md` (FR-003/FR-004/FR-007): el pedido no debe
  quedar registrado para el staff mientras el pago no esté resuelto. Crear la orden primero y el
  intento después —como hace spec 024 hoy— deja inevitablemente una ventana donde el pedido existe
  sin ningún intento de pago, exactamente lo que esta spec busca eliminar.
- **Alternatives considered**: mantener dos llamadas separadas pero ocultar la orden "recién
  creada, sin intento" del panel del staff con un filtro — descartado: reintroduce el mismo estado
  intermedio "pedido sin pago" que la spec prohíbe explícitamente (FR-003), solo que ocultándolo
  en vez de no crearlo; además duplica la superficie de fallo (dos escrituras en vez de una,
  con una ventana real entre ambas donde un fallo de red deja el pedido huérfano sin intento).

## Decisión 2 — Presign genérico, sin atar el archivo a un intento que todavía no existe

- **Decisión**: nuevo endpoint `POST /cart/payment-receipt/presign`, que solo exige un comensal
  autenticado (`Depends(get_session_context)`) y un `content_type` soportado — sin `attempt_id` ni
  ninguna otra validación de propiedad, porque no hay ningún recurso todavía que "poseer". Reutiliza
  las mismas primitivas de `app/core/storage.py` (`build_object_key`, `generate_presigned_put_url`,
  `public_url_for`) que ya usa el presign ligado a un intento (spec 024,
  `cart/service.py:presign_receipt`).
- **Rationale**: `build_object_key(tenant_schema, folder, extension)` ya genera una key aleatoria
  (`uuid4().hex`) — nunca dependió del `attempt_id` para nada más que la validación de propiedad
  del endpoint que lo envolvía. Quitar esa validación (innecesaria cuando no hay nada que
  poseer todavía) es la única diferencia real con el presign existente; duplicar la lógica de
  construir la key y firmar la URL sería repetir código sin ganar nada.
- **Alternatives considered**: crear el `OrderPaymentAttempt` "huérfano" (sin `order_id`, con esa
  columna nullable) solo para tener algo a lo que atar el presign — descartado de plano: el schema
  de `OrderPaymentAttempt` (spec 024) tiene `order_id` `NOT NULL` a propósito (todo intento
  pertenece a una orden), y relajar esa restricción para este caso reintroduce el mismo problema
  de "hay un registro sin pedido asociado" que la spec 024 nunca tuvo que resolver y esta spec no
  debería inventar.

## Decisión 3 — Ningún estado "borrador" nuevo en `CustomerOrder.status`

- **Decisión**: no se agrega ningún valor al `CHECK` de `CustomerOrder.status`
  (`recibida|abierta|bloqueada|pagada|cancelada`). Mientras el comensal está en la pantalla de
  revisión o en la de un método de transferencia, **no existe ninguna fila** de `CustomerOrder` —
  no es un estado nuevo del pedido, es la ausencia del pedido.
- **Rationale**: spec 024 (research.md, Decisión 1) ya estableció el mismo principio para
  "pendiente de pago" — derivar el estado en vez de agregar una columna. Aquí es incluso más
  directo: no hace falta derivar nada, porque antes de completar el pago simplemente no hay una
  `CustomerOrder` que consultar. Agregar un estado "borrador" obligaría a decidir qué CHECK
  constraints, índices y consultas existentes deben ignorarlo (por ejemplo, `list_orders` del
  staff, `compute_bill`, `release_table`), multiplicando la superficie de cambio sin necesidad.
- **Alternatives considered**: estado `"borrador"` visible solo al comensal, oculto para el staff
  — descartado por la razón de arriba, y porque contradice FR-003 en su forma más literal ("el
  sistema NO DEBE registrar el pedido" — un borrador en la base de datos ya es un registro).

## Decisión 4 — La garantía "nunca dos pedidos por una confirmación duplicada" se apoya en el mismo índice único que ya exige "una orden activa a la vez"

- **Decisión**: se sube el chequeo de "una orden activa por comensal" (spec 024, FR-005,
  hoy solo una validación de aplicación vía `SELECT` en `submit_cart`) a un índice único parcial
  real: `CREATE UNIQUE INDEX idx_active_order_per_participant ON customer_orders (participant_id)
  WHERE status NOT IN ('pagada', 'cancelada')`. `submit_cart` conserva el `SELECT` previo (para dar
  un `409` con mensaje claro en el camino feliz) y además captura `IntegrityError` alrededor del
  `db.commit()`, traduciéndolo al mismo `409` — mismo patrón de dos capas que ya usa
  `idx_pending_payment_attempt_per_order` (spec 024) y `idx_open_cart_per_participant`/
  `idx_active_session_per_table` (specs 007/015).
- **Rationale**: FR-013 exige que una confirmación duplicada (doble toque, reintento de red) nunca
  cree dos pedidos. Como cada envío de esta spec crea la orden y su primer intento juntos, "no
  puede haber una segunda orden no-terminal para el mismo comensal" ya es, por sí sola, exactamente
  la garantía que FR-013 pide — no hace falta una clave de idempotencia generada por el cliente ni
  ningún mecanismo nuevo. Un índice único de Postgres es correcto bajo concurrencia real (a
  diferencia de un `SELECT` seguido de un `INSERT`, vulnerable a una carrera entre dos peticiones
  casi simultáneas) sin requerir ningún `WITH FOR UPDATE` adicional.
- **Alternatives considered**: clave de idempotencia generada por el cliente (UUID por envío,
  deduplicada por el servidor) — descartada por añadir un concepto nuevo (almacenamiento y
  expiración de claves de idempotencia) para resolver un problema que la regla de negocio ya
  vigente ("una orden activa a la vez") resuelve gratis con un simple índice; `WITH FOR UPDATE`
  sobre alguna fila existente antes del `INSERT` — no aplica aquí porque, a diferencia de
  `confirm_order` (que bloquea una fila que ya existe), en este punto todavía no hay ninguna fila
  de la orden que bloquear.
- **Efecto colateral verificado, no un riesgo**: `participant_id` es `NULL` en las órdenes de
  mostrador/mesero (`channel` `counter`/`waiter`, `customer_order.py:38-40`) — un índice único de
  Postgres no considera dos `NULL` iguales, así que el índice nuevo no impone ningún límite sobre
  esas órdenes, solo sobre las del flujo QR con `participant_id` no nulo.

## Decisión 5 — Reintento sin volver a cargar el archivo (FR-012): sin mecanismo nuevo, es una consecuencia del diseño

- **Decisión**: si `POST /cart/submit` falla después de que el comprobante ya se subió a R2, el
  frontend conserva el `public_url` ya obtenido (en memoria del componente) y simplemente reintenta
  la misma llamada a `submit_cart(paymentMethodId, receiptUrl)` con la misma URL — sin volver a
  llamar al presign ni a subir nada de nuevo.
- **Rationale**: el archivo ya quedó durablemente en R2 desde el primer intento (la subida a R2 y
  la creación del pedido son dos pasos independientes, no una transacción conjunta) — reintentar
  solo la creación del pedido es simplemente volver a llamar el mismo endpoint con el mismo cuerpo,
  sin necesitar ningún estado nuevo del lado del servidor. No hay ningún comprobante "huérfano" que
  limpiar (FR-012): si `submit_cart` nunca llega a completar su `commit`, no queda ninguna fila en
  la base de datos que lo referencie; el archivo en R2 simplemente queda sin usar hasta que el
  reintento (u otro) lo reclame.
- **Alternatives considered**: que el backend registre el comprobante subido antes de que exista el
  pedido (para poder "recuperarlo" en un reintento sin que el cliente lo recuerde) — descartado por
  innecesario: el cliente ya tiene el dato (`public_url`) en memoria desde el momento en que
  `presign_payment_receipt` + la subida a R2 se completaron; agregar persistencia del lado del
  servidor para algo que el cliente ya conserva duplicaría el estado sin necesidad.

## Decisión 6 — Archivos de R2 huérfanos al cambiar de método de transferencia antes de enviar: costo aceptado, no una regresión

- **Decisión**: si el comensal sube un comprobante para un método de transferencia y luego cambia
  a otro método (o vuelve a la carta) antes de completar el envío, el archivo ya subido a R2 queda
  sin ningún registro que lo referencie — no se borra activamente.
- **Rationale**: el resto del sistema ya trata el borrado en R2 como best-effort y no garantizado
  en todos los caminos (ver `app/core/storage.py:delete_object`, y la propia spec 021, que corrige
  el *orden* de un borrado pero no introduce ninguna garantía nueva de limpieza). Construir un
  mecanismo de limpieza de archivos huérfanos específico para este caso sería una funcionalidad
  nueva no pedida por `spec.md` (que no la menciona) y fuera de proporción frente al costo real
  (un archivo de imagen sin referenciar en un bucket).
- **Alternatives considered**: borrar el archivo activamente al cambiar de método — descartado:
  requeriría rastrear del lado del cliente o del servidor cuál era el `public_url` "abandonado" y
  bajo qué condición borrarlo con seguridad (¿qué pasa si el comensal vuelve a subirlo?), complejidad
  no justificada por el problema (Principio V, no diseñar para un caso que la spec no pidió).

## Decisión 7 — Qué pasa con los pedidos `recibida` sin intento de pago que ya existan antes de este cambio

- **Decisión**: ninguna migración de datos ni reconciliación retroactiva. Los pedidos ya creados
  bajo el flujo de spec 024 (orden primero, pago después) que quedaron `recibida` sin ningún
  `OrderPaymentAttempt` siguen exactamente como están; el índice único parcial nuevo no los afecta
  (ya existen, no se vuelve a insertar nada para ellos) y `confirm_order` (spec 024, FR-017) ya los
  bloquea igual que a cualquier otro pedido sin pago confirmado.
- **Rationale**: Constitución Principio VII — ningún cambio retroactivo sobre datos ya existentes.
  Estos pedidos, si los hay, son datos operativos de un sistema pre-lanzamiento, no facturas
  emitidas, pero el mismo principio de no tocar retroactivamente aplica por defecto salvo decisión
  de negocio explícita, que `spec.md` no pide aquí.
- **Alternatives considered**: script de backfill que cancele o marque esos pedidos — descartado,
  no lo pidió `spec.md` y el propio `confirm_order` ya los deja inofensivos (nunca podrán avanzar a
  comanda sin un pago confirmado, con o sin este cambio).
