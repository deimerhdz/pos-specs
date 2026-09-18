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
3. **Given** el administrador no quiere asociar una variante a ninguna presentación del catálogo,
   **When** deja el selector en "Sin presentación", **Then** puede seguir escribiendo el nombre de
   esa variante libremente, igual que hoy.
4. **Given** una presentación del catálogo con una o más variantes de producto ya asociadas a ella,
   **When** el administrador la renombra desde la sección "Presentaciones", **Then** el nombre de
   cada variante asociada se actualiza para reflejar el nuevo nombre, sin que el administrador
   tenga que editar cada producto uno por uno.
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

### Edge Cases

- ¿Qué pasa si el administrador cambia las presentaciones asociadas a la categoría después de
  crear el producto? No se modifican retroactivamente las variantes ya creadas (spec 083 FR-008,
  sin cambio) — esta spec no toca ese mecanismo de herencia por categoría.
- ¿Qué pasa si renombrar una presentación (FR-004) haría que una de sus variantes asociadas quedara
  con el mismo nombre que otra variante sin presentación del mismo producto? El sistema rechaza el
  renombre completo (ninguna variante se actualiza) y le explica al administrador cuál producto y
  variante están en conflicto — el administrador debe renombrar o asociar esa otra variante primero
  antes de poder completar el renombre de la presentación.
- ¿Qué pasa si una promoción `Activa` tiene su vigencia (fecha/hora) ya vencida (badge "Vencida")
  pero su `status` sigue siendo `Activa` porque nadie la pausó ni finalizó? El botón "Configurar"
  sigue deshabilitado (User Story 2, escenario 4) — el administrador debe pausarla o esperar a que
  el sistema la marque `Finalizada` según las reglas ya vigentes de spec 063, no hay atajo nuevo.
- ¿Qué pasa si, al aplicar una regla de forma masiva (User Story 4), el administrador desmarca
  todos los productos coincidentes? El sistema no crea ninguna fila y deja la regla sin confirmar,
  en lugar de crear una fila vacía o sin variante.
- ¿Qué pasa si el menú de acciones de una fila (User Story 6) está abierto y el administrador
  redimensiona la ventana o rota el dispositivo? El menú se cierra, igual que ante cualquier cambio
  de layout que invalide su posición calculada.
- ¿Qué pasa si el administrador quiere mover un producto de una promoción activa a otra? Debe
  pausar la promoción activa que lo cubre primero (User Story 2); solo entonces ese producto queda
  disponible para seleccionarse en otra promoción, porque una promoción `Pausada` deja de contar
  para la exclusividad de User Story 5.
- ¿Qué pasa si, en la pestaña "Promociones" del menú QR, dos reglas vigentes de la misma promoción
  cubren distintas variantes del mismo producto? La tarjeta (User Story 3, escenario 2) muestra el
  precio equivalente por unidad más bajo entre ellas, identificado como precio mínimo.

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

### Key Entities

- **Presentación**: entidad de catálogo ya existente desde spec 083, sin cambio propio; esta spec
  la conecta por primera vez con `ProductVariant` mediante una relación real (FR-001 a FR-006).
- **Variante de producto (`ProductVariant`)**: **extiende su modelo** respecto a spec 083 (que la
  declaraba "sin cambio"): gana una referencia opcional a una Presentación del catálogo del mismo
  tenant. Cuando esa referencia está presente, el nombre de la variante se deriva del nombre de la
  presentación (FR-002/FR-003/FR-004); cuando no está presente, el nombre sigue siendo libre, igual
  que hoy. Dos variantes del mismo producto no pueden compartir la misma presentación (FR-006).
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
