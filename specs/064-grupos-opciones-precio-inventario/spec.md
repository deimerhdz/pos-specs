# Feature Specification: Tipo de precio e inventario condicional en grupos de opciones

**Feature Branch**: `064-grupos-opciones-precio-inventario`

**Created**: 2026-09-01

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad **nueva** de catálogo (fase de evolución funcional,
Principio I de la [Constitución](../../.specify/memory/constitution.md)). No reabre las reglas de
consumo de inventario ya definidas en [spec 003](../003-consumo-de-inventario-por-receta-y-opcion/spec.md)
(receta fija, `quantity_per_option` manda sobre `item_quantity`, chequeo de disponibilidad) —
se reutilizan sin cambios para toda opción que sí descuente inventario. **Sí cambia** un
comportamiento ya documentado en spec 004 (anomalía A-32, `RN-CAT-39`): unifica en un único criterio
explícito las dos definiciones distintas que hoy existen para "¿esta opción descuenta inventario?".
También **extiende** spec 027 (Control de Inventario por Producto) para que su switch "maneja
inventario" alcance de forma consistente a los grupos de opciones de sus presentaciones, y extiende
la regla de acceso por plan ya establecida en spec 033/062 para que el módulo Inventario gobierne
también estos campos.

**Input**: User description: "necesito mejorar el modelo actual de grupos de opciones, que permite
configurar varios casos de usos, el primero es para definir por ejemplo en un producto helados un
usuario puede definir varios sabores de helado para que el cliente seleccione desde el menu qr los
sabores y se descuente del inventario, y tambien creo que puede definir toppings pero actualmente
creo que hay una reglas de condicion de carrera que le agregan complejidad, porque un sabor de
helado, no se le define precio porque ya viene incluido en el producto pero el topping si lleva
precios, entonces quiero mejorar esa parte y lo otro es que como ahora un producto no necesaria
mente tiene que manejar inventario entonces el topping/ sabor tampoco deberia manejar inventario y
todo debe coincidir con las reglas de acceso en el plan."

## Clarifications

### Session 2026-09-01

- Q: ¿Cómo debe distinguirse un grupo de opciones tipo "sabor incluido" (sin precio propio) de uno
  tipo "topping con recargo"? → A: Campo explícito por grupo. El grupo de opciones gana un campo
  tipo "Incluido" / "Con recargo" que el administrador fija una sola vez; si es "Incluido", el
  formulario bloquea el precio de sus opciones en 0 — deja de depender de que el administrador
  recuerde dejarlo en 0 a mano.
- Q: ¿Qué debe pasar con los campos de inventario (insumo ligado, cantidad por opción/tamaño) de un
  grupo de opciones cuando el producto al que pertenece tiene el switch "maneja inventario"
  apagado? → A: Ocultar/deshabilitar esos campos. Si el producto no maneja inventario, el
  formulario de sus grupos de opciones oculta el enlace a insumo y las cantidades de consumo — solo
  queda visible el precio. Mismo criterio que ya aplica hoy a la receta fija (spec 027).
- Q: ¿Se corrige, como parte de esta mejora, la inconsistencia ya documentada (anomalía A-32) entre
  los dos criterios distintos que hoy responden "¿este grupo descuenta inventario?"? → A: Sí,
  unificar el criterio. Se define una sola regla explícita para "esta opción consume inventario" y
  se usa en ambos puntos del sistema (validación de selección y confirmación de venta), eliminando
  la discrepancia de raíz.
- Q: ¿Debe el switch "maneja inventario" de un producto (y los campos de inventario de sus
  opciones) quedar bloqueado por completo si el plan del tenant no incluye el módulo Inventario? →
  A: Sí, mismo criterio que spec 062. Sin el módulo Inventario en el plan, el switch del producto y
  los campos de insumo/cantidad de sus opciones se ocultan o deshabilitan igual que Unidades de
  Medida y Margen — un tenant sin ese módulo solo puede usar grupos de opciones para precio/menú,
  nunca para descontar stock.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Un grupo de sabores incluido nunca puede llevar recargo (Priority: P1)

Un administrador crea el grupo "Sabores" para una copa de helado y lo marca como **Incluido** (el
precio ya está cubierto por el precio de la presentación). Al agregar sus opciones ("Fresa",
"Chocolate", "Vainilla"), el formulario no ofrece ningún campo de precio editable — todas quedan en
$0 automáticamente, sin que el administrador tenga que recordarlo ni pueda equivocarse.

**Why this priority**: es el problema central que reportó el usuario — hoy "sabor sin precio" es
solo una convención (dejar `extra_price` en 0 a mano); nada impide que alguien le ponga precio a un
sabor por error, mezclando sin querer el caso de uso de sabor incluido con el de topping con
recargo.

**Independent Test**: se puede probar creando un grupo de opciones nuevo, marcándolo como
"Incluido", agregando una opción, y verificando que no hay forma de asignarle un precio distinto de
cero desde el formulario.

**Acceptance Scenarios**:

1. **Given** el formulario de creación de un grupo de opciones, **When** el administrador lo marca
   como "Incluido", **Then** el campo de precio de cada opción que agregue a ese grupo queda fijo
   en $0, sin control editable para cambiarlo.
2. **Given** un grupo "Sabores" ya marcado como "Incluido" con varias opciones, **When** se vende
   una presentación con un sabor de ese grupo elegido, **Then** el precio de la línea es
   exactamente el precio de la presentación — ningún sabor de un grupo "Incluido" puede sumar
   recargo (coherente con `RN-CAT-01`, spec 002).
3. **Given** un grupo "Incluido" existente, **When** un administrador intenta cambiarlo a "Con
   recargo", **Then** el sistema se lo permite, y a partir de ese momento el precio de sus opciones
   se vuelve editable (Historia 2) — la reclasificación no borra las opciones existentes, solo
   habilita su precio.

---

### User Story 2 - Un grupo de toppings con recargo exige y muestra precio por opción (Priority: P1)

Un administrador crea el grupo "Toppings" y lo marca como **Con recargo**. Al agregar sus opciones
("Chocolate en trozos", "Maní"), el formulario sí ofrece el campo de precio, tal como funciona hoy,
para que cada topping pueda cobrar un adicional distinto.

**Why this priority**: es la otra mitad del mismo problema — sin esta distinción explícita, no hay
forma de garantizar que un grupo de toppings efectivamente cobre lo que el administrador espera sin
revisar opción por opción.

**Independent Test**: se puede probar creando un grupo marcado como "Con recargo", agregando una
opción con un precio mayor a cero, y verificando que el precio de línea de una venta con esa opción
elegida lo suma correctamente (igual que hoy, spec 002, `RN-CAT-01`).

**Acceptance Scenarios**:

1. **Given** el formulario de creación de un grupo de opciones, **When** el administrador lo marca
   como "Con recargo", **Then** cada opción que agregue a ese grupo permite capturar un precio
   mayor o igual a cero, editable libremente (igual que el comportamiento actual, `RN-CAT-04`).
2. **Given** un grupo "Toppings" (Con recargo) con la opción "Maní" en $500, **When** se vende una
   presentación con "Maní" elegido, **Then** el precio de línea suma ese recargo, sin cambios
   respecto al cálculo ya vigente (`RN-CAT-01`).
3. **Given** un grupo "Con recargo" con al menos una opción cuyo precio hoy es mayor a $0, **When**
   un administrador intenta cambiarlo a "Incluido", **Then** el sistema le advierte que todas las
   opciones del grupo quedarán en $0 y pide confirmación explícita antes de aplicar el cambio —
   evita que un administrador borre por accidente un recargo ya configurado.

---

### User Story 3 - Sin "maneja inventario" en el producto, sus grupos de opciones tampoco descuentan stock (Priority: P1)

Un administrador edita un producto cuyo switch "maneja inventario" (spec 027) está apagado — por
ejemplo, un helado de paleta simple que no controla insumos por unidad. Al configurar los grupos de
opciones de sus presentaciones (sabores o toppings), el formulario no ofrece ningún campo para
ligar un insumo ni definir cantidad de consumo — solo puede configurar nombre y precio de cada
opción.

**Why this priority**: es el segundo problema central reportado — hoy nada impide configurar
insumo y cantidad de consumo en las opciones de un producto que ya declaró explícitamente que no
maneja inventario (spec 027), dejando datos fantasma que nunca se aplican y que confunden al
administrador sobre si ese producto realmente descuenta algo o no.

**Independent Test**: se puede probar apagando el switch "maneja inventario" de un producto,
abriendo el formulario de configuración de sus grupos de opciones, y verificando que los campos de
insumo y cantidad de consumo no están disponibles, mientras el precio de cada opción sigue siendo
editable con normalidad.

**Acceptance Scenarios**:

1. **Given** un producto con el switch "maneja inventario" apagado, **When** el administrador
   configura los grupos de opciones de sus presentaciones, **Then** el formulario oculta o
   deshabilita el enlace a insumo y la cantidad de consumo por opción y por tamaño, dejando
   únicamente editable el nombre y el precio de cada opción.
2. **Given** ese mismo producto, **When** se vende una presentación con una opción elegida,
   **Then** no se genera ningún movimiento de inventario asociado a esa opción, sin importar si esa
   opción tenía datos de insumo configurados de antes (mismo criterio de no aplicar datos
   "fantasma" que ya rige la receta fija, spec 027 FR-005/FR-006).
3. **Given** un producto con el switch "maneja inventario" apagado cuyas opciones ya tenían insumo
   y cantidad configurados de una época anterior a esta funcionalidad, **When** se activa esta
   funcionalidad, **Then** esos datos no se borran — quedan guardados pero ocultos/no editables
   mientras el switch del producto siga apagado (mismo criterio de no pérdida de datos que spec
   027, FR-008).
4. **Given** ese mismo producto, **When** el administrador enciende el switch "maneja inventario",
   **Then** los campos de insumo y cantidad de consumo de sus grupos de opciones vuelven a estar
   disponibles para editar, mostrando los valores que tenían guardados antes de ocultarse, sin
   exigir que se vuelvan a capturar.

---

### User Story 4 - Un solo criterio decide si una opción descuenta inventario (Priority: P2) — corrige anomalía A-32

Un administrador de catálogo configura una opción con cantidad de consumo puesta pero sin insumo
enlazado (catálogo a medio terminar). El sistema, en cualquier punto donde necesite responder "¿esta
opción descuenta inventario?" (al validar la selección del cliente o al confirmar la venta), usa
siempre el mismo criterio y llega siempre a la misma respuesta.

**Why this priority**: spec 004 (`RN-CAT-39`, anomalía A-32) documentó que hoy existen dos
funciones que responden esa pregunta de forma distinta, lo que puede mostrarle al cajero el mensaje
de error equivocado ("no tiene receta configurada" cuando el problema real es un insumo sin
enlazar). Resolver esto de raíz es necesario para que la separación de precio/inventario de las
Historias 1 a 3 sea confiable: un administrador debe poder confiar en que, si configuró una opción
como "no descuenta inventario", ningún otro punto del sistema la trate como si sí lo hiciera.

**Independent Test**: se puede probar configurando una opción con cantidad de consumo puesta pero
sin insumo enlazado (o con insumo desactivado), y verificando que tanto la validación de selección
al armar la venta como la confirmación al cobrar coinciden en que esa opción no descuenta
inventario.

**Acceptance Scenarios**:

1. **Given** una opción con cantidad de consumo configurada pero sin insumo de inventario enlazado
   (o con su insumo desactivado), **When** el sistema evalúa, en cualquier punto del flujo de
   venta, si esa opción descuenta inventario, **Then** la respuesta es "no" de forma consistente en
   todos los puntos — la validación de selección y la confirmación de venta nunca discrepan.
2. **Given** una opción con insumo enlazado, activo y cantidad de consumo mayor a cero, **When** el
   sistema evalúa si descuenta inventario en cualquier punto del flujo, **Then** la respuesta es
   "sí" de forma consistente en todos los puntos.
3. **Given** el mensaje de rechazo que recibe un cajero cuando una variante no descontaría nada al
   venderse (spec 003, `RN-CAT-34`), **When** la causa real es una opción con insumo sin enlazar
   (no una variante sin ningún grupo configurado), **Then** el mensaje refleja la causa correcta,
   distinguible del caso de "sin receta ni grupo configurado en absoluto".

---

### User Story 5 - Sin el módulo Inventario en el plan, ningún producto puede activar "maneja inventario" (Priority: P2)

Un Tenant Admin cuyo plan no incluye el módulo Inventario abre el formulario de un producto. El
switch "maneja inventario" (spec 027) y los campos de insumo/cantidad de sus grupos de opciones no
están disponibles — igual que el resto de superficies de Inventario ya bloqueadas por plan (spec
033/062) — porque activar ese switch sin acceso al módulo dejaría al administrador con un control
que no puede usar de verdad (no puede dar de alta insumos, no puede ver el stock).

**Why this priority**: cierra el pedido explícito del usuario de que "todo debe coincidir con las
reglas de acceso en el plan" — sin esta historia, un tenant sin Inventario podría activar el switch
y configurar campos de insumo que nunca podrán completarse de forma útil (la pantalla de insumos ya
está bloqueada por spec 062), dejando el formulario de producto en un estado confuso e
inconsistente con el resto del sistema.

**Independent Test**: con un tenant en un plan sin acceso a Inventario, se abre el formulario de un
producto nuevo o existente y se verifica que el switch "maneja inventario" no está disponible (o
aparece deshabilitado con explicación), y que los grupos de opciones de ese producto solo permiten
configurar nombre y precio, nunca insumo ni cantidad.

**Acceptance Scenarios**:

1. **Given** el plan vigente de un tenant no incluye el módulo Inventario (o está vencido), **When**
   el Tenant Admin abre el formulario de un producto, **Then** el switch "maneja inventario" no
   está disponible para activarse — mismo criterio de ocultar/deshabilitar ya usado para Unidades
   de Medida y Margen (spec 062).
2. **Given** ese mismo tenant, **When** configura los grupos de opciones de un producto, **Then**
   el formulario nunca ofrece campos de insumo ni cantidad de consumo en ninguna opción, sin
   importar el estado guardado del switch "maneja inventario" del producto.
3. **Given** un producto que ya tenía el switch "maneja inventario" activado y opciones con insumo
   configurado **antes** de que al tenant se le retirara el acceso a Inventario, **When** se le
   retira el acceso, **Then** esos datos no se borran — el switch y los campos de insumo quedan
   ocultos/no editables, y el producto se comporta como si no manejara inventario (sin bloquear
   ventas por falta de receta) mientras dure la restricción de plan, sin afectar productos de otros
   tenants (mismo criterio de no pérdida de datos que spec 033/062).
4. **Given** ese mismo tenant, **When** el Super Admin le reasigna el acceso al módulo Inventario,
   **Then** el switch "maneja inventario" y los campos de insumo de sus productos y opciones
   vuelven a estar disponibles de inmediato, mostrando exactamente los valores que tenían guardados
   antes de la restricción, sin intervención manual adicional (mismo criterio de spec 033 FR-014 y
   spec 062 FR-008).

---

### User Story 6 - Los grupos de opciones existentes se reclasifican sin cambiar su comportamiento de venta (Priority: P2)

Antes de esta funcionalidad, ningún grupo de opciones tenía un tipo explícito de "Incluido" o "Con
recargo" — el precio de cada opción era simplemente un número libre. Al entrar en vigor esta
funcionalidad, cada grupo ya existente se clasifica automáticamente según su configuración actual,
sin que ningún administrador tenga que revisar el catálogo completo a mano ni se altere ningún
precio ya vigente.

**Why this priority**: sin esta historia, la funcionalidad rompería el catálogo existente
(Principio II de la Constitución) — un grupo "Toppings" con precios ya configurados no puede
aparecer de golpe como "Incluido" y perder esos precios, ni un grupo "Sabores" en $0 debería
quedar bloqueado de tener recargo si el negocio decide cobrarlo en el futuro.

**Independent Test**: se puede probar revisando, sobre un catálogo existente, que todo grupo con al
menos una opción con precio mayor a $0 quedó clasificado como "Con recargo" conservando esos
precios exactamente igual, y que todo grupo con todas sus opciones en $0 quedó clasificado como
"Incluido".

**Acceptance Scenarios**:

1. **Given** un grupo de opciones existente con al menos una opción con precio mayor a $0, **When**
   se activa esta funcionalidad, **Then** ese grupo queda clasificado como "Con recargo",
   conservando los precios existentes sin ningún cambio.
2. **Given** un grupo de opciones existente donde todas sus opciones están en $0, **When** se
   activa esta funcionalidad, **Then** ese grupo queda clasificado como "Incluido", sin ningún
   cambio visible para quien lo vende — sigue sumando $0 a la línea, igual que antes.
3. **Given** cualquier grupo reclasificado automáticamente, **When** se vende una presentación que
   lo usa, **Then** el precio y el consumo de inventario resultantes son idénticos a los que
   producía antes de esta funcionalidad — la reclasificación es puramente de datos, no cambia
   ningún cálculo ya vigente.

---

### Edge Cases

- **Grupo "Incluido" compartido entre un producto que maneja inventario y otro que no** (spec 004,
  `RN-CAT-36`, un grupo puede asignarse a varias presentaciones): el enlace de insumo y la cantidad
  de consumo de una opción son datos propios de esa opción, no del producto — al editar el grupo
  desde un producto sin inventario esos campos se ocultan para esa edición, pero no se borran ni
  afectan la configuración que otro producto con inventario activo siga usando sobre la misma
  opción.
- **Cambiar un grupo de "Con recargo" a "Incluido" cuando alguna opción tiene precio mayor a $0**:
  requiere la confirmación explícita de la Historia 2, escenario 3 — sin confirmación, el cambio de
  tipo no se aplica y los precios existentes permanecen intactos.
- **Retirar el acceso a Inventario a un tenant con productos que ya manejaban inventario**: no se
  fuerza el switch "maneja inventario" a apagado en los datos — el valor guardado se preserva, pero
  mientras dure la restricción de plan el producto se comporta como si no manejara inventario
  (Historia 5, escenario 3), igual que un producto con el switch realmente apagado.
- **Un grupo "Incluido" con una opción que antes de esta funcionalidad ya tenía insumo y cantidad
  de consumo configurados** (caso legítimo hoy: un sabor puede descontar inventario sin cobrar
  recargo): "Incluido" solo bloquea el precio, no el consumo de inventario — ambos ejes (precio e
  inventario) son independientes entre sí; un grupo puede ser "Incluido" y seguir descontando stock
  si el producto al que pertenece maneja inventario.
- **Tenant sin módulo Inventario que ya tenía grupos "Incluido" y "Con recargo" configurados solo
  por precio, sin ningún insumo enlazado**: no se ve afectado por la Historia 5 — esos grupos
  siguen funcionando con normalidad porque nunca dependieron de campos de insumo.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Todo grupo de opciones DEBE tener un tipo explícito, "Incluido" o "Con recargo",
  elegido por el administrador al crearlo.
- **FR-002**: Cuando un grupo de opciones es de tipo "Incluido", el sistema NO DEBE permitir
  capturar un precio distinto de $0 en ninguna de sus opciones — el campo de precio queda fijo en
  $0, sin control editable.
- **FR-003**: Cuando un grupo de opciones es de tipo "Con recargo", el sistema DEBE permitir
  capturar un precio mayor o igual a $0 en cada una de sus opciones, con el mismo comportamiento de
  validación ya vigente (`RN-CAT-04`, spec 002).
- **FR-004**: El sistema DEBE permitir reclasificar un grupo existente entre "Incluido" y "Con
  recargo" en cualquier momento. Si el grupo tiene al menos una opción con precio mayor a $0 y se
  reclasifica a "Incluido", el sistema DEBE pedir confirmación explícita antes de aplicar el cambio,
  advirtiendo que todos los precios de sus opciones quedarán en $0; si el administrador cancela, el
  tipo y los precios permanecen sin cambios.
- **FR-005**: El tipo de un grupo de opciones ("Incluido"/"Con recargo") es independiente de si sus
  opciones descuentan inventario — un grupo "Incluido" puede seguir teniendo opciones que consumen
  insumo, y un grupo "Con recargo" puede tener opciones que no consumen ningún insumo.
- **FR-006 [Extiende spec 027]**: Cuando el switch "maneja inventario" de un producto está
  apagado, el sistema DEBE ocultar o deshabilitar, en el formulario de configuración de los grupos
  de opciones de sus presentaciones, el enlace a insumo de inventario y la cantidad de consumo (por
  opción y por tamaño) — dejando editable únicamente el nombre y el precio de cada opción.
- **FR-007**: Cuando el switch "maneja inventario" de un producto está apagado, ninguna venta de
  sus presentaciones DEBE generar movimiento de inventario a partir de las opciones elegidas, sin
  importar si esas opciones tienen datos de insumo/cantidad guardados de una configuración anterior
  (mismo criterio no destructivo de spec 027, FR-008).
- **FR-008**: Al encender el switch "maneja inventario" de un producto que ya tenía insumo y
  cantidad de consumo guardados en sus opciones desde antes de apagarlo, el sistema DEBE mostrarlos
  de nuevo tal como quedaron, sin exigir que se vuelvan a capturar.
- **FR-009 [Corrige anomalía A-32, `RN-CAT-39`, spec 004]**: El sistema DEBE usar un único criterio
  explícito para determinar si una opción descuenta inventario (insumo de inventario enlazado, ese
  insumo activo, y cantidad de consumo resultante mayor a $0), y DEBE aplicar ese mismo criterio en
  todos los puntos donde esa pregunta se responde — tanto al validar la selección del cliente como
  al confirmar/cobrar la venta. Los dos criterios distintos documentados en `RN-CAT-39` quedan
  reemplazados por este único criterio.
- **FR-010**: El mensaje de rechazo que recibe un cajero cuando una línea de venta no descontaría
  ningún inventario (spec 003, `RN-CAT-34`) DEBE seguir distinguiendo si la causa es "sin receta ni
  grupo configurado en absoluto" o "grupo configurado pero sin insumo enlazado o sin opción
  elegida", usando el criterio unificado de FR-009 para esa distinción.
- **FR-011 [Gating por plan, extiende spec 033/062]**: Cuando el plan vigente de un tenant no
  incluye el módulo Inventario, o el tenant está vencido, el sistema DEBE ocultar o deshabilitar el
  switch "maneja inventario" de todo producto de ese tenant, y DEBE ocultar el enlace a insumo y la
  cantidad de consumo en todos sus grupos de opciones — mismo criterio de ocultamiento total que
  spec 062 (FR-001, FR-005, FR-007).
- **FR-012**: El sistema DEBE denegar cualquier intento de activar el switch "maneja inventario" o
  de guardar un enlace a insumo/cantidad de consumo en una opción, por cualquier vía (no solo
  ocultar el control en pantalla), cuando el plan del tenant no incluye el módulo Inventario o está
  vencido — mismo criterio de denegación más allá de la pantalla que spec 062 (FR-003, FR-006).
- **FR-013 [No pérdida de datos]**: Cuando a un tenant se le retira el acceso al módulo Inventario
  teniendo productos con el switch "maneja inventario" ya activado y opciones con insumo
  configurado, el sistema NO DEBE borrar ni modificar esos datos — el switch y los campos de insumo
  quedan ocultos/no editables y el producto se comporta como si no manejara inventario mientras dure
  la restricción, sin bloquear sus ventas por falta de receta.
- **FR-014**: Cuando el Super Admin reasigna a un tenant el acceso al módulo Inventario (o renueva
  su plan vencido), el sistema DEBE restaurar de inmediato, sin intervención adicional del Tenant
  Admin, el switch "maneja inventario" de sus productos y los campos de insumo/cantidad de sus
  opciones, mostrando exactamente los valores que tenían guardados antes de la restricción (mismo
  criterio de aplicación inmediata que spec 033 FR-014 y spec 062 FR-008).
- **FR-015 [Migración de datos existentes]**: Todo grupo de opciones que ya existía antes de esta
  funcionalidad, con al menos una opción con precio mayor a $0, DEBE quedar clasificado
  automáticamente como "Con recargo", conservando exactamente los precios ya configurados. Todo
  grupo cuyas opciones estén todas en $0 DEBE quedar clasificado automáticamente como "Incluido".
  Ninguna reclasificación automática DEBE alterar el precio, el insumo, la cantidad de consumo ni
  el comportamiento de venta que el grupo ya tenía.

### Key Entities *(include if feature involves data)*

- **OptionGroup**: grupo de opciones (spec 004). Gana un atributo nuevo: tipo de precio
  ("Incluido" / "Con recargo"), que determina si el precio de sus opciones puede ser distinto de
  $0. Sin cambios en `min_select`/`max_select` ni en su vínculo con variantes (`RN-CAT-36`).
- **Option**: valor dentro de un grupo (spec 002/004). Su `extra_price` queda gobernado por el tipo
  del grupo al que pertenece (FR-002/FR-003); su enlace a insumo de inventario y cantidad de
  consumo (spec 003) quedan sujetos, además, al switch "maneja inventario" del producto (FR-006) y
  al acceso por plan al módulo Inventario (FR-011).
- **Producto**: su switch "maneja inventario" (spec 027) extiende su alcance para gobernar también
  la disponibilidad de los campos de insumo/cantidad en los grupos de opciones de sus
  presentaciones (FR-006), y queda además sujeto al acceso por plan al módulo Inventario (FR-011).
- **Plan de suscripción / acceso a módulo Inventario**: (spec 033) determina, además de las
  superficies ya cubiertas por spec 062, si el switch "maneja inventario" de un producto y los
  campos de insumo/cantidad de sus opciones están disponibles para ese tenant (FR-011 a FR-014).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las opciones de un grupo "Incluido" tienen precio $0, sin ninguna
  excepción capturable desde el formulario.
- **SC-002**: El 100% de los grupos de opciones existentes antes de esta funcionalidad quedan
  reclasificados automáticamente ("Incluido" o "Con recargo") sin alterar ningún precio, insumo,
  cantidad de consumo ni resultado de venta ya vigente.
- **SC-003**: El 100% de los productos con el switch "maneja inventario" apagado ocultan los campos
  de insumo y cantidad de consumo en todos sus grupos de opciones, sin perder esos datos si ya
  existían.
- **SC-004**: Cero discrepancias entre la validación de selección y la confirmación de venta sobre
  si una opción descuenta inventario — el mismo criterio (FR-009) responde igual en ambos puntos
  para el 100% de las opciones evaluadas.
- **SC-005**: El 100% de los tenants sin el módulo Inventario en su plan (o con el plan vencido) no
  pueden activar el switch "maneja inventario" de ningún producto ni guardar un enlace de insumo o
  cantidad de consumo en ninguna opción, por ninguna vía.
- **SC-006**: El 100% de los tenants que recuperan el acceso al módulo Inventario ven restaurado, en
  el siguiente intento de acceso, el switch y los campos de insumo de sus productos y opciones
  exactamente como quedaron guardados, sin intervención manual.

## Out of Scope

- Cambiar cómo se calcula el consumo de inventario por receta fija, por opción, o el chequeo de
  disponibilidad de stock — spec 003 sigue vigente sin ninguna modificación salvo la unificación de
  criterio de FR-009.
- Cambiar el rango de selección (`min_select`/`max_select`), la tolerancia de migración
  (`STRICT_OPTION_SELECTION`) o cualquier otra regla de validación de forma de spec 004 distinta a
  la corregida en FR-009.
- Configurar el switch "maneja inventario" o el tipo de grupo de opciones de forma independiente
  por presentación/variante dentro de un mismo producto — ambos siguen siendo atributos a nivel de
  producto y de grupo respectivamente, no de variante.
- Definir nuevos límites numéricos de plan para grupos u opciones de catálogo — esta spec solo
  extiende el acceso on/off al módulo Inventario ya existente (spec 033), no agrega límites nuevos.
- Resolver la anomalía A-06 (opción de un grupo ajeno a la variante que se cuela con la tolerancia
  de `STRICT_OPTION_SELECTION=False`) — sigue con el tratamiento de riesgo aceptado ya documentado
  en spec 004, sin relación con el tipo de precio ni con el switch de inventario de esta spec.
- Definir el diseño visual concreto del control de tipo de grupo ("Incluido"/"Con recargo") ni de
  los mensajes de campos deshabilitados — esta especificación define su comportamiento y los
  criterios de éxito; el diseño concreto se resuelve en la fase de planeación.

## Assumptions

- **El tipo de grupo ("Incluido"/"Con recargo") es un atributo del grupo de opciones, no de la
  opción individual**: el pedido original distingue casos de uso completos ("sabores" vs
  "toppings"), no opciones sueltas dentro de un mismo grupo mixto — permitir tipos distintos dentro
  del mismo grupo añadiría una complejidad que nadie pidió y contradice la idea de que un grupo
  representa un caso de uso coherente.
- **El enlace a insumo y la cantidad de consumo de una opción son datos propios de la opción, no
  del producto**: dado que un mismo grupo de opciones puede compartirse entre varias presentaciones
  y productos (`RN-CAT-36`), ocultar esos campos "por producto sin inventario" es un
  comportamiento de la pantalla de edición en ese contexto, no una eliminación ni un bloqueo de los
  datos para otros productos que sí compartan esa opción y sí manejen inventario.
- **Retirar el acceso a Inventario nunca fuerza el switch "maneja inventario" a apagado en los
  datos guardados**: se prioriza no perder la configuración ya capturada, igual que el criterio ya
  usado en spec 027 (apagar el switch no borra insumos) y en spec 033/062 (retirar un módulo no
  borra datos) — el switch guardado se preserva y solo se ignora mientras dure la restricción de
  plan.
- **El criterio unificado de FR-009 ("insumo enlazado, activo, y cantidad resultante mayor a $0")
  se elige por ser el más estricto de los dos que hoy compiten** (`RN-CAT-39`): evita que una opción
  con un insumo desactivado o sin enlazar se trate como si sí descontara inventario, que es
  precisamente el caso que hoy puede confundir al administrador con un mensaje de error equivocado.
- **La reclasificación automática de grupos existentes (Historia 6) usa únicamente el precio de las
  opciones como criterio** ("con recargo" si alguna opción tiene precio mayor a $0, "incluido" si
  todas están en $0), sin considerar si el grupo descuenta inventario — el tipo de precio y el
  consumo de inventario son ejes independientes (FR-005), así que la migración no necesita cruzar
  ambos datos para decidir el tipo inicial.
