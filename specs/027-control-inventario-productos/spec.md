# Feature Specification: Control de Inventario por Producto (Switch de Insumos)

**Feature Branch**: `027-control-inventario-productos`

**Created**: 2026-08-19

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad **nueva** de gestión de catálogo (fase de evolución
funcional, Principio I de la [Constitución](../../.specify/memory/constitution.md)). No reabre las
reglas de cálculo de consumo ya definidas en [spec 003](../003-consumo-de-inventario-por-receta-y-opcion/spec.md)
(receta fija, consumo por opción, chequeo de disponibilidad) — se reutilizan sin cambios para todo
producto que sí maneje inventario. **Sí cambia** un comportamiento ya definido en spec 003
(`RN-CAT-34`, FR-013: toda variante sin receta fija ni grupo de opciones configurado se rechaza al
venderse con `409`, "no tiene receta configurada") — a partir de esta spec, un producto marcado
explícitamente como que **no** maneja inventario queda exento de ese rechazo y puede venderse sin
descontar nada; se autoriza aquí explícitamente como cambio de comportamiento, acotado únicamente a
los productos que declaren no manejar inventario (Principio II). Tampoco reabre la spec 002
(`002-catalogo-productos-variantes-y-precios`, creación de producto/variante y SKU automático) —
esta spec agrega un atributo nuevo al producto y una condición nueva sobre cuándo aplica la
validación de spec 003, sin modificar cómo se crean variantes, precios o SKUs.

**Input**: User description: "necesito implementar una nueva funcionalidad para gestionar los
productos. El usuario debe poder seleccionar si desea que un producto contenga inventario, quiero
que sea mediante un componente switch cuando este activado debe habilitarse la sección de insumos
para asociarlos al producto que descuentan del inventario, caso contrario debe estar inhabilitada
la sección de insumos y el usuario podrá crear un producto sin descontar del inventario, por
defecto el switch debe estar apagado".

## Clarifications

### Session 2026-08-19

- Q: Cuando un administrador guarda un producto con el switch de inventario activado pero sin
  ningún insumo configurado todavía, ¿el formulario debe mostrarle una advertencia visible de
  inmediato, o basta con que el aviso siga apareciendo solo después, al intentar vender? → A:
  Advertencia visible al guardar — el formulario avisa de inmediato que el producto no es vendible
  aún, antes de que un cajero lo descubra frente a un cliente.
- Q: Al desactivar el switch de un producto que ya tiene insumos configurados (que hoy sí descuenta
  inventario), ¿el sistema debe pedir confirmación explícita antes de guardar, o basta con guardarlo
  directamente? → A: Pedir confirmación explícita — evita detener el descuento de inventario de un
  producto por accidente, sin darse cuenta de la consecuencia operativa.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Crear un producto que no maneja inventario, sin ningún insumo (Priority: P1)

Un administrador da de alta un producto que no debe descontar nada del inventario al venderse (por
ejemplo, un domicilio, una propina, un servicio o cualquier ítem que no consuma insumos físicos
controlados). Dado que el switch de "maneja inventario" viene apagado por defecto, el administrador
completa el resto del formulario y guarda el producto sin necesidad de asociar ningún insumo. Ese
producto queda disponible para vender de inmediato, sin generar ningún movimiento de inventario en
cada venta.

**Why this priority**: es el problema central que reportó el usuario — hoy todo producto vendible
necesita, sí o sí, una receta o un grupo de opciones que descuente inventario, o no puede venderse
(spec 003, `RN-CAT-34`). Sin esta historia, un producto legítimamente sin inventario no tiene forma
de crearse y venderse en el sistema.

**Independent Test**: se puede probar creando un producto dejando el switch en su estado por
defecto (apagado), guardándolo sin insumos, y verificando que puede agregarse a una venta y
cobrarse por completo sin ningún rechazo relacionado con inventario y sin que se genere ningún
movimiento de stock.

**Acceptance Scenarios**:

1. **Given** el formulario de creación de un producto nuevo, **When** el administrador lo abre,
   **Then** el switch "maneja inventario" aparece apagado por defecto y la sección de insumos
   aparece deshabilitada, sin exigir ningún dato de receta para continuar.
2. **Given** un producto nuevo con el switch apagado y ningún insumo asociado, **When** el
   administrador lo guarda, **Then** el sistema lo crea con éxito, sin bloquear el guardado por
   falta de insumos.
3. **Given** ese mismo producto ya guardado, **When** se agrega a una venta y se completa el cobro,
   **Then** la venta se completa sin ningún rechazo por "no tiene receta configurada" y sin generar
   ningún movimiento de inventario asociado a esa línea.

---

### User Story 2 - Activar el switch para asociar los insumos que descuenta el producto (Priority: P1)

Un administrador da de alta o edita un producto que sí debe descontar insumos del inventario en
cada venta (por ejemplo, una copa de helado con sabores e insumos físicos). Activa el switch
"maneja inventario", lo que habilita la sección de insumos, y allí asocia la receta fija y/o los
grupos de opciones que descuentan inventario, exactamente como el sistema ya permite hoy (spec
003).

**Why this priority**: es el otro extremo del mismo control — sin poder activarlo, ningún producto
nuevo podría volver a configurarse con descuento de inventario, rompiendo el flujo actual que sí
funciona correctamente hoy.

**Independent Test**: se puede probar activando el switch en el formulario de un producto,
verificando que la sección de insumos se habilita, asociando al menos un insumo o grupo de opciones
que descuente, guardando el producto, y confirmando que una venta posterior descuenta el inventario
exactamente igual que hoy (spec 003).

**Acceptance Scenarios**:

1. **Given** el formulario de un producto con el switch apagado, **When** el administrador lo
   activa, **Then** la sección de insumos pasa de deshabilitada a habilitada, permitiendo agregar
   receta fija y configurar el consumo de los grupos de opciones del producto.
2. **Given** un producto con el switch activado y al menos un insumo o grupo de opciones que
   descuenta asociado, **When** se guarda y luego se vende, **Then** el sistema descuenta el
   inventario exactamente con las mismas reglas ya vigentes (spec 003: receta fija, consumo por
   opción, chequeo de disponibilidad), sin ningún cambio de comportamiento para este caso.
3. **Given** un producto con el switch activado pero sin ningún insumo ni grupo configurado que
   descuente, **When** se intenta vender, **Then** el sistema lo rechaza exactamente igual que hoy
   (`409`, "no tiene receta configurada" — spec 003, `RN-CAT-34`) — activar el switch sin configurar
   insumos no exime de esa validación.
4. **Given** un producto con el switch activado y ninguna de sus presentaciones con receta fija ni
   grupo de opciones que descuente configurado, **When** el administrador guarda el formulario,
   **Then** el sistema muestra de inmediato, en el mismo formulario, una advertencia visible
   indicando que el producto no podrá venderse hasta que se le configure al menos un insumo — sin
   esperar a que un cajero lo descubra al intentar venderlo.

---

### User Story 3 - Cambiar el switch de un producto existente sin perder los insumos ya guardados (Priority: P2)

Un administrador edita un producto que ya tiene insumos asociados y apaga el switch (por ejemplo,
para dejar de descontar inventario de ese producto temporalmente, sin borrar la configuración que ya
tenía). Más adelante, otro administrador vuelve a activar el switch sobre ese mismo producto y
encuentra los insumos exactamente como quedaron, sin tener que volver a capturarlos desde cero.

**Why this priority**: evita pérdida de datos accidental al usar el switch como una simple
preferencia de "activo/inactivo" en vez de un borrado — es una garantía de seguridad de los datos ya
capturados, priorizada después de que ambos extremos del switch (Historias 1 y 2) ya funcionen.

**Independent Test**: se puede probar asociando insumos a un producto con el switch activado,
apagando el switch y guardando, verificando que el producto ya no exige ni aplica esos insumos al
venderse, y luego reactivando el switch para confirmar que los insumos siguen exactamente donde
estaban.

**Acceptance Scenarios**:

1. **Given** un producto con insumos ya asociados y el switch activado, **When** el administrador
   apaga el switch e intenta guardar, **Then** el sistema le pide una confirmación explícita antes
   de guardar, advirtiéndole que ese producto dejará de descontar inventario.
2. **Given** esa advertencia de confirmación, **When** el administrador la acepta, **Then** el
   sistema guarda el cambio: la sección de insumos se deshabilita y esos insumos no se eliminan —
   permanecen guardados aunque no sean visibles ni editables mientras el switch esté apagado.
3. **Given** esa misma advertencia de confirmación, **When** el administrador la cancela, **Then**
   el switch vuelve a quedar activado tal como estaba, sin guardar ningún cambio.
4. **Given** ese mismo producto con el switch ya apagado (confirmado), **When** se vende, **Then**
   no se genera ningún movimiento de inventario, sin importar que ese producto haya tenido insumos
   configurados en el pasado.
5. **Given** ese mismo producto, **When** el administrador vuelve a activar el switch, **Then** la
   sección de insumos se habilita de nuevo mostrando exactamente los insumos que tenía guardados
   antes de apagarlo, sin pedir que se vuelvan a capturar, y sin exigir ninguna confirmación
   adicional para encenderlo (la confirmación solo aplica al apagarlo con insumos configurados).

---

### User Story 4 - Los productos que ya existían en el sistema no pierden ni ganan comportamiento por accidente (Priority: P2)

Antes de esta funcionalidad, todo producto vendible en el sistema necesitaba, por la validación
existente (spec 003), tener al menos un insumo de receta fija o un grupo de opciones que descontara
inventario — de lo contrario no podía venderse. Al entrar en vigor esta funcionalidad, cada producto
que ya existía en el catálogo queda clasificado automáticamente: los que ya tenían insumos
configurados conservan su switch activado y su comportamiento de venta exactamente igual que antes;
los que no tenían ningún insumo configurado (y que, por lo tanto, hoy no se pueden vender) quedan
con el switch apagado, y pasan a poder venderse sin descuento de inventario por primera vez.

**Why this priority**: sin esta historia, la funcionalidad rompería el catálogo existente (Principio
II de la Constitución) o dejaría productos huérfanos en un estado indefinido — es una condición de
seguridad de datos, no una historia de valor nuevo por sí sola, por eso queda después de las tres
anteriores.

**Independent Test**: se puede probar revisando, sobre un catálogo existente, que todo producto con
al menos un insumo o grupo configurado quedó con el switch activado y que una venta suya se comporta
exactamente igual que antes de esta funcionalidad, y que todo producto sin ningún insumo configurado
quedó con el switch apagado y ahora puede venderse sin ser rechazado.

**Acceptance Scenarios**:

1. **Given** un producto existente antes de esta funcionalidad, con receta fija o grupo de opciones
   que ya descontaba inventario, **When** se activa esta funcionalidad, **Then** ese producto queda
   con el switch activado automáticamente y su comportamiento de venta no cambia en absoluto.
2. **Given** un producto existente antes de esta funcionalidad, sin ningún insumo ni grupo
   configurado (y que hoy no puede venderse por `RN-CAT-34`), **When** se activa esta funcionalidad,
   **Then** ese producto queda con el switch apagado automáticamente, y a partir de ese momento
   puede venderse sin descuento de inventario, sin que un administrador tenga que intervenir
   manualmente para destrabarlo.

---

### Edge Cases

- **Producto con switch apagado al que, por error, ya le quedaron insumos asociados de antes**: el
  producto sigue vendiéndose sin descontar inventario mientras el switch esté apagado — el switch,
  no la mera presencia de insumos guardados, es lo que determina si se aplican al vender (Historia
  3).
- **Producto con switch activado recién creado, sin insumos todavía**: el formulario permite
  guardarlo así (la sección de insumos solo queda habilitada para editarse, no exige contenido para
  guardar el producto), pero muestra de inmediato una advertencia visible de que aún no es vendible
  (Historia 2, escenario 4), y queda igual de bloqueado para la venta que hoy (Historia 2, escenario
  3) hasta que se le configure al menos un insumo o grupo que descuente.
- **Producto con varias presentaciones/variantes (tamaños)**: el switch aplica de forma uniforme a
  todas las presentaciones del mismo producto — no existe una presentación que maneje inventario y
  otra del mismo producto que no lo maneje.
- **Apagar y volver a encender el switch varias veces seguidas sin guardar entre medio**: solo el
  estado del switch al momento de guardar el formulario determina si la sección de insumos se
  habilita o deshabilita y si se exige o no la validación de venta — los cambios intermedios sin
  guardar no tienen ningún efecto persistente.
- **Desactivar el switch de un producto que nunca tuvo insumos asociados**: no dispara la
  confirmación de FR-014 — esa advertencia solo aplica cuando hay insumos configurados que
  realmente dejarían de descontarse; apagar el switch de un producto ya vacío de insumos se guarda
  directamente, igual que en Historia 1.
- **Cancelar la confirmación de FR-014**: el formulario deja el switch tal como estaba (activado) y
  no persiste ningún cambio — el administrador puede simplemente intentarlo de nuevo si en realidad
  sí quería apagarlo.
- **Migración de productos con insumos parcialmente configurados** (por ejemplo, un grupo de
  opciones vinculado pero sin ningún insumo con cantidad de consumo mayor a cero): se considera
  "tenía insumos configurados" únicamente si, con los datos ya existentes, ese producto **ya
  superaba** la validación de spec 003 antes de esta funcionalidad (es decir, ya era vendible); si
  no la superaba, migra con el switch apagado como cualquier otro producto sin insumos efectivos.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El formulario de creación y edición de producto DEBE mostrar un control tipo switch
  que indique si el producto maneja inventario, apagado por defecto en todo producto nuevo.
- **FR-002**: Cuando el switch esté activado, el sistema DEBE habilitar la sección de insumos del
  producto (tanto la receta fija como la configuración de consumo de los grupos de opciones) para
  que puedan asociarse o editarse.
- **FR-003**: Cuando el switch esté desactivado, el sistema DEBE deshabilitar la sección de insumos
  del producto — no editable ni exigida para guardar el formulario.
- **FR-004**: El sistema DEBE permitir crear y guardar un producto con el switch desactivado sin
  exigirle ningún insumo de receta fija ni ningún grupo de opciones que descuente inventario.
- **FR-005 [Cambia comportamiento de spec 003, `RN-CAT-34`/FR-013]**: Una venta que incluya un
  producto cuyo switch esté desactivado NO DEBE rechazarse por falta de receta configurada, y NO
  DEBE generar ningún movimiento de inventario para ese producto — queda explícitamente exento de la
  validación que hoy bloquea con `409` a toda variante sin receta ni grupo configurado.
- **FR-006**: Cada presentación de un producto cuyo switch esté activado DEBE seguir sujeta, sin
  ningún cambio, a las reglas ya vigentes de spec 003, evaluadas de forma independiente por
  presentación (no agregadas a nivel de producto): si la presentación vendida no tiene receta fija
  ni grupo de opciones configurado que descuente, esa venta se rechaza igual que hoy (`RN-CAT-34`),
  aunque otra presentación del mismo producto sí tenga insumos configurados; si sí tiene insumos
  configurados, el cálculo de consumo y el chequeo de disponibilidad se aplican exactamente igual
  que hoy.
- **FR-007**: El switch DEBE aplicar de forma uniforme a todas las presentaciones/variantes de un
  mismo producto — no existe una configuración distinta de "maneja inventario" por presentación
  dentro del mismo producto.
- **FR-008**: Al desactivar el switch de un producto que ya tenía insumos asociados, el sistema NO
  DEBE eliminar esos insumos guardados — deja de exigirlos y de mostrarlos como editables mientras
  el switch permanezca apagado, pero la información persiste.
- **FR-009**: Al reactivar el switch de un producto que tenía insumos guardados de antes, el sistema
  DEBE mostrarlos nuevamente tal como quedaron, sin exigir que se vuelvan a capturar.
- **FR-010 [Migración de datos existentes]**: Todo producto que ya existía antes de esta
  funcionalidad y que, con su configuración actual, ya superaba la validación de venta de spec 003
  (es decir, ya tenía al menos un insumo de receta fija o un grupo de opciones que descontaba
  inventario en alguna de sus presentaciones) DEBE quedar migrado con el switch activado,
  preservando exactamente su comportamiento de venta y descuento de inventario previo, sin ninguna
  intervención manual.
- **FR-011 [Migración de datos existentes]**: Todo producto que ya existía antes de esta
  funcionalidad y que no tenía ningún insumo ni grupo configurado que descontara inventario (y que,
  por lo tanto, no podía venderse por `RN-CAT-34`) DEBE quedar migrado con el switch apagado,
  quedando vendible sin descuento de inventario inmediatamente después de esta funcionalidad, sin
  requerir que un administrador lo configure manualmente.
- **FR-012**: Un producto con el switch desactivado DEBE poder agregarse a una venta u orden con el
  mismo flujo que cualquier otro producto (mismo carrito, misma pantalla de armado de venta) —
  ninguna pantalla de venta debe tratarlo como un caso especial visible para el cajero o el
  comensal.
- **FR-013**: Al guardar un producto con el switch activado cuando ninguna de sus presentaciones
  tiene receta fija ni grupo de opciones configurado que descuente inventario, el sistema DEBE
  mostrar de inmediato, en el mismo formulario, una advertencia visible de que el producto no podrá
  venderse hasta que se le configure al menos un insumo — sin esperar a que el rechazo de FR-006 lo
  revele recién en el momento de una venta.
- **FR-014**: Al desactivar el switch de un producto que ya tiene al menos un insumo de receta fija
  o un grupo de opciones que descuenta inventario asociado, el sistema DEBE pedir una confirmación
  explícita antes de guardar ese cambio, advirtiendo que el producto dejará de descontar inventario.
  Si el administrador cancela esa confirmación, el switch DEBE volver a quedar activado sin guardar
  ningún cambio. Esta confirmación NO DEBE exigirse al activar el switch, ni al desactivarlo sobre
  un producto que no tiene ningún insumo asociado.

### Key Entities *(include if feature involves data)*

- **Producto**: gana un atributo nuevo que indica si maneja inventario (booleano, apagado por
  defecto en todo producto nuevo). Este atributo determina, para todas sus presentaciones, si la
  sección de insumos está habilitada en el formulario y si la validación de venta de spec 003
  (`RN-CAT-34`) se le exige o no.
- **Presentación / variante del producto**: sigue siendo la unidad real de venta (spec 002); su
  receta fija y sus grupos de opciones que descuentan inventario (spec 003) solo se exigen y se
  aplican al venderse cuando el producto al que pertenece maneja inventario. Sin cambios en su
  estructura.
- **Insumo de receta / grupo de opciones que descuenta**: sin cambios en su estructura ni en cómo se
  captura (spec 003); lo único nuevo es que su exigencia y su aplicación en el momento de la venta
  ahora dependen del atributo nuevo del producto al que pertenece su presentación.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los productos guardados con el switch apagado pueden agregarse a una venta
  y completar el cobro sin ningún rechazo relacionado con inventario, y sin generar ningún
  movimiento de inventario.
- **SC-002**: El 100% de los productos guardados con el switch apagado se crean con éxito sin exigir
  ningún insumo asociado.
- **SC-003**: El 100% de los productos que ya existían antes de esta funcionalidad y ya tenían
  insumos configurados conservan exactamente su comportamiento de venta y descuento de inventario
  después de esta funcionalidad — cero regresiones.
- **SC-004**: El 100% de los productos que ya existían antes de esta funcionalidad sin ningún insumo
  configurado (y que antes no se podían vender) quedan vendibles inmediatamente después de esta
  funcionalidad, sin que un administrador tenga que configurar nada manualmente.
- **SC-005**: Un administrador identifica, de un vistazo y sin ayuda, si un producto maneja
  inventario o no, con solo mirar el estado del switch en su formulario.
- **SC-006**: El 100% de los insumos ya guardados de un producto siguen intactos después de apagar y
  volver a activar su switch, sin necesidad de volver a capturarlos.
- **SC-007**: El 100% de los productos guardados con el switch activado y sin ningún insumo
  configurado muestran, en el mismo momento de guardar, una advertencia visible de que aún no son
  vendibles — ningún administrador se entera de esto por primera vez a través de una venta fallida.
- **SC-008**: El 100% de los intentos de apagar el switch de un producto con insumos ya configurados
  pasan primero por una confirmación explícita; el 100% de esos productos conserva su switch
  activado y su configuración intacta cuando esa confirmación se cancela.

## Out of Scope

- Cambiar cómo se calcula el consumo de inventario, la receta fija, el consumo por opción o el
  chequeo de disponibilidad de stock cuando el switch está activado — spec 003 sigue vigente sin
  ninguna modificación.
- Configurar el switch de forma independiente por presentación/variante dentro de un mismo
  producto — el switch es exclusivamente a nivel de producto (FR-007).
- Reportes, filtros o indicadores del catálogo que listen o agrupen productos según si manejan
  inventario o no.
- Modificar o recalcular movimientos de inventario históricos ya generados por ventas pasadas.
- Cambios a la validación de grupos de opciones (spec 004) o a la conversión de unidades de compra
  (spec 005) — se reutilizan tal como están.
- Definir el diseño visual concreto del componente switch (color, tamaño, ubicación exacta en el
  formulario) — esta especificación define su comportamiento y los criterios de éxito que cualquier
  diseño debe cumplir; el diseño concreto se resuelve en la fase de planeación.

## Assumptions

- **El switch es un atributo de producto, no de presentación/variante**: el pedido original habla de
  "un producto" sin distinguir por presentación, y separar esta configuración por cada tamaño
  añadiría una complejidad (y una superficie de validación cruzada) que nadie pidió — si el negocio
  necesita esa granularidad más adelante, es una funcionalidad independiente (FR-007).
- **Apagar el switch nunca borra insumos ya guardados**: se prioriza no perder información capturada
  por accidente al usar el switch como preferencia de "activo/inactivo"; el usuario no pidió un
  comportamiento destructivo y ninguna otra spec de este proyecto borra datos de forma implícita al
  cambiar una bandera.
- **La migración de productos existentes es automática y determinista**, basada exclusivamente en si
  el producto ya superaba o no la validación de venta de spec 003 antes de esta funcionalidad — evita
  tanto romper productos que hoy sí descuentan inventario correctamente (Principio II) como dejar
  productos existentes en un estado indefinido que ningún administrador sepa que debe revisar.
- **"La sección de insumos" incluye tanto la receta fija como la configuración de consumo de los
  grupos de opciones** de las presentaciones del producto — ambos mecanismos de descuento de
  inventario ya documentados en spec 003 quedan bajo el mismo switch, porque el pedido original habla
  de "insumos que descuentan del inventario" en general, sin distinguir entre ambos mecanismos.
- **Un producto con el switch activado sigue pudiendo quedar bloqueado para la venta si no se le
  configura ningún insumo** (FR-006): activar el switch habilita la sección, no sustituye la
  necesidad de configurarla — esta spec no relaja esa validación existente para el caso en que el
  switch está encendido.
- **El componente switch se construye siguiendo el estilo visual ya usado en el resto del formulario
  de producto**: el proyecto no cuenta hoy con un componente de switch/toggle reutilizable (no usa
  Angular Material ni tiene uno propio); su apariencia final concreta queda para la fase de
  planeación, sin que eso cambie el comportamiento aquí especificado.
