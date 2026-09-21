# Feature Specification: Corrección de Bugs en Promociones y Productos

**Feature Branch**: `084-fix-promociones-productos`

**Created**: 2026-09-17

**Status**: Draft

**Input**: User description: "necesito que hagas una revision para resolver los siguientes bugs
encontrados en el modulo de promociones y productos: 1. Ajustar card del producto en la sección
de promociones en el menú QR, debe mostrar el precio mínimo de la promoción. 2. Hay un catálogo
de presentación, pero ésta no está relacionada con las variantes de los productos en el
formulario de crear producto y editar producto. 3. Validar que promoción aplique el descuento a
las presentaciones relacionadas con las variantes del producto, es decir, al definir una regla
(por ejemplo presentación única x 2 unidades $15.000), debe poder aplicar el descuento a todos los
productos seleccionados que hagan parte de la promoción. 4. La promoción no se puede configurar
cuando esté vigente; actualmente se permite configurar cuando está vigente desde el listado de
promociones. 5. Arreglar el problema del modal de acciones para evitar el scroll en la tabla."

**Enmienda 2026-09-20** (sobre una spec ya implementada, 42/42 tareas): tras probar el formulario
de producto con la asociación variante↔presentación en pantalla, el propietario pidió dos ajustes
al formulario de crear y editar producto — (1) quitar el campo "Nombre" de cada tamaño, porque la
presentación ya lo trae, y eliminarlo también del modelo de base de datos (User Story 7,
FR-029 a FR-036, anomalía **A-79**); y (2) mover la tabla de tamaños para que aparezca justo debajo
del encabezado "Tamaños del producto", y no debajo del interruptor "Maneja inventario", que hoy
hace parecer que la tabla es lo que ese interruptor habilita (User Story 8, FR-037 a FR-039). La
enmienda **sustituye** parte de lo dicho en User Story 1 y FR-001 a FR-007 (opcionalidad de la
presentación, nombre libre, cascada de renombre): esos textos quedan marcados abajo con *(enmendado)*.
Ver `## Clarifications` → sesión 2026-09-20.

## Contexto y relación con specs anteriores

Estos cinco bugs se detectaron probando el comportamiento en producción de funcionalidad ya
especificada en [063 — Promociones por variante](../063-promociones-por-variante/spec.md),
[066 — Promociones legibles y precios reales en el menú QR](../066-promociones-legibles-menu/spec.md),
[081 — Pestaña Promociones del menú QR](../081-tab-promociones-menu-qr/spec.md) y
[083 — Catálogo de Presentaciones y Rediseño de Promociones](../083-presentaciones-y-promociones/spec.md)
(esta última, la más reciente, con anomalías **A-74** y **A-75** ya registradas en
[`registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md)). Esta spec **no
reabre** el motor de cálculo de descuentos de esas specs; corrige comportamientos puntuales que no
siguen lo ya definido, cierra vacíos que quedaron sin cubrir por la interacción entre ellas, y
agrega dos decisiones de negocio nuevas confirmadas con el propietario del producto (ver
`## Clarifications`). En particular:

- El bug 1 aparece porque la spec 083 (FR-025) retira, para reglas nuevas, el patrón de precio de
  paquete con `minQuantity = 1` que spec 066 (FR-015) usaba como la única condición bajo la cual
  la tarjeta del menú QR mostraba un precio con descuento. Con reglas nuevas siempre en
  `minQuantity >= 2`, la tarjeta se queda solo con la insignia genérica "🎉 Promo" (spec 066
  FR-013/FR-014) y ningún precio visible — el vacío que este bug cierra.
- El bug 2, tras confirmarlo con el propietario del producto, **no** es sobre herencia automática
  de variantes por categoría (eso ya lo cubre spec 083 FR-005/FR-006 sin cambios de esta spec):
  es que el catálogo de Presentaciones (spec 083) no tiene ninguna relación real con las variantes
  — el formulario de crear/editar producto permite definir nombre y precio de cada variante, pero
  no permite asociarla con una presentación del catálogo. Esta spec agrega esa asociación como una
  extensión nueva del modelo de `ProductVariant` (ver `## Clarifications` y Key Entities), algo que
  spec 083 declaraba explícitamente fuera de su alcance ("sin cambio de modelo").
- El bug 3, tras confirmarlo, tampoco reabre el alcance automático de una regla por presentación
  (A-65, spec 063 FR-003/FR-010 no cambian): pide un atajo de captura para aplicar una regla, de
  una sola acción, a los productos ya seleccionados en el Paso 1 que comparten esa presentación. Al
  resolver esta pregunta, el propietario del producto introdujo además una regla de negocio nueva
  (no reportada como bug, pero necesaria para acotar el bug 1): un producto no puede seleccionarse
  en más de una promoción vigente a la vez.
- El bug 4 endurece, para el estado `Activa`, la edición parcial que permitía spec 083 FR-018.
- El bug 5 no está cubierto por ninguna spec anterior; es un defecto de interacción nuevo.

## Clarifications

### Session 2026-09-17

- Q: El bug 3 describe que, al configurar una regla (por ejemplo "Presentación única x 2 unidades
  $15.000"), el descuento debe poder aplicarse "a todos los productos seleccionados que hagan
  parte de la promoción". Spec 063 (FR-003/FR-010, anomalía A-65) estableció deliberadamente que
  el alcance de una regla de promoción **no** es automático por nombre de presentación — es
  siempre una lista explícita de variantes elegidas a mano. ¿Este bug pide reabrir esa decisión, o
  pide que, dentro del conjunto de productos ya elegidos en el Paso 1 (spec 083 FR-012), una sola
  regla configurada una vez se aplique de una sola acción a la variante correspondiente de cada uno
  de esos productos ya seleccionados? → A: Aplicar de una sola acción a la variante correspondiente
  de cada producto ya seleccionado en el Paso 1 (bulk-apply acotado al conjunto ya elegido). No se
  reabre alcance automático por presentación fuera de ese conjunto — A-65 y spec 063 FR-003/FR-010
  no cambian. Cada fila de regla resultante sigue guardando una lista explícita de variantes (una
  por producto), igual que hoy; el cambio es de **flujo de captura**, no de **modelo de datos** ni
  de motor de cálculo.

- Q: El bug 4 pide que una promoción "no se pueda configurar cuando esté vigente". Spec 083
  (FR-018) ya permite, en estado `Activa` o `Pausada`, seguir editando nombre, fin de vigencia,
  días y horas, bloqueando solo tipo/valor/unidades mínimas/conjunto de variantes de las reglas ya
  guardadas. ¿El bug pide bloquear **toda** edición (deshabilitar por completo el botón
  "Configurar" del listado) mientras el estado real sea `Activa`, o pide mantener la edición
  parcial ya definida por FR-018 y el problema reportado es que hoy ni siquiera esa restricción
  parcial se aplica? → A: El botón "Configurar" del listado debe quedar deshabilitado (sin poder
  abrir la pantalla de configuración) para toda promoción cuyo estado real sea `Activa`. Solo se
  puede configurar una promoción en estado `Borrador`, `Pausada` o `Finalizada`; para modificar
  cualquier campo de una promoción `Activa` — incluida una extensión de fecha de fin — el
  administrador primero debe pausarla desde el listado. Esto reemplaza, para el estado `Activa`, la
  edición parcial que permitía spec 083 FR-018; el resto de FR-018 (bloqueo de tipo/valor/unidades
  mínimas/conjunto de variantes) sigue aplicando sin cambios para el estado `Pausada`.

- Q: Al aplicar de una sola acción una regla a varios productos ya seleccionados que comparten la
  presentación elegida (bug 3), ¿el administrador debe poder excluir a alguno de esos productos
  antes de que se generen las filas, o la aplicación es incondicional para todos los coincidentes?
  → A: El sistema muestra la lista de productos coincidentes con una casilla de verificación por
  producto, todas premarcadas; el administrador puede desmarcar los que no quiere incluir antes de
  confirmar. Solo se genera una fila de regla por cada producto que quede marcado al confirmar.

- Q: El bug 2, tal como se redactó inicialmente (herencia automática de variantes al crear un
  producto según las presentaciones asociadas a su categoría, spec 083 FR-005/FR-006), ¿es
  realmente el problema reportado? → A: No. La categoría no cambia ni es el problema. Hoy, al crear
  una variante en el formulario de producto, solo se puede definir su nombre (texto libre) y su
  precio; lo que falta es poder asociar esa variante con una de las presentaciones ya definidas en
  el catálogo (spec 083), como una relación adicional a nombre y precio — no como una lista que se
  genera sola al elegir la categoría.

- Q: Al asociar una variante con una presentación del catálogo (pregunta anterior), ¿el nombre de
  la variante debe tomarse automáticamente del nombre de la presentación elegida y quedar
  sincronizado, o la variante conserva su propio nombre libre y la presentación queda solo como una
  referencia adicional independiente del nombre? → A: El nombre de la variante se toma del nombre
  de la presentación elegida y queda sincronizado — si la presentación se renombra después desde el
  catálogo, o el administrador elige otra presentación para esa variante, el nombre visible de la
  variante se actualiza para no quedar desalineado del catálogo.

- Q: Al definir una regla de precio de paquete o porcentaje que cubre variantes de un producto,
  ¿qué pasa si ese mismo producto (por alguna de sus otras variantes) ya hace parte de otra
  promoción vigente en ese momento? → A: Por ahora, un producto no puede incluirse en otra
  promoción si ya hace parte de una promoción vigente — es una restricción nueva, a nivel de
  producto completo (no solo de la variante ya cubierta), que se agrega junto con la corrección del
  bug 3.

- Q: Esa restricción de exclusividad, ¿bloquea cualquier variante del producto mientras alguna de
  sus variantes ya esté cubierta por otra promoción vigente, o solo bloquea reutilizar la misma
  variante ya cubierta (dejando libres las demás variantes de ese producto)? → A: Bloqueo a nivel
  de producto completo: mientras cualquier variante de un producto ya esté cubierta por una regla
  vigente de una promoción, ninguna variante de ese mismo producto puede seleccionarse en una
  promoción distinta. Dentro de la **misma** promoción, seleccionar varias variantes del mismo
  producto (p. ej. "Pequeña" y "Grande") sigue permitido sin cambio.

### Session 2026-09-20

- Q: Al quitar el campo "Nombre" de la variante, ¿cómo se nombran las variantes que hoy no tienen
  presentación — la variante default "Presentación única" de un producto sin tamaños y las de
  nombre libre ya guardadas con "Sin presentación"? → A: La presentación pasa a ser **obligatoria**
  para toda variante. "Presentación única" deja de ser un literal guardado en la variante y pasa a
  ser una fila del catálogo de presentaciones (se crea sola por tenant si no existe). La migración
  crea, para cada nombre libre ya guardado, una fila del catálogo con ese nombre (o reutiliza la
  que ya lo tenga) y enlaza la variante — ningún nombre existente se pierde. La opción "Sin
  presentación" desaparece del selector.
- Q: Con el nombre derivado de la presentación, ¿el campo `name` de una variante se mantiene en las
  respuestas de la API como dato calculado de solo lectura, o se elimina de la API? → A: Se elimina
  de la API. Todas las respuestas que hoy traen el nombre de una variante pasan a traer
  `presentation_id` y `presentation_name`; el payload de guardado deja de aceptar `name`.
- Q: El segundo ajuste ("el catálogo de variantes parece la opción que se habilita al marcar
  Maneja inventario"), ¿pide mover la tabla de tamaños para que quede justo debajo del encabezado
  "Tamaños del producto", con "Maneja inventario" después de la tabla? → A: Sí. Orden final dentro
  de la tarjeta: encabezado y interruptor de tamaños → tabla de tamaños (con sus presentaciones
  desactivadas) → "Maneja inventario" → detalle del tamaño activo (insumos fijos y sabores).

- Q: El selector de presentación del Paso 2 de una promoción listaba solo las presentaciones de
  las variantes de los productos ya elegidos en el Paso 1 (spec 083 FR-013): con el Paso 1 vacío,
  quedaba vacío. ¿Debe listar el catálogo, y cómo se guarda el alcance? → A: Debe listar **todas
  las presentaciones activas del catálogo**, independientes de los productos, y la regla se
  resuelve **al configurar**: se guarda la lista explícita de variantes (una fila por producto
  seleccionado que tenga esa presentación), como hoy, sin modo dinámico en la venta (A-65 y spec
  063 FR-003/FR-010 no cambian). Se descartó la variante dinámica (la regla guarda la presentación
  y el motor la resuelve en cada venta) por exigir cambios de modelo, motor, menú QR y guardas de
  exclusividad. Registrado como **A-80**.

### Session 2026-09-21

- Q: Al agregar la regla «8 onzas × 2 = $12.000» con varios productos seleccionados, el sistema
  generaba una fila por producto (A-77): cambiar de productos obligaba a quitar esas filas y
  crear otras, y las mismas presentaciones y precios se duplicaban por producto. ¿Cómo debe ser?
  → A: La lista debe tener **una regla por presentación** («8 onzas · 2 × $12.000», «12 onzas · 2
  × $17.000»), independiente de los productos. Al seleccionar o quitar productos en el Paso 1, las
  reglas se aplican solas a la variante de cada producto seleccionado que tenga esa presentación
  (el producto ya trae la presentación en su variante). Registrado como **A-81**.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Asociar cada variante de un producto con una presentación del catálogo (Priority: P1)

Hoy, al crear o editar un producto, cada variante solo permite escribir un nombre libre y un
precio; el catálogo de Presentaciones (spec 083) no tiene ninguna relación real con esas variantes.
El administrador necesita, al agregar o editar una variante, poder elegir una presentación del
catálogo activo — al elegirla, el nombre de la variante se completa automáticamente con el nombre
de esa presentación y queda sincronizado con ella — en lugar de escribir el nombre a mano sin
ninguna relación verificable con el catálogo.

**Why this priority**: sin esta asociación, el catálogo de Presentaciones (spec 083) sigue siendo
una simple lista de nombres sugeridos, sin ningún vínculo real con los productos; y la aplicación
masiva de una regla a varios productos (User Story 4) depende de poder identificar con certeza qué
variante de cada producto corresponde a la presentación elegida en la regla.

**Independent Test**: crear o editar un producto, agregar una variante nueva, seleccionar una
presentación activa del catálogo (p. ej. "Grande") y verificar que el nombre de la variante se
completa solo con "Grande" y queda guardada esa asociación; luego renombrar "Grande" a "Extra
Grande" desde el catálogo de Presentaciones y verificar que el nombre de esa variante se actualiza
también a "Extra Grande".

**Acceptance Scenarios**:

1. **Given** el administrador está creando o editando un producto y agrega una variante nueva,
   **When** elige una presentación activa del catálogo para esa variante, **Then** el nombre de la
   variante se completa automáticamente con el nombre de esa presentación y queda asociada a ella.
2. **Given** una variante ya asociada a una presentación, **When** el administrador elige otra
   presentación distinta para esa misma variante, **Then** el nombre de la variante se actualiza al
   nombre de la nueva presentación elegida.
3. ~~**Given** el administrador no quiere asociar una variante a ninguna presentación del
   catálogo, **When** deja el selector en "Sin presentación", **Then** puede seguir escribiendo el
   nombre de esa variante libremente, igual que hoy.~~ *(enmendado 2026-09-20 — reemplazado por
   User Story 7: la presentación es obligatoria y no existe "Sin presentación")*
4. **Given** una presentación del catálogo con una o más variantes de producto ya asociadas a ella,
   **When** el administrador la renombra desde la sección "Presentaciones", **Then** el nombre que
   ve cada variante asociada refleja el nuevo nombre, sin que el administrador tenga que editar
   cada producto uno por uno. *(enmendado 2026-09-20 — ya no se copia ni se "actualiza" ningún
   nombre en las variantes: lo muestran leyendo la presentación, ver FR-030)*
5. **Given** un producto con dos variantes ya asociadas cada una a una presentación distinta,
   **When** el administrador intenta asociar una tercera variante del mismo producto a una
   presentación que ya usa otra variante de ese mismo producto, **Then** el sistema lo impide y
   explica que esa presentación ya está en uso por otra variante del mismo producto.

---

### User Story 2 - Una promoción activa no se puede reconfigurar sin pausarla primero (Priority: P1)

Desde el listado de promociones, el botón "Configurar" de una promoción actualmente `Activa`
permite hoy abrir la pantalla de configuración y editar sus reglas, lo cual expone a error de
cobro sobre ventas en curso. El administrador necesita que, mientras una promoción esté en estado
`Activa`, el botón quede deshabilitado y la única forma de modificarla sea pausarla primero desde
el mismo listado.

**Why this priority**: es un riesgo de negocio — permite alterar las condiciones de una promoción
que ya se está cobrando a clientes en ese momento, algo que specs anteriores (063, 083) ya
buscaban impedir a nivel de reglas y que esta corrección extiende a la pantalla completa.

**Independent Test**: activar una promoción de prueba, confirmar que su botón "Configurar" en el
listado queda deshabilitado, pausarla desde el listado, y confirmar que "Configurar" vuelve a
estar disponible y abre la pantalla de configuración con sus campos editables.

**Acceptance Scenarios**:

1. **Given** una promoción con estado real `Activa`, **When** el administrador ve su fila en el
   listado, **Then** el botón "Configurar" aparece deshabilitado (o con una indicación equivalente
   de no disponible), sin permitir abrir la pantalla de configuración.
2. **Given** una promoción con estado real `Borrador`, `Pausada` o `Finalizada`, **When** el
   administrador presiona "Configurar", **Then** la pantalla de configuración se abre con sus
   campos editables, igual que hoy.
3. **Given** una promoción `Activa` que el administrador necesita modificar, **When** la pausa
   desde el listado, **Then** su botón "Configurar" pasa a estar disponible y, al presionarlo, la
   pantalla de configuración se abre con todos sus campos editables (no solo nombre/fin de
   vigencia/días/horas).
4. **Given** el estado visual derivado de una promoción `Activa` (p. ej. badge "Vencida" o "Fuera
   de horario" por vigencia ya pasada o fuera de horario), **When** el administrador ve su fila,
   **Then** "Configurar" sigue deshabilitado igual que cualquier promoción `Activa`, porque la
   decisión se basa en el estado real (`status`), no en el badge — mismo criterio ya usado por
   spec 083 FR-021/FR-022 para el botón "Eliminar" y las pestañas de filtro.

---

### User Story 3 - La tarjeta de un producto en promoción muestra un precio, no solo la insignia (Priority: P2)

Un comensal que abre la pestaña "Promociones" del menú QR (spec 081) ve tarjetas de producto que
hoy, para la mayoría de las promociones nuevas (precio de paquete con 2 o más unidades, tras spec
083 FR-025), solo muestran la insignia genérica "🎉 Promo" sin ningún precio — tiene que abrir el
producto para enterarse de cuánto cuesta o cuántas unidades necesita llevar. La tarjeta debe
mostrarle, sin abrir el producto, el precio mínimo al que puede acceder con la promoción vigente.

**Why this priority**: afecta directamente la experiencia del comensal en la pestaña que spec 081
creó específicamente para que las promociones fueran fáciles de encontrar y entender sin tener que
abrir cada producto.

**Independent Test**: con una regla de precio de paquete "2 x $15.000" vigente sobre la única
variante de un producto, abrir la pestaña "Promociones" del menú QR y verificar que la tarjeta
muestra el precio de paquete ($15.000 por 2 unidades) sin necesidad de abrir el producto.

**Acceptance Scenarios**:

1. **Given** un producto con una única variante cubierta por una regla de precio de paquete
   vigente (p. ej. "2 x $15.000"), **When** el comensal ve su tarjeta en la pestaña
   "Promociones", **Then** la tarjeta muestra ese precio de paquete (o su equivalente por unidad,
   con el mismo formato ya definido por spec 066 FR-008 para el modal), además de la insignia
   genérica "🎉 Promo".
2. **Given** un producto con varias variantes propias (p. ej. "Pequeña" y "Grande") cubiertas por
   distintas reglas de precio de paquete de la **misma** promoción vigente (dentro de una sola
   promoción sigue permitido cubrir varias variantes del mismo producto, ver User Story 1 de spec
   083 y la restricción de exclusividad entre promociones más abajo), **When** el comensal ve su
   tarjeta, **Then** la tarjeta muestra el precio equivalente por unidad más bajo entre esas reglas,
   identificado como precio mínimo (p. ej. "Desde $X"), sin prometer ese precio para una variante
   que no lo tiene.
3. **Given** un producto cubierto por una regla de tipo porcentaje con cantidad mínima 1 (caso ya
   cubierto por spec 066 FR-015), **When** el comensal ve su tarjeta, **Then** sigue mostrando el
   precio normal tachado junto al precio con descuento, sin cambio respecto al comportamiento
   actual.
4. **Given** un producto sin ninguna variante cubierta por una regla vigente en este momento,
   **When** el comensal lo ve fuera de la pestaña "Promociones", **Then** su tarjeta no muestra
   ningún precio de promoción ni la insignia, igual que hoy.

---

### User Story 4 - Definir una regla la aplica de una vez a todos los productos ya seleccionados (Priority: P2)

En la pantalla de configuración de una promoción, el administrador ya seleccionó varios productos
en el Paso 1 (spec 083 FR-012) que comparten la misma presentación (p. ej. "Presentación única").
Al definir una sola regla de precio para esa presentación (unidades mínimas y precio), espera que
el sistema la aplique de una vez a la variante correspondiente de cada uno de los productos ya
seleccionados que la tengan, pudiendo excluir de esa aplicación a los productos que no le
interesen, en lugar de repetir manualmente una fila por producto.

**Why this priority**: reduce el trabajo repetitivo de armar una promoción con muchos productos
que comparten condición — sin este cambio, el rediseño guiado de spec 083 sigue exigiendo tantas
filas como productos, aunque compartan exactamente la misma regla.

**Independent Test**: seleccionar tres productos que comparten la variante "Presentación única" en
el Paso 1, definir una sola regla "Presentación única x 2 unidades $15.000", desmarcar uno de los
tres productos en la lista de confirmación, y verificar que se crean solo dos filas de regla (una
por cada producto que quedó marcado, cada una con su propia variante).

**Acceptance Scenarios**:

1. **Given** dos o más productos ya seleccionados en el Paso 1, cada uno con una variante cuyo
   nombre coincide con la presentación elegida en la regla, **When** el administrador define esa
   regla (presentación, unidades mínimas, precio), **Then** el sistema muestra la lista de esos
   productos coincidentes con una casilla de verificación por producto, todas premarcadas.
2. **Given** la lista de productos coincidentes con sus casillas premarcadas, **When** el
   administrador desmarca uno o más productos y confirma, **Then** el sistema agrega una fila de
   regla solo por cada producto que quedó marcado, cada una con la variante correspondiente de su
   propio producto — sin crear fila para los productos desmarcados.
3. **Given** entre los productos ya seleccionados en el Paso 1, alguno no tiene una variante cuyo
   nombre coincida con la presentación elegida en la regla, **When** el administrador ve la lista
   de aplicación masiva, **Then** ese producto no aparece como opción marcable (o aparece
   deshabilitado con la razón), y no se le crea ninguna fila.
4. **Given** las filas de regla ya generadas por la aplicación masiva, **When** el administrador
   necesita un precio distinto para uno de esos productos, **Then** puede editar o eliminar esa
   fila individualmente sin afectar las demás, igual que si la hubiera creado una por una.
5. **Given** una regla ya aplicada de forma masiva a varios productos, **When** el administrador
   guarda la promoción, **Then** cada fila se guarda como una regla independiente con su propia
   lista explícita de variantes (una por producto), igual que si se hubieran creado a mano — el
   alcance por variante explícita de spec 063 (FR-003/FR-010) no cambia.

---

### User Story 5 - Un producto no puede quedar cubierto por dos promociones vigentes a la vez (Priority: P1)

Al armar el Paso 1 de una promoción nueva (o editar el de una existente en `Borrador`/`Pausada`),
el administrador necesita que el sistema le impida seleccionar un producto que ya hace parte de
otra promoción vigente en ese momento — sin importar si la variante que quiere usar en la nueva
promoción es la misma u otra distinta a la ya comprometida — para que nunca existan dos precios
promocionales distintos compitiendo por el mismo producto al mismo tiempo.

**Why this priority**: es una condición de integridad de precios — sin ella, un mismo producto
podría anunciar y cobrar condiciones distintas según cuál de dos promociones activas "gane", algo
que ni spec 063 (exclusividad por variante) ni spec 083 evitaban a nivel de producto completo.

**Independent Test**: con una promoción `Activa` que ya cubre la variante "Grande" de un producto,
intentar seleccionar ese mismo producto (incluso por su variante "Pequeña") en el Paso 1 de una
promoción distinta, y verificar que el sistema lo impide y explica por qué.

**Acceptance Scenarios**:

1. **Given** un producto con al menos una variante cubierta por una regla vigente de una promoción
   con estado real `Activa`, **When** el administrador intenta seleccionarlo en el Paso 1 de una
   promoción distinta, **Then** el sistema lo impide y le indica que ese producto ya hace parte de
   la promoción activa que lo cubre.
2. **Given** un producto ya seleccionado y con reglas guardadas dentro de la **misma** promoción,
   **When** el administrador agrega otra variante de ese mismo producto (p. ej. "Grande" además de
   "Pequeña") a esa misma promoción, **Then** el sistema lo permite sin cambio — la restricción
   aplica solo entre promociones distintas, no dentro de la misma.
3. **Given** dos promociones que intentan guardarse o activarse casi al mismo tiempo compartiendo un
   producto, **When** la segunda intenta guardar o activarse, **Then** el sistema rechaza esa
   operación con el mismo mensaje de exclusividad, aunque la selección en pantalla la haya
   permitido momentáneamente por una condición de carrera.
4. **Given** promociones ya activas hoy que comparten un producto entre sí (posible bajo el
   criterio de exclusividad por variante de spec 063 FR-014, vigente antes de esta spec), **When**
   ninguna de ellas se edita ni se reactiva, **Then** siguen funcionando y cobrándose sin cambio —
   la restricción nueva no es retroactiva.

---

### User Story 6 - Abrir el menú de acciones de una fila no desplaza ni recorta la tabla (Priority: P3)

Al abrir el menú de acciones (⋮) de una fila en un listado con muchas filas (promociones o
productos), el menú se corta contra el borde del contenedor con scroll de la tabla, o su apertura
mueve el scroll de la tabla, obligando al administrador a desplazarse para ver las opciones
completas.

**Why this priority**: es un defecto de interacción molesto pero no bloquea ninguna funcionalidad
— el administrador puede completar la acción, solo con fricción visual.

**Independent Test**: en un listado con más filas de las que caben en pantalla, desplazarse hasta
una fila cercana al borde inferior o superior del contenedor con scroll, abrir su menú de
acciones, y verificar que el menú se muestra completo y visible sin que la tabla se desplace ni
recorte ninguna opción.

**Acceptance Scenarios**:

1. **Given** un listado (promociones o productos) con scroll propio y varias filas, **When** el
   administrador abre el menú de acciones de una fila cercana al borde del contenedor visible,
   **Then** el menú se muestra completo, sin quedar recortado por el borde del contenedor ni
   generar una barra de scroll adicional dentro de la tabla.
2. **Given** el menú de acciones de una fila está abierto, **When** el administrador hace clic
   fuera del menú o selecciona una opción, **Then** el menú se cierra y la posición de scroll de
   la tabla queda exactamente donde estaba antes de abrirlo.
3. **Given** el menú de acciones de una fila está abierto, **When** el administrador desplaza la
   tabla (scroll), **Then** el menú se cierra o se reposiciona junto a su fila, sin quedar flotando
   sobre una fila distinta a la que lo abrió.

---

### User Story 7 - El nombre de una variante lo da su presentación: el campo "Nombre" se elimina (Priority: P1)

*(Enmienda 2026-09-20.)* Con la asociación de User Story 1, cada fila de tamaño del formulario
mostraba a la vez un campo "Nombre" (de solo lectura, copia del de la presentación) y el selector
de "Presentación": dos columnas con el mismo dato. El administrador pidió quitar el nombre: una
variante se identifica por su presentación y nada más. Esto implica que toda variante tenga una
presentación (no puede haber variante "sin presentación") y que el nombre deje de existir como dato
propio de la variante, en el formulario, en la API y en la base de datos.

**Why this priority**: es un cambio de modelo con paso de datos y de contrato de API; cuanto más
se demore, más lecturas nuevas de `variant.name` se escriben. Además elimina la cascada de renombre
(FR-004 original) y su caso límite de colisión de nombres, que ya no tienen razón de ser.

**Independent Test**: en un tenant con un producto de dos tamaños "Pequeña"/"Grande" y otro con
una variante de nombre libre "Familiar" sin presentación, aplicar la migración; abrir ambos
productos en el formulario y verificar que no hay columna "Nombre", que los tres tamaños muestran su
presentación ("Pequeña", "Grande", "Familiar") y que "Familiar" ahora existe en el catálogo de
Presentaciones. Renombrar "Grande" a "Extra Grande" desde el catálogo y verificar que el menú QR,
el carrito de mesa y el formulario de producto ya muestran "Extra Grande" sin ninguna otra acción.

**Acceptance Scenarios**:

1. **Given** el formulario de crear o editar un producto con tamaños, **When** el administrador ve
   la tabla, **Then** cada fila muestra solo el selector de presentación y el precio (más el
   arrastre, el número y "Eliminar"); no hay columna ni campo "Nombre".
2. **Given** una fila de tamaño nueva sin presentación elegida, **When** el administrador intenta
   guardar el producto, **Then** el sistema lo impide indicando qué fila no tiene presentación —
   no existe la opción "Sin presentación".
3. **Given** un producto sin tamaños (interruptor "Tamaños del producto" apagado), **When** el
   administrador lo guarda, **Then** su única variante queda asociada a la presentación
   "Presentación única" del catálogo, creada automáticamente si el tenant aún no la tenía.
4. **Given** el sistema con variantes ya guardadas con nombre libre y sin presentación, **When** se
   aplica la migración, **Then** por cada nombre distinto se crea una presentación con ese nombre
   en el catálogo (o se reutiliza la que ya lo tenga) y cada variante queda enlazada a ella, sin
   que ningún nombre visible cambie.
5. **Given** cualquier consumidor que hoy lee el nombre de una variante (menú QR, carrito de mesa
   y checkout, selector de producto, selector de reglas de promoción), **When** se carga tras el
   cambio, **Then** muestra el nombre de la presentación, con el mismo texto que mostraba antes.
6. **Given** una presentación con variantes asociadas, **When** el administrador la renombra desde
   "Presentaciones", **Then** la operación siempre se acepta si el nombre nuevo es único en el
   catálogo — ya no puede rechazarse por un choque con otra variante del mismo producto.
7. **Given** una presentación con al menos una variante asociada, **When** alguien intenta
   eliminarla del catálogo de forma física, **Then** la base de datos lo impide (no existe
   endpoint de borrado; solo se puede desactivar, y una presentación desactivada sigue nombrando
   las variantes que ya la usan).

---

### User Story 8 - La tabla de tamaños va justo debajo del encabezado, antes de "Maneja inventario" (Priority: P3)

*(Enmienda 2026-09-20.)* En la tarjeta "Tamaños del producto", el interruptor "Maneja inventario"
aparece entre el encabezado y la tabla de tamaños. Eso hace parecer que la tabla es lo que se
habilita al activar "Maneja inventario", cuando en realidad la tabla (presentación y precio de cada
tamaño) existe siempre que el producto tenga tamaños, con o sin inventario. Solo el bloque de
insumos fijos y sabores de cada tamaño depende de ese interruptor.

**Why this priority**: cambio de orden de presentación, sin efecto en datos ni en la API; confunde
pero no bloquea.

**Independent Test**: abrir el formulario de un producto con tamaños con "Maneja inventario"
apagado y verificar que la tabla de tamaños se ve completa y editable justo debajo del encabezado,
y que el interruptor "Maneja inventario" queda debajo de ella.

**Acceptance Scenarios**:

1. **Given** el formulario de producto con el interruptor de tamaños encendido, **When** se
   renderiza la tarjeta "Tamaños del producto", **Then** el orden vertical es: encabezado con su
   interruptor → tabla de tamaños con "+ Agregar tamaño" → lista de presentaciones desactivadas
   (si hay) → "Maneja inventario" (y su aviso, si aplica) → detalle del tamaño activo.
2. **Given** "Maneja inventario" apagado, **When** el administrador ve la tarjeta, **Then** la tabla
   de tamaños sigue visible y editable, y solo el detalle del tamaño activo muestra el aviso "Activa
   'Maneja inventario' arriba…", que apunta al interruptor que está justo encima.
3. **Given** el interruptor de tamaños apagado (producto sin tamaños), **When** se renderiza la
   tarjeta, **Then** no hay tabla, "Maneja inventario" queda justo debajo del encabezado y el
   precio y el detalle de la única variante siguen debajo de él, sin cambio de comportamiento.

---

> **Enmienda 2026-09-21 (A-81) a User Story 4**: la lista de reglas es por presentación, no por
> producto (FR-047 a FR-052); el panel de confirmación con casilla por producto desaparece. Los
> escenarios de la historia que hablan de «una fila por producto» se leen ahora como «una regla de
> presentación que se expande a una regla de backend por producto».
>
> **Enmienda 2026-09-20 (A-80) a User Story 4**: el selector "Presentación / Tamaño" del Paso 2
> ya no depende de los productos del Paso 1: lista todas las presentaciones activas del catálogo
> (FR-040) y la aplicación masiva empareja por presentación (FR-041). El resto de la historia
> (casilla por producto, una fila por producto) no cambia.

### Edge Cases

- ¿Qué pasa si el administrador cambia las presentaciones asociadas a la categoría después de
  crear el producto? No se modifican retroactivamente las variantes ya creadas (spec 083 FR-008,
  sin cambio) — esta spec no toca ese mecanismo de herencia por categoría.
- ~~¿Qué pasa si renombrar una presentación (FR-004) haría que una de sus variantes asociadas
  quedara con el mismo nombre que otra variante sin presentación del mismo producto? El sistema
  rechaza el renombre completo…~~ *(enmendado 2026-09-20 — el caso deja de existir: la variante no
  guarda nombre, así que no hay nada que pueda colisionar; el renombre solo valida la unicidad del
  nombre dentro del catálogo de presentaciones, spec 083 FR-003)*
- ¿Qué pasa si, durante la migración de US7, dos variantes distintas de **productos distintos**
  tenían el mismo nombre libre (p. ej. "Familiar" en dos productos)? Comparten la misma fila del
  catálogo: la migración empareja por nombre exacto, y la unicidad `(producto, presentación)` solo
  impide repetirla dentro del **mismo** producto — que el `UNIQUE(product_id, name)` anterior ya
  impedía, así que no puede haber colisión.
- ¿Qué pasa si el nombre libre de una variante existente coincide con una presentación del
  catálogo que está desactivada? La migración reutiliza esa fila (no crea un duplicado, que
  chocaría con `presentations.name UNIQUE`); la variante sigue funcionando y el selector del
  formulario sigue mostrando esa presentación para esa variante aunque esté inactiva (ya lo hace
  hoy), pero no la ofrece a otras variantes.
- ¿Qué pasa si el administrador desactiva o renombra la presentación "Presentación única"? Las
  variantes que ya la usan siguen mostrando su nombre actual. Al guardar un producto nuevo sin
  tamaños, el sistema busca la presentación por el nombre literal "Presentación única" (sin
  importar si está activa); si el administrador la renombró y no existe ninguna con ese nombre,
  crea una nueva — no hay marca especial de "presentación del sistema".
- ¿Qué pasa si el administrador intenta guardar un producto con tamaños y dos filas con la misma
  presentación? Se rechaza como hoy (FR-006); ahora el mensaje se muestra sobre la fila
  duplicada, ya que no hay un nombre distinto que la diferencie.
- ¿Qué pasa al encender el interruptor de tamaños en un producto que tenía una sola variante
  "Presentación única"? Se crean tres filas (Grande, Mediana, Pequeña) que heredan su precio,
  receta y grupos de opciones; a cada una se le preselecciona la presentación del catálogo con ese
  nombre si existe (si no, queda sin elegir y el administrador debe escogerla antes de guardar). Al
  apagarlo, se conserva la primera fila y se reasocia a "Presentación única".

## Requirements *(mandatory)*

### Asociación de variante con presentación del catálogo (bug 2)

- **FR-001**: El formulario de crear o editar producto MUST permitir, para cada variante (nueva o
  ya existente), seleccionar una presentación activa del catálogo (spec 083) como parte de los
  datos de esa variante, además de los campos ya existentes de nombre y precio, con la opción "Sin
  presentación" para no asociarla a ninguna.
- **FR-002**: Al seleccionar una presentación para una variante, el sistema MUST completar
  automáticamente el nombre de esa variante con el nombre de la presentación elegida; mientras la
  variante tenga una presentación asociada, su nombre MUST mostrarse como derivado de esa
  presentación (no editable de forma independiente).
- **FR-003**: Al elegir una presentación distinta para una variante que ya tenía una asociada, el
  sistema MUST actualizar el nombre de la variante al nombre de la nueva presentación elegida.
- **FR-004**: Al renombrar una presentación desde la sección "Presentaciones" del catálogo, el
  sistema MUST actualizar el nombre de cada variante de producto actualmente asociada a ella, para
  que nunca queden desalineados. Si ese renombre haría que una variante quede con el mismo nombre
  que otra variante sin presentación asociada del **mismo producto** (violando la unicidad de
  nombre por producto ya vigente), el sistema MUST rechazar el renombre completo de la presentación
  con un error que liste el producto y la variante en conflicto, sin renombrar ninguna variante.
- **FR-005**: Al poner el selector de una variante en "Sin presentación", el sistema MUST dejar el
  nombre de esa variante editable libremente, conservando como punto de partida el texto que tenía
  en ese momento (no lo borra).
- **FR-006**: El sistema MUST impedir asociar dos variantes del mismo producto a la misma
  presentación del catálogo, explicando que esa presentación ya está en uso por otra variante de
  ese producto.
- **FR-007**: Esta asociación es opcional y no retroactiva: una variante creada antes de esta
  funcionalidad, o cuyo administrador elija no asociarla, MUST seguir funcionando exactamente igual
  que hoy (nombre libre, sin presentación vinculada), sin que el sistema la fuerce a asociarse.

> **Enmienda 2026-09-20 (A-79)** — FR-001 a FR-007 se leen ahora así: FR-001 *(enmendado)*: la
> presentación es **obligatoria**, no existe la opción "Sin presentación"; FR-002 y FR-003
> *(enmendados)*: no se "completa" ni "actualiza" un nombre, la variante no tiene nombre propio y
> lo muestra leyendo su presentación (FR-030); FR-004 *(reemplazado)*: sin cascada de renombre y
> sin guarda de colisión (FR-031); FR-005 *(reemplazado)*: no hay nombre libre (FR-029); FR-006
> *(sin cambio)*; FR-007 *(reemplazado)*: la asociación deja de ser opcional y **sí** es
> retroactiva mediante la migración de datos de FR-033. Las secciones siguientes son la fuente
> vigente.

### Bloqueo de configuración de promociones activas (bug 4)

- **FR-008**: En el listado de promociones, el botón "Configurar" de una fila MUST aparecer
  deshabilitado (sin abrir la pantalla de configuración) cuando el estado real (`status`) de esa
  promoción sea `Activa`, sin importar el estado visual derivado que muestre su badge (p. ej.
  "Vencida" o "Fuera de horario" con `status=Activa` sigue deshabilitado) — mismo criterio de
  estado real ya usado por spec 083 FR-021/FR-022.
- **FR-009**: El botón "Configurar" MUST seguir habilitado, abriendo la pantalla de configuración
  con todos sus campos editables, para toda promoción cuyo estado real sea `Borrador`, `Pausada` o
  `Finalizada`.
- **FR-010**: Al pausar una promoción `Activa` desde el listado (acción ya existente), su botón
  "Configurar" MUST quedar disponible de inmediato, sin recargar la página, reflejando el nuevo
  estado real `Pausada`.
- **FR-011**: Esta restricción reemplaza, para el estado `Activa`, la edición parcial que permitía
  spec 083 FR-018 (nombre, fin de vigencia, días, horas editables en `Activa`); para el estado
  `Pausada`, FR-018 sigue aplicando sin cambios (bloqueo de tipo/valor/unidades mínimas/conjunto de
  variantes de reglas ya guardadas, resto de campos editable).

### Precio visible en la tarjeta de promoción del menú QR (bug 1)

- **FR-012**: La tarjeta de un producto en la pestaña "Promociones" del menú QR (spec 081 FR-003)
  MUST mostrar, además de la insignia genérica "🎉 Promo" (spec 066 FR-013), el precio de la
  regla vigente que cubre su variante — usando el mismo formato ya definido por spec 066 FR-008
  para el equivalente por unidad ("N x $valor · $equivalente c/u") — sin exigir que el comensal
  abra el producto para conocerlo.
- **FR-013**: Cuando más de una variante del mismo producto esté cubierta por reglas vigentes de la
  misma promoción con precios distintos, la tarjeta MUST mostrar el precio equivalente por unidad
  más bajo entre esas reglas, identificado como precio mínimo (p. ej. "Desde $X"), sin mostrarlo
  como el precio garantizado de toda presentación del producto.
- **FR-014**: Para el caso ya cubierto por spec 066 FR-015 (precio unitario con descuento: regla de
  porcentaje o de precio de paquete con cantidad mínima 1), la tarjeta MUST seguir mostrando el
  precio normal tachado junto al precio vigente, sin cambio de ese comportamiento.
- **FR-015**: Un producto sin ninguna variante cubierta por una regla vigente en el momento actual
  MUST NOT mostrar precio de promoción ni la insignia genérica en su tarjeta, dentro ni fuera de la
  pestaña "Promociones" (spec 066 FR-013, sin cambio).

### Aplicación masiva de una regla a los productos ya seleccionados (bug 3)

- **FR-016**: En la pantalla de configuración de una promoción, al definir una regla de precio
  (presentación, unidades mínimas, precio), si más de un producto ya seleccionado en el Paso 1
  (spec 083 FR-012) tiene una variante cuyo nombre coincide con la presentación elegida, el sistema
  MUST mostrar la lista de esos productos coincidentes con una casilla de verificación por
  producto, todas premarcadas.
- **FR-017**: Al confirmar, el sistema MUST generar una fila de regla independiente solo por cada
  producto que haya quedado marcado, cada una con la variante correspondiente a ese producto — sin
  crear ni requerir un mecanismo de alcance automático por presentación fuera del conjunto de
  productos ya seleccionado (spec 063 FR-003/FR-010, anomalía A-65, sin cambio).
- **FR-018**: Un producto ya seleccionado que no tenga ninguna variante cuyo nombre coincida con la
  presentación elegida en la regla MUST NOT aparecer como opción marcable en esa lista (o MUST
  mostrarse deshabilitado con la razón), y el sistema MUST NOT crearle ninguna fila.
- **FR-019**: Cada fila de regla generada por la aplicación masiva MUST seguir siendo editable o
  eliminable de forma individual (unidades mínimas, precio, o quitarla) sin afectar las demás filas
  generadas por la misma acción, igual que una fila creada manualmente.
- **FR-020**: Las validaciones ya vigentes por fila de regla — mínimo de 2 unidades y precio menor
  a la suma de precios regulares para precio de paquete (spec 083 FR-025/FR-026), y ahorro real
  para porcentaje — MUST seguir aplicando a cada fila generada por la aplicación masiva de forma
  individual, usando el precio regular propio de la variante de cada producto.

### Exclusividad de producto entre promociones vigentes (nuevo, bug 3)

- **FR-021**: Al armar el Paso 1 de una promoción nueva, o editar el de una promoción en estado
  `Borrador` o `Pausada`, el sistema MUST impedir seleccionar un producto si alguna de sus
  variantes ya está cubierta por una regla vigente de **otra** promoción con estado real `Activa`
  — el bloqueo aplica a nivel de todo el producto, sin importar si la variante que se quiere usar
  es la misma u otra distinta a la ya comprometida.
- **FR-022**: El buscador/selector de productos del Paso 1 MUST indicarle al administrador por qué
  un producto no está disponible para seleccionar (p. ej., el nombre de la promoción activa que ya
  lo cubre), en lugar de omitirlo sin explicación.
- **FR-023**: Esta restricción MUST aplicarse también como validación de backend al guardar o
  activar una promoción, para cubrir el caso de que dos promociones que comparten un producto se
  guarden o activen casi al mismo tiempo.
- **FR-024**: Dentro de la **misma** promoción, seleccionar más de una variante del mismo producto
  (p. ej. "Pequeña" y "Grande") sigue permitido sin cambio — la restricción de exclusividad aplica
  solo entre promociones distintas.
- **FR-025**: Esta restricción no es retroactiva: promociones ya activas que hoy comparten un
  producto entre sí (posible bajo el criterio de exclusividad por variante de spec 063 FR-014,
  vigente antes de esta spec) no se ven afectadas hasta que alguna de ellas se edite o se intente
  activar de nuevo.

### Menú de acciones sin desplazar la tabla (bug 5)

- **FR-026**: El menú de acciones (⋮) de una fila, en los listados de promociones y de productos,
  MUST mostrarse completo y visible al abrirse, sin quedar recortado por el borde del contenedor
  con scroll de la tabla ni generar una barra de scroll adicional dentro de ese contenedor.
- **FR-027**: Abrir o cerrar el menú de acciones de una fila MUST NOT cambiar la posición de scroll
  de la tabla en la que vive esa fila.
- **FR-028**: Si el administrador desplaza el scroll de la tabla mientras el menú de acciones de
  una fila está abierto, el sistema MUST cerrar ese menú o reposicionarlo junto a su fila, de modo
  que nunca quede flotando sobre una fila distinta a la que lo abrió.

### Nombre de variante derivado de la presentación (enmienda 2026-09-20, A-79)

- **FR-029**: El formulario de crear o editar producto MUST NOT mostrar un campo "Nombre" para las
  variantes: la tabla de tamaños muestra por fila solo el selector de presentación y el precio (más
  arrastre, número y "Eliminar"). El selector MUST NOT ofrecer la opción "Sin presentación"; una
  fila sin presentación elegida impide guardar y se señala con un mensaje sobre esa fila.
- **FR-030**: El sistema MUST eliminar el nombre como dato propio de la variante: la columna
  `product_variants.name` y su unicidad `(product_id, name)` desaparecen, y el nombre que ve
  cualquier consumidor (formulario, menú QR, carrito, checkout, promociones) es
  siempre el de la presentación asociada, leído en el momento. Toda presentación renombrada se
  refleja de inmediato en todas sus variantes sin ningún `UPDATE` sobre `product_variants`.
- **FR-031**: Renombrar una presentación MUST NOT ejecutar ninguna cascada sobre variantes ni
  validar colisiones contra otras variantes; solo aplica la unicidad de nombre del catálogo de
  presentaciones (spec 083 FR-003). Esto retira la cascada y la guarda de spec 084 FR-004.
- **FR-032**: `product_variants.presentation_id` MUST ser obligatorio (`NOT NULL`) con integridad
  referencial restrictiva: una presentación con variantes asociadas no puede eliminarse
  físicamente. La unicidad `(product_id, presentation_id)` (FR-006) queda como única garantía de
  que un producto no repita un tamaño, incluidas las variantes desactivadas.
- **FR-033**: La migración MUST conservar el nombre visible de toda variante existente: por cada
  variante sin presentación, empareja su nombre exacto con una fila del catálogo (creándola activa
  si no existe, reutilizándola aunque esté desactivada) y la enlaza; las variantes que ya tenían
  presentación no se modifican. La migración MUST ser reversible sin pérdida de datos.
- **FR-034**: La variante de un producto sin tamaños MUST asociarse a la presentación del catálogo
  llamada "Presentación única" (literal de A-74). El sistema MUST crearla si el tenant no la tiene,
  tanto al guardar un producto sin tamaños como al heredar/crear la variante default; un `null` en
  `presentation_id` dentro del payload de guardado MUST interpretarse como "usar la Presentación
  única" (atajo de entrada; el valor guardado y devuelto nunca es nulo).
- **FR-035**: Las variantes que un producto nuevo hereda de las presentaciones de su categoría
  (spec 083 FR-005) MUST crearse ya asociadas a esa presentación (`presentation_id`), sin copiar
  nombres.
- **FR-036**: Todas las respuestas de la API que hoy devuelven el nombre de una variante MUST
  devolver `presentation_id` y `presentation_name` en su lugar, y los payloads de guardado MUST
  dejar de aceptar `name` para variantes. Los recibos y ventas ya emitidos no cambian (guardan su
  descripción como texto al momento de la venta). La aplicación masiva de reglas (User Story 4)
  sigue emparejando la variante de cada producto por la etiqueta de presentación; como el nombre
  de cada presentación es único en el catálogo (spec 083 FR-003), es equivalente a emparejar por
  presentación.

### Presentaciones del Paso 2 de una promoción (enmienda 2026-09-20, A-80)

- **FR-040**: El selector "Presentación / Tamaño" del Paso 2 MUST listar todas las presentaciones
  activas del catálogo (spec 083), ordenadas por nombre, con o sin productos seleccionados en el
  Paso 1, incluidas las que ningún producto usa. Las presentaciones desactivadas no se listan.
  Esto sustituye a spec 083 FR-013 (unión de las variantes de los productos candidatos).
- **FR-041**: Al agregar la regla, el sistema MUST emparejar, por producto ya seleccionado, la
  variante cuya `presentation_id` sea la elegida (no por texto), y aplicar el flujo ya vigente de
  spec 084 FR-016 a FR-019 (casilla por producto, una fila independiente por producto). Un
  producto de una sola variante se empareja por su presentación real; ya no se le asigna la
  etiqueta "Presentación única".
- **FR-042**: La presentación elegida en el Paso 2 MUST conservarse al cambiar la selección de
  productos del Paso 1.
- **FR-043**: Con una presentación elegida, la interfaz MUST indicar a cuántos de los productos
  seleccionados se aplicaría ("Se aplicará a N de M…"), avisar cuando no hay productos
  seleccionados y avisar cuando ninguno la tiene; en ese último caso, agregar la regla se rechaza
  con "Ningún producto seleccionado tiene esa presentación" (mensaje ya existente).
- **FR-044**: La regla guardada sigue siendo una lista explícita de variantes, resuelta en el
  momento de configurar. Una presentación que un producto reciba después NO se agrega sola a una
  promoción ya configurada.

### Reglas por presentación en el Paso 2 (enmienda 2026-09-21, A-81)

- **FR-047**: La lista de reglas del Paso 2 MUST tener **una regla por presentación**, con sus
  unidades mínimas y su precio o porcentaje, sin repetir la presentación por producto. Una
  presentación solo admite una regla por promoción: intentar agregarla de nuevo se rechaza con un
  mensaje que indica quitarla para cambiarla. Esto sustituye a la fila por producto de FR-017 y a
  la casilla de exclusión por producto de FR-017/FR-018 (excluir un producto se hace en el Paso 1).
- **FR-048**: Cambiar la selección de productos del Paso 1 (agregar o quitar) MUST NOT exigir
  quitar ni rehacer reglas: las reglas de presentación se mantienen y se aplican a las variantes
  de los productos seleccionados que tengan esa presentación.
- **FR-049**: Al guardar, cada regla de presentación MUST expandirse a una regla de backend por
  cada producto seleccionado que tenga esa presentación, con solo la variante de ese producto
  (spec 084 FR-017/FR-019, A-77: un paquete nunca mezcla productos). El backend, su modelo y el
  motor de descuentos no cambian.
- **FR-050**: Se MUST poder agregar una regla de presentación aunque aún no haya productos
  seleccionados o ninguno la tenga; la lista la marca como «sin productos seleccionados con esta
  presentación», la interfaz avisa que no se guardará mientras siga así, y no genera reglas de
  backend. El formulario solo es válido si al menos una regla alcanza algún producto.
- **FR-051**: Al abrir una promoción guardada, el sistema MUST reconstruir la lista por
  presentación agrupando las reglas guardadas de igual presentación, valor y unidades, y recuperar
  la selección de productos de las variantes de esas reglas. Las variantes que ya no aparezcan en
  el menú (p. ej. de un producto inactivo) no se muestran y se descartan al guardar.
- **FR-052**: Los cambios de la selección de productos y de la lista de reglas MUST contar como
  cambios para habilitar «Guardar y sincronizar» (FR-046).

- **FR-053**: El anuncio de la promoción en el menú QR (`GET /menu/promotions`) MUST mostrar **una
  sola línea por presentación**: las reglas de backend de una misma promoción que producen el mismo
  texto (una por producto, por FR-049) se fusionan en una, y `variant_count` suma las variantes de
  las reglas fusionadas. Reglas con texto distinto no se fusionan.

- **FR-054** (A-82): Cuando el conjunto de una regla se nombra con **un solo nombre**, el texto de
  condición MUST leerse «Llevando {nombre} x {n} pagas {precio}» (paquete) o «{p}% llevando
  {nombre} x {n}» (porcentaje), p. ej. «Llevando 8 onzas x 2 pagas $12.000»; backend y réplica del
  frontend MUST coincidir. Los conjuntos con varios nombres y los textos de cantidad mínima 1 no
  cambian.
- **FR-055** (A-82): La tarjeta de producto de la pestaña «Promociones» del menú QR MUST mostrar
  solo la condición corta de la regla más barata (p. ej. «Desde 2 x $12.000»), sin el equivalente
  por unidad («· $6.000 c/u»). Esto ajusta spec 084 FR-012/FR-013; el modal de producto conserva el
  equivalente por unidad.

### Duplicar una promoción con un nombre ya usado (enmienda 2026-09-21, A-83)

- **FR-056**: Al duplicar una promoción con un nombre que ya usa otra, el sistema MUST avisar en el
  diálogo que la existente se eliminará con todas sus reglas y ofrecer «Reemplazar y duplicar»; al
  confirmar, MUST eliminarla y crear la copia (en Borrador, con las reglas y la vigencia de la
  fuente) con ese nombre, de forma atómica. Sin confirmar (`replace_existing` ausente), un nombre
  repetido sigue siendo un 409.
- **FR-057**: Una promoción `Activa` MUST NOT reemplazarse (409: hay que pausarla antes). Duplicar
  con el mismo nombre de la fuente la reemplaza por su copia. La promoción eliminada MUST quedar
  registrada en auditoría.

### Guardado de la configuración de una promoción (enmienda 2026-09-20)

- **FR-045**: En la pantalla de configuración de una promoción, el botón "Guardar y sincronizar"
  MUST estar al final del formulario (después de las reglas y de los mensajes de conflicto), no en
  la cabecera. No se muestra en una promoción `Finalizada` (solo lectura).
- **FR-046**: El botón MUST permanecer deshabilitado mientras el formulario no difiera de como se
  abrió la configuración (nombre, fechas, días, horas y reglas; el orden de días o de variantes no
  cuenta como cambio) y mientras no sea válido; con cualquier cambio válido se habilita, y deshacer
  el cambio lo vuelve a deshabilitar. Sin cambios, un texto discreto lo explica ("No hay cambios
  por guardar").

### Orden de la tarjeta "Tamaños del producto" (enmienda 2026-09-20)

- **FR-037**: Con el interruptor de tamaños encendido, la tabla de tamaños MUST renderizarse justo
  debajo del encabezado de la tarjeta "Tamaños del producto" (título, descripción e interruptor),
  antes de cualquier otro bloque.
- **FR-038**: El interruptor "Maneja inventario" (con su descripción y el aviso de "no podrá
  venderse…") MUST renderizarse después de la tabla de tamaños y de la lista de presentaciones
  desactivadas, y antes del detalle del tamaño activo. Con tamaños apagados, queda justo debajo
  del encabezado.
- **FR-039**: El orden de FR-037/FR-038 no cambia qué se habilita con cada interruptor: la tabla de
  tamaños (presentación y precio) está disponible con o sin "Maneja inventario"; solo el bloque de
  insumos fijos y la parte de inventario de "Sabores a elegir" dependen de él (spec 027/064, sin
  cambio).

### Key Entities

- **Presentación**: entidad de catálogo ya existente desde spec 083, sin cambio propio; esta spec
  la conecta por primera vez con `ProductVariant` mediante una relación real (FR-001 a FR-006).
- **Variante de producto (`ProductVariant`)**: **cambia su modelo** respecto a spec 083 (que la
  declaraba "sin cambio"): referencia **obligatoria** a una Presentación del catálogo del mismo
  tenant y **sin nombre propio** (enmienda 2026-09-20, FR-029 a FR-036) — su nombre es el de la
  presentación. Dos variantes del mismo producto no pueden compartir la misma presentación
  (FR-006). Una versión intermedia de esta spec (2026-09-17) la dejó con referencia opcional y
  nombre sincronizado; esa versión queda sustituida.
- **Asociación Categoría↔Presentación**: entidad ya existente desde spec 083, sin cambio; esta spec
  no la modifica ni depende de ella.
- **Promoción / Regla / conjunto de variantes**: entidades ya existentes desde spec 063, sin cambio
  de modelo; esta spec agrega una restricción de UI (bloqueo del botón "Configurar" en `Activa`),
  un atajo de captura (aplicación masiva de una regla a varios productos ya seleccionados, con
  exclusión por casilla), y una nueva regla de exclusividad de producto entre promociones vigentes
  distintas — sin tocar el alcance por variante explícita ni el motor de cálculo de descuentos.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las variantes de producto a las que el administrador asocia una
  presentación del catálogo muestran el nombre de esa presentación, incluso después de que el
  administrador la renombre desde el catálogo.
- **SC-002**: El 0% de las promociones con estado real `Activa` permite abrir su pantalla de
  configuración desde el listado sin pausarla primero.
- **SC-003**: El 100% de las tarjetas de producto visibles en la pestaña "Promociones" del menú QR
  muestran un precio de promoción (paquete, mínimo entre reglas, o precio con descuento según
  corresponda), no solo la insignia genérica.
- **SC-004**: Configurar una regla de precio para una presentación compartida por 5 productos ya
  seleccionados toma una sola acción del administrador (definir la regla una vez y confirmar la
  lista), en lugar de 5 repeticiones manuales.
- **SC-005**: El 100% de los intentos de seleccionar, en una promoción nueva o en edición, un
  producto que ya hace parte de otra promoción vigente son rechazados con una explicación, tanto
  desde la interfaz como desde el backend.
- **SC-006**: El 100% de las aperturas del menú de acciones de una fila, en cualquier posición del
  listado (incluidas las filas cercanas al borde del contenedor con scroll), muestran el menú
  completo sin recortarlo ni mover la posición de scroll de la tabla.

- **SC-007**: El 0% de las variantes existentes o nuevas queda sin presentación asociada tras la
  migración de US7, y el 100% de los textos que hoy muestran el nombre de una variante (menú QR,
  carrito de mesa, checkout, selector de producto, reglas de promoción) muestra el mismo texto que
  mostraba antes de la migración.
- **SC-008**: Renombrar una presentación desde el catálogo se refleja en el 100% de los productos
  que la usan sin ejecutar ninguna escritura sobre `product_variants`.
- **SC-009**: En el formulario de producto con tamaños, la tabla de tamaños es el primer bloque
  bajo el encabezado de la tarjeta y "Maneja inventario" nunca aparece por encima de ella.
- **SC-011**: Con las reglas «8 onzas × 2 = $12.000» y «12 onzas × 2 = $17.000» definidas una vez, el
  administrador puede cambiar los productos del Paso 1 cualquier número de veces sin quitar ni
  volver a crear ninguna regla, y la lista nunca muestra más de una regla por presentación.
- **SC-010**: Con el Paso 1 vacío, el selector del Paso 2 muestra el 100% de las presentaciones
  activas del catálogo; con productos seleccionados, la lista es la misma.

## Assumptions

- Estos cinco bugs se corrigen sobre el comportamiento ya especificado en specs 063, 066, 081 y
  083. La única excepción de modelo de datos es la nueva referencia opcional de `ProductVariant` a
  `Presentación` (bug 2); el resto de los cambios son de interfaz, flujo de captura y validación,
  sin tocar el motor de cálculo de descuentos.
- Los cambios de comportamiento acordados en esta sesión de Clarifications (bug 2: relación
  variante↔presentación, nueva en el modelo; bug 3: flujo de captura masivo con exclusión por
  casilla, y la nueva regla de exclusividad de producto entre promociones vigentes; bug 4: bloqueo
  total de "Configurar" para `Activa`, en lugar de la edición parcial de spec 083 FR-018) son
  decisiones de negocio que, según el Principio II de la constitución del proyecto, MUST quedar
  registradas como anomalías (mismo tratamiento que A-74/A-75 de spec 083) antes de implementarse.
- El bug 5 (menú de acciones y scroll de tabla) aplica, como mínimo, al listado de promociones y al
  listado de productos; si existieran otros listados con el mismo patrón de menú de acciones por
  fila, se corrigen con el mismo criterio (FR-026 a FR-028) al encontrarlos, sin que eso amplíe el
  alcance de esta spec a listados no reportados.
- "Precio mínimo" en la tarjeta de la pestaña "Promociones" (bug 1) se refiere al precio
  equivalente por unidad más bajo (mismo cálculo que spec 066 FR-008) entre las reglas vigentes que
  cubren alguna variante del producto en ese instante — no al precio total de un paquete comparado
  directamente contra el precio de otro con distinta cantidad mínima, y no a un precio histórico ni
  proyectado a futuro. Gracias a la nueva exclusividad de producto (bug 3) y a que spec 083 FR-010
  ya fija un único tipo de descuento por promoción, esas reglas simultáneas siempre pertenecen a la
  misma promoción y al mismo tipo de descuento.
- No se requiere migrar datos ni tocar promociones o reglas ya guardadas: los cambios de esta spec
  son de interfaz, flujo de captura y una relación nueva y opcional en `ProductVariant`, no
  retroactivos (Principio VII de la constitución).
- **Enmienda 2026-09-20**: la única excepción de modelo de datos que declaraba esta sección (una
  referencia *opcional* de `ProductVariant` a `Presentación`) queda ampliada por A-79: la
  referencia pasa a ser obligatoria, la columna `name` se elimina y la migración sí toca datos
  existentes (paso de datos de FR-033), a diferencia de lo que afirma la viñeta anterior sobre "no
  migrar datos". Sigue sin tocarse ninguna promoción, regla ni el motor de cálculo. El cambio de
  contrato de la API (FR-036) no es retrocompatible; se despliega en tres pasos (research.md D10).
