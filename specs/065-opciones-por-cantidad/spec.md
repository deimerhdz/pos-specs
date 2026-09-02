# Feature Specification: Selección por cantidad en grupos de opciones

**Feature Branch**: `065-opciones-por-cantidad`

**Created**: 2026-09-01

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad **nueva** de catálogo y de pedido (fase de evolución
funcional, Principio I de la [Constitución](../../.specify/memory/constitution.md)). No reabre el
cálculo de precio de línea ni el consumo por opción ya definidos en
[spec 002](../002-catalogo-productos-variantes-y-precios/spec.md)/
[spec 003](../003-consumo-de-inventario-por-receta-y-opcion/spec.md) para el modo de selección
actual ("conteo") — los extiende con un segundo modo ("cantidad") que coexiste con el primero, sin
cambiar su comportamiento por defecto. Tampoco reabre spec 004 (validación de forma) ni spec 063
(tipo de precio "incluido"/"con_recargo", switch de inventario condicional) — ambas siguen
aplicando sin cambios sobre grupos en modo "conteo", y se extienden explícitamente para definir su
interacción con el modo "cantidad" nuevo.

**Input**: User description: "El grupo de opciones actual solo permite un modo de selección: elegir
N de M opciones distintas (min_select/max_select, contadas como on/off). Necesito un segundo modo,
'por cantidad': el cliente elige libremente cuántas unidades de cada opción agregar (ej. 2 Bobombún
+ 1 Gomitas), con el precio y — si el grupo maneja inventario — el consumo multiplicándose por esa
cantidad. Un OptionGroup debe declarar cuál de los dos modos usa; el modo actual (conteo) sigue
siendo el comportamiento por defecto para no romper catálogos existentes. Y en la orden se debe
tener en cuenta al momento de realizar los cálculos si el producto viene con toppings o no, y
registrar pagos y ventas."

## Clarifications

### Session 2026-09-01

- Q: Para un grupo en modo "por cantidad", ¿el administrador debe poder poner un tope máximo (por
  topping individual y/o total del grupo), o la cantidad debe ser completamente libre? → A: Tope
  por opción Y tope total, ambos configurables y opcionales — el administrador puede fijar un
  máximo por topping individual (ej. "máx. 3 Bobombún"), un máximo total de unidades del grupo (ej.
  "máx. 5 toppings en total"), ambos, o ninguno (sin tope, solo limitado por stock real si el grupo
  maneja inventario).
- Q: ¿Un grupo "por cantidad" puede exigir un mínimo de unidades, o los toppings siempre deben ser
  opcionales? → A: Siempre opcional, sin mínimo posible — un grupo en modo "por cantidad" nunca
  bloquea el pedido por no haber elegido nada; el cliente puede avanzar con cero unidades de todas
  sus opciones.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Declarar que un grupo se elige "por cantidad" (Priority: P1)

Un administrador de catálogo crea o edita el grupo "Toppings" y lo marca con modo de selección
"Por cantidad", en vez del modo "Conteo" (N de M) que ya existe hoy. Todo grupo nuevo o ya
existente que no se toque sigue siendo "Conteo" — el modo "por cantidad" es una elección explícita,
nunca implícita.

**Why this priority**: es la base sobre la que se construyen todas las demás historias — sin un
modo declarado explícitamente, ningún otro comportamiento nuevo puede activarse sin arriesgar el
catálogo existente.

**Independent Test**: se puede probar creando un grupo nuevo, marcándolo "Por cantidad", y
verificando que un grupo creado sin tocar ese campo (o uno migrado del catálogo anterior) queda en
"Conteo".

**Acceptance Scenarios**:

1. **Given** el formulario de un grupo de opciones nuevo, **When** el administrador no elige
   ningún modo de selección, **Then** el grupo se crea en modo "Conteo" — el comportamiento de hoy
   (`min_select`/`max_select` contando opciones distintas) sigue siendo el default.
2. **Given** el mismo formulario, **When** el administrador elige explícitamente "Por cantidad",
   **Then** el grupo se crea en ese modo, y el formulario deja de pedir `min_select`/`max_select`
   como rango de opciones distintas — en su lugar ofrece los topes de cantidad de la Historia 4.
3. **Given** un grupo "Conteo" ya existente, **When** el administrador lo cambia a "Por cantidad" (o
   viceversa), **Then** el sistema aplica el cambio para selecciones futuras, sin alterar pedidos ya
   confirmados que usaron el modo anterior (Principio VII, compatibilidad con datos históricos).

---

### User Story 2 - El cliente elige libremente cuántas unidades de cada topping agregar (Priority: P1)

Un comensal que arma su pedido en el menú QR (o un mesero/cajero armando una venta) llega a un
grupo "Toppings" marcado "Por cantidad". En vez de solo poder marcar/desmarcar cada topping, ve un
control de cantidad (`+`/`-`) por cada opción, y puede agregar, por ejemplo, 2 unidades de
"Bobombún" y 1 de "Gomitas" en la misma línea de producto.

**Why this priority**: es el problema central reportado — hoy el cliente está forzado a elegir
"1 de 1" (o el rango que el administrador configuró como conteo de opciones distintas), sin poder
pedir más de una unidad de un mismo topping.

**Independent Test**: se puede probar abriendo un producto con un grupo "Toppings" en modo "Por
cantidad", incrementando la cantidad de una opción a 2 y la de otra a 1, y verificando que ambas
quedan reflejadas en el carrito con sus cantidades respectivas.

**Acceptance Scenarios**:

1. **Given** un grupo "Toppings" en modo "Por cantidad" con las opciones "Bobombún" y "Gomitas",
   **When** el comensal incrementa "Bobombún" a 2 y "Gomitas" a 1, **Then** el carrito registra esa
   línea con ambas cantidades, distinguibles entre sí (2 de una opción, 1 de la otra, nunca
   fusionadas en un solo número).
2. **Given** el mismo grupo, **When** el comensal no incrementa ninguna opción (todas en 0),
   **Then** el pedido avanza sin ningún topping de ese grupo — un grupo "por cantidad" nunca
   bloquea por no elegir nada (Clarifications, sesión 2026-09-01).
3. **Given** una opción ya en cantidad 1, **When** el comensal la baja a 0, **Then** esa opción deja
   de formar parte de la selección — bajar a 0 es equivalente a nunca haberla elegido.
4. **Given** un grupo "Toppings" en modo "Conteo" (sin cambios de esta spec), **When** el comensal
   lo usa, **Then** sigue comportándose exactamente igual que hoy — marcar/desmarcar hasta el
   máximo de opciones distintas configurado, sin ningún control de cantidad visible.

---

### User Story 3 - El precio y el consumo de inventario del pedido reflejan la cantidad elegida (Priority: P1)

Un cajero confirma un pedido que incluye una línea con toppings en modo "Por cantidad". El sistema
calcula el precio de esa línea sumando el precio de la presentación más el recargo de **cada
unidad** elegida de cada topping (no solo una vez por topping), y — si el producto maneja
inventario — descuenta el insumo de cada topping multiplicado por la cantidad elegida.

**Why this priority**: es el requisito explícito del pedido original ("en la orden se debe tener en
cuenta al momento de realizar los cálculos si el producto viene con toppings o no") — sin este
cálculo correcto, la Historia 2 permite elegir cantidades que después no se cobran ni se descuentan
como corresponde.

**Independent Test**: se puede probar armando una línea con "Bobombún" (recargo $1.000) en cantidad
2 y "Gomitas" (recargo $800) en cantidad 1 sobre una presentación de $15.000, y verificando que el
precio de línea es $15.000 + (2×$1.000) + (1×$800) = $17.800; y, si el producto maneja inventario,
que el insumo de "Bobombún" se descuenta 2 veces su cantidad configurada y el de "Gomitas" 1 vez.

**Acceptance Scenarios**:

1. **Given** una presentación de $15.000 con "Bobombún" (recargo $1.000, cantidad elegida 2) y
   "Gomitas" (recargo $800, cantidad elegida 1), **When** se calcula el precio de la línea, **Then**
   el resultado es $15.000 + $2.000 + $800 = $17.800 — el recargo de una opción se multiplica por
   la cantidad elegida de esa opción, nunca se cuenta una sola vez sin importar cuántas unidades se
   pidieron.
2. **Given** el mismo producto **maneja inventario**, **When** se confirma la venta, **Then** el
   insumo ligado a "Bobombún" se descuenta dos veces la cantidad configurada para esa opción (o la
   que defina el tamaño, según la regla ya vigente de "una sola cantidad manda, nunca se suman" —
   spec 003), y el de "Gomitas" una vez — cada topping sigue generando su propio movimiento de
   inventario, ahora multiplicado por su cantidad elegida.
3. **Given** un grupo "Toppings" marcado como "Incluido" (spec 063) y además "Por cantidad", **When**
   el comensal elige 2 unidades de una opción de ese grupo, **Then** el precio de línea no suma
   ningún recargo por esas 2 unidades — "Incluido" sigue bloqueando el precio en $0 sin importar la
   cantidad elegida; solo el consumo de inventario (si aplica) se multiplica por la cantidad.
4. **Given** un grupo "Toppings" marcado "Con recargo" (spec 063) y "Por cantidad", **When** el
   comensal elige cantidades de varias opciones, **Then** cada una cobra su propio recargo
   multiplicado por su cantidad — los dos ejes (tipo de precio de spec 063, modo de selección de
   esta spec) son independientes entre sí y se combinan sin casos especiales.

---

### User Story 4 - El administrador limita cuántas unidades se pueden pedir (Priority: P2)

Un administrador configura el grupo "Toppings" en modo "Por cantidad" y quiere evitar pedidos
absurdos o que agoten el inventario de un solo topping: fija un máximo de 3 unidades por topping
individual y un máximo de 5 unidades en total para el grupo completo.

**Why this priority**: sin un tope configurable, un cliente podría pedir cantidades
desproporcionadas de un mismo topping, con impacto directo en costo e inventario — es una
protección necesaria pero secundaria a que la funcionalidad básica (Historias 1-3) exista primero.

**Independent Test**: se puede probar configurando ambos topes, intentando subir un topping
individual por encima de su máximo (el control deja de incrementar), y sumando varios toppings
hasta el máximo total (el sistema impide seguir sumando aunque un topping individual no haya
llegado a su propio tope).

**Acceptance Scenarios**:

1. **Given** un grupo "Toppings" con máximo de 3 unidades por opción, **When** el comensal intenta
   subir "Bobombún" a 4, **Then** el sistema no permite superar 3 para esa opción.
2. **Given** el mismo grupo con máximo total de 5 unidades, **When** el comensal ya tiene 3
   "Bobombún" y 2 "Gomitas" (5 en total) e intenta agregar una unidad más de cualquier topping,
   **Then** el sistema lo impide — el tope total aplica sobre la suma de todas las opciones del
   grupo, no solo sobre una individual.
3. **Given** un grupo "Por cantidad" sin ningún tope configurado, **When** el comensal agrega
   unidades, **Then** no hay ningún límite propio del grupo — si el producto maneja inventario, el
   único límite real es el stock disponible del insumo (chequeo preventivo ya vigente, spec 003).
4. **Given** un grupo con tope por opción configurado pero sin tope total (o viceversa), **When** el
   comensal agrega cantidades, **Then** cada tope configurado se valida de forma independiente —
   ambos son opcionales y no dependen uno del otro.

---

### User Story 5 - La comanda, el recibo y "Mis pedidos" muestran cuántas unidades de cada topping se pidieron (Priority: P2)

Un cocinero mirando la comanda, un comensal revisando "Mis pedidos" en el menú QR, o un cliente
leyendo su recibo impreso, ven claramente "2x Bobombún" en vez de ver "Bobombún" listado una vez sin
indicar cantidad, o repetido de forma confusa.

**Why this priority**: sin esto, la Historia 2 y 3 funcionan por dentro pero nadie en el punto de
preparación o de cobro puede ver cuánto topping preparar o qué se cobró exactamente — es de menor
prioridad que el cálculo correcto porque no bloquea la venta, pero sí la calidad operativa.

**Independent Test**: se puede probar confirmando un pedido con cantidades distintas de dos
toppings, y verificando que la comanda de cocina, el detalle de la orden, "Mis pedidos" y el recibo
impreso muestran la cantidad de cada uno.

**Acceptance Scenarios**:

1. **Given** una línea con "Bobombún" en cantidad 2 y "Gomitas" en cantidad 1, **When** se genera la
   comanda de cocina, **Then** se lee "2x Bobombún, Gomitas" (o formato equivalente que distinga la
   cantidad de cada topping) — nunca "Bobombún, Bobombún, Gomitas" ni "Bobombún, Gomitas" sin
   indicar que son 2 de la primera.
2. **Given** la misma línea, **When** el comensal consulta "Mis pedidos" en el menú QR, o el cajero
   revisa el detalle de la orden, o se imprime el recibo, **Then** las tres superficies muestran la
   misma información de cantidad por topping, de forma consistente entre sí.
3. **Given** una línea con un grupo en modo "Conteo" (sin cambios), **When** se muestra en cualquiera
   de esas superficies, **Then** se ve exactamente igual que hoy — un nombre por opción elegida, sin
   ningún prefijo de cantidad (porque en "Conteo" la cantidad siempre es 1 por definición).

---

### User Story 6 - El catálogo existente no cambia de comportamiento (Priority: P2)

Antes de esta funcionalidad, ningún grupo de opciones tenía un modo de selección explícito — todos
se comportaban como "Conteo" porque era la única forma que existía. Al entrar en vigor esta
funcionalidad, todo grupo ya existente queda clasificado automáticamente en modo "Conteo", sin que
ningún administrador tenga que revisar el catálogo ni se altere ningún precio, inventario o pedido
ya confirmado.

**Why this priority**: sin esta historia, la funcionalidad rompería el catálogo existente
(Principio II de la Constitución) — todo grupo de opciones hoy vigente debe seguir funcionando
exactamente igual el día que esta funcionalidad entre en producción.

**Independent Test**: se puede probar revisando, sobre un catálogo existente, que todo grupo quedó
clasificado en modo "Conteo" tras activar esta funcionalidad, y que vender una presentación que lo
usa produce el mismo precio y el mismo consumo que antes.

**Acceptance Scenarios**:

1. **Given** cualquier grupo de opciones existente antes de esta funcionalidad, **When** se activa
   esta funcionalidad, **Then** ese grupo queda clasificado en modo "Conteo", conservando
   exactamente su `min_select`/`max_select` y su comportamiento de venta actuales.
2. **Given** un pedido o venta ya confirmado antes de esta funcionalidad, **When** se consulta
   después de activarla, **Then** su precio, sus opciones y su consumo de inventario ya registrados
   no cambian — el modo de selección de un grupo nunca se aplica retroactivamente a ventas pasadas
   (Principio VII).

## Edge Cases

- **Cambiar un grupo de "Conteo" a "Por cantidad" (o viceversa) mientras está asignado a varias
  presentaciones**: el cambio de modo aplica de inmediato a cualquier selección nueva sobre
  cualquier presentación que use ese grupo — no existe una transición gradual ni una versión
  distinta del grupo por presentación.
- **Grupo "Por cantidad" que maneja inventario, con una opción cuyo insumo no tiene suficiente stock
  para la cantidad pedida**: aplica el mismo chequeo preventivo ya vigente (spec 003, `RN-CAT-24`),
  ahora comparando contra el total requerido ya multiplicado por la cantidad elegida — mismo
  criterio de "advertencia temprana, no reserva" que hoy.
- **Tope por opción o tope total configurado en cantidad 0**: equivale a que esa opción (o el grupo
  completo) no admita ninguna unidad — el administrador puede usarlo para desactivar
  temporalmente un topping sin desactivar la opción en el catálogo.
- **Un grupo "Por cantidad" que además es el único grupo que descuenta inventario de una
  presentación sin receta fija, y el cliente no elige ninguna unidad de ninguna opción**: dado que
  este modo nunca es obligatorio (Clarifications), este caso se trata igual que "grupo opcional sin
  elegir nada" ya documentado como tensión abierta en spec 003 (A-33) — esta spec no la reabre ni la
  resuelve.
- **Migración de un grupo con `min_select > 0` (obligatorio en modo Conteo) hacia "Por cantidad"**:
  al no existir mínimo en modo "Por cantidad" (Clarifications), ese `min_select` deja de aplicar en
  cuanto el administrador cambia el modo — es una consecuencia esperada del cambio de modo, no un
  caso a bloquear.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Todo grupo de opciones DEBE tener un modo de selección explícito: "Conteo" (el
  comportamiento actual, contar opciones distintas elegidas) o "Por cantidad" (contar unidades
  elegidas por opción). "Conteo" DEBE ser el valor por defecto para todo grupo nuevo que no elija
  otro explícitamente, y para todo grupo migrado del catálogo anterior a esta funcionalidad.
- **FR-002**: En un grupo "Por cantidad", el sistema DEBE permitir al cliente elegir, para cada
  opción del grupo de forma independiente, una cantidad entera mayor o igual a cero — nunca un
  simple marcado/desmarcado.
- **FR-003**: Un grupo "Por cantidad" NUNCA DEBE bloquear el avance del pedido por no haber elegido
  ninguna unidad de ninguna opción — no existe un mínimo total configurable para este modo
  (Clarifications, sesión 2026-09-01).
- **FR-004**: El precio de una línea que incluye opciones de un grupo "Por cantidad" DEBE sumar el
  recargo de cada opción elegida multiplicado por su cantidad elegida (`extra_price × cantidad`),
  sobre el mismo cálculo base ya vigente (variante + recargos, spec 002, `RN-CAT-01`).
- **FR-005**: Un grupo "Por cantidad" marcado como "Incluido" (spec 063) DEBE seguir bloqueando el
  precio de todas sus opciones en $0, sin importar la cantidad elegida de cada una — el tipo de
  precio (spec 063) y el modo de selección (esta spec) son ejes independientes.
- **FR-006**: Cuando el producto de una presentación maneja inventario (spec 027) y una de sus
  opciones "Por cantidad" tiene insumo ligado, el consumo de esa opción DEBE multiplicarse por la
  cantidad elegida del cliente, aplicada sobre la cantidad por unidad que ya resulte de la regla
  vigente de "una sola cantidad manda, nunca se suman" (spec 003, `RN-CAT-18`) entre el tamaño y la
  opción.
- **FR-007**: El chequeo preventivo de disponibilidad de stock (spec 003, `RN-CAT-24`) DEBE
  considerar la cantidad total requerida ya multiplicada por la cantidad elegida de cada opción de
  un grupo "Por cantidad", con el mismo criterio de "advertencia temprana, sin reserva" ya vigente.
- **FR-008**: El sistema DEBE permitir configurar, de forma independiente y opcional, un máximo de
  unidades por opción individual y un máximo de unidades totales para el grupo completo, en un
  grupo "Por cantidad". Ambos límites son opcionales; si no se configuran, no aplica ningún tope
  propio del grupo — solo el límite real de stock cuando el grupo maneja inventario (Clarifications,
  sesión 2026-09-01).
- **FR-009**: El sistema NO DEBE permitir que el cliente eleve la cantidad de una opción por encima
  de su tope individual configurado, ni que la suma de cantidades de todas las opciones del grupo
  supere el tope total configurado, cuando esos topes existen.
- **FR-010**: Toda superficie que muestre las opciones elegidas de una línea de pedido (comanda de
  cocina, detalle de orden, "Mis pedidos" del menú QR, recibo impreso) DEBE mostrar, para cada
  opción de un grupo "Por cantidad" con cantidad mayor a cero, cuántas unidades se eligieron —
  nunca repitiendo el nombre de la opción tantas veces como unidades, ni omitiendo la cantidad.
  Una opción de un grupo "Conteo" sigue mostrándose exactamente igual que hoy (un nombre, sin
  cantidad visible).
- **FR-011**: El registro de la venta y del pago de una línea con opciones "Por cantidad" DEBE
  conservar, en el detalle guardado de esa venta, la cantidad elegida de cada opción — de forma que
  un reporte o una consulta posterior pueda reconstruir exactamente qué y cuánto se cobró, igual que
  hoy se conserva el nombre y el recargo de cada opción elegida.
- **FR-012**: El precio, las opciones elegidas (incluida su cantidad) y el consumo de inventario ya
  registrados en un pedido o venta confirmados antes de esta funcionalidad, o con un grupo en modo
  "Conteo", NO DEBEN verse alterados por la existencia del modo "Por cantidad" (Principio VII de la
  Constitución).
- **FR-013**: Cambiar el modo de selección de un grupo ya existente (de "Conteo" a "Por cantidad" o
  viceversa) DEBE aplicar de inmediato a cualquier selección nueva sobre cualquier presentación que
  ofrezca ese grupo, sin afectar pedidos o ventas ya confirmados con el modo anterior.

### Key Entities *(include if feature involves data)*

- **OptionGroup**: grupo de opciones (spec 004), ya con un tipo de precio ("incluido"/"con_recargo",
  spec 063). Gana un atributo nuevo: modo de selección ("conteo"/"cantidad"), y — solo cuando el
  modo es "cantidad" — dos topes opcionales: máximo de unidades por opción individual y máximo de
  unidades totales del grupo. En modo "conteo", `min_select`/`max_select` siguen significando lo
  mismo de hoy (rango de opciones distintas); en modo "cantidad" esos dos campos no aplican.
- **Option**: valor dentro de un grupo (sabor, topping). Sin cambio de atributos propios — su
  `extra_price` (spec 063) y su enlace a insumo (spec 003) se interpretan igual, ahora multiplicados
  por la cantidad que el cliente elija de esa opción cuando su grupo está en modo "cantidad".
- **Selección del cliente sobre un grupo**: en modo "conteo", sigue siendo una lista de opciones
  elegidas (spec 004). En modo "cantidad", pasa a ser una lista de pares (opción, cantidad elegida)
  — cada opción con cantidad mayor a cero cuenta como "elegida" para efectos de precio, consumo y
  visualización; una cantidad de cero equivale a no haberla elegido.
- **Línea de pedido / línea de venta**: conserva, para cada opción elegida de un grupo "cantidad", la
  cantidad con la que se eligió — de la misma forma en que hoy conserva el nombre y el recargo de
  cada opción elegida, para que comanda, "Mis pedidos", recibo y reportes puedan reconstruirla.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los grupos de opciones existentes antes de esta funcionalidad quedan
  clasificados en modo "Conteo" tras activarla, sin ningún cambio de precio, inventario o
  comportamiento de venta.
- **SC-002**: Un cliente puede elegir cantidades distintas de al menos dos toppings distintos en la
  misma línea de producto, y ve el precio total de esa línea reflejando exactamente la suma de cada
  recargo multiplicado por su cantidad, antes de confirmar el pedido.
- **SC-003**: El 100% de las ventas con un grupo "Por cantidad" que maneja inventario descuentan el
  insumo de cada opción elegida multiplicado por su cantidad, verificable comparando el movimiento
  de inventario generado contra la cantidad configurada por opción.
- **SC-004**: El 100% de los intentos de superar un tope por opción o un tope total configurado son
  rechazados por el sistema antes de confirmar el pedido, sin excepción.
- **SC-005**: Un cocinero, un comensal consultando "Mis pedidos" y un cliente leyendo su recibo ven
  la misma cantidad por topping para la misma línea de pedido, sin discrepancias entre esas tres
  superficies.
- **SC-006**: El 100% de los pedidos y ventas confirmados antes de esta funcionalidad, al
  consultarse después, muestran exactamente el mismo precio, las mismas opciones y el mismo consumo
  de inventario que mostraban antes — cero regresiones sobre datos históricos.

## Out of Scope

- Cambiar el cálculo de precio o de consumo para grupos en modo "Conteo" — spec 002/003/004 siguen
  vigentes sin ninguna modificación para ese modo.
- Resolver la tensión abierta de spec 003 (A-33, un grupo opcional que es la única fuente de
  consumo de una presentación sin receta fija) — el modo "Por cantidad" hereda el mismo criterio ya
  documentado como pendiente, sin resolverlo aquí.
- Definir un tercer modo de selección distinto de "Conteo"/"Por cantidad", o permitir que una misma
  opción pertenezca a más de un modo a la vez.
- Cambios al motor de promociones y combos (specs 012/013) más allá de que sigan operando sobre el
  precio de línea ya calculado, sin importar qué modo de selección lo produjo.
- Definir el diseño visual concreto del control de cantidad (`+`/`-`, campo numérico, u otro) ni de
  cómo se presentan los topes al administrador — esta especificación define su comportamiento y
  los criterios de éxito; el diseño concreto se resuelve en la fase de planeación.
- Reportes o filtros nuevos que agrupen o analicen ventas específicamente por cantidad de topping
  pedida — esta spec solo exige que el dato quede conservado (FR-011), no que se construya un
  reporte nuevo sobre él.

## Assumptions

- **El modo de selección es un atributo del grupo completo, no de una opción individual dentro de
  él**: el pedido original distingue casos de uso completos ("elegir N distintas" vs. "elegir
  cantidad de cada una"), no opciones sueltas con comportamientos mezclados dentro del mismo grupo —
  mezclar modos dentro de un mismo grupo añadiría una complejidad de validación que nadie pidió.
- **Los dos topes de cantidad (por opción y total) son independientes y ambos opcionales**: decisión
  explícita de la sesión de Clarifications — cubre tanto al administrador que solo quiere un límite
  simple como al que quiere control fino por topping, sin obligar a configurar ninguno de los dos.
- **Un grupo "Por cantidad" nunca es obligatorio**: decisión explícita de la sesión de
  Clarifications — simplifica la validación de forma (no hay equivalente a "exige exactamente el
  máximo" de spec 004 para este modo) y evita que un comensal quede bloqueado por no querer ningún
  topping.
- **La cantidad elegida por opción multiplica tanto el precio como el consumo de inventario, sobre
  el mismo insumo y la misma cantidad-por-unidad que ya determina spec 003**: no se introduce un
  mecanismo de consumo paralelo — la cantidad del cliente es un multiplicador nuevo sobre una regla
  ya vigente (RN-CAT-18), no un reemplazo de ella.
- **El registro de venta/pago no necesita ningún cálculo nuevo propio**: por diseño ya vigente, la
  capa de venta y pago consume un precio de línea ya calculado y un detalle de opciones ya resuelto,
  sin recalcular nada por su cuenta — esta spec solo exige que ese detalle conserve la cantidad por
  opción (FR-011), sin cambiar cómo se registran ventas o pagos en sí.
- **Las cuatro combinaciones de tipo de precio (spec 063) × modo de selección (esta spec) son todas
  válidas y no requieren casos especiales**: "incluido"+"conteo" (el caso típico de sabores hoy),
  "con_recargo"+"conteo" (topping único con recargo, el caso típico de hoy), "incluido"+"cantidad"
  (extras gratuitos de los que se puede pedir más de una unidad, ej. "bombas de sirope") y
  "con_recargo"+"cantidad" (el caso central de esta spec, toppings con recargo por cantidad).
