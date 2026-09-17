# Feature Specification: Catálogo de Presentaciones y Rediseño de Promociones

**Feature Branch**: `083-presentaciones-y-promociones`

**Created**: 2026-09-16

**Status**: Draft

**Input**: User description: rediseñar el módulo de Promociones (UI y lógica de negocio) y crear
un catálogo central de presentaciones de producto (funcionalidad inexistente hoy), para que las
presentaciones se administren una sola vez, se asocien a Categorías de Producto, y todo producto
creado dentro de una categoría herede automáticamente sus presentaciones (o reciba
"Presentación única" si la categoría no maneja variaciones). El formulario de promociones se
rediseña siguiendo al 100% los prototipos `.html` en `/Escritorio/promociones`
(`listado-promociones.html`, `formulario-crear-promocion.html`, `configurar-promocion.html`), con
un flujo guiado: definir vigencia y luego buscar productos, elegir su presentación y configurar
reglas de precio por unidades.

## Contexto y relación con specs anteriores

La spec [063](../063-promociones-por-variante/spec.md) (en producción desde 2026-09-01)
**eliminó por completo** la entidad de catálogo `Presentation` que existía antes (spec
[040](../040-promociones-precio-por-presentacion/spec.md)), precisamente porque un catálogo
compartido implicaba precio uniforme por presentación entre productos distintos — algo falso en
el catálogo real del tenant (anomalías **A-63** y **A-65** en
[`registro-de-anomalias.md`](../000-reconocimiento/registro-de-anomalias.md)). Desde entonces, el
alcance de cada regla de promoción es siempre una lista explícita de variantes de producto
(`ProductVariant`) elegidas a mano; no existe alcance automático por presentación (spec 063
FR-003, FR-010).

Esta spec **reintroduce el nombre "Presentación"**, pero como un catálogo de **etiquetas
reutilizables** para estandarizar el nombre de las variantes y agilizar su creación por
categoría — **no** como un mecanismo de alcance de promoción. El modelo `Promoción`/`Regla`/
conjunto de variantes y las reglas de vigencia y solapamiento de la spec 063 **no cambian**: cada
regla de promoción sigue guardando una lista explícita de variantes, igual que hoy. El motor de
cálculo de descuentos tampoco cambia, **salvo** las validaciones de unidades mínimas y precio de
las reglas de precio de paquete, y la exclusión de toppings del descuento por porcentaje,
agregadas en la sesión de Clarifications 2026-09-17 (FR-025 a FR-027). Ver sección
"Clarifications" para el detalle de esta decisión.

## Clarifications

### Session 2026-09-16

- Q: ¿Cómo debe convivir el nuevo catálogo de Presentaciones con la regla de la spec 063 que
  prohíbe el alcance automático de una promoción por presentación (FR-003/FR-010)? → A:
  **Presentación es solo un catálogo de etiquetas/plantillas.** Sirve para estandarizar el nombre
  de las variantes y para ayudar a poblar el selector de variantes de cada regla de promoción; el
  alcance de cada regla sigue siendo una lista explícita de variantes elegidas a mano, igual que
  hoy. No se reabren ni se revierten FR-003/FR-010 de la spec 063 ni la anomalía A-65.
- Q: ¿La "Presentación única" que exige esta spec es un concepto nuevo o el mismo mecanismo de la
  variante `"Single"` que ya se crea automáticamente hoy cuando un producto no maneja variaciones
  (`RN-CAT-05`, specs 002/043)? → A: **Es un renombre en la interfaz** del mismo mecanismo ya
  existente; no cambia el comportamiento de creación automática ni el precio inicial de esa
  variante.
- Q: ¿Cómo se relaciona esta spec con la spec [081](../081-tab-promociones-menu-qr/spec.md)
  (pestaña "Promociones" del menú QR), ya redactada pero aún sin implementar, que asume el motor
  Promoción/Regla/conjunto de variantes vigente? → A: **Compatible, sin tocarla.** La spec 081
  sigue su curso sobre el mismo motor de cálculo; esta spec solo cambia **cómo** se configura una
  promoción (catálogo de presentaciones + pantallas nuevas), nunca **qué** se calcula ni cómo se
  consume en el menú QR.
- Q: La numeración automática de carpetas en `specs/` toma el siguiente número secuencial
  disponible (`083`, porque la última carpeta existente es `082`), distinto del `090` usado en el
  ejemplo de rama de implementación que el equipo ya definió (`feat/090-promotions`). ¿Cómo se
  numera? → A: **Carpeta secuencial `083-presentaciones-y-promociones`**; la rama de
  implementación en `pos-backend`/`pos-heladeria` se llama `090-...`, según la convención que el
  equipo ya fijó — la carpeta del spec y el nombre de rama son independientes por diseño de este
  flujo.
- Q: Al configurar una regla de promoción para un producto cuyas variantes no coinciden con
  ninguna presentación del catálogo (producto anterior a esta funcionalidad, o con una variante
  renombrada a mano), ¿qué variantes debe poder elegir el selector de la regla? (FR-013) → A:
  **El selector siempre muestra las variantes reales del producto elegido**, usando el nombre de
  la presentación del catálogo como etiqueta cuando coincide con una presentación activa asociada
  a la categoría del producto (o "Presentación única" si no maneja variaciones), y el nombre
  propio de la variante cuando no coincide con ninguna; ninguna variante queda excluida del
  selector.
- Q: ¿El catálogo de Presentaciones debe migrarse/sembrarse automáticamente a partir de los
  nombres de variante ya existentes en cada tenant, o arranca vacío y el administrador lo puebla a
  mano? → A: **Se siembra automáticamente al desplegar**: el sistema crea una presentación activa
  por cada nombre de variante distinto (comparación exacta de texto) ya existente en el tenant.
  Esa siembra solo crea entradas del catálogo; no crea asociaciones Categoría↔Presentación (FR-004
  sigue siendo manual) ni modifica productos o variantes existentes.
- Q: ¿El administrador puede reactivar una presentación que había desactivado previamente, o la
  desactivación es de un solo sentido? → A: **Sí, se puede reactivar** desde el mismo listado
  (toggle activa/inactiva), igual que otros catálogos del panel (categorías, productos).
- Q: ¿La unicidad de nombre (FR-003) también aplica al editar/renombrar una presentación existente,
  o solo al crearla? → A: **También aplica al editar.** Renombrar una presentación a un nombre que
  ya usa otra presentación del mismo tenant se bloquea igual que en la creación.
- Q: Cuando el administrador presiona "Continuar" en la pantalla de creación (con nombre y tipo,
  sin ninguna regla de precio todavía), ¿el sistema debe crear la promoción en la base de datos de
  inmediato con una lista de reglas vacía, o esperar a que se agregue la primera regla en la
  pantalla de configuración? → A: **Crear de inmediato al presionar "Continuar"**, con `status`
  `Borrador` y una lista de reglas vacía. Esto exige relajar la validación del backend
  `PromotionCreate.rules` (de `min_length=1` a `min_length=0`, **solo** cuando el `status` de
  creación es `draft`) — la validación de que una promoción necesita al menos una regla (y cada
  regla al menos una variante) para poder **activarse** ya existe hoy en `change_status` y no
  cambia.
- Q: Para las pestañas de filtro del listado ("Todas"/"Borradores"/"Activas"/"En pausa"/
  "Finalizadas") y para habilitar el botón "Eliminar" de cada fila, ¿debe usarse el estado real
  guardado en base de datos (`status`: draft/active/paused/finished) o el estado visual derivado
  que ya muestra el badge de cada fila (incluye "Vencida"/"Fuera de horario"/"Programada", variantes
  de `status=active` según fecha/hora)? → A: **Estado real (`status`) para ambos casos**, sin
  cambios de backend. Las pestañas ya filtran correctamente por `status` (no había bug de consulta);
  la percepción de "datos desalineados" es la diferencia esperada entre `status=active` y un badge
  como "Vencida" cuando la ventana de vigencia ya pasó — eso no cambia. "Eliminar" queda habilitado
  siempre que `status !== 'active'` (Borrador, En pausa, Finalizada); solo una promoción `Activa`
  (por estado real, sin importar lo que diga el badge) bloquea la eliminación desde el listado.
- Q: El grid de tarjetas de producto del "Paso 1" en `configurar-promocion.html` no define ningún
  `max-height`/`overflow` (solo muestra 4 tarjetas de ejemplo). Cuando el tenant tiene muchos
  productos que coinciden con la búsqueda/filtro, ¿qué altura debe tener el contenedor con scroll
  vertical que evite que el listado rompa la estructura de la página? → A: **Altura fija de 320px**
  con `overflow-y: auto`, reutilizando la clase `.custom-scrollbar` ya definida en el prototipo
  (mismo estilo de scrollbar que el resto de la pantalla) para mantener consistencia visual; el
  buscador y el filtro por categoría de FR-012 permanecen siempre visibles por encima del
  contenedor, sin scrollear junto con las tarjetas.

### Session 2026-09-17

- Q: Las nuevas reglas de validación de negocio para "Promoción por Paquete" y "Promoción por
  Porcentaje" (unidades mínimas, precio vs. suma regular, exclusión de toppings del descuento por
  porcentaje) ¿se integran en esta spec (083) o se registran como una spec nueva independiente? →
  A: **Se integran en esta spec (083)**, ya que 083 ya rediseña exactamente las pantallas de
  creación/configuración de estos dos tipos de promoción donde aplican estas reglas.
- Q: La spec 063 (vigente) permite hoy una regla de precio de paquete con `minQuantity = 1`
  ("precio unitario especial", ej. "Litro sin licor a $17.000 los lunes"). La nueva Regla 1 exige
  `unidades >= 2` siempre en promociones por paquete. ¿Qué pasa con ese caso? → A: **Se elimina el
  patrón `minQuantity = 1` para precio de paquete.** El mínimo de 2 unidades aplica sin excepción a
  toda regla nueva o editada de este tipo; si el negocio necesita un precio especial de una sola
  unidad, debe usar el tipo porcentaje en su lugar. No es retroactivo (ver clarificación de
  reglas existentes, más abajo).
- Q: En el nuevo flujo, cada fila de regla se arma con UN producto + UNA presentación + unidades
  mínimas + precio (FR-013). ¿La Regla 2 (suma de precios regulares > precio promocional) aplica
  sobre esa única variante repetida `unidades` veces, o el paquete puede combinar variantes
  distintas en una misma fila/regla? → A: **Una sola variante × N unidades.** La suma de precios
  regulares de una regla es `precio_unitario_regular × unidades`; esto es exactamente lo que ya
  bloquea spec 063 FR-016 para una regla de una sola variante, así que Regla 2 no exige un motor de
  cálculo nuevo — solo se refuerza el mensaje de error y se agrega el mínimo de 2 unidades
  (Regla 1). No se soporta combinar productos distintos dentro de una misma fila/regla.
- Q: ¿"Toppings" (Regla 3) se refiere a los grupos de opciones/adicionales de specs 064/065? ¿El
  cambio exige persistir un desglose base/toppings del descuento, dado que spec 063 FR-021 deja
  el desglose por línea fuera de alcance? → A: **Sí, son los `OptionGroup` de specs 064/065, y solo
  cambia el TOTAL calculado de la línea.** Cuando el tipo de promoción es porcentaje, el descuento
  se aplica solo sobre el precio base de la variante; el precio de los toppings elegidos se suma
  íntegro (sin descuento) al total. No se requiere extender la persistencia de Sale/SaleInvoice/
  CustomerOrder con un desglose base/toppings — FR-021 de spec 063 sigue sin cambios, el monto de
  descuento agregado ya refleja el efecto neto.
- Q: Al desplegar estas reglas más estrictas, ¿qué pasa con promociones/reglas ya guardadas
  (algunas quizá `Activa`) que las incumplen (`minQuantity=1` en paquete, o precio ≥ suma regular)?
  → A: **Se conservan tal cual, sin cambio retroactivo.** Las reglas ya guardadas siguen
  funcionando y cobrándose sin alteración; las nuevas validaciones (FR-025, FR-026) solo aplican al
  crear una regla nueva o al guardar una edición sobre una regla existente — si la edición no
  cumple el nuevo mínimo o la nueva comprobación de precio, el guardado se bloquea, igual que
  cualquier regla nueva.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Administrar el catálogo global de presentaciones (Priority: P1)

El administrador entra a una sección nueva del panel ("Presentaciones", dentro de Catálogo) y
crea las presentaciones que su negocio necesita reutilizar (p. ej. Pequeño, Mediano, Grande, 8oz,
16oz, 24oz). Puede listarlas, editar su nombre, y desactivar las que ya no usa.

**Why this priority**: es la funcionalidad que hoy no existe y de la que dependen las otras dos
historias — sin un catálogo que administrar, no hay nada que asociar a una categoría ni que
ofrecer como ayuda al configurar una promoción.

**Independent Test**: crear tres presentaciones nuevas ("Pequeño", "Mediano", "Grande"), verlas
en el listado, editar el nombre de una, desactivar otra, y comprobar que la desactivada deja de
ofrecerse al asociar presentaciones a una categoría.

**Acceptance Scenarios**:

1. **Given** el administrador en la sección "Presentaciones", **When** crea una presentación con
   nombre "Pequeño", **Then** queda disponible en el listado con estado activo.
2. **Given** una presentación existente, **When** el administrador edita su nombre, **Then** el
   cambio se refleja en el listado y en cualquier lugar donde se muestre esa presentación.
3. **Given** el administrador intenta crear, o renombrar una existente hacia, un nombre que ya
   existe en el catálogo del tenant, **When** guarda, **Then** el sistema lo rechaza y explica que
   el nombre ya existe.
4. **Given** una presentación sin ningún producto ni promoción que la referencie, **When** el
   administrador la desactiva, **Then** deja de aparecer como opción al asociar presentaciones a
   una categoría, pero sigue visible en el listado con su estado.
5. **Given** una presentación previamente desactivada, **When** el administrador la reactiva desde
   el listado, **Then** vuelve a ofrecerse como opción al asociar presentaciones a una categoría.

---

### User Story 2 - Asociar presentaciones a una categoría y heredarlas al crear un producto (Priority: P2)

El administrador edita una categoría (p. ej. "Granizados") y selecciona, del catálogo global, las
presentaciones que le corresponden (8oz, 16oz, 24oz). Al crear un producto nuevo dentro de esa
categoría, el sistema le crea automáticamente una variante por cada presentación asociada, lista
para que el administrador solo complete el precio. Si la categoría no tiene presentaciones
asociadas, el producto nace con una única variante "Presentación única".

**Why this priority**: es lo que hace útil al catálogo del Historia 1 — sin esta asociación, el
administrador seguiría creando cada variante a mano, nombre por nombre, en cada producto.

**Independent Test**: asociar "Pequeña", "Mediana" y "Grande" a la categoría "Ensaladas"; crear un
producto nuevo dentro de esa categoría y verificar que nace con esas tres variantes ya creadas (a
precio $0, pendientes de completar). Crear otro producto en una categoría sin presentaciones
asociadas y verificar que nace con una única variante "Presentación única".

**Acceptance Scenarios**:

1. **Given** el administrador editando la categoría "Granizados", **When** selecciona "8oz",
   "16oz" y "24oz" del catálogo de presentaciones, **Then** esa asociación queda guardada para la
   categoría.
2. **Given** la categoría "Granizados" con esas tres presentaciones asociadas, **When** el
   administrador crea el producto "Granizado de Mora" dentro de ella, **Then** el producto nace
   con tres variantes ("8oz", "16oz", "24oz"), cada una lista para que se le asigne su precio.
3. **Given** una categoría sin ninguna presentación asociada, **When** el administrador crea un
   producto dentro de ella, **Then** el producto nace con una única variante "Presentación única".
4. **Given** un producto ya creado con las variantes heredadas de su categoría, **When** el
   administrador cambia después las presentaciones asociadas a esa categoría, **Then** las
   variantes del producto ya existente no se modifican retroactivamente; el cambio solo aplica a
   productos creados de ahí en adelante.
5. **Given** un producto recién creado con sus variantes heredadas, **When** el administrador
   necesita agregar, renombrar o quitar una variante manualmente, **Then** el sistema se lo
   permite igual que hoy, sin restringir la edición a lo heredado de la categoría.

---

### User Story 3 - Configurar una promoción con vigencia y reglas de precio por presentación (Priority: P3)

El administrador entra al listado de promociones (rediseñado según `listado-promociones.html`),
crea una promoción nueva indicando su nombre y su tipo (descuento por porcentaje o precio de
paquete, según `formulario-crear-promocion.html`), y pasa a la pantalla de configuración
(`configurar-promocion.html`): define la vigencia (días de la semana, fecha de inicio y fin, hora
de inicio y fin), busca y selecciona los productos participantes, y por cada presentación
aplicable define las unidades mínimas requeridas y el precio promocional total, agregando cada
combinación como una fila de regla de precio. Puede agregar varias filas (distintas
presentaciones, cada una con su propia condición) antes de guardar.

**Why this priority**: es el objetivo de negocio explícito de esta spec (rediseño del módulo de
promociones) y el que le da uso final al catálogo de presentaciones; depende de que existan
presentaciones (Historia 1) y, opcionalmente, de que estén asociadas a categorías (Historia 2),
pero puede probarse con productos que ya tengan sus variantes creadas.

**Independent Test**: crear la promoción "2X Granizados 8oz" del ejemplo de negocio: tipo precio
de paquete, vigencia lunes a miércoles de 14:00 a 18:00, producto "Granizado de Mora", presentación
"8oz", 2 unidades, $12.000; guardarla y verificar que aparece en el listado con su vigencia y su
regla resumidas.

**Acceptance Scenarios**:

1. **Given** el administrador en el listado de promociones, **When** presiona "Nueva promoción",
   **Then** ve la pantalla de creación con campo de nombre y selección de tipo (porcentaje o
   precio de paquete), igual que `formulario-crear-promocion.html`.
2. **Given** el administrador guardó el nombre y el tipo, **When** continúa a la pantalla de
   configuración, **Then** ve el bloque de vigencia (días, fecha inicio/fin, hora desde/hasta) y,
   debajo, los pasos para elegir productos y configurar la regla de precio, igual que
   `configurar-promocion.html`.
3. **Given** el administrador definió la vigencia lunes a miércoles, 14:00 a 18:00, **When**
   busca y selecciona el producto "Granizado de Mora", elige la presentación "8oz", indica 2
   unidades mínimas y $12.000 de precio promocional, y presiona "Agregar a la lista", **Then** esa
   combinación aparece como una fila en la tabla de reglas configuradas, con el precio regular
   tachado y el ahorro calculado.
4. **Given** una promoción con una fila de regla ya agregada, **When** el administrador agrega una
   segunda fila con otra presentación (p. ej. "16oz") y su propio precio, **Then** ambas filas
   quedan configuradas en la misma promoción, cada una con su propia condición.
5. **Given** el administrador intenta guardar una regla cuyo precio promocional no representa un
   ahorro frente al precio regular de la presentación elegida, **When** confirma, **Then** el
   sistema lo bloquea y explica por qué (mismo criterio ya vigente, spec 063 FR-016).
6. **Given** una promoción ya `Activa`, **When** el administrador entra a su pantalla de
   configuración, **Then** ve el tipo de la promoción marcado como fijado en creación y no puede
   modificar el tipo, el valor, las unidades mínimas ni el conjunto de variantes de sus reglas ya
   guardadas (mismo bloqueo ya vigente, spec 063 FR-018), aunque sí puede editar nombre, fin de
   vigencia, días y horas (sin campo de descripción, FR-023).
7. **Given** el listado de promociones, **When** el administrador consulta cualquier promoción
   existente, **Then** ve su nombre, el tipo de promoción en la columna "Reglas" (FR-017), su
   vigencia en lenguaje llano, su estado y las mismas acciones ya disponibles hoy (configurar,
   duplicar, pausar/activar, eliminar — "Eliminar" habilitado salvo que la promoción esté `Activa`,
   FR-021), con el diseño de `listado-promociones.html`.
8. **Given** el administrador en la pantalla de creación, **When** ingresa el nombre, elige el tipo
   y presiona "Continuar", **Then** la promoción queda creada de inmediato en base de datos con
   estado `Borrador` y sin reglas todavía (FR-020), antes de llegar a la pantalla de configuración.

---

### Edge Cases

- **Eliminar o desactivar una presentación vinculada**: si una presentación del catálogo ya está
  referenciada por al menos una variante de producto o por una regla de promoción, el sistema
  bloquea su eliminación física; el administrador solo puede desactivarla, y la desactivación no
  afecta las asociaciones ya existentes (FR-002).
- **Categoría sin presentaciones asociadas**: todo producto creado dentro de ella recibe
  automáticamente "Presentación única" (FR-006), nunca queda sin ninguna variante.
- **Cambiar las presentaciones de una categoría con productos ya creados**: no altera
  retroactivamente las variantes de esos productos (FR-008); solo rige para productos creados
  después del cambio.
- **Solapamiento de vigencia y reglas de precio sobre la misma presentación de un mismo producto
  en promociones distintas**: se reutiliza sin cambios el bloqueo por solapamiento ya vigente
  (spec 063 FR-014/FR-014a) — dos reglas que comparten al menos una variante solo pueden coexistir
  si sus ventanas de fecha, día y hora no se intersectan.
- **Nombre de presentación duplicado dentro del mismo tenant**: se bloquea tanto al crear como al
  editar/renombrar (FR-003).
- **Búsqueda de productos con muchos resultados en el Paso 1 de configuración**: el grid de
  tarjetas de producto se muestra dentro de un contenedor de 320px de alto con scroll vertical
  propio; el buscador y el filtro por categoría permanecen fijos por encima, sin scrollear (FR-024).
- **Variante de producto sin presentación equivalente en el catálogo**: al configurar una regla de
  promoción, el selector igual la ofrece (mostrando el nombre propio de la variante en vez de una
  etiqueta de presentación); ninguna variante real queda inseleccionable por no coincidir con el
  catálogo (FR-013).
- **Editar una regla de precio de paquete ya guardada con `minQuantity = 1` o que incumple la nueva
  comprobación de precio**: la regla sigue activa y cobrándose sin cambios hasta que el
  administrador intente guardar una edición sobre ella; en ese momento se le exige cumplir el
  mínimo de 2 unidades y que el precio promocional sea menor a la suma de precios regulares
  (FR-025, FR-026), igual que a una regla nueva; si no se ajusta, el guardado de la edición se
  bloquea pero la regla activa no se ve afectada.
- **Promoción de porcentaje sobre un producto con toppings/adicionales seleccionados**: el
  descuento se calcula solo sobre el precio base de la variante; el precio de los toppings
  elegidos se suma íntegro, sin descuento, al total de la línea (FR-027).

## Requirements *(mandatory)*

### Catálogo de Presentaciones (nuevo)

- **FR-001**: El sistema MUST proveer una sección de administración donde el administrador pueda
  crear, listar, editar, desactivar y reactivar presentaciones del catálogo global de su tenant.
  Cada presentación MUST tener al menos un nombre y un estado (activa/inactiva), y ese estado MUST
  poder alternarse en ambos sentidos desde el listado.
- **FR-002**: El sistema MUST impedir la eliminación física de una presentación referenciada por
  al menos una variante de producto o por una regla de promoción; en su lugar MUST permitir
  desactivarla. Una presentación desactivada MUST dejar de ofrecerse para nuevas asociaciones a
  categorías, sin alterar las asociaciones y variantes ya existentes que la referencian.
- **FR-003**: El sistema MUST impedir crear o editar una presentación con un nombre que ya use otra
  presentación dentro del mismo tenant (la validación de unicidad aplica tanto a la creación como a
  la edición/renombre).
- **FR-019**: Al desplegar esta funcionalidad, el sistema MUST sembrar el catálogo de
  Presentaciones de cada tenant creando una presentación activa por cada nombre de variante de
  producto distinto (comparación exacta de texto) que ya exista en ese tenant. Esta siembra MUST
  NOT crear asociaciones Categoría↔Presentación (FR-004) ni modificar productos o variantes
  existentes; solo puebla el catálogo para que el administrador los reutilice y los asocie
  manualmente a las categorías que correspondan.

### Asociación Categoría↔Presentaciones y herencia en producto

- **FR-004**: Al crear o editar una Categoría, el sistema MUST permitir seleccionar cero o más
  presentaciones del catálogo global como las presentaciones habilitadas para esa categoría. El
  selector de la interfaz MUST ofrecer únicamente presentaciones activas como opciones nuevas
  (mismo criterio de disponibilidad que FR-002); la validación del sistema al guardar MUST
  verificar solo que cada id exista en el catálogo (esté activa o no), sin exigir `active=true`
  en ese momento — así se permite conservar una asociación a una presentación que se desactivó
  después (FR-002), sin reabrir su eliminación física.
- **FR-005**: Al crear un producto dentro de una categoría con una o más presentaciones
  asociadas, el sistema MUST crear automáticamente una variante del producto por cada presentación
  asociada a esa categoría, usando el nombre de la presentación como nombre de la variante y
  dejando su precio en $0 para que el administrador lo complete.
- **FR-006**: Al crear un producto dentro de una categoría sin presentaciones asociadas, el
  sistema MUST asignarle automáticamente una única variante llamada "Presentación única" (mismo
  mecanismo ya vigente para productos sin variaciones de tamaño, `RN-CAT-05`), garantizando que
  ningún producto quede sin al menos una variante.
- **FR-007**: El sistema MUST seguir permitiendo que el administrador agregue, edite o elimine
  variantes de un producto manualmente después de su creación automática, sin restringir la
  edición a lo heredado de la categoría.
- **FR-008**: Cambiar las presentaciones asociadas a una categoría después de que ya existan
  productos creados en ella MUST NOT modificar retroactivamente las variantes de esos productos;
  el cambio solo MUST aplicar a productos creados a partir de ese momento.

### Rediseño de promociones (fidelidad UI y flujo guiado)

- **FR-009**: El sistema MUST reemplazar las pantallas de listado, creación y configuración de
  promociones por el diseño y la estructura visual de los prototipos `listado-promociones.html`,
  `formulario-crear-promocion.html` y `configurar-promocion.html`, sin alterar las reglas de
  vigencia, estados, solapamiento o persistencia del descuento ya
  vigentes desde la spec 063.
- **FR-010**: Al crear una promoción, el sistema MUST solicitar primero su nombre y su tipo
  (descuento por porcentaje o precio de paquete). El tipo elegido en esta pantalla MUST aplicar
  como el tipo compartido de todas las reglas que se configuren después para esa promoción, y MUST
  quedar fijo una vez creada (no editable desde la pantalla de configuración).
- **FR-011**: En la pantalla de configuración, el sistema MUST permitir definir la vigencia de la
  promoción: días de la semana (selección múltiple, vacío = todos los días), fecha de inicio,
  fecha de fin (opcional), y hora de inicio y fin (opcionales, ambas o ninguna, con posibilidad de
  cruzar la medianoche) — reutilizando sin cambios las reglas de vigencia ya vigentes (spec 063
  FR-012, FR-013).
- **FR-012**: El sistema MUST permitir buscar y seleccionar uno o varios productos participantes,
  con filtro por categoría y por texto como ayuda para poblar la selección, igual que ya existe
  (spec 063 FR-004).
- **FR-013**: Para los productos seleccionados, el sistema MUST permitir elegir una de las
  variantes reales del producto — mostrando como etiqueta el nombre de la presentación del
  catálogo cuando el nombre de la variante coincide con una presentación activa asociada a la
  categoría del producto (o "Presentación única" si el producto no maneja variaciones), y el
  nombre propio de la variante en caso contrario, sin excluir ninguna variante del selector —,
  indicar las unidades mínimas requeridas y el precio promocional total, y agregar esa combinación
  como una fila de regla de precio de la promoción.
- **FR-014**: El sistema MUST permitir agregar varias filas de regla de precio (cada una con su
  propia presentación, unidades mínimas y precio) dentro de la misma promoción, igual que ya
  permite varias reglas por promoción (spec 063 FR-001).
- **FR-015**: Antes de guardar, el sistema MUST mostrar por cada fila/regla configurada un resumen
  legible: presentación, unidades mínimas, precio regular estimado, precio promocional y el
  ahorro resultante (mismo criterio de resumen ya exigido por spec 063 FR-005, adaptado a la
  tabla de reglas del nuevo diseño).
- **FR-016**: El sistema MUST seguir bloqueando el guardado de una regla de precio de paquete cuyo
  valor no represente un ahorro frente al precio normal de la presentación elegida (mismo criterio
  ya vigente, spec 063 FR-016).
- **FR-017**: El listado de promociones MUST mostrar, por cada promoción, su nombre, la columna
  "Reglas" MUST mostrar únicamente el tipo de promoción (descuento % o precio de paquete; sin
  condición/resumen de variantes en esta columna), su vigencia en lenguaje llano, su estado, y las
  acciones ya disponibles hoy (configurar, duplicar, pausar/activar, eliminar), reutilizando sin
  cambios el motor de estados vigente (spec 063 FR-015, FR-017).
- **FR-018**: En una promoción `Activa` o `Pausada`, el sistema MUST seguir bloqueando la edición
  del tipo, el valor, las unidades mínimas o el conjunto de variantes de cualquiera de sus reglas
  ya guardadas (spec 063 FR-018), y MUST reflejar ese bloqueo visualmente en la pantalla de
  configuración (aviso de tipo "fijado en creación"). La pantalla de configuración MUST NOT mostrar
  ni permitir editar un campo de descripción (ver FR-023); nombre, fin de vigencia, días y horas
  siguen editables en `Activa`/`Pausada`.
- **FR-020**: Al presionar "Continuar" en la pantalla de creación con el nombre y el tipo ya
  definidos, el sistema MUST crear inmediatamente la promoción en base de datos con estado
  `Borrador` y una lista de reglas vacía, sin esperar a que se configure ninguna regla de precio.
  El backend MUST permitir crear una promoción sin reglas únicamente cuando su `status` de creación
  es `draft` (relaja `PromotionCreate.rules` de `min_length=1` a `min_length=0` solo en ese caso);
  la exigencia de al menos una regla con al menos una variante para poder activarla (`status=active`)
  MUST seguir vigente sin cambios (spec 063 FR-018, ya validado en `change_status`).
- **FR-021**: El botón "Eliminar" de cada fila del listado MUST estar habilitado siempre que el
  `status` real de la promoción sea distinto de `active` (es decir: `draft`, `paused` o `finished`),
  sin importar el estado visual derivado que muestre su badge (p. ej. una promoción `active` cuya
  vigencia ya venció — badge "Vencida" — sigue sin poder eliminarse hasta pausarla o finalizarla).
  MUST seguir usando el endpoint `DELETE /promotions/{id}` ya existente, sin cambios de backend
  (ese endpoint no valida `status` hoy).
- **FR-022**: Las pestañas de filtro del listado ("Todas"/"Borradores"/"Activas"/"En pausa"/
  "Finalizadas") MUST seguir filtrando por el `status` real de la promoción (`GET /promotions?status=`),
  sin cambios de backend — el filtro ya funciona correctamente sobre `status`; el badge de estado de
  cada fila sigue mostrando el estado visual derivado (que puede diferir de la pestaña activa, p. ej.
  "Vencida" dentro de la pestaña "Activas", cuando `status=active` pero la vigencia ya pasó).
- **FR-023**: La pantalla de configuración de promociones MUST NOT incluir un campo de
  "Descripción": se retira de la interfaz y el frontend MUST NOT enviarlo en los payloads de crear
  ni actualizar una promoción. El campo `description` del modelo de datos permanece sin cambios
  (Principio VII): las promociones ya existentes que tengan una descripción guardada de antes de
  este cambio la conservan sin modificarse.
- **FR-024**: El grid de tarjetas de producto del Paso 1 (configuración de promoción, FR-012) MUST
  mostrarse dentro de un contenedor con altura fija de 320px y scroll vertical propio
  (`overflow-y: auto`) cuando la cantidad de productos que coinciden con la búsqueda/filtro exceda
  el espacio visible, reutilizando el estilo de scrollbar (`.custom-scrollbar`) ya definido en el
  prototipo `configurar-promocion.html`. El buscador y el filtro por categoría de FR-012 MUST
  permanecer visibles por encima del contenedor, sin desplazarse junto con las tarjetas.

### Validaciones de reglas: unidades mínimas, precio vs. suma regular y toppings (nuevo, sesión 2026-09-17)

- **FR-025**: Toda regla de tipo **precio de paquete** (nueva, o una regla existente que se edite)
  MUST exigir `unidades (minQuantity) >= 2`. El campo/selector de unidades del formulario MUST
  iniciar en 2 por defecto y MUST impedir ingresar o decrementar a un valor menor a 2, mostrando el
  mensaje "En promociones por paquete, el mínimo es de 2 unidades". El backend MUST aplicar el
  mismo mínimo en el DTO/request de creación y de edición de la regla, rechazando la operación si
  no se cumple. Esta validación retira, para reglas nuevas o editadas, el patrón `minQuantity = 1`
  ("precio unitario especial") documentado en spec 063; **no es retroactiva**: una regla ya
  guardada con `minQuantity = 1` (o que de otro modo incumpla este mínimo) sigue vigente y
  cobrándose sin cambios hasta que el administrador intente editarla, momento en el que debe
  cumplir el nuevo mínimo para poder guardarse.
- **FR-026**: Al crear o editar una regla de tipo precio de paquete, el sistema MUST bloquear el
  guardado si el precio promocional es mayor o igual que la suma de los precios regulares de las
  unidades del paquete (`precio_promocional >= precio_regular_de_la_variante × unidades`),
  mostrando "El precio promocional ($X) debe ser menor a la suma del precio regular de los
  productos seleccionados ($Y)." Dado que cada regla de este flujo se arma sobre una sola
  variante (FR-013), esta suma coincide exactamente con el "peor caso del conjunto" que ya bloquea
  spec 063 FR-016 — esta regla no introduce un motor de cálculo nuevo, solo refuerza el mensaje de
  error y se combina con el mínimo de 2 unidades de FR-025. El frontend MUST recalcular y comparar
  dinámicamente esta suma contra el precio promocional cada vez que cambie el producto, la
  presentación, las unidades o el precio dentro del formulario, antes de permitir "Agregar a la
  lista". El backend MUST reforzar la misma comprobación en el DTO/request de creación y edición de
  la regla. Tampoco es retroactiva, con el mismo criterio de FR-025.
- **FR-027**: Cuando el tipo de promoción es **porcentaje** y la variante vendida incluye grupos de
  opciones/adicionales ("Toppings", `OptionGroup`, specs 064/065) seleccionados, el motor de
  cálculo de precios (vista previa en frontend y cobro en backend) MUST aplicar el descuento
  porcentual únicamente sobre el precio base de la variante, sumando el precio regular íntegro
  (sin descuento) de cada opción/adicional elegida al total de la línea. Esta regla aplica
  ÚNICAMENTE a promociones de tipo porcentaje: en promociones de tipo precio de paquete, el precio
  de los toppings/adicionales sigue cobrándose aparte a precio regular sin cambios, porque el
  precio de paquete ya definido por el administrador cubre solo la variante base. No se requiere
  persistir un desglose separado de descuento base/toppings en `Sale`/`SaleInvoice`/
  `CustomerOrder`: FR-021 de spec 063 sigue sin cambios, ya que el monto de descuento agregado
  registrado refleja el efecto neto de esta exclusión.

### Key Entities

- **Presentación**: entidad de catálogo nueva, por tenant. Tiene nombre y estado
  (activa/inactiva, alternable en ambos sentidos). Es reutilizable entre categorías y productos; no
  lleva precio propio (el precio sigue viviendo en cada variante de producto). No participa en el
  alcance de una regla de promoción más que como ayuda de selección — el alcance real sigue siendo
  la lista explícita de variantes. Al desplegar la funcionalidad, cada tenant recibe una siembra
  inicial con una presentación por cada nombre de variante distinto ya existente (FR-019).
- **Asociación Categoría↔Presentación**: relación N a N entre una Categoría y las presentaciones
  del catálogo global que le corresponden. Determina qué variantes se crean automáticamente al dar
  de alta un producto en esa categoría.
- **Variante de producto (`ProductVariant`)**: sin cambio de modelo. Su nombre puede poblarse
  automáticamente a partir de las presentaciones asociadas a la categoría del producto al crearlo,
  o recibir "Presentación única" cuando la categoría no tiene ninguna asociada — mismo mecanismo
  ya vigente (`RN-CAT-05`), solo con nuevo nombre visible.
- **Promoción / Regla / conjunto de variantes**: entidades ya existentes desde la spec 063. Esta
  spec cambia cómo se configuran desde la interfaz de administración (usando el catálogo de
  Presentaciones como ayuda para poblar el conjunto de variantes de cada regla) y agrega, para
  reglas de precio de paquete, el mínimo de 2 unidades y el refuerzo de la comprobación de precio
  vs. suma regular (FR-025, FR-026, sesión 2026-09-17); el resto de su comportamiento (vigencia,
  solapamiento, estados) no cambia.
- **Grupo de opciones / Toppings (`OptionGroup`)**: entidad ya existente (specs 064/065), sin
  cambio de modelo. Esta spec solo ajusta cómo el motor de cálculo de precios trata su precio
  frente a una promoción de tipo porcentaje: queda excluido de la base sobre la que se aplica el
  descuento (FR-027).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El administrador puede crear una presentación una sola vez en el catálogo global y
  reutilizarla en al menos dos categorías distintas sin volver a escribir su nombre.
- **SC-002**: El 100% de los productos nuevos creados dentro de una categoría con presentaciones
  asociadas nace con esas presentaciones ya asignadas como variantes, sin que el administrador
  tenga que crearlas manualmente una por una.
- **SC-003**: El 100% de los productos nuevos creados en una categoría sin presentaciones
  asociadas nace con la variante "Presentación única" asignada automáticamente; ningún producto
  queda sin al menos una variante.
- **SC-004**: Las pantallas de listado, creación y configuración de promociones coinciden
  visualmente con los prototipos entregados, verificable por comparación directa sin necesidad de
  leer código.
- **SC-005**: Un administrador puede configurar una promoción completa (vigencia + al menos una
  regla de precio por presentación) en una sola sesión de formulario.
- **SC-006**: El 100% de los intentos de eliminar del catálogo una presentación vinculada a un
  producto o a una promoción se bloquean, ofreciendo desactivarla en su lugar.
- **SC-007**: El 100% de los intentos de crear o editar una regla de precio de paquete con menos
  de 2 unidades, o con un precio promocional mayor o igual a la suma de precios regulares del
  paquete, se bloquean tanto en el formulario como en el backend, mostrando el motivo del bloqueo.
- **SC-008**: En el 100% de las ventas con una promoción de porcentaje aplicada a una variante con
  toppings/adicionales seleccionados, el descuento cobrado corresponde solo al precio base de la
  variante; el precio de los toppings se cobra íntegro.

## Assumptions

- El catálogo de Presentaciones es un catálogo de **etiquetas/plantillas** por tenant; no
  reintroduce ningún mecanismo de alcance automático de promoción por presentación. El alcance de
  cada regla de promoción sigue siendo la lista explícita de variantes elegidas a mano (spec 063
  FR-003, FR-010, anomalía A-65 sin cambios).
- "Presentación única" es un renombre en la interfaz del mecanismo ya existente (`RN-CAT-05`,
  variante `"Single"` automática); no cambia su comportamiento de creación ni su precio inicial
  ($0, a completar por el administrador).
- Esta spec no modifica el modelo Promoción/Regla, las reglas de vigencia/solapamiento, la
  persistencia del descuento en la venta, ni el consumo de promociones en la terminal o en el
  menú QR (specs 063, 066, 071, 081), **salvo** las validaciones de unidades mínimas y suma de
  precios regulares en reglas de precio de paquete, y el ajuste del motor de cálculo para excluir
  los toppings/adicionales del descuento por porcentaje (FR-025 a FR-027, sesión de Clarifications
  2026-09-17). Fuera de eso, solo agrega el catálogo de Presentaciones, la asociación por
  categoría, la herencia al crear producto, y rediseña las pantallas de administración de
  promociones.
- El tipo de una promoción (porcentaje o precio de paquete) se elige una sola vez al crearla y
  aplica como tipo compartido de todas sus reglas, reflejando el bloqueo mostrado en el prototipo
  `configurar-promocion.html` ("Tipo seleccionado... Fijado en creación").
- Solo el administrador del tenant gestiona el catálogo de presentaciones, sus asociaciones por
  categoría y las promociones — mismo criterio de permisos ya vigente (spec 063 FR-019); el
  cajero solo visualiza.
- El catálogo de presentaciones vive con el mismo aislamiento por tenant (schema-per-tenant) que
  el resto del catálogo de productos y categorías.
- Los repositorios afectados son `pos-backend` (catálogo de presentaciones y su siembra inicial
  desde los nombres de variante existentes, asociación por categoría, herencia en producto, y
  lógica de administración de promociones) y `pos-heladeria`
  (las tres pantallas rediseñadas y el nuevo módulo de administración de presentaciones). La rama
  de implementación en ambos repositorios sigue la convención `<tipo>/090-<nombre>` ya definida
  por el equipo (p. ej. `feat/090-promotions`), independiente del número de esta carpeta de spec
  (`083-presentaciones-y-promociones`).
