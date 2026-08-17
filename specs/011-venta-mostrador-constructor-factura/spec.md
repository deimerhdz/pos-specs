# Feature Specification: Venta de mostrador, constructor de venta compartido (`build_sale`) y emisión de factura interna

**Feature Branch**: `011-venta-mostrador-constructor-factura`

**Created**: 2026-08-16

**Status**: Draft

**Naturaleza de esta spec**: **ingeniería inversa / characterization spec**. No describe una
funcionalidad nueva: documenta el comportamiento que el sistema **ya tiene hoy** en
`pos-backend/app/api/v1/sales/builder.py` (el constructor de venta compartido por los cuatro
caminos de cobro), `app/api/v1/sales/service.py` (el camino de mostrador propiamente dicho,
`checkout`) y `app/api/v1/invoices/service.py` (la emisión de factura interna), para que sirva de
contrato formal de cara a la modernización (Principio I y Principio III de la
[Constitución](../../.specify/memory/constitution.md)). Es el dominio de "cuánto cuesta esta
venta, quién puede tocar ese número, y cómo queda su factura sin que nadie tenga que acordarse de
pulsar un botón": la fórmula única del total, el candado del pago contra el turno de caja, la
regla `[PROTEGIDA]` A-14a que unificó la emisión de factura en un solo punto tras perder facturas
reales en producción, y la regla propia **A-11**, de la que esta spec es dueña — el mecanismo de
descuento manual del cajero, endurecido por decisión de negocio explícita hasta la prohibición
total. Incluye una regla protegida (A-14a), una regla propia con decisión de negocio cerrada
(A-11), un bug matemático sin condicionalidad (A-14, con una pieza de contexto A-49 pendiente de
ratificación real), dos anomalías accidentales de bajo riesgo actual (A-12, A-43), una decisión
fiscal intencional con riesgo residual (A-41), y una anomalía `PENDIENTE` sin decisión de negocio
(A-29, porción de mostrador).

**Input**: User description: "Spec de ingeniería inversa: documenta el comportamiento EXISTENTE
de la venta de mostrador, el constructor de venta compartido por los cuatro caminos de cobro, y
la emisión de factura interna del sistema POS Heladería, tomado de `reglas-de-negocio.md`
(RN-VENTA-01 a RN-VENTA-17, RN-FACT-01 a RN-FACT-07) y de `registro-de-anomalias.md` (A-11, A-12,
A-14, A-14a, A-29, A-41, A-43, A-49), para que sirva de contrato en la modernización."

## User Scenarios & Testing *(mandatory)*

<!--
  Cada escenario documenta un comportamiento OBSERVADO en `sales/builder.py`, `sales/service.py`
  e `invoices/service.py`, no uno deseado — con la única excepción explícita de A-11 (User Story
  4), donde el propio alcance de esta spec es especificar el mecanismo endurecido que el negocio
  ya decidió, no el estado actual sin freno. Las demás anomalías se marcan inline con su
  tratamiento acordado (`registro-de-anomalias.md`).
-->

### User Story 1 - El total de la venta es una fórmula única, sin redondeo intermedio, y nunca negativo (Priority: P1)

`build_sale` calcula el total como `subtotal − descuento + impuesto + propina`, en ese orden
aritmético exacto sobre `Decimal` puro, sin ningún paso de redondeo intermedio. Si el resultado
es negativo, la venta se rechaza antes de aceptar ningún pago.

**Why this priority**: es el número que define cuánto le cobra el negocio al cliente y cuánto
entra al cajón — un error aquí es el núcleo económico de toda la spec, y el docstring del propio
módulo confirma que existir como constructor único es deliberado.

**Independent Test**: se puede probar invocando `build_sale(db, lines=[...], shift=..., cashier=...,
payments=[...], discount=..., tax=..., tip=...)` con distintas combinaciones de subtotal,
descuento, impuesto y propina, y verificando el total resultante o el `422` de total negativo, sin
pasar por HTTP ni por cocina.

**Acceptance Scenarios**:

1. **Given** 2 helados a $8.000 y 1 topping a $2.000 (subtotal $18.000), descuento manual $1.000 +
   promoción automática $500 (descuento total $1.500), impuesto $0, propina $2.000, **When** se
   arma la venta, **Then** el total resulta $18.000 − $1.500 + $0 + $2.000 = $18.500, calculado en
   un único paso aritmético sobre `Decimal` (`RN-VENTA-02`, evidencia: `sales/builder.py:118-132`).
2. **Given** una venta con `items: []`, **When** se invoca `build_sale`, **Then** el sistema
   rechaza con `409` "La venta no tiene ítems cobrables", sin crear ninguna `Sale`, `SaleItem` ni
   `Payment` (`RN-VENTA-01`, evidencia: `sales/builder.py:97-98`).
3. **Given** un subtotal de $5.000 con un descuento manual de $9.000, **When** se calcula el
   total, **Then** el sistema rechaza con `422` "El total no puede ser negativo" antes de evaluar
   ningún pago (`RN-VENTA-03`, evidencia: `sales/builder.py:132-136`).

---

### User Story 2 - El dinero: el pago debe cubrir el total sin excepción, el vuelto solo sale de efectivo, y solo se cobra contra un turno abierto (Priority: P1)

El pago recibido debe cubrir el total exacto o más — no existe cobro parcial. Si el pago incluye
efectivo de sobra, el sistema calcula vuelto; si lo que sobra viene de un medio electrónico
(tarjeta, transferencia), se rechaza. Cualquier cobro exige un turno de caja `open` vigente.

**Why this priority**: mismo mecanismo exacto que rige el cierre de mesa (spec 010, User Story 4)
— es el segundo bloque de reglas con impacto económico directo e inmediato: dinero real mal
calculado o mal repartido en el mismo cobro.

**Independent Test**: se puede probar invocando `build_sale` con distintas combinaciones de pago
(insuficiente, con sobra en efectivo, con sobra electrónica) sobre un total conocido, y por
separado invocando `ensure_open_shift` sobre un turno `closed`, sin depender de otros módulos.

**Acceptance Scenarios**:

1. **Given** un total de $18.500, **When** se paga con un único pago en efectivo de $18.000,
   **Then** el sistema rechaza con `422` "El pago (18000) no cubre el total (18500)"; no se
   comitea nada (`RN-VENTA-04`, evidencia: `sales/builder.py:138-153`).
2. **Given** un total de $10.000 pagado con tarjeta por $15.000 sin efectivo, **When** se cobra,
   **Then** el sistema rechaza con `422` "Los pagos que no son en efectivo (15000) no pueden
   superar el total (10000): el vuelto solo sale del efectivo" (`RN-VENTA-05`, evidencia:
   `sales/builder.py:154-163`).
3. **Given** el mismo total de $10.000 pagado con tarjeta $10.000 + efectivo $5.000, **When** se
   cobra, **Then** el sistema acepta el pago, fija `paid_amount=15.000` y `change_given=5.000` —
   ese vuelto se descuenta del efectivo esperado en el arqueo del turno, no se resta bruto
   (`RN-VENTA-06`, evidencia: `sales/builder.py:165-172`, comentario "RF-029").
4. **Given** un turno de caja con `status="closed"`, **When** se intenta cobrar contra él,
   **Then** el sistema rechaza con `409` "El turno de caja está cerrado", antes de armar ninguna
   línea (`RN-VENTA-07`, evidencia: `sales/builder.py:60-64`, función `ensure_open_shift`).

---

### User Story 3 - A-14a [PROTEGIDA]: toda venta pagada emite automáticamente su factura, en la misma transacción de base de datos (Priority: P1)

`build_sale` cierra sus totales, marca la venta `paid`, y en la misma transacción invoca
`issue_for_sale` — no existe ningún botón separado para facturar. Si cualquier paso posterior
falla (por ejemplo, el descuento de inventario del camino de mostrador), el rollback deshace venta
y factura juntas. La numeración es un consecutivo estrictamente secuencial y sin huecos por
prefijo, protegido con `SELECT...FOR UPDATE` sobre el contador, y la emisión es idempotente por
`sale_id`.

**Why this priority**: es la regla `[PROTEGIDA]` central de esta spec — dos testigos (código +
`memoria-historica.md` #6, 2026-07-29, commit `27711065`, Deimer Hernandez): "hay cuatro formas de
cobrar... emitir por separado en cada una garantizaba que alguna se quedara fuera — que es
exactamente lo que pasaba" — el propio commit cita el resultado real: "20 ventas reales, cero
facturas". **Confirmado en P11** (entrevista de negocio): el `pay_order` legado (spec 008) sigue
sin uso real hoy — lo que el negocio usa para cobrar por partes es la cuenta dividida (spec 010),
no ese endpoint. Los otros tres caminos (mostrador aquí, unificado y dividido en spec 010) quedan
confirmados como usuarios reales de `build_sale`, lo que cierra a favor la duda que dejó abierta
`RN-FACT-06` sobre cobertura parcial del análisis original.

**Independent Test**: se puede probar completamente ejecutando `python -m app.scripts.test_facturacion`
contra un `pos-backend` en ejecución — el script cubre los cinco invariantes con datos desechables
propios y los borra al terminar.

**Acceptance Scenarios**:

1. **Given** una venta recién armada y pagada con `build_sale`, **When** se inspecciona la venta
   inmediatamente después, **Then** ya tiene una `Invoice` asociada, con el prefijo del tenant y
   copiando el total exacto de la venta — no existe estado intermedio "pagada sin factura"
   (`RN-FACT-01`, evidencia: `sales/builder.py:174-179`). **Verificado por** `test_facturacion.py`,
   caso 1: "cobrar emite factura".
2. **Given** dos ventas cobradas consecutivamente contra el mismo prefijo, **When** se inspeccionan
   sus números de factura, **Then** avanzan sin huecos ni repeticiones (`RN-FACT-03`, evidencia:
   `invoices/service.py:30-42`). **Verificado por** `test_facturacion.py`, caso 2: "el consecutivo
   avanza sin huecos" (`[2, 3]` tras el primero).
3. **Given** una venta cuyo pago es insuficiente, **When** `build_sale` lanza `422` a mitad de la
   construcción, **Then** el rollback deshace todo — ni venta ni factura quedan persistidas, y el
   contador del prefijo no avanza (`RN-FACT-01`, `RN-FACT-03`). **Verificado por**
   `test_facturacion.py`, caso 3: "un cobro fallido NO consume número" y "ni deja facturas
   huérfanas".
4. **Given** una venta ya facturada, **When** se invoca `issue_for_sale` una segunda vez sobre ella
   (fuera de la ruta normal de cobro), **Then** devuelve la factura ya existente en vez de chocar
   con la constraint única de `sale_id` (`RN-FACT-02`, evidencia: `invoices/service.py:45-58`).
   **Verificado por** `test_facturacion.py`, caso 4: "emitir dos veces devuelve la misma factura".
5. **Given** un `split` de dos comensales que produce dos ventas independientes (mecanismo de la
   spec 010, no de esta), **When** se inspeccionan sus facturas, **Then** cada venta tiene su
   propia factura — nunca una factura compartida por pedido: la unidad de facturación es la
   **venta**, no el pedido (`RN-FACT-01`). **Verificado por** `test_facturacion.py`, caso 6: "dos
   ventas (split) → dos facturas distintas".
6. **Given** un tenant sin `invoice_prefix` configurado, **When** se cobra una venta, **Then** la
   factura se emite igual, con `prefix=""` — la ausencia de configuración no bloquea la emisión
   (`RN-FACT-04`, evidencia: `sales/router.py:53`, `tenant.invoice_prefix or ""`). **Verificado
   por** `test_facturacion.py`, caso 5.

**Especificación de esta spec**: `RN-FACT-01`, `RN-FACT-02` y `RN-FACT-03` se fijan como
invariantes de test obligatorios en cualquier reimplementación — **especificar tal cual, no
tocar**, precisamente porque el propio historial documenta el costo real de no tenerlos (facturas
faltantes en producción).

---

### User Story 4 - A-11 [regla propia — dueña de la spec]: el cajero no debe poder aplicar descuento manual en absoluto (Priority: P1)

**Comportamiento actual (evidencia)**: el campo `discount` de `SaleCreate` (`sales/schemas.py:63`)
no declara ningún tope superior (`le=`) — solo `ge=0`. El único freno que existe hoy es que el
total resultante no quede negativo (`sales/builder.py:132-136`, User Story 1). El endpoint
`POST /sales` (`create_sale`) solo exige `Depends(get_current_user)` — cualquier cajero
autenticado, sin rol elevado — a diferencia de `create_payment_method` en el mismo router
(`sales/router.py:31`), que sí exige `Depends(require_tenant_admin)`. Es una asimetría verificable
entre dos endpoints del mismo router que protegen con distinto rigor.

**Comportamiento objetivo, ya decidido por negocio**: en la primera ronda de entrevista (P5), tras
repregunta, el negocio cerró el tratamiento en un sentido más estricto que un tope numérico: "me
preocupa que un cajero regale una venta por error, no debería poder aplicar descuento de forma
manual" — prohibición total, no un límite configurable. En la ronda 3 (simulada, P30), se cerró
explícitamente el alcance: la prohibición aplica **a los tres caminos de cobro por igual**
(mostrador aquí; mesa unificada/dividida en spec 010; `pay_order` legado en spec 008, sin uso real
confirmado en P11) — "no tendría sentido prohibirlo en uno y dejarlo abierto en otro, el cajero
simplemente usaría el camino que se lo permite".

**Why this priority**: es la regla propia de mayor impacto económico de esta spec — un cajero con
acceso al sistema hoy puede regalar una venta completa sin que quede señalado, y la decisión de
negocio que lo cierra ya está tomada, solo falta su mecanismo de aplicación.

**Independent Test**: se puede verificar hoy inspeccionando `discount` en `SaleCreate`
(`sales/schemas.py:63`) y la ausencia de dependencia de rol en `create_sale`
(`sales/router.py:46-53`), contrastando contra `create_payment_method` (línea 31); el estado
objetivo se prueba invocando el checkout con `discount > 0` desde una cuenta de rol cajero y
verificando el rechazo, una vez implementado.

**Acceptance Scenarios**:

1. **Given** el esquema `SaleCreate` actual, **When** se inspecciona el campo `discount`, **Then**
   solo exige `ge=0` — ningún límite superior propio (evidencia actual, no el contrato objetivo).
2. **Given** el endpoint `POST /sales` actual, **When** se compara su dependencia de autorización
   con `POST /sales/payment-methods` del mismo router, **Then** `create_sale` solo exige
   `Depends(get_current_user)` mientras que `create_payment_method` exige
   `Depends(require_tenant_admin)` — asimetría verificable, sin justificación documentada
   distinta de un descuido (evidencia actual).
3. **Given** la decisión de negocio cerrada (P5 + P30), **When** un cajero (sin rol elevado)
   intenta cobrar una venta de mostrador con `discount > 0`, **Then** el contrato objetivo de esta
   spec exige que el sistema rechace la operación — el descuento manual del cajero queda prohibido
   sin excepción, no limitado a un tope (`RN-VENTA` — regla propia de esta spec, sin número
   `RN-VENTA-XX` asignado en el reconocimiento original porque nace de la entrevista de negocio,
   no de una lectura de código previa).
4. **Given** la misma prohibición, **When** se aplica a los otros dos caminos de cobro (unificado
   y dividido, spec 010), **Then** el mecanismo es el mismo — esta spec es la dueña del mecanismo
   compartido; spec 010 solo documenta que queda alcanzada por la misma decisión, sin
   respecificarlo.

**Tratamiento acordado**: corregir en fase de modernización sobre el mecanismo objetivo ya
decidido — prohibición total del descuento manual de cajero, no un tope superior configurable. **No
retroactivo**: no se pueden recalcular ventas ya cobradas con descuento excesivo; solo se puede
auditar el histórico para dimensionar el impacto ya ocurrido, con una consulta que hoy no existe.

---

### User Story 5 - Cómo se arma y valora cada línea cobrable de mostrador (Priority: P2)

Cada ítem del carrito de mostrador se resuelve y valora antes de construir la venta: las opciones
seleccionadas deben estar activas y pertenecer a la variante (los IDs repetidos se deduplican), el
precio de la línea es el de la variante más la suma de `extra_price` de las opciones sin redondeo
adicional, los ítems que forman parte de un combo se cobran a precio normal individual con su
ahorro calculado aparte como descuento, y el descuento total de mostrador es la suma de tres
fuentes: el manual del cajero, las promociones automáticas (percent/fixed) y el ahorro de combos.

**Why this priority**: es la mecánica de valoración que alimenta el total de User Story 1 — de
menor riesgo económico directo que las reglas de dinero (P1), pero necesaria para reproducir el
número exacto que llega a `build_sale`.

**Independent Test**: se puede probar invocando `checkout` con un carrito que combine ítems
normales, ítems con opciones repetidas/inactivas, y un `combo_id`, e inspeccionando el `discount`
y las líneas resultantes antes de que `build_sale` calcule el total.

**Acceptance Scenarios**:

1. **Given** un combo "helado + topping" cuyo precio normal combinado es $11.000 y se vende en
   $9.000 (ahorro $2.000), **When** se cobra, **Then** se generan dos `SaleLine` a precio normal
   ($8.000 y $3.000, subtotal $11.000) y `combo_discount_for_lines` añade $2.000 al descuento total
   — el total refleja los $9.000 reales (`RN-VENTA-12`, evidencia: `sales/service.py:46-65,99-104`).
2. **Given** un descuento manual de $500 y una promoción automática del 10% sobre una línea de
   $8.000 ($800), sin combos, **When** se calcula el descuento total, **Then** resulta $1.300 —
   suma de las tres fuentes (`RN-VENTA-13`, evidencia: `sales/service.py:99-119`).
3. **Given** un carrito cuyas líneas usan exactamente un `combo_id`, **When** se calcula
   `final_promotion_id`, **Then** registra ese combo específico; con cero combos o dos combos
   distintos, cae al resultado de la evaluación general de promociones, que puede ser `None`
   (`RN-VENTA-14 [DUDOSA]`, evidencia: `sales/service.py:104` — ver User Story 10, A-29).
4. **Given** una línea con IDs de opción repetidos y una opción inactiva mezclados, **When** se
   valida la selección, **Then** los IDs repetidos se deduplican y la opción inactiva se rechaza
   antes de calcular precio (`RN-VENTA-15`, evidencia: `sales/service.py:71-74`,
   `catalog/line_pricing.py:44-52`).
5. **Given** una variante base y sus opciones seleccionadas válidas, **When** se calcula el precio
   de la línea, **Then** resulta precio de variante + suma de `extra_price`, sin ningún redondeo
   adicional (`RN-VENTA-16`, mismo mecanismo que `RN-CAT-01`, evidencia:
   `catalog/line_pricing.py:191-196`, `sales/builder.py:54-55`).

---

### User Story 6 - A-12 [ACCIDENTAL]: la venta de mostrador cobra variantes sin receta ni opción configurada, sin bloquear ni descontar nada (Priority: P2)

A diferencia de la confirmación de pedidos por QR (que bloquea con `409` si no hay ningún plan de
consumo, `RN-CAT-34`, spec 003), el descuento de inventario del camino de mostrador no bloquea el
cobro si la variante no tiene ninguna regla de consumo definida: la venta se cobra y factura sin
generar ningún movimiento de inventario.

**Why this priority**: el propio comentario del código lo describe como un "agujero" conocido y
creciente ("con slots el agujero crece"); el equipo mismo documenta la intención contraria a este
comportamiento — no requiere testigo de negocio adicional para confirmarse como no deseado, solo
para dimensionar su alcance.

**Independent Test**: se puede probar invocando `deduct_sale` sobre una venta cuya única línea es
una variante sin receta ni opción con consumo configurado, y verificando que no se lanza ningún
error ni se genera ningún `record_movement`.

**Acceptance Scenarios**:

1. **Given** una variante activa sin receta ni opción configurada que consuma inventario,
   **When** se cobra en mostrador, **Then** la venta se cobra y factura con normalidad, sin
   generar ningún movimiento de inventario y sin ningún rechazo (`RN-VENTA-11`, evidencia:
   `sales/consumption.py:46-51`).
2. **Given** el mismo escenario pero por el camino de confirmación de pedidos QR, **When** se
   confirma el pedido, **Then** el sistema bloquea con `409` (`RN-CAT-34`, spec 003) —
   contraste directo con el camino de mostrador, que no tiene esa guarda.
3. **Given** la entrevista de negocio (P8, administrador de catálogo/inventario), **When** se
   pregunta si todos los productos activos tienen receta cargada, **Then** la respuesta confirma
   sospecha de que varios no la tienen, con una repregunta que estima un rango de 6 a 20 productos
   activos sin receta — cifra aproximada, no verificada por consulta real.

**Tratamiento acordado**: corregir en fase de modernización, replicando el bloqueo de `RN-CAT-34`
(409 si el plan de consumo agregado es vacío) también en el camino de mostrador. **No
retroactivo**: no hay forma de reconstruir el inventario no descontado de ventas pasadas.

---

### User Story 7 - A-14 [BUG A SECAS]: el número de factura se formatea distinto en Python y en SQL, y ambos divergen desde el consecutivo 1.000.000 (Priority: P2)

`Invoice.full_number` (Python, `invoices/schemas.py:40-43`, `f"{self.prefix}{self.number:06d}"`)
nunca trunca un número que exceda 6 dígitos. La reconstrucción SQL usada para buscar por
referencia (`sales/service.py:142-172`, `func.lpad(cast(Invoice.number, String), 6, "0")`) sí
trunca a 6 caracteres, descartando el dígito sobrante. Ambas fórmulas coinciden exactamente por
debajo de 1.000.000 y divergen matemáticamente a partir de ahí, sin ninguna condición de carrera
de por medio.

**Why this priority**: es un bug matemáticamente cierto en el código actual — no depende de datos
ni de negocio para confirmarse, solo de aritmética. La verificación técnica del 2026-08-16
(`entrevista-negocio.md` §8) recuperó su prioridad tras inspeccionar `app/core/scheduler.py` y
confirmar que **no existe ningún mecanismo de reinicio anual del consecutivo** — solo registra
`sweep_orphan_table_sessions` y `expire_promotions`, ninguno relacionado con facturación.

**Independent Test**: se puede probar de forma determinista con el caso `1234567` →
`f"{1234567:06d}"` produce `"1234567"` (Python no trunca) mientras que
`lpad('1234567', 6, '0')` produce `"234567"` (SQL trunca) — `"FAC-123456"` (lo que el Python
generaría) nunca se encontraría buscando por la reconstrucción SQL.

**Acceptance Scenarios**:

1. **Given** una factura con `number=1234567` y `prefix="FAC-"`, **When** se calcula
   `Invoice.full_number` en Python, **Then** resulta `"FAC-1234567"` (7 dígitos, sin truncar)
   (evidencia: `invoices/schemas.py:40-43`).
2. **Given** la misma factura, **When** se reconstruye el número vía `list_sales_query` en SQL,
   **Then** resulta `"FAC-234567"` (6 dígitos, truncado) — un usuario que busque por el número que
   ve en el ticket (`"FAC-1234567"`) no encontrará esa venta con la búsqueda SQL
   (`RN-FACT-07 [DUDOSA] → confirmada como divergencia real por esta spec`, evidencia:
   `sales/service.py:142-172`).
3. **Given** la verificación técnica del 2026-08-16, **When** se inspecciona
   `app/core/scheduler.py`, `invoices/service.py` (`_next_number`) y `app/models/invoice.py`,
   **Then** no aparece ningún job, endpoint admin ni script de reinicio del consecutivo — la única
   vez que `next_number` vale `1` es al crear un prefijo nuevo (`invoices/service.py:37`), no como
   reinicio periódico (A-49). El testimonio que en la ronda 3 (simulada, P29) descarta la premisa
   de "se reinicia cada año" (P10, primera ronda) es **simulado**, no un testigo de negocio real —
   pendiente de ratificación con la gestoría antes de cerrarse en sentido estricto.
4. **Given** la búsqueda por referencia de factura (`invoice_reference` en `GET /sales`),
   **When** se filtra con `ILIKE '%42%'` sobre el número reconstruido, **Then** también coincide
   con `"FAC-004200"` o cualquier factura que contenga "42" en cualquier posición, no solo la
   que empieza por ese número (`RN-FACT-05 [DUDOSA]`, evidencia: `sales/service.py:165-172`).

**Tratamiento acordado**: adoptar la corrección segura, independiente de si existe o no un
reinicio real — ampliar el padding a **7+ dígitos** en ambos lados (`Invoice.full_number` en
Python y la reconstrucción SQL en `list_sales_query`), sin depender de una política de reinicio
que la verificación técnica no encontró en el código. **No retroactivo** en el sentido de que los
números ya emitidos con 6 dígitos o menos no cambian de valor; sí requiere migrar la lógica de
búsqueda antes de que el primer tenant cruce el millón.

---

### User Story 8 - A-41: los impuestos del ticket de mostrador quedan fijos en $0, a propósito (Priority: P2)

El campo de impuestos editable de la terminal se deprecó deliberadamente: el total del ticket ya
no calcula impuestos, siempre queda en 0 (`const tax = 0`, comentario propio: "Impuestos
deprecado: se guarda/calcula siempre en 0").

**Why this priority**: es la única entrada `INTENCIONAL` de todo el registro de anomalías con una
decisión de negocio explícitamente pendiente encima — confirmado como correcto en la entrevista
(P2), pero con el mayor riesgo fiscal/legal residual del reconocimiento pese a su bajo impacto
técnico actual.

**Independent Test**: se puede verificar por inspección directa de
`pos-heladeria/src/app/modules/tables/services/pos-terminal.store.ts:497` (`const tax = 0`), sin
necesidad de reproducir un cobro real.

**Acceptance Scenarios**:

1. **Given** cualquier venta armada desde la terminal de mesas, **When** se calculan sus totales,
   **Then** `tax` siempre resulta `0`, sin importar el subtotal o la categoría de los productos
   (evidencia: `pos-terminal.store.ts:497`).
2. **Given** el commit histórico que introdujo este cambio (`memoria-historica.md` #10,
   2026-08-03, commit `8166ea9e`, Leonardo Gomez, mensaje "fix(tables): deprecate editable tax
   field in table terminal"), **When** se contrasta con la entrevista de negocio (P2), **Then** el
   negocio confirma que es correcto — el ticket no discrimina impuesto, por decisión deliberada.

**Especificación de esta spec**: se especifica **tal cual** el comportamiento actual (`tax=0`
fijo) como punto de partida — pero sin cerrar la especificación de facturación fiscal completa;
queda como la pregunta de mayor riesgo legal/fiscal residual del registro, a priorizar en la
próxima conversación con negocio pese a su bajo impacto técnico hoy.

---

### User Story 9 - A-43 [ACCIDENTAL]: la idempotencia de emisión de factura no tiene lock ni captura de `IntegrityError` (Priority: P3)

`issue_for_sale` se documenta como idempotente ("si la venta ya tiene factura, devuelve la
existente"), pero el patrón real es `SELECT` seguido de `INSERT` sin `with_for_update()` ni
captura de `IntegrityError`. Bajo dos llamadas concurrentes para la misma venta, la segunda podría
lanzar una excepción no controlada en vez de devolver la factura existente.

**Why this priority**: hoy solo existe un llamador (`build_sale`, dentro de la misma transacción
de cobro, User Story 3), lo que limita la exposición real a cero — es un riesgo de diseño
prospectivo, no un incidente activo.

**Independent Test**: se puede verificar por inspección de código — el patrón `SELECT`+`INSERT`
de `issue_for_sale` (`invoices/service.py:54-58,60-76`) sin `with_for_update()` — sin necesidad de
reproducir la condición de carrera con un segundo llamador real, porque no existe todavía.

**Acceptance Scenarios**:

1. **Given** el código de `issue_for_sale`, **When** se inspecciona la consulta de existencia
   previa a `INSERT`, **Then** no usa `with_for_update()` — es un `SELECT` simple sin lock de fila
   (evidencia: `invoices/service.py:54-58`).
2. **Given** el único llamador conocido hoy (`build_sale`, dentro de la misma transacción de
   cobro), **When** se traza el flujo completo, **Then** no hay ningún segundo camino que invoque
   `issue_for_sale` fuera de esa transacción — la exposición real de la condición de carrera es
   hoy cero (evidencia: `sales/builder.py:176-178`, único llamador en todo el código).

**Tratamiento acordado**: corregir en fase de modernización **solo si se introduce un segundo
llamador** fuera de la transacción de `build_sale`; documentar como precaución de diseño mientras
tanto, sin acción correctiva inmediata.

---

### User Story 10 - A-29 (porción mostrador) [PENDIENTE]: con dos o más combos distintos, se pierde la trazabilidad del combo específico (Priority: P3)

Si las líneas cobradas en una venta de mostrador usan exactamente un combo distinto,
`promotion_id` de la venta registra ese combo. Con cero combos, o con dos o más combos distintos,
el sistema no registra ningún combo específico — cae al resultado de la evaluación general de
promociones, que puede ser `None`. El descuento monetario de todos los combos se suma
correctamente al total en cualquier caso; solo se pierde la trazabilidad en reportes agrupados por
promoción/combo específico.

**Why this priority**: mismo mecanismo documentado también en las specs 008 y 010 para sus
respectivos caminos de cobro — sin impacto práctico confirmado (P21, entrevista de negocio: el
negocio no revisa hoy ningún reporte de ventas por combo/promoción). Prioridad P3, la más baja de
esta spec.

**Independent Test**: se puede probar cobrando una venta de mostrador cuyas líneas incluyan dos
`combo_id` distintos, e inspeccionando que `promotion_id` de la `Sale` resultante queda `None` (o
el resultado de `promotions.evaluate`) pese a que el descuento monetario de ambos combos sí se
sumó al total (mismo mecanismo ya cubierto en User Story 5, escenario 3).

**Acceptance Scenarios**:

1. **Given** un carrito de mostrador cuyas líneas usan dos `combo_id` distintos, **When** se
   calcula `final_promotion_id`, **Then** no queda ligado a ninguno de los dos — cae al resultado
   de `promotions.evaluate` sobre las líneas sin combo, que puede ser `None`, aunque el descuento
   monetario de ambos combos sí se sumó correctamente al total (`RN-VENTA-14 [DUDOSA]`).
2. **Given** la entrevista de negocio (P21, dueño/gerente, sobre uso de reportes), **When** se
   pregunta si usan algún reporte de ventas por combo/promoción, **Then** la respuesta confirma
   que no lo revisan — sin impacto práctico hoy.

**Tratamiento acordado** (`registro-de-anomalias.md`, A-29): documentar sin especificar hasta
respuesta de negocio. Queda como decisión de negocio abierta si la pérdida de trazabilidad por
promoción en reportes es aceptable; esta spec no la fija como comportamiento deseado ni
obligatorio para la modernización.

---

### Edge Cases

- **Total exactamente en cero tras descuento y sin impuesto/propina**: no está excluido por
  ninguna validación adicional — solo se rechaza si el total queda negativo, no si queda en cero
  (`RN-VENTA-03`, User Story 1).
- **Pago exacto, sin sobra**: `change_given` resulta `0`, sin rechazo (`RN-VENTA-06`, User Story
  2, caso límite del contraste positivo).
- **Turno de caja inexistente** (`cash_shift_id` que no resuelve): `ensure_open_shift` usa
  `get_or_404`, así que el rechazo es `404`, no `409` — distinto código de estado del caso "turno
  cerrado" (`sales/builder.py:60-64`).
- **Descuento de inventario que falla a mitad de `checkout`**: revierte venta y factura juntas,
  ya construidas en memoria — no queda ninguna de las dos persistida (`RN-VENTA-10`, User Story
  3).
- **Consecutivo de factura cruzando el millón sin reinicio real**: divergencia matemática
  garantizada entre `Invoice.full_number` (Python) y la búsqueda SQL — ver User Story 7 (A-14,
  A-49).
- **Combo cuyo `option_ids` viene poblado**: rechazado en el propio schema (`SaleItemIn`,
  `sales/schemas.py:46-47`) — "Los combos no admiten option_ids en esta versión", antes de llegar
  a `build_sale`.
- **Ítem que no trae ni `product_variant_id` ni `combo_id`, o trae ambos**: rechazado por el
  validador del schema (`sales/schemas.py:42-45`) antes de resolver ninguna línea.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: `build_sale` DEBE calcular el total como `subtotal − descuento + impuesto +
  propina`, en ese orden aritmético exacto sobre `Decimal` puro, sin ningún redondeo intermedio
  (`RN-VENTA-02`).
- **FR-002 [Regla crítica]**: El total resultante NO DEBE poder ser negativo; DEBE rechazarse con
  `422` "El total no puede ser negativo" (`RN-VENTA-03`).
- **FR-003**: Una venta DEBE tener al menos un ítem cobrable; sin ítems, DEBE rechazarse con `409`
  sin persistir `Sale`, `SaleItem` ni `Payment` (`RN-VENTA-01`).
- **FR-004 [Regla crítica]**: El pago recibido DEBE cubrir el total exacto o más; el cobro parcial
  DEBE rechazarse con `422` (`RN-VENTA-04`).
- **FR-005 [Regla crítica]**: El vuelto SOLO DEBE poder salir de un exceso pagado en efectivo; un
  exceso pagado por un medio electrónico DEBE rechazarse con `422` (`RN-VENTA-05`).
- **FR-006**: El cambio entregado DEBE calcularse como `pagado − total` y quedar registrado en
  `change_given`, de forma que el arqueo del turno lo descuente del efectivo esperado en vez de
  contarlo bruto (`RN-VENTA-06`).
- **FR-007**: Cobrar cualquier venta DEBE exigir un turno de caja con `status="open"`; contra un
  turno cerrado DEBE rechazarse con `409` (`RN-VENTA-07`).
- **FR-008**: La venta DEBE transicionar de `issued` a `paid` dentro de la misma construcción,
  sin quedar visible en ningún estado intermedio para otros lectores concurrentes
  (`RN-VENTA-08`).
- **FR-009**: `build_sale` NO DEBE tocar inventario bajo ninguna circunstancia; el descuento de
  stock es responsabilidad exclusiva de quien llama. El camino de mostrador SIEMPRE descuenta
  inventario al cobrar, porque no hubo un paso previo de confirmación (`RN-VENTA-09`).
- **FR-010**: Si el descuento de inventario del camino de mostrador falla por falta de stock, la
  venta completa (incluida su factura, ya emitida en la misma transacción) DEBE revertirse
  (`RN-VENTA-10`).
- **FR-011 [Regla crítica, A-14a PROTEGIDA]**: Toda venta que alcanza `status="paid"` DEBE emitir
  automáticamente su factura, dentro de la misma transacción de base de datos que la crea — sin
  ningún paso manual adicional (`RN-FACT-01`).
- **FR-012 [A-14a PROTEGIDA]**: La emisión de factura DEBE ser idempotente por `sale_id` — una
  segunda invocación para la misma venta DEBE devolver la factura ya existente, nunca crear una
  duplicada (`RN-FACT-02`).
- **FR-013 [A-14a PROTEGIDA]**: La numeración de facturas DEBE ser un consecutivo estrictamente
  secuencial y sin huecos por prefijo, serializado con `SELECT...FOR UPDATE` sobre el contador; un
  cobro fallido NO DEBE consumir ningún número (`RN-FACT-03`).
- **FR-014**: La numeración de facturas DEBE ser independiente por tenant (schema separado) y,
  dentro de cada tenant, por prefijo — la unicidad real en base de datos es sobre
  `(prefix, number)` (`RN-FACT-04`).
- **FR-015 [Hallazgo — A-11, ACCIDENTAL, comportamiento actual]**: hoy, el campo `discount` de
  `SaleCreate` no impone ningún tope superior propio — el único freno es que el total no quede
  negativo (FR-002); `create_sale` no exige ningún rol elevado, a diferencia de
  `create_payment_method` en el mismo router (asimetría verificable, sin justificación
  documentada distinta de un descuido).
- **FR-016 [Regla crítica — A-11, regla propia, decisión de negocio cerrada]**: El sistema DEBE
  impedir que el cajero aplique cualquier descuento manual, en los tres caminos de cobro
  (mostrador, cierre unificado, cierre dividido) por igual — prohibición total, no un tope
  numérico configurable. **No retroactivo**: ventas ya cobradas con descuento excesivo no se
  recalculan.
- **FR-017**: Los ítems que forman parte de un combo DEBEN cobrarse a precio normal individual; su
  ahorro DEBE calcularse aparte y sumarse al descuento total (`RN-VENTA-12`).
- **FR-018**: El descuento total de una venta de mostrador DEBE ser la suma de tres fuentes: el
  descuento manual escrito por el cajero, las promociones automáticas (percent/fixed) evaluadas
  sobre las líneas normales, y el ahorro de los combos seleccionados (`RN-VENTA-13`).
- **FR-019 [`[DUDOSA]`, anomalía A-29, `PENDIENTE` — documentada sin especificar como contrato]**:
  si las líneas cobradas usan exactamente un combo distinto, `promotion_id` DEBE registrar ese
  combo; con cero o con dos o más combos distintos, el sistema usa el resultado de la evaluación
  general de promociones (que puede ser `None`) — con dos o más combos, ninguno queda registrado
  individualmente, aunque el descuento monetario de todos se sume correctamente al total
  (`RN-VENTA-14`).
- **FR-020**: Las opciones seleccionadas para una línea DEBEN estar activas y pertenecer a la
  variante vendida; IDs repetidos DEBEN deduplicarse antes de calcular el precio (`RN-VENTA-15`).
- **FR-021**: El precio de una línea DEBE resultar del precio de la variante más la suma de
  `extra_price` de las opciones seleccionadas, sin ningún redondeo adicional (`RN-VENTA-16`).
- **FR-022 [`[ACCIDENTAL]`, anomalía A-12 — comportamiento actual, corrección pendiente]**: el
  descuento de inventario del camino de mostrador NO bloquea el cobro si la variante no tiene
  ninguna regla de consumo definida (sin receta ni opción con consumo); la venta se cobra y
  factura sin generar ningún movimiento — a diferencia de la confirmación de pedidos QR, que
  bloquea con `409` (`RN-CAT-34`, spec 003). Alcance estimado: 6-20 productos activos sin receta
  (cifra aproximada, no verificada). **No retroactivo**.
- **FR-023**: El descuento de inventario DEBE pre-bloquearse en orden canónico de UUID entre venta
  de mostrador y confirmación de pedidos de mesa, para evitar deadlocks entre ambos caminos que
  compartan insumos (`RN-VENTA-17`).
- **FR-024 [Hallazgo — A-14, BUG A SECAS]**: el número de factura visible (`prefix + number`) se
  formatea de forma matemáticamente distinta en Python (`f"{n:06d}"`, nunca trunca) y en la
  reconstrucción SQL usada para búsqueda por referencia (`lpad(...,6,'0')`, sí trunca a 6
  caracteres); ambas coinciden por debajo del consecutivo 1.000.000 y divergen a partir de ahí de
  forma determinista, sin depender de ninguna condición de carrera.
- **FR-025 [Tratamiento acordado — A-14, informado por A-49]**: corregir ampliando el padding a
  **7+ dígitos** en ambos lados (`Invoice.full_number` en Python y la reconstrucción SQL en
  `list_sales_query`) como corrección segura, sin depender de un mecanismo de reinicio anual del
  consecutivo que la verificación técnica del 2026-08-16 no encontró en el código
  (`app/core/scheduler.py` solo registra `sweep_orphan_table_sessions` y `expire_promotions`) —
  pendiente de ratificación real con la gestoría antes de cerrarse en sentido estricto.
- **FR-026 [`[DUDOSA]`]**: la búsqueda de ventas por `invoice_reference` usa `ILIKE '%...%'` sobre
  el número reconstruido, aceptando coincidencias parciales en cualquier posición del string, no
  solo al inicio (`RN-FACT-05`).
- **FR-027 [Regla confirmada — A-41, INTENCIONAL con decisión fiscal pendiente]**: el total de
  cualquier venta armada desde la terminal de mesas DEBE mantener `tax=0` fijo — comportamiento
  deliberado (commit `8166ea9e`, "deprecate editable tax field"), confirmado correcto por negocio
  (P2). Queda como la pregunta de mayor riesgo fiscal/legal residual del reconocimiento, pese a su
  bajo impacto técnico actual, sin decisión de negocio que cierre la especificación fiscal
  completa.
- **FR-028 [Hallazgo — A-43, ACCIDENTAL, sin acción correctiva inmediata]**: `issue_for_sale` se
  documenta idempotente, pero su patrón `SELECT` seguido de `INSERT` no usa `with_for_update()` ni
  captura `IntegrityError`; bajo dos llamadas concurrentes para la misma venta, la segunda podría
  fallar sin control. La exposición real hoy es cero: el único llamador conocido
  (`build_sale`) opera dentro de la misma transacción de cobro. Corregir solo si se introduce un
  segundo llamador fuera de esa transacción.

### Key Entities *(include if feature involves data)*

- **Sale**: venta emitida. Atributos relevantes a esta spec: `subtotal`/`discount`/`tax`/`tip`/
  `total` (fórmula única, `RN-VENTA-02`), `status` (`issued`→`paid`, `RN-VENTA-08`),
  `paid_amount`/`change_given` (arqueo, `RN-VENTA-06`), `promotion_id` (regla de combo único,
  `RN-VENTA-14`), `cash_shift_id` (exige turno abierto, `RN-VENTA-07`). Su emisión de factura
  ocurre en la misma transacción que su creación (`RN-FACT-01`).
- **SaleItem**: línea de una venta, snapshot inmutable de precio y opciones al momento del cobro.
  Atributos relevantes: `product_variant_id`, `options` (snapshot JSONB), `quantity`,
  `unit_price`, `line_total`, `combo_id` (marca de qué combo proviene la línea, `RN-VENTA-12`).
- **Payment**: pago aplicado a una venta. Atributos relevantes: `payment_method_id`, `amount`; su
  suma frente al total gobierna `RN-VENTA-04`/`RN-VENTA-05`.
- **PaymentMethod**: método de pago del tenant. Atributo relevante: `is_cash` (gobierna qué exceso
  de pago puede generar vuelto, `RN-VENTA-05`); su creación exige `require_tenant_admin`, a
  diferencia de `create_sale` (contraste central de A-11).
- **CashShift**: turno de caja. Solo `status="open"` permite cobrar contra él (`RN-VENTA-07`,
  `ensure_open_shift`).
- **Invoice**: factura interna, snapshot inmutable de una venta. Atributos relevantes: `sale_id`
  (único → idempotencia, `RN-FACT-02`), `prefix`/`number` (consecutivo por tenant y prefijo,
  `RN-FACT-03`/`RN-FACT-04`), `full_number` (propiedad Python que diverge de la reconstrucción SQL
  desde 1.000.000, A-14).
- **InvoiceCounter**: contador de facturación por prefijo. `next_number` se lee y actualiza con
  `SELECT...FOR UPDATE`, serializando la emisión concurrente del mismo prefijo (`RN-FACT-03`).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas `RN-VENTA-01` a `RN-VENTA-17` y `RN-FACT-01` a `RN-FACT-07`
  puede verificarse ejecutando los pasos descritos en esta spec contra un `pos-backend` en
  ejecución, sin necesitar leer `sales/builder.py`, `sales/service.py` ni `invoices/service.py`
  para entender el comportamiento esperado.
- **SC-002**: Los cinco invariantes de la regla protegida A-14a (toda venta emite factura, factura
  por venta no por pedido, consecutivo sin huecos, atomicidad venta+factura, idempotencia por
  `sale_id`) quedan cubiertos punto por punto por `app/scripts/test_facturacion.py` — ningún
  cambio futuro en `build_sale`/`issue_for_sale` puede reintroducir el escenario histórico "20
  ventas reales, cero facturas" sin que este script lo detecte.
- **SC-003**: Ninguna venta puede quedar en `status="paid"` sin una `Invoice` asociada en el mismo
  commit — verificable inspeccionando que la emisión de factura vive dentro de la misma
  transacción que cierra los totales de la venta, para los tres caminos de cobro que usan
  `build_sale` (mostrador aquí; unificado y dividido en spec 010).
- **SC-004**: El bug matemático de A-14 (divergencia Python vs SQL en el número de factura) queda
  reproducible de forma determinista con un solo caso de prueba (`number=1234567`), sin depender
  de datos de producción ni de condiciones de carrera.
- **SC-005**: La regla propia A-11 (prohibición total de descuento manual del cajero) queda
  especificada con su decisión de negocio citada textualmente (P5, P30) y su alcance a los tres
  caminos de cobro explícito, de forma que las specs 008 y 010 puedan referenciarla sin
  respecificarla.
- **SC-006**: Las anomalías `PENDIENTE`/de bajo riesgo actual de esta spec (A-29 porción
  mostrador, A-43) quedan documentadas con su evidencia de código y su tratamiento acordado, de
  forma que el equipo de modernización no las reintroduzca por accidente ni las trate como si ya
  tuvieran una decisión de negocio cuando no la tienen.
- **SC-007**: La entrada A-49 (verificación técnica de ausencia de mecanismo de reinicio del
  consecutivo) queda señalada explícitamente como pendiente de ratificación real con la gestoría,
  distinta de una decisión de negocio cerrada, para que la próxima ronda real de entrevista la
  incluya sin necesidad de releer el código.

## Out of Scope

- **El cierre de cuenta de mesa que invoca `build_sale`** (`billing_mode=unified`/`split`, el
  reparto de comensales, la regla protegida A-15 del cobro dividido) — cubierto por la spec 010;
  esta spec solo documenta el constructor compartido y su emisión de factura, no los caminos de
  mesa que lo invocan.
- **La confirmación y cancelación de pedidos, y el ciclo de cobro legado `pay_order`** (el punto
  real de descuento de inventario de mesa, la asimetría de cancelación) — cubierto por la spec
  008; esta spec solo confirma, vía P11, que `pay_order` sigue sin uso real hoy.
- **El arqueo de caja que agrega estas ventas** (`reconcile`, el cálculo de efectivo esperado a
  partir de `Payment` y `change_given`) — cubierto por la spec 006; esta spec documenta que
  `change_given` existe y se calcula (`RN-VENTA-06`), no cómo lo consume el arqueo.
- **El motor de evaluación de promociones y combos** (`promotions.evaluate`,
  `combo_discount_for_lines`, prioridad entre promociones aplicables) — cubierto por specs
  futuras del reconocimiento (012/013, aún no escritas); esta spec solo documenta **cuándo** y
  **cómo** el checkout de mostrador invoca ese motor y registra su resultado (`RN-VENTA-13`,
  `RN-VENTA-14`), no cómo decide el motor mismo.
- **La facturación electrónica DIAN** (`cufe`, `dian_status`, `dian_sent_at` del modelo
  `Invoice`) — campos presentes en el modelo pero explícitamente "DIAN-ready, no usados en v1"; no
  hay comportamiento que documentar todavía.
- **El mecanismo de aplicación técnica de la prohibición de descuento manual (A-11)** más allá de
  su especificación de contrato (validación de schema, chequeo de rol, remoción del campo del
  payload del cajero, u otro mecanismo equivalente) — esta spec fija **qué** debe cumplirse
  (FR-016) como contrato de negocio ya decidido; el diseño técnico exacto del mecanismo es del
  módulo de implementación que lo construya.

## Assumptions

- **Esta es una spec de ingeniería inversa, no de una feature nueva**: a diferencia del resto de
  las guías de este template ("evitar detalles de implementación"), aquí los endpoints, códigos
  de estado HTTP, nombres de campo y fórmulas exactas **son** el contrato observable que se está
  documentando — se citan explícitamente porque los criterios de aceptación deben ser verificables
  directamente contra `pos-backend` en ejecución o contra `test_facturacion.py`.
- **A-14a (`RN-FACT-01`, `RN-FACT-02`, `RN-FACT-03`) se especifica tal cual, sin tocar**: es una
  regla `[PROTEGIDA]` con dos testigos (CÓDIGO + `memoria-historica.md` #6) y un costo histórico
  documentado explícitamente ("20 ventas reales, cero facturas"). Se fija como invariante de test
  obligatorio de máxima prioridad en cualquier reimplementación.
- **A-11 es la única regla de esta spec donde el contrato objetivo difiere del comportamiento
  actual observado**: a diferencia de las demás anomalías (que documentan sin corregir, Principio
  III de la Constitución), A-11 tiene una decisión de negocio explícita y por escrito (P5, P30)
  que autoriza fijar el comportamiento deseado como parte de esta spec — no solo el actual. Esto
  es coherente con el Principio I: "ningún cambio de comportamiento observable se introduce sin
  una decisión explícita y por escrito del negocio", y aquí esa decisión existe.
- **A-49 no cuenta como testigo NEGOCIO genuino todavía**: el testimonio que descarta la premisa
  de "el consecutivo se reinicia cada año" (P29, ronda 3) es explícitamente simulado, según el
  propio aviso de método de `entrevista-negocio.md` §8. La verificación técnica de código (ausencia
  de mecanismo de reinicio) sí es concluyente por sí sola; la corrección adoptada en A-14 (ampliar
  el padding) no depende de que A-49 se ratifique, precisamente para no bloquear la corrección
  segura en una confirmación pendiente.
- **A-29 (porción mostrador) y A-43 se documentan pero NO se especifican como contrato**:
  siguiendo instrucción explícita de alcance, ambas quedan con su clasificación de
  `registro-de-anomalias.md` (`PENDIENTE` y `ACCIDENTAL` de bajo riesgo actual respectivamente) —
  se describe el comportamiento observado hoy, pero no se fija como comportamiento correcto ni
  obligatorio para la modernización.
- **A-41 se especifica tal cual, con la reserva fiscal explícita**: el negocio confirmó que
  `tax=0` es correcto para el ticket (P2), pero esa confirmación no resuelve la pregunta legal más
  amplia de cómo se maneja el IVA fuera del POS si el negocio lo paga — esta spec no asume una
  respuesta en ningún sentido sobre esa pregunta más amplia.
- **`build_sale` es compartido por los cuatro caminos de cobro, pero esta spec solo controla
  mostrador**: los otros tres (unificado, dividido — spec 010; `pay_order` legado — spec 008)
  invocan el mismo constructor documentado aquí (User Stories 1-4), pero sus propios flujos de
  armado de líneas, descuento de inventario y reglas de mesa se especifican en sus specs
  correspondientes, no en esta.
