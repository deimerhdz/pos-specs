# Feature Specification: Promociones legibles y precios reales en el menú QR

**Feature Branch**: `066-promociones-legibles-menu`

**Created**: 2026-09-01

**Status**: Draft

**Naturaleza de esta spec**: **corrección de comportamiento en producción** sobre las
superficies de consumo que dejó la spec
[063](../063-promociones-por-variante/spec.md) (FR-022, FR-023). No cambia el modelo de
promociones, ni el motor de cálculo del cobro, ni la administración de reglas: cambia **qué
ve** el comensal (y, de rebote, el cajero y el administrador) sobre una promoción vigente.
Un punto sí corrige un importe visible que hoy no coincide con el que se cobra
(FR-010), por lo que se listan explícitamente los cambios de comportamiento respecto de
producción (Principio II de la [Constitución](../../.specify/memory/constitution.md)) y los
tests afectados (Principio III).

**Input**: User description: tres defectos reportados sobre el menú QR con el modelo de
promociones de la spec 063 ya en producción:
(1) el cartel de promociones del menú describe el conjunto elegible de forma genérica
("Llevando 2 de estas 1 variantes pagas $12.000") en vez de nombrar qué se lleva
("Llevando 2 Pequeño 8oz pagas $12.000");
(2) el modal que muestra las presentaciones de un producto no refleja el costo real cuando
la variante tiene una promoción vigente;
(3) las tarjetas del menú no señalan que un producto tiene promoción, ni por paquete ni por
descuento.

---

## Estado actual (lo que se corrige)

Línea base tomada del código en producción tras la spec 063:

- **Texto de la condición**: lo construye el backend en una única función
  (`variant_set_condition_text`, `pos-backend/app/api/v1/promotions/service.py`) y solo usa
  el **conteo** de variantes del conjunto, nunca sus nombres. Produce
  `"Llevando 2 de estas 8 variantes pagas $12.000"`,
  `"Cada una de estas 8 variantes a $6.000"`, `"10% en estas 8 variantes"` y
  `"15% llevando 3 de estas 8 variantes"`. El mismo texto viaja al cartel del menú QR
  (`MenuPromotionRule.text`) y al listado de administración (`condition_text`); el
  formulario de administración lo replica localmente para su vista previa
  (`ruleConditionPreview`, `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.ts`).
- **Precio con descuento en el menú**: el backend solo resuelve un precio unitario con
  descuento (`MenuVariantResponse.discounted_price`) para reglas de tipo **porcentaje con
  cantidad mínima 1**; para **precio de paquete** (cualquier cantidad mínima) y para
  **porcentaje con cantidad mínima > 1** devuelve nada. En consecuencia:
  - el modal de presentaciones muestra el precio normal, sin ninguna señal de la promoción;
  - una regla de **precio de paquete con cantidad mínima 1** (un precio unitario especial)
    se **cobra** al precio de la regla pero se **muestra** al precio normal: el menú
    anuncia un importe mayor que el que termina cobrando el backend.
- **Insignia de la tarjeta**: la tarjeta del menú QR muestra una insignia
  (`🏷️ -10%` / `🏷️ -$2.000`) derivada exclusivamente de `discounted_price`, así que solo
  aparece para porcentaje con cantidad mínima 1. Los demás tipos de regla no producen
  ninguna señal en la tarjeta.
- **Terminal de staff**: muestra una insignia por producto (`-10%` / `Paquete $12.000`)
  calculada desde las reglas vigentes, pero **no** muestra la condición en lenguaje llano
  que exige la spec 063 FR-023, ni ningún equivalente por unidad.
- **Vigencia**: el cartel del menú solo lista promociones vigentes **en ese instante**
  (estado, fechas, días y franja horaria en la zona horaria del tenant). Ese criterio no
  cambia en esta spec.

---

## Clarifications

### Session 2026-09-01

- Q: ¿Cómo debe nombrar la condición a las variantes del conjunto? → A: **Por su nombre de
  variante**. Si todas las variantes del conjunto comparten el mismo nombre, se usa ese
  nombre ("Llevando 2 Pequeño 8oz pagas $12.000"). Si el conjunto mezcla nombres distintos,
  se listan hasta tres y se resume el resto ("Llevando 2 entre Grande 16oz, Mediano 12oz y
  Pequeño 8oz pagas $15.000" — los nombres van en **orden alfabético**, no en el orden en que
  se seleccionaron; con más de tres nombres distintos, los tres primeros y "y N más")
  (FR-001 a FR-005).
- Q: En el modal de presentaciones, con una promoción de paquete (p. ej. 2 x $12.000 sobre
  una variante de $8.000), ¿qué precio debe verse? → A: **El precio normal como precio
  vigente, más el equivalente por unidad del paquete**: "2 x $12.000 · $6.000 c/u". El
  cálculo **no** depende de lo que ya haya en el carrito: es estable entre aperturas del
  modal y explica de dónde sale el ahorro sin prometer un precio que una unidad suelta no
  tiene (FR-007, FR-008, FR-009).
- Q: ¿Cómo debe indicar la tarjeta del menú que el producto tiene promoción? → A: Con una
  **insignia genérica única** ("🎉 Promo"), igual para descuento por porcentaje y para
  precio de paquete; el detalle se ve al abrir el producto (FR-013).
- Q: ¿Hasta dónde llega el alcance? → A: **Menú QR + terminal de staff + formulario de
  administración**. Las tres superficies muestran el mismo texto de condición con nombres
  de variante y la misma forma de presentar el precio de paquete. La insignia genérica de
  FR-013 es **solo del menú QR**: la insignia de la terminal es una herramienta del cajero y
  conserva su etiqueta informativa actual (FR-016, FR-018).
- Q: ¿El equivalente por unidad cambia el importe que se cobra? → A: **No**. Es información
  para el comensal. El importe vinculante lo sigue calculando el cobro en el backend
  (spec 063 FR-020, FR-023); esta spec no toca el motor de cálculo.
- Q: ¿Y el caso de precio de paquete con cantidad mínima 1, donde el menú muestra un precio
  mayor que el cobrado? → A: Es un **defecto de importe**, no de presentación: esa regla es
  un precio unitario especial y DEBE reflejarse como precio con descuento en el menú, igual
  que hoy hace el porcentaje con cantidad mínima 1 (FR-010).
- Q: Con una regla de precio de paquete de cantidad mínima 1 cuyo valor resulta **mayor** que
  el precio normal de alguna variante de su conjunto, ¿qué muestra el menú? → A: El valor de
  la regla es **siempre** el precio vigente, para que lo mostrado coincida con lo cobrado; el
  precio normal tachado aparece **solo** cuando el valor de la regla es menor que él. Si el
  valor es mayor o igual, se muestra sin tachado y sin ninguna señal de descuento (FR-010,
  FR-015).
- Q: ¿Cómo se redondea y se marca el equivalente por unidad cuando no da exacto en pesos?
  → A: **Redondeo al peso más cercano, medio hacia arriba** — el mismo criterio que ya usa el
  motor de cobro para su descuento —, y marca de aproximado ("≈") **siempre que el importe
  exacto no sea entero en pesos**, tanto en precio de paquete como en porcentaje (FR-009).
- Q: ¿Quién registra A-66, A-67 y A-68 en `registro-de-anomalias.md` y cuándo? → A: Es un
  **paso previo externo a esta feature**: las tres entradas se registran a mano antes de
  planificar la implementación. Las tareas de esta feature no las redactan; solo **verifican
  que existen** como condición de arranque (Principio II).
- Q: ¿Quién arma la información de promoción por presentación que pinta el modal? → A: **El
  backend**, ya calculada en la respuesta del menú (condición corta, equivalente por unidad
  con su marca de aproximado, y precio vigente cuando la cantidad mínima es 1). El frontend
  solo la pinta y deriva de ella la insignia genérica de la tarjeta; no recalcula importes ni
  redondeos (FR-007, FR-013).

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El comensal entiende qué le ofrece el cartel de promociones (Priority: P1)

Un comensal abre el menú por QR y, mientras hay una promoción vigente, ve el cartel con el
nombre de la promoción y una línea por cada una de sus reglas. Cada línea nombra **qué**
tiene que llevar para pagar ese precio, no cuántas filas de configuración tiene la regla.

**Why this priority**: es el defecto reportado con más impacto. Hoy el cartel de
Springfield dice tres veces "Llevando 2 de estas 1 variantes pagas $X", que no le dice al
comensal nada accionable: ni qué producto, ni qué tamaño. Sin esto, el cartel ocupa espacio
sin vender.

**Independent Test**: con la promoción "Semana feliz en granizados" activa y sus tres
reglas de precio de paquete (conjuntos de tamaño Pequeño 8oz, Mediano 12oz y Grande 16oz),
abrir el menú QR dentro de su horario y comprobar que las tres líneas nombran el tamaño.

**Acceptance Scenarios**:

1. **Given** una regla de precio de paquete con cantidad mínima 2, valor $12.000 y un
   conjunto de 8 variantes que se llaman todas "Pequeño 8oz", **When** el comensal ve el
   cartel, **Then** lee "Llevando 2 Pequeño 8oz pagas $12.000".
2. **Given** una regla de precio de paquete con cantidad mínima 2, valor $12.000 y un
   conjunto de **una sola** variante llamada "Pequeño 8oz", **When** el comensal ve el
   cartel, **Then** lee "Llevando 2 Pequeño 8oz pagas $12.000" — nunca "de estas 1
   variantes".
3. **Given** una regla cuyo conjunto mezcla variantes llamadas "Pequeño 8oz", "Mediano
   12oz" y "Grande 16oz", **When** el comensal ve el cartel, **Then** lee "Llevando 2 entre
   Grande 16oz, Mediano 12oz y Pequeño 8oz pagas $15.000" — en orden alfabético (FR-002), no
   en el orden en que el administrador eligió las variantes.
4. **Given** una regla cuyo conjunto tiene cinco nombres de variante distintos ("Pequeño
   8oz", "Mediano 12oz", "Grande 16oz", "Jumbo 20oz", "Familiar 24oz"), **When** el comensal
   ve el cartel, **Then** lee los tres primeros **en orden alfabético** y el resumen del
   resto: "Llevando 2 entre Familiar 24oz, Grande 16oz, Jumbo 20oz y 2 más pagas $15.000".
5. **Given** una regla de porcentaje del 10% con cantidad mínima 1 sobre un conjunto cuyas
   variantes se llaman todas "Pequeño 8oz", **When** el comensal ve el cartel, **Then** lee
   "10% en Pequeño 8oz".
6. **Given** una promoción activa pero fuera de su ventana de día u hora en ese instante,
   **When** el comensal abre el menú, **Then** el cartel no muestra ninguna de sus reglas
   (sin cambio respecto de hoy).

---

### User Story 2 - El comensal ve el costo real de la presentación que va a elegir (Priority: P1)

Al abrir un producto, el comensal ve sus presentaciones con precio. Cuando una presentación
está cubierta por una regla de una promoción vigente, ve además cuánto le costaría de
verdad aprovechando esa promoción.

**Why this priority**: es donde el comensal decide. Hoy el modal es la única superficie que
muestra precio por presentación y es exactamente la que ignora las promociones de paquete;
y en el caso de precio de paquete con cantidad mínima 1 el modal muestra un importe **mayor
que el que se cobra**, lo que es un defecto de importe, no de estética.

**Independent Test**: con "2 x $12.000" activa sobre las variantes "Pequeño 8oz" (precio
normal $8.000), abrir el producto y comprobar que la fila de esa presentación muestra
$8.000 y, debajo, "2 x $12.000 · $6.000 c/u"; luego, con una regla de precio unitario
especial de $6.000 (cantidad mínima 1), comprobar que la fila muestra $8.000 tachado y
$6.000 como precio vigente, y que el carrito cobra $6.000.

**Acceptance Scenarios**:

1. **Given** una regla de precio de paquete de cantidad mínima 2 y valor $12.000 vigente
   sobre la variante "Pequeño 8oz" de $8.000, **When** el comensal abre el modal del
   producto, **Then** la fila de esa presentación muestra el precio normal $8.000 y, como
   información adicional, "2 x $12.000 · $6.000 c/u".
2. **Given** la misma regla, **When** el comensal agrega **una** unidad al carrito,
   **Then** el carrito la cobra a $8.000 — el equivalente por unidad es informativo y no
   cambia el importe.
3. **Given** una regla de porcentaje del 15% con cantidad mínima 3 vigente sobre la
   variante "Mediano 12oz" de $11.000, **When** el comensal abre el modal, **Then** la fila
   muestra $11.000 y "3 x -15% · $9.350 c/u".
4. **Given** una regla de **precio de paquete con cantidad mínima 1** de $6.000 vigente
   sobre "Pequeño 8oz" (precio normal $8.000), **When** el comensal abre el modal,
   **Then** la fila muestra $8.000 tachado y $6.000 como precio vigente, y el total del
   botón de agregar usa $6.000.
5. **Given** la regla del punto 4, **When** el comensal agrega la unidad y el pedido se
   cobra, **Then** el importe cobrado coincide con el que mostró el modal.
6. **Given** una regla de porcentaje del 10% con cantidad mínima 1 (el caso que ya
   funcionaba), **When** el comensal abre el modal, **Then** sigue viendo el precio normal
   tachado y el precio con descuento **sin ningún cambio de importe**, y además la línea
   informativa "1 x -10% · $7.200 c/u" que FR-008 añade a toda presentación cubierta.
7. **Given** un producto con tres presentaciones donde solo una pertenece al conjunto de
   una regla vigente, **When** el comensal abre el modal, **Then** solo esa fila lleva la
   información de la promoción; las otras dos se ven como siempre.
8. **Given** una promoción activa pero fuera de su ventana horaria en ese instante,
   **When** el comensal abre el modal, **Then** ninguna fila muestra información de esa
   promoción.

---

### User Story 3 - Las tarjetas del menú señalan qué productos tienen promoción (Priority: P2)

Recorriendo la carta, el comensal distingue de un vistazo qué productos tienen alguna
promoción vigente, sea por descuento o por paquete, y abre el que le interesa para ver el
detalle.

**Why this priority**: es descubrimiento. Sin la insignia, una promoción de paquete solo
existe en el cartel de arriba, y el comensal tiene que abrir producto por producto para
saber cuál está incluido. No bloquea el cobro correcto (User Story 2), por eso va después.

**Independent Test**: con una promoción de paquete vigente sobre las variantes "Pequeño
8oz" de dos productos distintos, recorrer la carta y comprobar que esos dos productos —y
solo esos— llevan la insignia.

**Acceptance Scenarios**:

1. **Given** un producto con al menos una variante en el conjunto de una regla de **precio
   de paquete** de una promoción vigente, **When** el comensal ve su tarjeta, **Then** la
   tarjeta lleva la insignia genérica "🎉 Promo".
2. **Given** un producto con al menos una variante en el conjunto de una regla de
   **porcentaje** de una promoción vigente, **When** el comensal ve su tarjeta, **Then** la
   tarjeta lleva la misma insignia genérica "🎉 Promo" — no una distinta por tipo.
3. **Given** un producto sin ninguna variante en ninguna regla vigente, **When** el
   comensal ve su tarjeta, **Then** la tarjeta no lleva insignia.
4. **Given** un producto cuya única promoción está activa pero fuera de su ventana de día u
   hora en ese instante, **When** el comensal ve su tarjeta, **Then** la tarjeta no lleva
   insignia.
5. **Given** un producto con una regla de porcentaje de cantidad mínima 1 vigente (que hoy
   ya muestra precio tachado en la tarjeta), **When** el comensal ve su tarjeta, **Then**
   sigue viendo el precio tachado y el precio con descuento, ahora acompañados por la
   insignia genérica.

---

### User Story 4 - El cajero y el administrador leen la misma condición que el comensal (Priority: P3)

La terminal del cajero y el módulo de administración de promociones describen cada regla
con el mismo texto que ve el comensal, para que las tres superficies hablen del mismo
conjunto con las mismas palabras.

**Why this priority**: consistencia operativa. Un cajero que lee "Llevando 2 de estas 8
variantes" no puede decirle al cliente qué le falta para completar el paquete; un
administrador que revisa el resumen antes de guardar tampoco puede verificar que eligió las
variantes correctas. No bloquea nada del flujo del comensal, por eso es la última.

**Independent Test**: con la misma regla vigente, comprobar que el cartel del menú, la
terminal del cajero y el resumen del formulario de administración muestran la misma frase.

**Acceptance Scenarios**:

1. **Given** una regla de precio de paquete vigente sobre variantes llamadas "Pequeño 8oz",
   **When** el cajero mira ese producto en la terminal, **Then** ve la condición "Llevando
   2 Pequeño 8oz pagas $12.000" y el equivalente "$6.000 c/u", con las mismas palabras que
   el menú (spec 063 FR-023).
2. **Given** la misma regla, **When** el administrador la revisa en el listado de
   promociones, **Then** la columna de condición muestra la misma frase.
3. **Given** el administrador armando una regla en el formulario, **When** selecciona las
   variantes y mira el resumen previo a guardar, **Then** la condición usa el mismo formato
   con nombres de variante, calculado sobre las variantes que tiene seleccionadas en ese
   momento.
4. **Given** una regla cuyo conjunto tiene más de tres nombres de variante distintos,
   **When** el administrador ve el resumen, **Then** el texto resume igual que en el menú
   (tres nombres y "y N más"), y la lista completa de variantes con su precio normal sigue
   estando disponible en el propio resumen (spec 063 FR-005, sin cambio).
5. **Given** la insignia por producto de la terminal ("-10%" / "Paquete $12.000"), **When**
   el cajero recorre el catálogo, **Then** la insignia conserva su etiqueta informativa
   actual — la insignia genérica de FR-013 es exclusiva del menú QR.

---

### Edge Cases

- **Conjunto de una sola variante**: el texto nombra esa variante; nunca aparece "de estas
  1 variantes" (FR-002, el defecto reportado).
- **Variantes con el mismo nombre en productos distintos**: los nombres se deduplican, así
  que ocho variantes "Pequeño 8oz" de ocho productos producen un único nombre en el texto
  (FR-003). Es el caso real de Springfield y la razón por la que el texto describe el
  **tamaño**, no el producto.
- **Más de tres nombres distintos**: se listan los tres primeros en orden alfabético y se
  resume el resto con "y N más", donde N es la cantidad de nombres distintos restantes
  (FR-002).
- **Variante sin nombre**: se usa el nombre de su producto. Si tampoco hay nombre
  utilizable en ninguna variante del conjunto, la regla vuelve al texto genérico por conteo
  que existe hoy (FR-006).
- **Equivalente por unidad que no da exacto en pesos**: se redondea al peso más cercano
  (medio hacia arriba) y se marca como aproximado ("≈ $4.333 c/u"), en paquete y en
  porcentaje, porque el importe exacto depende de qué unidades entren al grupo
  (spec 063 FR-008) (FR-009).
- **Precio de paquete cuyo conjunto mezcla precios normales distintos**: el equivalente por
  unidad del paquete es el mismo para todas las variantes del conjunto (valor de la regla
  dividido entre la cantidad mínima), porque el precio del paquete es único; el precio
  normal que se tacha o se muestra al lado sí es el de cada variante (FR-008).
- **Porcentaje con cantidad mínima > 1**: el equivalente por unidad se calcula sobre el
  precio normal **de esa variante**, no sobre el conjunto (FR-008).
- **Precio de paquete con cantidad mínima 1 cuyo valor supera el precio normal de una
  variante**: el valor de la regla sigue siendo el precio vigente (es el que se cobra), pero
  se muestra sin precio tachado y sin señal de descuento — nunca se presenta un aumento como
  ahorro (FR-010, FR-015).
- **Producto con varias variantes y solo algunas en promoción**: la tarjeta lleva insignia
  (hay al menos una variante cubierta) pero el modal solo marca las filas cubiertas
  (FR-014, FR-007).
- **Dos promociones vigentes sobre la misma variante**: no puede ocurrir en un mismo
  instante — la spec 063 FR-014 bloquea el solapamiento real —, así que cada variante tiene
  a lo sumo una regla vigente que describir (FR-012).
- **Regla de un tipo retirado** (histórica, de una promoción ya `Finalizada`): no tiene
  texto de condición, no se anuncia y no genera insignia ni información en el modal, igual
  que hoy (FR-006).
- **Promoción vigente cuyo conjunto quedó sin variantes elegibles** (todas desactivadas o
  eliminadas, spec 063 FR-011): no genera insignia ni información en el modal; el cartel
  no anuncia esa regla.

---

## Requirements *(mandatory)*

### Texto de la condición en lenguaje llano

- **FR-001**: El texto de condición de una regla DEBE describir su conjunto elegible por los
  **nombres de las variantes** que lo componen, no por su cantidad. El texto lo construye el
  backend como fuente única y todas las superficies lo consumen (menú QR, terminal,
  administración), salvo la vista previa del formulario de administración, que lo replica
  localmente con la misma regla porque las variantes todavía no están guardadas (FR-018).
- **FR-002**: El **descriptor del conjunto** DEBE construirse a partir de la lista de
  nombres de variante **distintos** del conjunto (D), ordenados alfabéticamente sin
  distinguir mayúsculas ni tildes para que el texto sea determinista:
  - un solo nombre distinto → ese nombre ("Pequeño 8oz");
  - dos o tres nombres distintos → los nombres unidos por comas y "y" antes del último
    ("Grande 16oz, Mediano 12oz y Pequeño 8oz");
  - más de tres nombres distintos → los tres primeros y "y N más", con N = (cantidad de
    nombres distintos − 3) ("Familiar 24oz, Grande 16oz, Jumbo 20oz y 2 más").
- **FR-003**: Los nombres DEBEN deduplicarse: varias variantes con el mismo nombre en
  productos distintos cuentan como un solo nombre en el descriptor. El texto NO DEBE
  mencionar la cantidad de variantes del conjunto.
- **FR-004**: Los textos por tipo de regla DEBEN ser, con `n` = cantidad mínima, `d` =
  descriptor de FR-002 y `e` = "entre " cuando el conjunto tiene más de un nombre distinto y
  cadena vacía cuando tiene uno solo:
  - **precio de paquete** con `n` > 1: `Llevando {n} {e}{d} pagas {valor}`;
  - **precio de paquete** con `n` = 1: `Cada {e}{d} a {valor}`;
  - **porcentaje** con `n` = 1: `{porcentaje}% en {d}`;
  - **porcentaje** con `n` > 1: `{porcentaje}% llevando {n} {e}{d}`.
- **FR-005**: Los importes DEBEN escribirse con el mismo formato de moneda que ya usa el
  sistema ("$12.000") y el porcentaje sin ceros decimales sobrantes ("10%", "12.5%" — con
  **punto** decimal, que es el separador que el sistema produce hoy), sin cambio respecto de
  hoy. Cambiar el separador a coma sería un cambio de comportamiento visible fuera del
  alcance de A-66, y rompería SC-005 contra la vista previa del formulario.
- **FR-006**: Si ninguna variante del conjunto aporta un nombre utilizable (nombre de
  variante vacío y nombre de producto vacío), el texto DEBE volver al descriptor genérico
  por conteo que existe hoy ("de estas N variantes"). Una regla de un tipo retirado
  (histórica) DEBE seguir sin texto de condición y sin anunciarse.

### Precio real por presentación en el modal del producto

- **FR-007**: El modal de presentaciones de un producto DEBE mostrar, en cada presentación
  cubierta por el conjunto de una regla cuya promoción esté vigente **en ese instante**
  (estado, fechas, días y franja horaria en la zona horaria del tenant), la información de
  esa promoción junto a su precio. Las presentaciones no cubiertas NO DEBEN mostrar ninguna.
  Esa información la DEBE entregar el backend **ya calculada por variante** en la respuesta
  del menú —condición corta, equivalente por unidad con su marca de aproximado, y precio
  vigente cuando la cantidad mínima es 1—, como fuente única y por el mismo motivo que
  FR-001: es lo que hace verificable SC-005 y evita duplicar el redondeo de FR-009 en dos
  lenguajes. El menú NO DEBE recalcular importes, redondeos ni textos por su cuenta.
- **FR-008**: Para **toda** regla vigente que cubra la presentación —con `n` = cantidad
  mínima, incluido `n` = 1—, la información DEBE ser la condición corta más el **equivalente
  por unidad**:
  - **precio de paquete**: `{n} x {valor} · {valor ÷ n} c/u`, con el mismo equivalente para
    todas las variantes del conjunto (el precio del paquete es único);
  - **porcentaje**: `{n} x -{porcentaje}% · {precio normal de esa variante × (1 −
    porcentaje ÷ 100)} c/u`, calculado sobre el precio normal de la variante mostrada.

  Con `n` = 1 la línea se emite igual ("1 x -10% · $7.200 c/u") y convive con el precio
  vigente de FR-010: es información adicional bajo el precio, no lo sustituye ni cambia el
  tachado.
- **FR-009**: El equivalente por unidad DEBE redondearse **al peso más cercano, con el medio
  hacia arriba** — el mismo criterio de redondeo que aplica el cobro al calcular su
  descuento —, y DEBE presentarse marcado como aproximado ("≈ $4.333 c/u") **siempre que el
  importe exacto no sea entero en pesos**. La marca aplica por igual a los dos tipos de
  regla: en precio de paquete porque el valor puede no dividirse exacto entre la cantidad
  mínima, y en porcentaje porque el importe vinculante lo calcula el cobro sobre el total del
  grupo y luego lo reparte, de modo que el equivalente por variante puede diferir en un peso.
- **FR-010**: Una regla de **precio de paquete con cantidad mínima 1** es un precio unitario
  especial y el valor de la regla DEBE ser el **precio vigente** de todas las variantes de su
  conjunto, tanto en el modal como en el total del botón de agregar y en el carrito. Esto
  corrige la discrepancia actual entre el importe que el menú muestra y el que el cobro
  aplica. El **precio normal tachado** DEBE acompañarlo **solo cuando el valor de la regla es
  menor** que el precio normal de esa variante; cuando el valor es mayor o igual (posible
  porque un conjunto puede mezclar precios normales distintos), la variante muestra el valor
  de la regla **sin tachado y sin ninguna señal de descuento**, porque no hay ahorro que
  anunciar. El campo `discount_kind` de la respuesta del menú pasa a llevar el tipo real de
  la regla (`percent` o `package_price`), en vez de `percent` siempre.
- **FR-011**: El equivalente por unidad de FR-008 es **informativo**: NO DEBE alterar el
  precio unitario con el que la unidad entra al carrito, ni el subtotal del carrito, ni el
  importe del cobro. El importe vinculante lo sigue calculando el cobro en el backend
  (spec 063 FR-020).
- **FR-012**: La información mostrada DEBE corresponder a la **única** regla vigente que
  cubre esa variante en ese instante. El sistema no necesita criterio de desempate: la
  spec 063 FR-014 impide que dos reglas vigentes compartan una variante en el mismo momento.

### Insignia de promoción en las tarjetas del menú QR

- **FR-013**: La tarjeta de un producto en el menú QR DEBE mostrar una **insignia genérica
  única** ("🎉 Promo") cuando al menos una de sus variantes pertenece al conjunto de una
  regla cuya promoción está vigente en ese instante, **sin distinguir** entre porcentaje y
  precio de paquete. La tarjeta deriva esa señal de la información por variante que entrega
  el backend (FR-007), no de una evaluación propia de las reglas. Esta insignia reemplaza a
  la insignia por tipo que la tarjeta muestra hoy ("🏷️ -10%" / "🏷️ -$2.000").
- **FR-014**: La insignia DEBE aparecer aunque la regla sea de precio de paquete o de
  porcentaje con cantidad mínima > 1 — es decir, aunque el producto no tenga un precio
  unitario con descuento que mostrar.
- **FR-015**: Cuando además exista un precio unitario con descuento (porcentaje con cantidad
  mínima 1, o precio de paquete con cantidad mínima 1 según FR-010), la tarjeta DEBE seguir
  mostrando el precio normal tachado junto al precio vigente, ahora acompañado de la
  insignia genérica. El tachado sigue el mismo criterio de FR-010: si el valor de la regla no
  es menor que el precio normal, la tarjeta muestra el precio vigente sin tachado, con la
  insignia genérica como única señal.

### Terminal de staff y administración

- **FR-016**: La terminal de staff DEBE mostrar, para un producto/variante cubierto por una
  regla vigente, la condición en lenguaje llano de FR-004 y el equivalente por unidad de
  FR-008, con el mismo texto que ve el comensal — completando lo que la spec 063 FR-023 ya
  exigía. La insignia por producto de la terminal CONSERVA su etiqueta informativa actual
  ("-10%" / "Paquete $12.000"): la insignia genérica de FR-013 es exclusiva del menú QR.
- **FR-017**: La terminal NO DEBE calcular ni aplicar descuento por su cuenta a partir de
  esta información (spec 063 FR-023, sin cambio).
- **FR-018**: El módulo de administración de promociones DEBE usar el texto de FR-004 tanto
  en el listado de promociones como en el resumen previo a guardar del formulario. En el
  formulario, el texto se calcula sobre las variantes seleccionadas en ese momento, con la
  misma regla de FR-002 a FR-004. La lista completa de variantes con su precio normal que el
  resumen ya muestra (spec 063 FR-005) NO cambia.

### Límites del cambio

- **FR-019**: Esta spec NO DEBE modificar el modelo de datos de promociones, el motor de
  cálculo del descuento, el criterio de vigencia, el bloqueo de solapamiento, la máquina de
  estados, los permisos, ni la persistencia del descuento en la venta (spec 063 FR-001 a
  FR-021 y FR-024 a FR-027).
- **FR-020**: El cambio NO es retroactivo: no altera ninguna venta, factura ni pedido ya
  emitido, ni su representación histórica (Principio VII de la Constitución).

### Key Entities

- **Regla de promoción**: sin cambios respecto de la spec 063 (tipo, valor, cantidad mínima
  y conjunto de variantes). Esta spec solo cambia **cómo se describe** y **qué se deriva**
  de ella para mostrar.
- **Descriptor del conjunto**: representación legible del conjunto de una regla, derivada de
  los nombres de variante distintos que lo componen (FR-002). No se persiste: se calcula al
  momento de mostrar, con los nombres vigentes del catálogo.
- **Información de promoción por variante (menú)**: para una variante cubierta por una regla
  vigente, el paquete de datos que el menú necesita para pintarla — condición corta,
  equivalente por unidad, y precio unitario con descuento cuando la cantidad mínima es 1
  (FR-007 a FR-010). La calcula y la entrega el backend en la respuesta del menú; es
  información derivada, no persistida.
- **Insignia de promoción (tarjeta)**: señal booleana por producto — tiene o no al menos una
  variante con información de promoción en la respuesta del menú (FR-007, FR-013).

---

## Cambios de comportamiento respecto de producción

Cada punto es un cambio de comportamiento visible que exige decisión de negocio registrada
(Principio II). Se proponen para registro en
[`specs/000-reconocimiento/registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md)
como continuación de A-65.

**Cómo se registran**: es un **paso previo externo a esta feature**. Las tres entradas se
escriben a mano en el registro de anomalías —con quién tomó la decisión y cuándo, en el
formato de A-62 a A-65— **antes** de planificar la implementación. Las tareas de esta feature
NO redactan el registro: solo **verifican que A-66, A-67 y A-68 existen** como condición de
arranque. Sin ese registro, el cambio de comportamiento no está autorizado.

1. **A-66 (propuesta)** — El texto de condición de una regla pasa de describir el conjunto
   por **conteo** ("de estas 8 variantes") a describirlo por **nombres de variante**
   ("Pequeño 8oz"), en las tres superficies. Motivo: el texto por conteo no le dice al
   comensal qué debe llevar, y en el caso real de Springfield (conjuntos de una variante por
   tramo de tamaño) resulta directamente absurdo ("de estas 1 variantes").
2. **A-67 (propuesta)** — Dos señales nuevas en el menú QR: (a) la insignia de la tarjeta
   pasa de ser **por tipo de descuento** ("🏷️ -10%", derivada de que exista precio unitario
   con descuento) a una **insignia genérica** ("🎉 Promo") que cubre también las promociones
   de paquete y las de porcentaje con cantidad mínima > 1; y (b) **cada presentación cubierta
   por una regla vigente gana una línea informativa** con su condición corta y su equivalente
   por unidad, incluidas las de cantidad mínima 1, que hoy solo muestran precio tachado
   (FR-008). Motivo: hoy una promoción de paquete no produce ninguna señal en la carta, y una
   presentación con descuento no dice de qué promoción viene.
3. **A-68 (propuesta)** — Una regla de **precio de paquete con cantidad mínima 1** pasa a
   reflejarse en el menú como precio unitario con descuento. Motivo: hoy el menú muestra el
   precio normal y el cobro aplica el de la regla — el comensal ve un importe distinto del
   que se le cobra. Es una corrección de importe visible, no una decisión de presentación.

**Tests afectados** (ninguno lleva el prefijo `"CONGELA comportamiento actual:"`, verificado
en `pos-backend/app/characterization_tests/`; son tests de aceptación de la spec 063 y DEBEN
actualizarse citando esta spec, Principio III):

- `pos-backend/app/characterization_tests/test_menu_router.py::test_vigente_se_anuncia_con_texto_legible`
  — afirma el texto exacto `"Llevando 2 de estas 8 variantes pagas $12.000"`.
- `pos-backend/app/characterization_tests/test_promotions_rules_admin.py::test_ca1_ca6_paquete_nace_borrador_con_condicion`
  — afirma el mismo texto en `condition_text`.
- `pos-backend/app/characterization_tests/test_promotions_router.py::test_el_header_no_cambia_la_forma_de_la_respuesta`
  — afirma `condition_text == "10% en estas 1 variantes"` (línea 75). No figuraba en la
  primera redacción de esta spec; lo encontró el reconocimiento de código del plan
  (`research.md` D-8). Su variante se crea sin nombre explícito, así que el fixture le asigna
  uno no determinista: hay que darle un `name` fijo para poder afirmar el texto nuevo.
- `pos-backend/app/scripts/test_promotions_rules.py` — verificación de `_regla_texto` con el
  texto por conteo.
- `pos-heladeria/src/app/modules/promotions/pages/promotions-page.component.spec.ts`
  — afirma que la vista previa contiene `'Llevando 2 de estas 3 variantes'`.

**Redacción de la spec 063 que esta spec refina** (sin derogarla): los ejemplos de texto de
FR-022 ("Llevando 2 de estos sabores pagas $12.000") y FR-023 ("Llevando 2 pagas $12.000")
quedan sustituidos por el formato de FR-004 de esta spec. El resto de FR-022 y FR-023
(cuándo se anuncia, qué evalúa la vigencia, que la terminal no aplica descuento) sigue
vigente sin cambio.

---

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas anunciadas en el cartel del menú QR nombran al menos una
  variante de su conjunto; ninguna muestra el descriptor por conteo salvo el caso de FR-006
  (ninguna variante con nombre utilizable).
- **SC-002**: El 100% de las presentaciones cubiertas por una regla vigente muestran, en el
  modal del producto, su condición y su equivalente por unidad; el 0% de las presentaciones
  no cubiertas muestran información de promoción.
- **SC-003**: El importe que el menú muestra para una variante cubierta por una regla de
  cantidad mínima 1 coincide con el importe cobrado en el 100% de los casos (hoy difiere en
  todas las reglas de precio de paquete con cantidad mínima 1).
- **SC-004**: El 100% de los productos con al menos una variante en una regla vigente
  muestran insignia en su tarjeta, y el 0% de los productos sin ninguna variante cubierta la
  muestran — evaluado en el mismo instante en que se evalúa la vigencia.
- **SC-005**: El texto de condición de una misma regla es idéntico, carácter por carácter,
  en el cartel del menú, en la terminal del cajero y en el listado de administración.
- **SC-006**: El total cobrado de un pedido no cambia en ningún escenario respecto del
  comportamiento anterior, salvo en el caso de FR-010, donde el importe **mostrado** pasa a
  coincidir con el que ya se cobraba. Verificable con la batería de pruebas de cálculo de la
  spec 063 sin modificar sus asertos de importe.
- **SC-007**: Un comensal que abre el menú identifica sin abrir ningún producto qué
  artículos tienen promoción y qué debe llevar para obtener el precio anunciado.

---

## Assumptions

- El nombre de variante del catálogo real es descriptivo del tamaño/presentación ("Pequeño
  8oz", "Mediano 12oz"), que es lo que el comensal necesita leer. Esta spec asume ese
  significado; no crea ningún campo nuevo para nombrar la presentación (la entidad
  `Presentation` fue eliminada por la spec 063 FR-027).
- El orden alfabético de los nombres distintos (FR-002) se elige por ser determinista y
  reproducible entre superficies; no representa ninguna jerarquía de negocio.
- El límite de tres nombres antes de resumir (FR-002) se elige para que la línea del cartel
  quepa en una pantalla de móvil sin truncarse; no hay una regla de negocio detrás.
- La insignia genérica del menú QR no distingue tipo porque el detalle está a un toque de
  distancia (el modal del producto) y una insignia por tipo obligaría a decidir cuál mostrar
  cuando un producto tiene variantes en reglas distintas.
- El criterio de vigencia "en este instante" (estado, fechas, días y franja horaria en la
  zona horaria del tenant, con la corrección A-57 para ventanas que cruzan la medianoche) es
  el que ya usa el cartel del menú y no cambia.
- Las tres superficies siguen consumiendo el texto que construye el backend; solo la vista
  previa del formulario de administración lo replica localmente, porque describe variantes
  seleccionadas que todavía no se han guardado.
- El límite por plan de suscripción sobre la cantidad de reglas (mencionado como fuera de
  alcance en la spec 063) sigue fuera de alcance aquí.

## Out of Scope

- Cualquier cambio al modelo `Promoción`/`Regla`, al motor de cálculo, al bloqueo de
  solapamiento, a la máquina de estados o a la persistencia del descuento (spec 063).
- El desglose del descuento **por línea de venta** (variante ↔ regla ↔ monto), que la
  spec 063 FR-021 dejó explícitamente para una spec aparte.
- Mostrar en el menú un precio efectivo que dependa del contenido actual del carrito
  (opción evaluada y descartada en Clarifications: el modal muestra un equivalente estable,
  no un precio marginal).
- Cambiar la insignia por producto de la terminal de staff (FR-016).
- Reportes, analítica o arqueo sobre promociones.
