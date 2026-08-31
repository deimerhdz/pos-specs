# Feature Specification: Promociones de Precio por Cantidad Configuradas por Presentación

**Feature Branch**: `040-promociones-precio-por-presentacion`

**Created**: 2026-08-26

**Status**: Draft

**Naturaleza de esta spec**: **funcionalidad nueva** (fase de evolución funcional, Principio I de
la [Constitución](../../.specify/memory/constitution.md)), no la corrección de una anomalía ya
registrada en `registro-de-anomalias.md` — ninguna entrada existente cubre el problema de fondo
(un producto con presentaciones de precio distinto, y sabores distintos en la misma presentación
que hoy no combinan para una promoción). Se construye conceptualmente sobre el dominio ya
caracterizado por las specs [012](../012-motor-de-evaluacion-de-promociones-y-combos/spec.md)
(motor de evaluación: vigencia, mejor promoción por línea) y
[013](../013-administracion-de-promociones-y-combos/spec.md) (administración y máquina de
estados), pero introduce una dimensión de alcance que el sistema no tiene hoy: hoy una regla de
"precio por cantidad" apunta a un producto o a una categoría — no existe ningún concepto de
**presentación compartida entre productos** al que una regla pueda apuntar. Definir esa entidad y
su modelo de datos es evolución del modelo de datos (Principio VIII) y corresponde a la fase de
`/speckit-plan`, no a esta especificación. Esta spec no modifica el cuerpo de ningún test
`"CONGELA comportamiento actual:"`: las promociones de precio por cantidad a nivel de producto que
ya existen siguen funcionando sin cambios (ver FR-016), y esta funcionalidad se suma como una
modalidad adicional, no como un reemplazo. **Una excepción acotada**: FR-004 corrige `_valid_now`
para que las horas posteriores a la medianoche de una ventana que cruza medianoche pertenezcan al
día de inicio; ese arreglo aplica a todos los tipos de promoción y su cambio de comportamiento
observable queda registrado como A-55 en `registro-de-anomalias.md` (Principio II). No es
retroactivo.

**Input**: User description: hoy una promoción de "precio por cantidad" (ej. 2 x Granizado Ojo de
Diablo por $12.000) se configura sobre un producto puntual, lo que falla en dos frentes: un
producto puede tener varias presentaciones con precio unitario distinto que la promoción no
distingue, y un cliente que lleva dos sabores distintos en la misma presentación no recibe el
descuento aunque comercialmente sea el mismo caso. Se pide que una promoción de precio por
cantidad se pueda configurar por presentación: cada regla define una presentación, una cantidad
mínima y un precio total de paquete, aplica a todos los productos del tenant que tengan esa
presentación (incluidos los creados después), y el paquete puede combinar productos distintos que
compartan presentación. Una misma promoción puede tener varias reglas, una por presentación. Se
adjuntó además un mockup de la pantalla de administración (formulario de información general +
configuración de reglas, con un panel lateral de "Productos Aplicables" y "Resumen de la Regla")
como referencia visual — ver Assumptions.

## Clarifications

### Session 2026-08-26

- Q: ¿Cómo debe resolverse el conflicto cuando dos promociones activas de precio-por-cantidad
  tienen una regla para la misma presentación (CL-4)? → A: Bloquear al crear/activar — el sistema
  impide guardar o activar una promoción si alguna de sus reglas apunta a una presentación ya
  cubierta por una regla de otra promoción de este tipo que esté activa (FR-006).
- Q: ¿Qué debe pasar si se elimina o desactiva una presentación que tiene una regla en una
  promoción activa (CL-2)? → A: Bloquear la baja — el sistema impide eliminar o desactivar la
  presentación mientras esté referenciada por una regla activa, y muestra qué promoción(es) la
  usan (FR-020).
- Q: Si una promoción heredada a nivel de producto y una nueva a nivel de presentación pueden
  aplicar a la misma línea del pedido, ¿cómo coexisten? → A: Excluyentes por línea — cada línea
  del pedido recibe como máximo el descuento de una sola promoción; si ya fue tomada por una, no
  entra en el cálculo de otra (FR-013).
- Q: Con precios no uniformes entre las variantes de una presentación y la regla guardada igual
  (Historia 3), ¿sobre qué precio se calculan el descuento del paquete y las unidades sobrantes?
  → A: Un precio unitario de referencia único por presentación —el menor precio vigente entre las
  variantes elegibles que aportan unidades a esa presentación en el pedido— aplicado por igual a
  todas sus unidades (en paquetes y sueltas); el descuento nunca se calcula variante por variante
  (FR-011, FR-017).
- Q: Cuando una línea es elegible tanto para una promoción heredada a nivel de producto como para
  una de precio por presentación (FR-013), ¿cuál gana? → A: La que deje menor total para esa
  línea, resuelto con el mismo mecanismo de "mejor promoción por línea" que el motor ya usa hoy
  (spec 012), tratando la promoción por presentación como una candidata más (FR-013).
- Q: ¿Cómo se reparte el descuento entre líneas y cómo se decide qué línea queda con la unidad
  sobrante cuando varias empatan? → A: El descuento se aplica SOLO a las unidades que forman
  paquetes completos; cada una de esas unidades se cobra a `precio_paquete ÷ cantidad_mínima`
  (redondeado a peso colombiano; el residuo se asigna a la unidad tomada de la línea con
  identificador de variante más alto). Las unidades sobrantes se cobran a precio unitario normal y
  se toman de la(s) línea(s) con el identificador de variante más alto (desempate: identificador
  de línea más alto), de forma que el resultado no depende del orden de las líneas del pedido
  (FR-011, CL-9).
- Q: ¿Qué pasa si el precio de paquete de una regla no representa un descuento real
  (`precio_paquete ÷ cantidad_mínima` ≥ precio unitario normal)? → A: Al guardar, el sistema
  advierte y exige confirmación explícita (igual que la verificación de uniformidad de FR-017); la
  regla puede guardarse igual pero nunca en silencio. En el cobro, el motor nunca aplica la
  promoción a una línea si la deja con un total mayor que sin promoción (FR-022, FR-023).
- Q: ¿El anuncio del menú QR se ve siempre que la promoción esté activa, o solo dentro de su
  ventana de día y horario? → A: Solo cuando la promoción está vigente en ese momento según su
  ventana de día y horario (zona horaria del tenant); fuera de esa ventana no se anuncia, aunque
  no esté pausada (FR-021).

## User Scenarios & Testing *(mandatory)*

<!--
  Nota de trazabilidad (Principio XII de la Constitución): las etiquetas entre paréntesis
  E-N/RF-N/CA-N/CL-N remiten a la numeración de la solicitud original que dio origen a esta spec
  (sección "Input" arriba), preservada aquí solo para rastrear de dónde salió cada
  escenario/requisito. El contrato vinculante es el texto de esta spec (User Stories, FR-XXX,
  SC-XXX), no esas etiquetas.
-->

### User Story 1 - El administrador configura una promoción con una o más reglas por presentación (Priority: P1)

Un administrador crea una promoción de precio por cantidad y, en vez de elegir un producto
puntual, elige una presentación del catálogo del tenant (p. ej. "8oz"), define cuántas unidades
mínimas y a qué precio total de paquete. Puede repetir esto para otra presentación dentro de la
misma promoción (p. ej. "16oz" con su propio precio de paquete), y ve un resumen de todas las
reglas — con cuántos productos alcanza cada una — antes de guardar.

**Why this priority**: sin esto no existe ninguna promoción que evaluar; es el punto de entrada
de todo lo demás.

**Independent Test**: crear una promoción con nombre, fechas, días y horas, agregar dos reglas de
presentaciones distintas, ver el resumen antes de guardar, e intentar agregar una tercera regla
repitiendo una presentación ya usada en la misma promoción.

**Acceptance Scenarios**:

1. **Given** un administrador que agrega reglas para dos presentaciones distintas, **When**
   revisa el resumen antes de guardar, **Then** ve ambas reglas listadas junto con cuántos
   productos alcanza cada una (E1, RF-11, CA-1).
2. **Given** un administrador que intenta agregar una segunda regla para una presentación ya
   usada en la misma promoción, **When** intenta guardar, **Then** el sistema lo impide y explica
   por qué (RF-3, CA-2).
3. **Given** una promoción activa con una regla sobre la presentación "8oz", **When** un
   administrador intenta activar otra promoción de este tipo con una regla también sobre "8oz",
   **Then** el sistema lo impide y señala cuál promoción existente genera el conflicto (CL-4,
   decisión de aclaración).
4. **Given** un horario de 22:00 a 02:00, **When** el administrador lo guarda, **Then** el
   sistema lo acepta como una ventana válida que cruza la medianoche (RF-10).

---

### User Story 2 - El cajero cobra pedidos que combinan distintos productos de una misma presentación (Priority: P1)

Un cliente lleva unidades de distintos sabores/productos que comparten presentación. Al cobrar, el
sistema agrupa esas unidades sin importar de qué producto sean, arma tantos paquetes completos
como permita la cantidad de la regla, cobra cada unidad de un paquete completo a
`precio_paquete ÷ cantidad_mínima` y cada unidad sobrante a precio normal. Cuando la unidad
sobrante podría salir de varias líneas, se toma de forma determinista (línea con el identificador
de variante más alto) para que el reparto por línea no dependa del orden del pedido.

**Why this priority**: es el problema de negocio central que motiva la spec — hoy dos sabores
distintos en la misma presentación no se combinan para la promoción, aunque comercialmente sea el
mismo caso.

**Independent Test**: con una regla "2 x 8oz por $12.000" activa y variantes de 8oz a $7.000
c/u, cobrar combinaciones de líneas con distintos productos en 8oz (2, 3, 5 unidades; con y sin
otra presentación en el mismo pedido) y verificar el total y el reparto del descuento por línea,
en cualquier orden de las líneas.

**Acceptance Scenarios**:

1. **Given** 1 unidad de Ojo de Diablo 8oz + 1 unidad de Fresa Boom 8oz (regla 2x$12.000, precio
   normal $7.000 c/u), **When** se cobra, **Then** el total es $12.000 y la terminal muestra la
   etiqueta de la promoción aplicada (E2, CA-3).
2. **Given** 1 unidad de Ojo de Diablo + 1 de Fresa Boom + 1 de Maracumango, todas 8oz, **When**
   se cobra, **Then** el total es $19.000: dos líneas se cobran a $6.000 (su unidad entró al
   paquete de $12.000) y una a $7.000 (unidad suelta); qué línea queda como suelta se decide de
   forma determinista (línea con el identificador de variante más alto), no por el orden de
   captura (E3, CA-4).
3. **Given** el mismo pedido del punto anterior con las líneas en otro orden, **When** se cobra,
   **Then** el total y el reparto del descuento entre líneas son idénticos (CA-5).
4. **Given** reglas activas "2 x 8oz por $12.000" y "2 x 16oz por $16.500", con variantes de 16oz a
   $9.500 c/u, **When** se cobra un pedido de 2 unidades en 8oz (sabores distintos) + 2 unidades en
   16oz, **Then** el total es $28.500 ($12.000 + $16.500: un paquete completo por cada
   presentación) y nunca se combinan unidades de 8oz con unidades de 16oz en un mismo paquete.
5. **Given** 5 unidades en 8oz, **When** se cobra, **Then** el total es $31.000 (2 paquetes de
   $12.000 + 1 unidad suelta de $7.000).
6. **Given** 1 unidad en 8oz + 1 unidad en 16oz, ninguna alcanzando la cantidad mínima de su
   regla, **When** se cobra, **Then** el total es $16.500 sin ninguna etiqueta de promoción (E4,
   CA-6).
7. **Given** el pedido del punto 2, cobrado un día de la semana no incluido en la promoción,
   **When** se cobra, **Then** se cobra sin descuento (CA-7).
8. **Given** una promoción con ventana horaria de 08:00 a 22:00, **When** el mismo pedido se
   cobra a las 07:59, **Then** no hay descuento; **When** se cobra a las 08:01, **Then** sí lo
   hay (CA-8).
9. **Given** una regla activa "3 x 8oz por $10.000" y tres líneas de sabores distintos en 8oz a
   $7.000 c/u, **When** se cobra en cualquier orden de captura, **Then** el total es exactamente
   $10.000: `redondear($10.000 ÷ 3)` = $3.333 por unidad de paquete y el residuo de $1 se cobra en
   la unidad de la línea con el identificador de variante más alto ($3.334); la suma de las tres
   líneas y la suma de los descuentos por línea ($3.667 + $3.667 + $3.666 = $11.000) cuadran al
   peso (CL-9, SC-005).
10. **Given** una regla activa "2 x 8oz por $12.000", 1 unidad de una variante activa en 8oz y 1
    unidad de otra variante en 8oz que quedó desactivada (`active = false`) después de agregarse al
    pedido, ambas a $7.000, **When** se cobra, **Then** el total es $14.000 sin etiqueta de
    promoción: la unidad de la variante desactivada NO cuenta como unidad elegible, así que no se
    forma ningún paquete (CL-1c, FR-015).

---

### User Story 3 - El sistema advierte si los productos de una presentación no tienen precio uniforme (Priority: P2)

Al guardar una regla, el sistema verifica que todos los productos que la regla alcanza tengan el
mismo precio unitario en esa presentación. Si alguno difiere, se lo muestra al administrador y
exige confirmación explícita antes de guardar — la regla puede guardarse igual, pero nunca en
silencio.

**Why this priority**: sin esto, una regla puede guardarse y luego el cobro usar un precio
unitario normal único (FR-011, el menor de la presentación) sobre productos que en realidad no
cuestan lo mismo en esa presentación, sin que el administrador se entere de que las variantes más
caras se cobrarán como si costaran ese precio más bajo.

**Independent Test**: con dos productos en la misma presentación a precios distintos, crear una
regla sobre esa presentación y verificar que el sistema muestra la diferencia y bloquea el guardado
hasta que el administrador confirme explícitamente.

**Acceptance Scenarios**:

1. **Given** un producto con precio distinto al resto de su presentación, **When** un
   administrador guarda una regla sobre esa presentación, **Then** la interfaz lo advierte y exige
   confirmación explícita antes de guardar (CA-10, RF-8).
2. **Given** una promoción ya activa, **When** un producto de su presentación cambia de precio
   después (rompiendo la uniformidad), **Then** el sistema no revalida retroactivamente — la
   verificación solo corre al momento de guardar la regla (CL-1).
3. **Given** una promoción activa con una regla sobre "8oz", **When** se crea una variante nueva y
   se le asigna la presentación "8oz", **Then** esa variante entra en la promoción sin pasar por
   esta verificación de uniformidad (CL-1b).

---

### User Story 4 - Un producto nuevo con la presentación de una regla activa entra automáticamente a la promoción (Priority: P2)

Un administrador crea un producto (o una variante) con la presentación "8oz" mientras una
promoción con una regla sobre "8oz" está activa. Sin editar la promoción, ese producto queda
alcanzado por la regla la próxima vez que se cobre.

**Why this priority**: sin esto, cada producto nuevo obligaría a editar todas las promociones
vigentes que compartan su presentación — justo el problema de mantenimiento que esta
funcionalidad busca evitar.

**Independent Test**: con una regla activa sobre "8oz", crear un producto nuevo con una variante
en presentación "8oz", y verificar que un pedido con ese producto nuevo combina para el paquete
sin haber tocado la promoción.

**Acceptance Scenarios**:

1. **Given** una promoción activa con una regla sobre la presentación "8oz", **When** se crea un
   producto nuevo con una variante en "8oz", **Then** ese producto entra en la promoción sin que
   el administrador la edite (E4, CA-9).
2. **Given** una presentación referenciada por una regla de una promoción activa, **When** un
   administrador intenta eliminarla o desactivarla, **Then** el sistema lo impide y muestra qué
   promoción(es) la usan (CL-2, decisión de aclaración).

---

### User Story 5 - El cliente ve la promoción anunciada en el menú QR (Priority: P3)

Un cliente que consulta el menú público por QR ve que existe una promoción de precio por
presentación vigente en ese momento, junto con su condición en lenguaje llano, sin necesidad de
agregar nada al carrito.

**Why this priority**: valor de descubrimiento — anima a pedir para alcanzar la promoción, pero
no bloquea el cobro correcto (User Story 2), que es el valor central.

**Independent Test**: con una promoción activa "2 x 8oz por $12.000", abrir el menú QR sin
agregar productos al carrito dentro de su horario y verificar que la condición de la promoción es
visible; volver a abrirlo fuera de horario y verificar que el anuncio ya no aparece.

**Acceptance Scenarios**:

1. **Given** una promoción de precio por presentación vigente en ese momento, **When** un cliente
   abre el menú QR sin agregar nada al carrito, **Then** ve la promoción anunciada con su
   condición legible (p. ej. "Llevando 2 de cualquier sabor en 8oz") (CA-11).
2. **Given** una promoción de precio por presentación activa pero fuera de su ventana de día u
   horario en ese momento, **When** un cliente abre el menú QR, **Then** no ve esa promoción
   anunciada (aclaración 2026-08-26).

---

### Edge Cases

- **CL-1 / CL-1b** — cubiertos por User Story 3, Acceptance Scenarios 2 y 3: la verificación de
  uniformidad de precio (FR-017) solo corre al guardar la regla, nunca retroactivamente.
- **CL-1c**: una variante desactivada (`active = false`) no cuenta como unidad elegible para
  completar un paquete, aunque el pedido tenga unidades de ella (FR-015).
- **CL-3**: si el precio de una o más variantes de la presentación cambia después de creada la
  regla, el precio de paquete queda fijo y el precio unitario normal se recalcula en cada cobro
  (FR-011) — el descuento efectivo varía sin que el sistema avise de nuevo más allá de la
  verificación que ya corrió al guardar la regla (FR-017). Es el comportamiento esperado, no un
  error.
- **CL-5**: si se elimina una línea del pedido y eso deshace un paquete, el siguiente cálculo de
  cobro recalcula desde cero el conteo de unidades elegibles y los paquetes completos (FR-014) —
  nunca ajusta un descuento mostrado previamente.
- **CL-6**: si la promoción se pausa mientras una sesión de mesa tiene líneas cargadas, el
  siguiente cálculo de cobro (FR-014) ya no la encuentra activa y no aplica su descuento — no
  existe un "descuento congelado" de una promoción pausada.
- **CL-7**: una regla con cantidad mínima 1 es válida — equivale a un precio especial por unidad
  de esa presentación, y se calcula con la misma lógica de paquetes completos (FR-010).
- **CL-8**: cuando la ventana horaria cruza la medianoche, el día de aplicación se determina por
  el día en que inicia la ventana — las horas después de medianoche siguen perteneciendo al día
  de inicio (FR-004).
- **CL-9**: cuando `precio_total_del_paquete ÷ cantidad_mínima` no da un valor exacto en pesos,
  el precio por unidad de paquete se redondea a peso colombiano y el residuo se asigna a la unidad
  del paquete que provenga de la línea con el identificador de variante más alto, de modo que la
  suma de las unidades del paquete cuadre exactamente con el precio del paquete (FR-011).
- **Coexistencia con promociones heredadas**: una línea nunca recibe el descuento de una
  promoción heredada a nivel de producto y de una nueva a nivel de presentación al mismo tiempo
  (FR-013, decisión de aclaración).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001 [Historia 1]** (RF-1): Una promoción de tipo "precio por cantidad por presentación"
  DEBE contener una o más reglas. Cada regla es la tripleta (presentación, cantidad mínima,
  precio total del paquete).
- **FR-002 [Historia 1]** (RF-9): La promoción DEBE tener: nombre, descripción opcional, fecha de
  inicio (obligatoria), fecha de fin (opcional), días de la semana en los que aplica, y hora de
  inicio y fin (opcionales).
- **FR-003 [Historia 1]** (RF-10): Cuando la promoción define hora de inicio y fin, el sistema
  DEBE permitir que la ventana cruce la medianoche (ej. 22:00 a 02:00), y DEBE evaluar días y
  horas en la zona horaria del tenant.
- **FR-004 [Historia 1]** (CL-8): Cuando la ventana horaria cruza la medianoche, el día de
  aplicación de la promoción DEBE determinarse por el día en que INICIA la ventana — las horas
  posteriores a la medianoche siguen perteneciendo al día de inicio para efectos de evaluar
  vigencia.
- **FR-005 [Historia 1]** (RF-11): Antes de confirmar la creación o edición, la interfaz DEBE
  mostrar un resumen legible de todas las reglas de la promoción, incluyendo cuántos productos
  alcanza cada una.
- **FR-006 [Historia 1, decisión de aclaración]** (RF-3, CL-4): Dentro de una misma promoción, el
  sistema NO DEBE permitir dos reglas para la misma presentación (RF-3). Además, el sistema NO
  DEBE permitir que una regla de una promoción de este tipo, al guardarse o activarse, apunte a
  una presentación ya cubierta por una regla de OTRA promoción de precio-por-cantidad-por-
  presentación que esté activa al mismo tiempo — DEBE explicar cuál promoción existente genera el
  conflicto.
- **FR-007 [Historia 2]** (RF-2, RF-2b): Cada regla DEBE aplicar automáticamente a todas las
  variantes de producto activas del tenant que referencien la presentación de la regla, incluidas
  las creadas después de guardar la promoción. El alcance se resuelve por la referencia a la
  presentación — nunca comparando el nombre de la variante.
- **FR-008 [Historia 2]** (RF-2c): Una variante sin presentación asignada NO DEBE entrar en
  ninguna regla por presentación. En particular, la variante "Single" de un producto sin tamaños
  definidos no participa de ninguna promoción por presentación salvo que un administrador le
  asigne una presentación de forma explícita.
- **FR-009 [Historia 2]** (RF-4): Al cobrar, el sistema DEBE poder formar un paquete con
  cualquier combinación de unidades elegibles de la presentación de la regla, sin importar de qué
  producto provenga cada unidad. El sistema NUNCA DEBE mezclar unidades de presentaciones
  distintas dentro de un mismo paquete.
- **FR-010 [Historia 2]** (RF-5): El sistema DEBE descontar únicamente paquetes completos — tantos
  como permita la división entera del total de unidades elegibles de esa presentación entre la
  cantidad mínima de la regla — y DEBE cobrar toda unidad sobrante al precio unitario normal de la
  presentación definido en FR-011.
- **FR-011 [Historia 2]** (RF-6, aclaración 2026-08-26): A todos los efectos del cálculo de esta
  promoción, el precio unitario normal de una presentación es ÚNICO: el menor precio unitario
  vigente entre las variantes elegibles que aportan unidades a esa presentación en el pedido,
  aplicado por igual a todas esas unidades sin importar de qué variante provengan. El descuento
  NUNCA se calcula variante por variante. El descuento se aplica ÚNICAMENTE a las unidades que
  entran en un paquete completo: cada una de esas unidades se cobra a `precio_total_del_paquete ÷
  cantidad_mínima` (redondeado a peso colombiano, sin decimales; el residuo de esa división se
  asigna a la unidad que provenga de la línea con el identificador de variante más alto). Las
  unidades sobrantes se cobran al precio unitario normal, sin descuento. Cuando la unidad (o
  unidades) sobrante pudiera provenir de varias líneas empatadas, DEBE tomarse de la(s) línea(s)
  con el identificador de variante más alto y, si dos líneas comparten variante, la de
  identificador de línea más alto — de modo que el total cobrado y el reparto por línea NO
  dependan del orden de las líneas del pedido. El descuento de cada línea es una magnitud derivada
  (unidades de la línea en esa presentación × precio normal − lo efectivamente cobrado a esa
  línea) y la suma de los descuentos por línea DEBE cuadrar exactamente con el descuento total, al
  peso.
- **FR-012 [Historia 2]** (RF-7): Una presentación sin ninguna regla en la promoción NO DEBE
  recibir descuento de esta promoción.
- **FR-013 [Historia 2, decisión de aclaración]** (CL-4, coexistencia): Cuando una línea del
  pedido sea elegible tanto para el descuento de una promoción heredada configurada a nivel de
  producto (FR-016) como para el de una promoción de precio por presentación, el sistema DEBE
  aplicarle el descuento de una sola de ellas — nunca DEBE acumular el descuento de ambas sobre
  la misma línea. Entre las dos, DEBE prevalecer la que deje MENOR total para esa línea, resuelto
  con el mismo mecanismo de "mejor promoción por línea" que el motor de promociones ya usa hoy
  (spec 012), tratando la promoción de precio por presentación como una candidata más.
- **FR-014 [Historia 2]** (RF-12): El descuento de esta promoción NUNCA DEBE persistirse; DEBE
  calcularse en el momento de cobrar, recalculando desde el estado actual del pedido cada vez
  (igual que el resto del motor de promociones existente).
- **FR-015 [Historia 2]** (CL-1c): Una variante desactivada (`active = false`) NO DEBE contar
  como unidad elegible para completar un paquete, aunque el pedido tenga unidades de ella.
- **FR-016 [Historia 2]** (RF-13): Las promociones existentes configuradas a nivel de producto
  DEBEN seguir funcionando sin cambios hasta que un administrador las edite explícitamente.
- **FR-017 [Historia 3]** (RF-8): Al guardar una regla, el sistema DEBE verificar que todas las
  variantes alcanzadas por la regla tengan el mismo precio unitario en esa presentación. Si
  alguna difiere, DEBE mostrarlo al administrador y exigir confirmación explícita antes de
  guardar. Haya o no diferencia de precios, el cálculo del cobro NO usa el precio de cada variante
  por separado: usa el precio unitario normal único de la presentación definido en FR-011 (el
  menor precio vigente entre las variantes elegibles). La verificación de uniformidad existe para
  que el administrador sepa que, si confirma, las variantes más caras se cobrarán como si costaran
  ese precio más bajo.
- **FR-018 [Historia 3]** (CL-1, CL-1b): Esta verificación de uniformidad de precio (FR-017) DEBE
  ejecutarse únicamente en el momento de guardar la regla — un producto que rompe la uniformidad
  después de que la promoción ya está activa, o una variante nueva a la que se le asigna la
  presentación de una regla activa, NO pasan por esta verificación retroactivamente.
- **FR-019 [Historia 4]** (RF-2, E4): Un producto o variante nuevo creado con la presentación de
  una regla que pertenece a una promoción activa DEBE quedar incluido en esa regla
  automáticamente, sin que el administrador edite la promoción.
- **FR-020 [Historia 4, decisión de aclaración]** (CL-2): El sistema NO DEBE permitir eliminar ni
  desactivar una presentación que esté referenciada por una regla de una promoción activa. DEBE
  explicar qué promoción(es) la referencian y exigir que el administrador edite o pause la
  promoción antes de continuar.
- **FR-021 [Historia 5]** (CA-11): El menú QR público DEBE mostrar, para cada promoción de precio
  por presentación activa, un anuncio con su condición legible (p. ej. "Llevando 2 de cualquier
  sabor en presentación 8oz"), sin que el cliente necesite agregar ningún producto al carrito
  para verla. El anuncio DEBE mostrarse ÚNICAMENTE mientras la promoción está vigente en ese
  momento según su ventana de día y horario (en la zona horaria del tenant, FR-003); fuera de esa
  ventana NO se muestra, aunque la promoción no esté pausada.
- **FR-022 [Historia 1, aclaración 2026-08-26]**: Al guardar una regla, si
  `precio_total_del_paquete ÷ cantidad_mínima` es mayor o igual al precio unitario normal de la
  presentación (el menor precio vigente entre las variantes elegibles, FR-011) — es decir, si la
  regla no representa un descuento real — el sistema DEBE advertirlo al administrador y exigir
  confirmación explícita antes de guardar, con el mismo patrón que la verificación de uniformidad
  de FR-017. La regla PUEDE guardarse igual, pero NUNCA en silencio.
- **FR-023 [Historia 2, aclaración 2026-08-26]**: Al cobrar, el motor NUNCA DEBE aplicar la
  promoción de precio por presentación a una línea si aplicarla deja a esa línea con un total
  mayor que sin ninguna promoción. Se rige por el mismo criterio de "mejor promoción por línea"
  del motor existente (spec 012, ver FR-013), que ya contempla "ninguna promoción" como
  alternativa.

### Key Entities *(include if feature involves data)*

- **Promoción de precio por cantidad por presentación**: agrupa una o más reglas bajo un nombre,
  descripción, ventana de vigencia (fechas, días, horas) y estado. Es una modalidad adicional del
  tipo de promoción "precio por cantidad" existente, no un reemplazo.
- **Regla de presentación**: pertenece a una única promoción; define una presentación, una
  cantidad mínima de paquete y un precio total de paquete. No puede repetirse la misma
  presentación dos veces dentro de la misma promoción (FR-006).
- **Presentación**: concepto del catálogo compartido del tenant (p. ej. "8oz", "16oz") al que
  distintas variantes de distintos productos pueden hacer referencia. Es la unidad natural de
  precio de este negocio — todos los productos en una misma presentación cuestan lo mismo. Si en
  la práctica no cuestan lo mismo, el cálculo del cobro de esta promoción usa un único precio
  unitario normal para la presentación: el menor precio vigente entre sus variantes elegibles
  (FR-011, FR-017). Hoy no existe como entidad compartida entre productos (ver "Naturaleza de esta
  spec"); su modelado es trabajo de la fase de plan.
- **Variante de producto**: unidad vendible de un producto. Puede referenciar una presentación
  del catálogo compartido; si no la referencia, queda fuera de toda regla por presentación
  (FR-008). Su estado `active` determina si cuenta para completar paquetes (FR-015).
- **Línea de pedido**: cantidad de una variante dentro de un pedido. Aporta unidades elegibles a
  la presentación de su variante para efectos de formar paquetes; sus unidades se cobran a precio
  de paquete o a precio unitario normal según entren o no en un paquete completo (FR-011).
- **Paquete**: agrupación de unidades elegibles de una presentación, del tamaño definido por la
  regla, formada al momento de cobrar. Solo los paquetes completos generan descuento (FR-010); cada
  unidad de un paquete completo se cobra a `precio_total_del_paquete ÷ cantidad_mínima` (FR-011).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los pedidos que reúnen la cantidad mínima de una regla activa en una
  presentación —sin importar de qué producto sean las unidades ni el orden de las líneas del
  pedido— reciben el precio de paquete correspondiente.
- **SC-002**: El 100% de los intentos de guardar dos reglas de la misma presentación dentro de
  una promoción, o de activar una promoción con una regla que se solapa con otra promoción activa
  sobre la misma presentación, son rechazados con una explicación clara del conflicto.
- **SC-003**: El 0% de los pedidos cobrados fuera del día o del horario configurado de la
  promoción reciben su descuento.
- **SC-004**: El 100% de los productos nuevos creados con una presentación ya cubierta por una
  regla de una promoción activa quedan incluidos en su descuento sin que el administrador edite
  la promoción.
- **SC-005**: El 100% de los cobros con paquetes completados asignan a cada línea un cobro tal que
  la suma de los descuentos por línea (unidades de la línea en la presentación × precio normal −
  lo cobrado a la línea) cuadra exactamente con el descuento total, sin perder ni sobrar un peso,
  sin importar el orden de las líneas.
- **SC-006**: Un cliente del menú QR puede identificar la condición de una promoción de
  presentación vigente en ese momento sin agregar ningún producto al carrito, y no ve anunciadas
  las promociones que están fuera de su ventana de día u horario.

## Out of Scope

- Paquetes que mezclen presentaciones distintas dentro del mismo paquete.
- Excluir productos concretos de una regla mientras el precio se mantenga uniforme por
  presentación.
- El tipo de promoción "compra X lleva Y".
- Editar el tipo o el alcance de una promoción después de creada.
- Migrar automáticamente las promociones existentes al nuevo formato de reglas por presentación.
- Definir el modelo de datos concreto de la entidad "presentación compartida" (tablas, columnas,
  migraciones) — corresponde a `/speckit-plan` (Principio VIII de la Constitución).
- Registrar formalmente la **modalidad de descuento** en `registro-de-anomalias.md` — no aplica: no
  es la corrección de un comportamiento existente, es una modalidad nueva que se suma a la actual
  (ver "Naturaleza de esta spec"). La única entrada que sí corresponde es **A-55**, por la
  corrección de `_valid_now` que exige FR-004 (atribución de día al cruzar medianoche), que sí
  cambia comportamiento observable de promociones existentes.

## Assumptions

- **La "sección 9" de preguntas abiertas** referenciada en la solicitud original no llegó
  incluida en el texto recibido para esta spec. Las ambigüedades de mayor impacto se
  identificaron y resolvieron en su lugar mediante las preguntas de aclaración de esta
  conversación (ver Clarifications), priorizando alcance y correctitud del cálculo de cobro.
- **El mockup de interfaz adjunto** (formulario de "Información General" + "Configuración de
  Reglas", con panel lateral de "Productos Aplicables" y "Resumen de la Regla") se toma como
  referencia visual para la fase de plan/implementación. Confirma, sin contradecir, lo ya exigido
  por FR-002, FR-005 y User Story 1 (Acceptance Scenarios 1 y 2); no se trata como contenido
  normativo de esta especificación, que se mantiene libre de detalles de interfaz o
  implementación.
  Vinculado a: FR-002, FR-005.
- **"Excluyentes por línea" (decisión de aclaración, FR-013)** se asume satisfecha reutilizando el
  mecanismo que el motor de promociones ya usa hoy para elegir una única promoción ganadora por
  línea entre varias candidatas (spec 012), extendido para incluir esta nueva modalidad como
  candidata adicional — no se asume la introducción de un mecanismo de exclusión distinto.
  Vinculado a: FR-013.
  El reparto por línea dentro de una misma presentación ya quedó especificado (FR-011: precio de
  paquete por unidad, unidad sobrante a la línea de identificador de variante más alto) y la
  precedencia frente a la promoción de producto también (FR-013: gana la de menor total).
  Riesgo residual (identificado al redactar la spec): el mecanismo de "mejor promoción por línea" de
  spec 012 se evalúa hoy línea por línea y este descuento nace de agrupar varias líneas a la vez.
  **Resuelto en la fase de plan** (research.md D6): la modalidad entra como candidata con un
  recálculo del conjunto de líneas del paquete hasta punto fijo — cuando una línea se adjudica al
  motor línea-por-línea, sus unidades salen del pool y el paquete se recalcula, de modo que una
  línea nunca conserva el precio de paquete si sus unidades ya no completan uno.
- **"Presentación" es una entidad nueva y compartida del catálogo del tenant**, distinta de la
  variante de producto — hoy cada variante pertenece a un único producto sin ningún concepto
  compartido entre productos al que varias variantes de productos distintos puedan hacer
  referencia simultáneamente. Su creación y gestión como entidad de catálogo es una decisión de
  modelo de datos que corresponde a la fase de plan (Principio VIII de la Constitución), no a
  esta especificación.
  Vinculado a: FR-007, Key Entities (Presentación).
- **El cambio de precio de una variante de una presentación después de creada una regla (CL-3)**
  no dispara ningún aviso adicional al administrador más allá de la verificación que ya corre al
  guardar la regla (FR-017) — se documenta como comportamiento esperado, no como una alerta
  pendiente de construir. El precio de paquete queda fijo; el precio unitario normal se recalcula
  en cada cobro (FR-011).
