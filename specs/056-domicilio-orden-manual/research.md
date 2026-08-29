# Research: Habilitación del tipo de orden "Domicilio" en la creación manual de pedidos

Decisiones técnicas para implementar `specs/056-domicilio-orden-manual/spec.md`. El código vive en
dos repositorios hermanos de este (`pos-specs`): `../pos-backend` (FastAPI + SQLAlchemy 2.0 +
Alembic, multi-tenant por schema Postgres) y `../pos-heladeria` (Angular 21, signals). Todas las
citas de archivo:línea a continuación están verificadas contra el código vivo, no contra lo que
dice spec.md (que ya cita casi todo correctamente — se anota cualquier corrección puntual).

## Decisión 1 — Tipo de columna para `delivery_fee`: `Numeric(12, 2)`, mismo patrón que `Sale`

- **Decisión**: `delivery_fee: Mapped[Optional[Decimal]] = mapped_column(Numeric(12, 2),
  nullable=True)` en `CustomerOrder`. Ningún `server_default` ni default de aplicación — FR-006 y
  el Edge Case de spec.md son explícitos: un campo vacío se trata como faltante (Historia 2), nunca
  como `$0` implícito, así que la columna debe poder distinguir "no diligenciado" (`NULL`) de "cero
  explícito" (`0.00`).
- **Rationale**: `Numeric(12, 2)` es el tipo monetario ya usado en todo el proyecto —
  `Sale.subtotal`/`discount`/`tax`/`tip`/`total` (`app/models/sale.py:54-58`) y
  `SaleItem.unit_price`/`line_total` (líneas 124, 126) — todos `Mapped[Decimal] =
  mapped_column(Numeric(12, 2), ...)`. No existe ningún precedente de `Float` para dinero en esta
  base de código; usar otra cosa aquí introduciría el único campo monetario con un tipo distinto.
  Que los valores reales sean pesos colombianos sin decimales (spec.md, Assumptions) no cambia esta
  decisión — es exactamente la misma situación que ya tienen `Sale.total` y el resto de columnas
  monetarias, que también almacenan pesos sin parte decimal real dentro de un `Numeric(12,2)`.
- **Alternatives considered**: `Integer` (pesos enteros) — descartado por introducir un segundo
  tipo monetario en el modelo de datos sin ninguna necesidad real, rompiendo la consistencia con
  `Sale`. `Float` — descartado, nunca usado para dinero en este proyecto (riesgo de error de
  redondeo binario en un campo que se suma a un total facturado).

## Decisión 2 — `delivery_address` y `delivery_phone`: texto libre, mismo patrón que `customer_name`/`notes`

- **Decisión**: `delivery_address: Mapped[Optional[str]] = mapped_column(String(255),
  nullable=True)` (mismo largo que `customer_name`, `app/models/customer_order.py:49`) y
  `delivery_phone: Mapped[Optional[str]] = mapped_column(String(30), nullable=True)` (suficiente
  para cualquier formato de número telefónico colombiano con indicativo, sin necesidad del largo de
  `notes`, `String(500)`, línea 83).
- **Rationale**: spec.md (Assumptions) pide explícitamente el mismo nivel de validación que ya
  tienen otros campos de texto libre del sistema — no hay ninguna validación de formato de
  dirección real ni de número telefónico que implementar, así que el tipo de columna sigue el
  patrón más cercano ya existente (`customer_name`) en vez de inventar un tipo nuevo.
- **Alternatives considered**: modelar `delivery_address` como una entidad separada (línea,
  ciudad, barrio) — descartado, fuera de alcance de spec.md (que solo pide "la dirección" como un
  dato, no una estructura), y sin ningún requisito de geocodificación o reporte por componente de
  dirección.

## Decisión 3 — Restricción `delivery_fee >= 0` a nivel de base de datos

- **Decisión**: `CheckConstraint("delivery_fee IS NULL OR delivery_fee >= 0",
  name="ck_customer_order_delivery_fee_non_negative")`, mismo patrón ya usado para `status`
  (`app/models/customer_order.py:114-125`) y para `channel`/`order_type` (spec 055).
- **Rationale**: el Edge Case de spec.md ("¿Se puede escribir un valor de domicilio negativo? No")
  es una regla de datos, no solo de UI — sin este constraint, un valor negativo podría colarse por
  cualquier llamador futuro de `POST /orders` que no pase por la validación de `create_order`
  (defensa en profundidad, mismo criterio que ya aplica el proyecto a sus otros catálogos).

## Decisión 4 — Validación de campos obligatorios (FR-007) en `orders/service.py::create_order`, junto al guard de mesa existente

- **Hallazgo**: FR-002 (sin mesa para Domicilio) y la combinación canal×tipo de orden POS+DELIVERY
  **ya están completamente implementados desde spec 055** — nada que cambiar ahí:
  - `orders/service.py:140-146` ya rechaza con `422` cualquier `order_type in (TAKEAWAY, DELIVERY)`
    con `dining_table_id` no nulo.
  - `orders/service.py:58-63` ya incluye `DELIVERY` en el conjunto de tipos de orden permitidos
    para el canal `POS` (`frozenset({DINE_IN, TAKEAWAY, DELIVERY})`).
- **Decisión**: agregar un nuevo bloque de validación, inmediatamente después del guard de mesa
  existente (`orders/service.py:140-146`), específico de esta spec:
  ```python
  if data.order_type is OrderType.DELIVERY and (
      not (data.customer_name or "").strip()
      or not (data.delivery_address or "").strip()
      or data.delivery_fee is None
  ):
      raise HTTPException(422, "Un pedido a domicilio requiere nombre del cliente, "
                                "dirección y valor del domicilio.")
  ```
  `delivery_phone` queda fuera de esta condición a propósito (FR-008, opcional).
- **Rationale**: mismo criterio que la Decisión 4 de spec 055 (research.md) — `create_order` es el
  único punto donde estos campos llegan como datos arbitrarios de un llamador (`OrderCreate`); los
  otros dos caminos de creación de orden (`cart/service.py::submit_cart`, QR;
  `orders/consolidation.py::get_or_create_open_order`, mesero) nunca construyen órdenes `DELIVERY`
  y no necesitan pasar por esta validación.
- **Alternatives considered**: validar con un `field_validator`/`model_validator` de Pydantic en
  `OrderCreate` — descartado por el mismo motivo que spec 055 descartó validar la combinación
  canal×tipo en el schema: la obligatoriedad de estos campos depende de `order_type`, una regla de
  negocio condicional que ya vive junto a las otras reglas condicionales de `create_order`
  (`hold_for_payment`+`QR_MENU`, combinación canal×tipo, mesa+TAKEAWAY/DELIVERY) — mantenerlas
  todas en el mismo archivo evita dispersar la misma clase de validación entre dos capas.

## Decisión 5 — El valor del domicilio en el total facturado: 3 puntos coordinados, no solo `build_sale`

- **Hallazgo crítico** (no anticipado en spec.md, encontrado al auditar todos los cálculos de total
  que existen hoy en el checkout): agregar `delivery_fee` a la fórmula de `Sale.total` **no es un
  cambio de una sola línea**. Existen **tres cálculos de total independientes** en
  `app/api/v1/orders/checkout.py`, y los tres deben aprender sobre `delivery_fee` o el flujo de
  pago de una orden `DELIVERY` queda roto (no solo con un total mal mostrado, sino con un `422` real
  al aprobar el pago):
  1. **`sales/builder.py::build_sale`** (línea 132): `total = subtotal - Decimal(discount) +
     Decimal(tax) + Decimal(tip)` — el cálculo "oficial" que efectivamente persiste
     `Sale.total`/`Sale.subtotal` (líneas 165-166). Este es el que FR-011 pide extender.
  2. **`checkout.py::_order_total()`** (líneas 784-792) — suma `unit_price * quantity` sobre ítems
     no anulados, sin descuento/impuesto/propina/domicilio. Se usa en
     `confirm_cash_payment_attempt` (línea ~967) para el chequeo previo "`amount_received >=
     total`", **antes** de llamar a `build_sale`. Si no se actualiza, el mensaje de error de ese
     chequeo previo queda desalineado con el total real (aunque `build_sale` seguiría rechazando el
     pago insuficiente por su propio chequeo interno, línea 149) — inconsistencia de UX, no un
     agujero de corrección, pero sí una regresión visible.
  3. **`checkout.py::approve_payment_attempt`** (línea ~874) — computa su propio `total = sum(line.
     line_total for line in lines) - promo_discount - combo_discount`, usado en la línea ~881 para
     construir el **único pago automático** que se le pasa a `build_sale`
     (`payments=[PaymentIn(..., amount=total)]`). **Este es el punto de mayor riesgo real**: si no
     se le suma `delivery_fee`, el pago autogenerado para una orden `DELIVERY` aprobada por
     transferencia queda corto exactamente en el valor del domicilio, y el propio chequeo `if paid <
     total` de `build_sale` (línea 149, una vez que su `total` sí incluya `delivery_fee`) rechazaría
     ese pago con `422` — **`approve_payment_attempt` quedaría completamente roto para órdenes
     DELIVERY** si este punto se pasa por alto.
- **Decisión**: los tres puntos se actualizan juntos, en el mismo cambio:
  1. `build_sale(...)` gana un parámetro nuevo `delivery_fee: Decimal = Decimal("0")`; la fórmula
     de la línea 132 pasa a `total = subtotal - Decimal(discount) + Decimal(tax) + Decimal(tip) +
     Decimal(delivery_fee)`.
  2. Los 4 sitios que llaman a `build_sale` (todos en `checkout.py`, con el objeto `order` ya en
     alcance): `pay_order` (línea 280), `checkout_and_send` (línea 471), `approve_payment_attempt`
     (línea 876) y `confirm_cash_payment_attempt` (línea 995) pasan `delivery_fee=order.
     delivery_fee or Decimal("0")`.
  3. `_order_total()` (líneas 784-792) se extiende para sumar `order.delivery_fee or Decimal("0")`
     al final de su cálculo.
  4. El `total` local de `approve_payment_attempt` (línea ~874) se extiende de la misma forma antes
     de construir el `PaymentIn` automático.
- **Decisión adicional — `Sale` gana su propia columna `delivery_fee`** (`Numeric(12,2)`, nulable,
  sin default, mismo patrón que `discount`/`tax`/`tip`): aunque spec.md solo exige que el *total*
  incluya el valor (FR-011), no que la venta lo almacene como línea propia, seguir el patrón ya
  establecido por `discount`/`tax`/`tip` (cada término del total tiene su propia columna en `Sale`,
  no solo el total agregado) mantiene la factura auto-contenida y auditable sin depender de una
  consulta adicional a `CustomerOrder` para reconstruir de qué se compone un total ya emitido —
  consistente con Principio VII (una factura emitida es un hecho consumado; que sea auto-contenida
  evita que su desglose dependa de datos mutables de otra tabla).
- **Rationale**: sin esta auditoría de los 3 puntos, el cambio "obvio" (una línea en `build_sale`)
  habría dejado un bug de facturación real y silencioso en producción — exactamente el tipo de
  hallazgo que este research.md existe para capturar antes de escribir tasks.md.
- **Alternatives considered**: calcular `delivery_fee` dentro de `build_sale` leyendo
  `order.delivery_fee` directamente (sin parámetro explícito) — descartado: `build_sale` no recibe
  hoy el objeto `CustomerOrder` completo en todas sus formas de invocación de manera uniforme;
  pasar el valor ya resuelto como parámetro (igual que `discount`/`tax`/`tip`, que tampoco se leen
  de `order` dentro de `build_sale`) mantiene la función con la misma forma de dependencias que ya
  tiene, sin acoplarla a un nuevo atributo de `CustomerOrder`.

## Decisión 6 — Frontend: reusar el literal `'domicilios'` ya existente en `OrderTypeTab`, sin tocar el tipo

- **Hallazgo**: `OrderTypeTab` (`pos-terminal.store.ts:99`) ya es `'mesas' | 'domicilios' |
  'para-llevar'` — el valor `'domicilios'` existe desde antes (usado hoy únicamente por
  `pos-tables-panel.component.ts`, la pestaña "Domicilios" del panel general de Terminal de Mesas,
  que solo muestra un estado vacío, FR-014, fuera de alcance). `manual-order-page.component.ts`
  declara su propia instancia de `PosTerminalStore` (`providers: [PosTerminalStore]`, documentado
  en el propio archivo: "esta vista tiene su propia instancia de store, así que no hay ningún
  efecto cruzado entre ambas pantallas") — las dos pantallas nunca comparten el mismo signal en
  tiempo de ejecución pese a compartir el tipo.
- **Decisión**: conectar la pestaña "🛵 Domicilio" de `manual-order-page.component.ts` al literal
  `'domicilios'` ya existente, sin agregar un cuarto valor a `OrderTypeTab` ni introducir un signal
  paralelo. Mapeo: `'mesas'` ⇄ `DINE_IN`, `'para-llevar'` ⇄ `TAKEAWAY`, `'domicilios'` ⇄ `DELIVERY`.
- **Rationale**: mismo criterio que la Decisión 6 de spec 055 — reusar lo que el store ya modela en
  vez de duplicar el concepto; el aislamiento por instancia de store ya confirmado hace que esto
  sea seguro pese a que el mismo literal técnicamente "existe" en otra pantalla con otro propósito
  visual.
- **Alternatives considered**: agregar un cuarto valor al tipo (p. ej. `'domicilio-manual'`) para
  distinguirlo explícitamente del `'domicilios'` de `pos-tables-panel.component.ts` — descartado,
  añade una distinción sin ninguna consecuencia real dado el aislamiento por instancia ya
  confirmado, y complica innecesariamente cualquier código futuro que itere sobre `OrderTypeTab`.

## Decisión 7 — Nuevos signals en `PosTerminalStore`: `deliveryAddress`, `deliveryPhone`, `deliveryFee`

- **Decisión**: agregar tres signals nuevos a `PosTerminalStore`, mismo patrón que `customerName`
  (línea 314, `signal('')`): `readonly deliveryAddress = signal('')`, `readonly deliveryPhone =
  signal('')`, `readonly deliveryFee = signal<number | null>(null)`. `totals` (líneas 792-801) se
  extiende:
  ```ts
  readonly totals = computed(() => {
    const subtotal = this.subtotal();
    const discount = 0;
    const tax = 0;
    const deliveryFee = this.orderTypeTab() === 'domicilios' ? (this.deliveryFee() ?? 0) : 0;
    const total = Math.max(0, Math.round(subtotal - discount + tax + deliveryFee));
    return { subtotal, discount, tax, deliveryFee, total };
  });
  ```
  `createManualOrderFromDraft()` (líneas 1056-1095) gana una tercera rama:
  ```ts
  const esParaLlevar = this.orderTypeTab() === 'para-llevar';
  const esDomicilio = this.orderTypeTab() === 'domicilios';
  const tableId = this.selectedTableId();
  if ((!esParaLlevar && !esDomicilio && !tableId) || this.draftLines().length === 0) return false;
  if (esDomicilio && (
    !this.customerName().trim() || !this.deliveryAddress().trim() || this.deliveryFee() == null
  )) return false;
  ...
  const order = await this.api.createManualOrder({
    channel: 'POS',
    order_type: esDomicilio ? 'DELIVERY' : esParaLlevar ? 'TAKEAWAY' : 'DINE_IN',
    dining_table_id: (esParaLlevar || esDomicilio) ? null : tableId,
    customer_name: this.customerName().trim() || null,
    delivery_address: esDomicilio ? this.deliveryAddress().trim() : null,
    delivery_phone: esDomicilio ? (this.deliveryPhone().trim() || null) : null,
    delivery_fee: esDomicilio ? this.deliveryFee() : null,
    items,
    hold_for_payment: true,
  });
  ```
- **Rationale**: mantener estos tres campos como signals del store (no estado local del
  componente) sigue el mismo patrón que `customerName` — el store es quien construye el payload
  final en `createManualOrderFromDraft()`, así que necesita leer los tres directamente; y el guard
  defensivo dentro del propio método (no solo el botón deshabilitado del componente) sigue el mismo
  criterio ya usado para `tableId` en la línea 1061 — dos capas de protección para el mismo
  requisito (FR-007), consistente con cómo ya está protegido el caso de mesa faltante.
- **Alternatives considered**: mantener dirección/teléfono/valor como estado local de
  `manual-order-page.component.ts` y pasarlos como parámetros a `createManualOrderFromDraft()` —
  descartado por romper la simetría con `customerName` (ya un signal de store leído directamente
  por ese mismo método) sin ninguna ventaja; hubiera significado dos patrones distintos para datos
  con el mismo rol (campos de "quién/cómo entregar" en el borrador del pedido).

## Decisión 8 — `manual-order-page.component.ts`: habilitar pestaña, campos nuevos, y el hallazgo de `applyDefaultCustomerName()`

- **Habilitar la pestaña**: quitar `disabled`/`title="Todavía no disponible..."` del botón "🛵
  Domicilio" (líneas 136-143), conectado a `setOrderTypeTab('domicilios')` igual que las otras dos
  pestañas.
- **Campos nuevos**, visibles solo con `@if (store.orderTypeTab() === 'domicilios')`: "Cliente"
  (input simple, siempre editable, sin el patrón de solo-lectura+botón editar que sí usan "En
  Mesa"/"Para Llevar" — no hay nada que proteger con un toggle de edición cuando no existe ningún
  valor por defecto que preservar), "Dirección", "Teléfono", "Valor del domicilio" (input numérico,
  `min="0"`).
- **Bloque "Mesas" (buscador de mesa, spec 053)**: se oculta también con `orderTypeTab() ===
  'domicilios'` (ya se oculta hoy con `'para-llevar'`; se extiende la misma condición).
- **Botón "Confirmar y Enviar"** (`[disabled]`, líneas 234-238): gana un tercer disyunto:
  ```html
  [disabled]="
    store.cartEmpty() ||
    store.submitting() ||
    (store.orderTypeTab() === 'mesas' && !store.selectedTableId()) ||
    (store.orderTypeTab() === 'domicilios' && (
      !store.customerName().trim() || !store.deliveryAddress().trim() || store.deliveryFee() == null
    ))
  "
  ```
- **Bloque de totales** (líneas 221-230): gana una fila "Domicilio" (visible solo si
  `tot.deliveryFee > 0` o si la pestaña activa es `'domicilios'`, a definir en tasks.md el criterio
  exacto de visibilidad — decisión de detalle visual, no de comportamiento) antes de la fila
  "Total", leyendo `tot.deliveryFee` del `totals()` extendido (Decisión 7).
- **Hallazgo crítico**: `applyDefaultCustomerName()` (líneas 312-317) se llama hoy de forma
  **incondicional** en `ngOnInit`, `selectTable()`, el wrapper de `setOrderTypeTab()`, y —el punto
  que importa— **dentro de `confirm()` mismo, justo antes de llamar a
  `createManualOrderFromDraft()`** (línea 328):
  ```ts
  async confirm(): Promise<void> {
    this.applyDefaultCustomerName();
    const ok = await this.store.createManualOrderFromDraft();
    ...
  }
  ```
  Como FR-003 exige explícitamente que "Domicilio" **no** tenga ningún valor por defecto, este
  método debe dejar de rellenar "Consumidor final" cuando `orderTypeTab() === 'domicilios'` — de lo
  contrario, un campo "Cliente" vacío se sobrescribiría silenciosamente justo antes de enviar,
  anulando FR-003/FR-007 con código que ya existe, no con código ausente. Cambio: `if
  (this.store.orderTypeTab() === 'domicilios') return;` al inicio de `applyDefaultCustomerName()`.
- **Rationale**: ninguna decisión de diseño nueva más allá de lo que spec.md ya definió — el valor
  de este research es haber ubicado el punto exacto de código que, sin tocarlo, rompería
  silenciosamente el requisito "sin valor por defecto".

## Decisión 9 — `dining.interface.ts`: 3 campos opcionales nuevos, sin tocar `OrderType`

- **Decisión**: `OrderType` (línea 14) ya incluye `'DELIVERY'`, sin cambios. `OrderCreatePayload`
  (líneas 50-65) gana `delivery_address?: string | null; delivery_phone?: string | null;
  delivery_fee?: number | null;`. `DiningOrder` (líneas 191-214) gana los mismos tres campos como
  opcionales de respuesta, mismo patrón que `order_type?: string | null` (línea 194).
- **Rationale**: cambio puramente aditivo de forma — ningún consumidor existente de estas
  interfaces se rompe por campos opcionales nuevos (mismo criterio que spec 055 aplicó a
  `order_type`).

## Decisión 10 — Migración Alembic: mismo patrón `@for_each_tenant_schema` + `_has_table`, sin backfill

- **Decisión**: una migración nueva, encadenada al head real de `alembic/versions/` al momento de
  implementar (confirmar con `alembic heads`), que en un único `upgrade()` por schema de tenant:
  1. Agrega `delivery_address` (`String(255)`, nulable), `delivery_phone` (`String(30)`, nulable) y
     `delivery_fee` (`Numeric(12,2)`, nulable) a `customer_orders`.
  2. Agrega el `CheckConstraint` `ck_customer_order_delivery_fee_non_negative` (Decisión 3).
  3. Agrega `delivery_fee` (`Numeric(12,2)`, nulable) a `sales` (Decisión 5).
  - Mismo guard `_has_table(schema, "customer_orders")` / `_has_table(schema, "sales")` ya usado en
    la migración de spec 055 (`03c1cc5bfeb2_estandariza_canal_tipo_orden.py`) para saltar schemas
    de tenant aún no inicializados.
- **Sin backfill**: a diferencia de spec 055 (que sí necesitó reclasificar `channel`/`order_type`
  históricos), estas tres columnas nuevas son puramente aditivas y nulables — ningún pedido
  histórico es de tipo `DELIVERY` (ese valor no tenía ningún punto de creación real hasta esta
  spec), así que no existe ningún dato histórico que reclasificar; todo pedido/venta anterior a esta
  mejora simplemente queda con estos tres campos en `NULL`, sin necesidad de ningún `UPDATE`.
- **Downgrade**: simétrico — elimina el `CheckConstraint` y las 4 columnas nuevas (3 en
  `customer_orders`, 1 en `sales`). Sin remapeo de datos que revertir (a diferencia del `channel`
  de spec 055), porque ninguna columna existente cambia de significado.
- **Rationale**: mismo patrón ya validado por el proyecto (spec 055), aplicado a un caso más simple
  (sin backfill) — Principio VIII exige documentar migración y rollback, ambos quedan explícitos
  arriba.

## Decisión 11 — Characterization tests afectados

- **`manual-order-page.component.spec.ts:148-157`**: el caso `'"Para Llevar" ya no es un
  placeholder deshabilitado; "Domicilio" sigue siéndolo (FR-008, FR-012)'` (de spec 055) queda
  **contradicho** por esta spec — se reescribe para reflejar que ambas pestañas quedan habilitadas.
  Es un cambio de comportamiento explícitamente autorizado por esta spec (FR-001), no una
  regresión no autorizada — cumple el requisito de Principio III de que exista spec + decisión de
  negocio que justifique tocar un test protegido.
- **Casos nuevos a agregar** (no reemplazan comportamiento protegido, extienden cobertura):
  - Backend `test_orders_service.py`: rechazo `422` de una orden `DELIVERY` sin `customer_name`,
    sin `delivery_address`, o sin `delivery_fee` (Decisión 4); aceptación con los tres presentes y
    `delivery_phone` ausente.
  - Backend `test_orders_checkout.py`: `Sale.total` de una orden `DELIVERY` incluye
    `delivery_fee` en los 4 caminos de checkout (Decisión 5) — caso explícito para
    `approve_payment_attempt` (el punto de mayor riesgo encontrado en Decisión 5).
  - Frontend `pos-terminal.store.spec.ts`: siguiendo el patrón ya existente en la línea 1067 (caso
    de "Para Llevar" → `TAKEAWAY`), un caso equivalente `'domicilios'` → `order_type: 'DELIVERY'`
    con `delivery_address`/`delivery_phone`/`delivery_fee` en el payload; caso de `totals()` con
    `deliveryFee` sumado solo cuando la pestaña activa es `'domicilios'`.
  - Frontend `manual-order-page.component.spec.ts`: visibilidad de los 4 campos nuevos solo con
    "Domicilio" seleccionado; campo "Cliente" vacío (sin "Consumidor final") y sin `readOnly`;
    botón "Confirmar" deshabilitado con cualquiera de los 3 campos obligatorios vacío.
