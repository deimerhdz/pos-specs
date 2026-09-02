# Feature Specification: Mejoras de usabilidad en el formulario de administración de promociones

**Feature Branch**: `071-mejoras-formulario-promociones`

**Created**: 2026-09-02

**Status**: Draft

**Naturaleza de esta spec**: **corrección de usabilidad** sobre el formulario de administración
de promociones que dejaron las specs [063](../063-promociones-por-variante/spec.md) (modelo
`Promoción`/`Regla` por conjunto de variantes) y [066](../066-promociones-legibles-menu/spec.md)
(condición en lenguaje llano con nombres de variante, ya aplicada al menú QR y a la terminal).
No cambia el modelo de datos ni el motor de cálculo del cobro. El cuarto punto sí es un
**cambio de comportamiento en producción** sobre la spec 063 FR-018 (relaja el bloqueo de
edición de reglas —agregar, quitar reglas completas y editar el conjunto de variantes de las
existentes— en estado `Pausada`), por lo que se documenta explícitamente (Principio II de la
[Constitución](../../.specify/memory/constitution.md)).

**Input**: User description: cuatro mejoras sobre la pantalla de administración de promociones
(`pos-heladeria`, módulo de promociones) reportadas con una captura de la tenant "heladeria3":
(1) el resumen de cada regla debe nombrar el/los producto(s) de su conjunto en vez de solo la
condición numérica; (2) el listado de productos bajo una regla debe mostrar únicamente los ya
seleccionados, no el catálogo completo — la búsqueda con filtro de categoría es responsable de
encontrar y agregar productos, y desmarcar uno debe quitarlo también del listado; (3) una regla
nueva debe aparecer al principio del listado de reglas, no al final; (4) el conjunto de
productos de una promoción vigente (`Activa` o `Pausada`) debe poder modificarse cuando esa
promoción está en estado `Pausada` al momento de editar.

---

## Estado actual (lo que se corrige)

Línea base tomada de la captura de pantalla adjunta y de las specs 063/066:

- **Resumen de regla**: la tarjeta colapsada de cada regla en el listado ("Regla 1", "Regla 2",
  …) muestra un texto de condición que **no nombra ningún producto**, por ejemplo
  "Precio de paquete - Paga $ 12.000 llevando 2 unidades." aunque la regla tenga exactamente un
  producto seleccionado ("1 producto seleccionado" aparece debajo, en gris, como dato aparte).
  El comensal en el menú QR y el cajero en la terminal ya ven el nombre del producto en su
  propia condición (spec 066 FR-001–FR-005); el formulario de administración quedó fuera de esa
  corrección para este resumen específico.
- **Listado de productos de una regla**: bajo "CONJUNTO (N)" el formulario muestra **el
  catálogo completo del tenant** (todas las variantes, con casilla y precio), con un selector de
  categoría, un campo "Buscar variante…" y un botón "Agregar visibles". La lista no distingue
  entre "ya seleccionado" y "disponible para seleccionar": ambos aparecen juntos, scrolleables,
  sin importar cuántos productos tenga el catálogo del tenant.
- **Orden del listado de reglas**: al presionar "+ Agregar regla", la nueva regla se agrega
  **al final** del listado (después de "Regla 1", "Regla 2", …), obligando a scrollear para
  configurarla, sobre todo en promociones con varias reglas (p. ej. las seis reglas "2X" de
  Springfield descritas en la spec 063).
- **Edición del conjunto de variantes en promoción vigente**: por spec 063 FR-018, en estado
  `Activa` **o** `Pausada` el conjunto de variantes de cada regla queda bloqueado; la única vía
  para cambiar qué productos cubre una promoción que ya salió de `Borrador` es duplicarla. Esto
  incluye el caso de pausar la promoción a propósito para corregir un producto mal
  seleccionado: hoy pausar no lo habilita.

---

## Clarifications

### Session 2026-09-02

- Q: Al pausar una promoción vigente para corregirla, ¿el administrador podrá también agregar o
  quitar reglas completas, o solo cambiar los productos de una regla ya existente? → A:
  **También** podrá agregar y quitar reglas completas mientras la promoción está `Pausada`, no
  solo editar el conjunto de variantes de una regla que ya existía (FR-014).
- Q: ¿La búsqueda de variantes debe poder listar resultados usando solo el filtro de categoría,
  sin escribir texto, para agregar de una vez todos los productos de una categoría? → A: **Sí**,
  pero únicamente cuando se elige una categoría específica: con "Todas las categorías" y el
  buscador vacío no se muestra ningún resultado, para no volver a listar el catálogo completo
  por defecto (FR-007, FR-008).

---

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El resumen de una regla dice qué producto lleva (Priority: P1)

Un administrador configura una regla de precio de paquete o de descuento por porcentaje,
selecciona el/los producto(s) que la componen, y al colapsar la tarjeta de la regla lee de
inmediato qué productos están incluidos, sin tener que reabrirla.

**Why this priority**: es el defecto más visible del formulario — hoy dos reglas distintas de
la misma promoción pueden mostrar el mismo texto genérico ("Paga $12.000 llevando 2 unidades")
aunque cubran productos completamente distintos, obligando a abrir cada una para saber cuál es
cuál.

**Independent Test**: crear una regla de precio de paquete con cantidad mínima 2 y valor
$12.000 sobre la variante "Gaseosa - Única", colapsarla, y comprobar que el resumen la nombra;
repetir con "Banana Split Especial - Pequeña" y comprobar que el resumen cambia en consecuencia.

**Acceptance Scenarios**:

1. **Given** una regla de tipo "Precio de paquete" con cantidad mínima 2, valor $12.000 y como
   único producto seleccionado "Gaseosa - Única", **When** el administrador colapsa o guarda la
   regla, **Then** el resumen lee "Precio de paquete - Paga $ 12.000 llevando 2 unidades Gaseosa
   - Única.".
2. **Given** la misma regla pero con "Banana Split Especial - Pequeña" como único producto,
   **When** se muestra el resumen, **Then** lee "Precio de paquete - Paga $ 12.000 llevando 2
   unidades Banana Split Especial - Pequeña.".
3. **Given** una regla de tipo "Descuento %" con valor 10%, cantidad mínima 1 y un solo producto
   "Cono sencillo - Única", **When** se muestra el resumen, **Then** lee "Descuento % - 10% en
   Cono sencillo - Única.".
4. **Given** una regla cuyo conjunto tiene tres productos con nombres distintos, **When** se
   muestra el resumen, **Then** los nombra en orden alfabético con el conector "entre" (mismo
   criterio ya usado en el menú QR, spec 066 FR-002); con más de tres, se listan los tres
   primeros y "y N más" (spec 066 FR-004).
5. **Given** una regla recién creada sin ningún producto seleccionado, **When** se muestra el
   resumen, **Then** indica explícitamente que no tiene productos seleccionados en vez de
   mostrar una condición con cantidades pero sin nombre.

---

### User Story 2 - Elegir productos por búsqueda, no por catálogo completo (Priority: P2)

Un administrador con un catálogo de decenas o cientos de variantes agrega productos a una
regla escribiendo en el buscador (opcionalmente acotado por categoría) y marcando los que
quiere incluir. El listado bajo la regla solo muestra los productos que ya forman parte del
conjunto; si desmarca uno, deja de aparecer ahí.

**Why this priority**: hoy el listado de "productos aplicables" es en realidad el catálogo
completo del tenant, lo que no escala y mezcla "seleccionado" con "candidato" en la misma
lista, dificultando ver de un vistazo qué compone la regla.

**Independent Test**: con un catálogo de más de 6 variantes en dos categorías, abrir una regla
sin selección, filtrar por una categoría, buscar un texto, marcar dos variantes y comprobar que
solo esas dos aparecen en el listado del conjunto; desmarcar una y comprobar que desaparece de
ese listado.

**Acceptance Scenarios**:

1. **Given** una regla con el conjunto vacío, **When** el administrador la abre sin escribir
   nada en el buscador, **Then** el listado bajo "CONJUNTO (0)" no muestra ninguna variante del
   catálogo.
2. **Given** el mismo estado, **When** el administrador escribe "gaseosa" en "Buscar variante…",
   **Then** ve entre los resultados de búsqueda las variantes del catálogo que coinciden, estén
   o no ya seleccionadas.
3. **Given** los resultados de búsqueda, **When** el administrador marca "Gaseosa - Única",
   **Then** esa variante se agrega al conjunto y aparece de inmediato en el listado de
   seleccionados, con el contador "CONJUNTO (1)".
4. **Given** un conjunto con dos variantes seleccionadas, **When** el administrador elige una
   categoría en el filtro, **Then** una nueva búsqueda solo devuelve variantes de esa categoría,
   sin quitar del conjunto las ya seleccionadas que pertenezcan a otra categoría.
5. **Given** una variante ya en el conjunto, **When** el administrador la desmarca (desde el
   listado de seleccionados o desde un resultado de búsqueda que la muestre), **Then** se quita
   del conjunto y deja de aparecer en el listado de seleccionados.
6. **Given** varios resultados visibles tras buscar y filtrar por categoría, **When** el
   administrador presiona "Agregar visibles", **Then** todas las variantes que en ese momento
   coinciden con la búsqueda y la categoría se agregan al conjunto, sin afectar las que ya
   estaban seleccionadas fuera de ese filtro.
7. **Given** el conjunto vacío y el buscador sin texto, **When** el administrador elige la
   categoría "Granizados" en el filtro, **Then** ve como resultados todas las variantes de esa
   categoría, sin haber escrito nada; **When** presiona "Agregar visibles", **Then** todas esas
   variantes se agregan al conjunto de una sola vez.
8. **Given** el buscador sin texto, **When** el administrador deja el filtro en "Todas las
   categorías", **Then** no ve ningún resultado de búsqueda — evita listar el catálogo completo
   por defecto (FR-008).

---

### User Story 3 - Una regla nueva aparece primero (Priority: P3)

Al presionar "+ Agregar regla", la nueva regla aparece al principio del listado, lista para
configurarse sin scrollear.

**Why this priority**: es una fricción menor pero constante en promociones con varias reglas
(el caso real de las seis reglas "2X" de Springfield, spec 063): cada regla nueva obliga a
bajar hasta el final para encontrarla.

**Independent Test**: con una promoción que ya tiene dos reglas, presionar "+ Agregar regla" y
comprobar que la nueva ocupa la posición 1 y las anteriores se corren a la posición 2 y 3,
conservando su orden relativo entre sí.

**Acceptance Scenarios**:

1. **Given** una promoción con "Regla 1" y "Regla 2" existentes, **When** el administrador
   presiona "+ Agregar regla", **Then** la nueva regla se muestra en la primera posición del
   listado y las existentes se corren hacia abajo, sin cambiar su orden relativo.
2. **Given** una promoción sin reglas, **When** se agrega la primera, **Then** es la única del
   listado, sin cambio de comportamiento.

---

### User Story 4 - Corregir las reglas de una promoción vigente pausándola (Priority: P1)

Un administrador nota que una promoción `Activa` incluyó un producto equivocado, le falta una
regla, o tiene una regla de más. La pausa, corrige lo que haga falta —agregar productos a una
regla existente, quitar productos de ella, agregar una regla completamente nueva o quitar una
regla que ya no aplica— y la reactiva, sin tener que duplicar la promoción ni recrearla desde
cero.

**Why this priority**: hoy corregir cualquier aspecto de las reglas de una promoción que ya
salió de `Borrador` exige duplicarla, perdiendo el vínculo con la promoción original y
obligando a finalizar la vieja a mano; para un error operativo simple (marcó el producto
equivocado, o faltó una regla) es una fricción desproporcionada.

**Independent Test**: activar una promoción con una regla y un producto seleccionado, pausarla,
agregar y quitar productos del conjunto de esa regla, agregar una regla nueva, quitar una
regla existente, reactivarla, y comprobar que el cobro usa las reglas y el conjunto
actualizados.

**Acceptance Scenarios**:

1. **Given** una promoción `Activa` con una regla, **When** el administrador intenta marcar o
   desmarcar un producto del conjunto de esa regla, o agregar/quitar una regla, **Then** el
   sistema lo impide, igual que hoy (spec 063 FR-018 sin cambios para el estado `Activa`).
2. **Given** la misma promoción, **When** el administrador la pasa a `Pausada`, **Then** el
   conjunto de variantes de sus reglas existentes, y la posibilidad de agregar o quitar reglas
   completas, se habilitan para edición.
3. **Given** la promoción `Pausada`, **When** el administrador agrega un producto nuevo al
   conjunto de una regla y quita otro, y guarda, **Then** el cambio se persiste y, al reactivar
   la promoción, el descuento aplica sobre el conjunto ya corregido.
4. **Given** la promoción `Pausada`, **When** el administrador presiona "+ Agregar regla" y
   configura tipo, valor, cantidad mínima y conjunto de la regla nueva, y guarda, **Then** la
   regla se agrega a la promoción (en la primera posición del listado, FR-012) y aplica
   descuento en cuanto la promoción vuelve a `Activa`.
5. **Given** la promoción `Pausada` con dos o más reglas, **When** el administrador quita una de
   las reglas existentes y guarda, **Then** esa regla deja de existir en la promoción y de
   aplicar descuento al reactivar; las demás reglas no se ven afectadas.
6. **Given** la promoción `Pausada` con una sola regla, **When** el administrador intenta
   quitarla, **Then** el sistema lo impide porque una promoción debe conservar al menos una
   regla (FR-018); para eliminar la promoción por completo debe finalizarla.
7. **Given** la promoción `Pausada`, **When** el administrador intenta cambiar el tipo, el valor
   o la cantidad mínima de una regla que ya existía antes de pausar, **Then** el sistema lo
   sigue bloqueando in situ (sin cambio respecto de spec 063 FR-018); para lograr ese cambio
   debe quitar la regla y agregar una nueva con la configuración deseada, ambas acciones ya
   habilitadas en `Pausada`.
8. **Given** una edición en estado `Pausada` (conjunto de una regla existente, o una regla
   nueva), **When** el administrador intenta guardar un conjunto que compartiría producto y
   ventana de vigencia con otra regla vigente, o dentro de la misma promoción con otra de sus
   reglas, **Then** el sistema lo bloquea igual que al crear (spec 063 FR-014, FR-001a).
9. **Given** una promoción `Finalizada` o `Borrador`, **When** el administrador intenta aplicar
   esta corrección, **Then** no aplica: `Finalizada` sigue sin ninguna edición (terminal) y
   `Borrador` ya permite editar y agregar/quitar reglas libremente, sin cambio respecto de spec
   063.

---

### Edge Cases

- **Regla sin productos seleccionados**: el resumen debe decirlo explícitamente en vez de mostrar
  una condición con cantidades pero sin nombre (US1, escenario 5).
- **Búsqueda sin resultados**: si el texto o la combinación de categoría y texto no coincide con
  ninguna variante del catálogo, el listado de resultados de búsqueda queda vacío; el listado de
  seleccionados no cambia.
- **Categoría elegida sin texto de búsqueda**: se listan como resultados todas las variantes de
  esa categoría, permitiendo agregarla completa con "Agregar visibles" (FR-007, US2 escenario
  7). Sin una categoría específica (opción "Todas las categorías") y sin texto, no se muestra
  ningún resultado, para no listar el catálogo completo por defecto (FR-008, US2 escenario 8).
- **Desmarcar el único producto de una regla**: el conjunto queda vacío; la regla sigue
  existiendo pero no aplica descuento hasta que se le agregue al menos un producto (consistente
  con spec 063 FR-011).
- **Cambiar de categoría con productos ya seleccionados de otra categoría**: no se pierden ni se
  desmarcan; el filtro solo acota qué aparece en los *resultados de búsqueda*, no el listado de
  seleccionados (US2, escenario 4).
- **Pausar solo para editar el conjunto y no reactivar**: la promoción queda `Pausada` sin
  aplicar descuento hasta que el administrador la reactive explícitamente; pausar no reactiva
  sola al guardar (sin cambio respecto del comportamiento de estado de spec 063 FR-015).
- **Reactivar tras editar el conjunto en `Pausada`**: el cobro usa el conjunto vigente al momento
  de la venta; no queda ningún rastro del conjunto anterior a la edición (consistente con spec
  063 FR-020, que prohíbe descuentos "congelados").
- **Reglas nuevas consecutivas**: agregar dos reglas seguidas deja la segunda agregada arriba de
  la primera (cada nueva regla entra en la posición 1 en el momento de agregarla).
- **Quitar la última regla de una promoción pausada**: el sistema lo bloquea; una promoción
  siempre conserva como mínimo una regla, en `Pausada` igual que en cualquier otro estado
  (FR-018). Para eliminar la promoción por completo, el administrador la finaliza.
- **Agregar una regla nueva mientras la promoción está `Pausada`**: queda sujeta a las mismas
  validaciones de guardado que crear una regla en `Borrador` (solapamiento, conjuntos disjuntos
  dentro de la misma promoción, precio de paquete que sí represente un descuento) — FR-016.
- **Cambiar tipo, valor o cantidad mínima de una regla que ya existía al pausar**: sigue
  bloqueado in situ; la vía habilitada por esta spec es quitar esa regla y agregar una nueva con
  la configuración deseada, ambas acciones posibles en `Pausada` (FR-014, FR-015).

---

## Requirements *(mandatory)*

### Resumen legible de la regla

- **FR-001**: El resumen de cada regla (tarjeta colapsada o vista previa en el listado de
  reglas) DEBE nombrar el/los producto(s)-variante(s) de su conjunto; NUNCA DEBE mostrar
  únicamente una cantidad de unidades sin nombrar qué producto se lleva.
- **FR-002**: Para una regla de tipo "Precio de paquete", el resumen DEBE tener el formato
  "Precio de paquete - Paga $ {valor} llevando {cantidad mínima} unidades {nombre(s) de
  variante}.".
- **FR-003**: Para una regla de tipo "Descuento %", el resumen DEBE tener el formato "Descuento
  % - {valor}% en {nombre(s) de variante}." cuando la cantidad mínima es 1, o "Descuento % -
  {valor}% llevando {cantidad mínima} unidades {nombre(s) de variante}." cuando es mayor a 1.
- **FR-004**: Cuando el conjunto de una regla tenga más de un nombre de variante distinto, el
  resumen DEBE listarlos en orden alfabético con el conector "entre", hasta un máximo de tres
  nombres, seguido de "y N más" si hay nombres adicionales — mismo criterio de la spec 066
  (FR-002 a FR-005), para que el formulario de administración use la misma convención que el
  menú QR y la terminal.
- **FR-005**: Si el conjunto de variantes de una regla está vacío, el resumen DEBE indicarlo
  explícitamente (p. ej. "Sin productos seleccionados") en vez de mostrar una condición con
  cantidades pero sin nombre.

### Selección de productos por búsqueda

- **FR-006**: El listado de productos bajo el selector de una regla DEBE mostrar únicamente las
  variantes que ya forman parte del conjunto de esa regla; NO DEBE listar el catálogo completo
  del tenant por defecto.
- **FR-007**: El campo de búsqueda de variantes DEBE permitir localizar, entre todo el catálogo
  del tenant, cualquier variante coincidente con el texto ingresado, esté o no ya en el conjunto
  de la regla. Si el administrador elige una categoría específica (FR-008) y deja el campo de
  búsqueda vacío, el listado de resultados DEBE mostrar igualmente todas las variantes de esa
  categoría, sin exigir texto.
- **FR-008**: El filtro de categoría DEBE acotar los resultados de la búsqueda a la categoría
  elegida. Con "Todas las categorías" (sin categoría específica) y el campo de búsqueda vacío,
  el listado de resultados NO DEBE mostrar ninguna variante — evita volver a listar el catálogo
  completo por defecto (FR-006).
- **FR-009**: Al marcar una variante desde los resultados de búsqueda, el sistema DEBE agregarla
  al conjunto de la regla, y esa variante DEBE aparecer de inmediato en el listado de
  seleccionados descrito en FR-006.
- **FR-010**: Al desmarcar una variante (desde el listado de seleccionados o desde un resultado
  de búsqueda que la muestre marcada), el sistema DEBE quitarla del conjunto de la regla, y esa
  variante DEBE dejar de aparecer en el listado de seleccionados.
- **FR-011**: La acción "Agregar visibles" DEBE agregar al conjunto todas las variantes que en
  ese momento coincidan con el texto de búsqueda y la categoría vigentes, sin quitar del
  conjunto ninguna variante ya seleccionada que no coincida con ese filtro.

### Orden del listado de reglas

- **FR-012**: Al agregar una nueva regla a una promoción, el sistema DEBE insertarla en la
  primera posición del listado de reglas; las reglas existentes DEBEN conservar su orden
  relativo, desplazadas una posición hacia abajo.

### Edición de reglas de una promoción vigente pausada

- **FR-013**: En una promoción vigente en estado `Activa`, agregar o quitar reglas y editar el
  conjunto de variantes de cualquiera de ellas DEBE seguir bloqueado, sin cambio respecto de la
  spec 063 FR-018.
- **FR-014**: En una promoción vigente que se encuentre en estado `Pausada`, el sistema DEBE
  permitir, sin necesidad de duplicar la promoción: (a) agregar productos a o quitar productos
  del conjunto de variantes de cualquiera de sus reglas existentes; (b) agregar una regla
  completamente nueva a la promoción; y (c) quitar una regla existente de la promoción.
- **FR-015**: El tipo, el valor y la cantidad mínima de una regla que ya existía al pausar la
  promoción DEBEN permanecer bloqueados para edición **in situ** en los estados `Activa` y
  `Pausada`. Para cambiarlos, el administrador DEBE quitar esa regla y agregar una regla nueva
  con la configuración deseada —ambas acciones habilitadas por FR-014 en `Pausada`— o bien
  duplicar la promoción completa (spec 063 FR-017).
- **FR-016**: Al guardar cualquier cambio hecho mientras la promoción está `Pausada` (agregar o
  quitar una regla, o editar el conjunto de variantes de una regla existente), el sistema DEBE
  aplicar las mismas validaciones que al crear: bloqueo de solapamiento con otra regla vigente
  que comparta variante y ventana de vigencia (spec 063 FR-014, FR-014a), conjuntos disjuntos
  entre las reglas de la misma promoción (spec 063 FR-001a), y bloqueo de precio de paquete que
  no represente un descuento real (spec 063 FR-016).
- **FR-017**: Al reactivar una promoción cuyas reglas fueron modificadas en estado `Pausada`
  (conjunto editado, regla agregada o regla quitada), el sistema DEBE calcular el descuento
  sobre las reglas y el conjunto vigentes al momento de la venta, sin conservar ningún rastro de
  la configuración anterior a la edición.
- **FR-018**: Una promoción DEBE conservar como mínimo una regla en todo momento; el sistema NO
  DEBE permitir quitar la última regla restante de una promoción en estado `Pausada` (ni en
  ningún otro estado). Para eliminar la promoción por completo, el administrador debe
  finalizarla (spec 063 FR-015).

### Key Entities *(include if feature involves data)*

- **Promoción**: agrupa una o más `Regla`, comparte estado (`Borrador`/`Activa`/`Pausada`/
  `Finalizada`) y vigencia (fecha, días, horas). Definida en spec 063.
- **Regla**: pertenece a una `Promoción`; define tipo (precio de paquete / descuento %), valor,
  cantidad mínima y su propio conjunto de variantes. Esta spec no agrega atributos nuevos: solo
  cambia cuándo es editable su conjunto y cómo se resume y se selecciona.
- **Variante de producto**: unidad elegible dentro del conjunto de una regla; tiene nombre y
  precio normal, y pertenece a una categoría.
- **Categoría**: agrupa variantes de producto; se usa aquí solo como filtro de búsqueda, sin
  cambio de modelo.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las reglas con al menos un producto seleccionado muestran, en su
  resumen, el nombre de cada producto de su conjunto — ninguna queda con una condición genérica
  sin nombre.
- **SC-002**: Con un catálogo de más de 100 variantes, un administrador encuentra y agrega un
  producto específico al conjunto de una regla en menos de 10 segundos, sin necesidad de
  scrollear una lista con todo el catálogo.
- **SC-003**: El listado de productos bajo una regla nunca muestra más filas que la cantidad de
  productos actualmente seleccionados en esa regla.
- **SC-004**: El 100% de las reglas agregadas quedan visibles en la primera posición del
  listado, sin necesidad de scroll, inmediatamente después de crearlas.
- **SC-005**: Un administrador corrige las reglas de una promoción vigente (agregar o quitar una
  regla, o cambiar el conjunto de productos de una regla existente) pausándola y editándola, sin
  necesitar duplicarla ni recrearla, en el 100% de los casos donde el cambio necesario no
  requiere editar el tipo, el valor o la cantidad mínima de una regla ya existente in situ.

## Assumptions

- El formato de resumen de regla (FR-002, FR-003) aplica a los dos tipos de regla vigentes tras
  la spec 063 (precio de paquete y descuento por porcentaje); no hay un tercer tipo activo hoy.
- El criterio de listar nombres de variante en orden alfabético, con tope de tres y resumen "y N
  más" (FR-004), reutiliza exactamente el ya validado y clarificado en la spec 066 para el menú
  QR, para no introducir una segunda convención de redacción en la misma promoción.
- El alcance de "modificar cualquier producto" (pedido del usuario) se resolvió en
  Clarifications: en `Pausada` incluye agregar y quitar reglas completas, además de editar el
  conjunto de variantes de una regla existente. El tipo, el valor y la cantidad mínima de una
  regla que ya existía al pausar siguen sin poder editarse in situ — la vía es quitarla y
  agregar una nueva, o duplicar la promoción completa (FR-015).
- Pausar una promoción para editar su conjunto no dispara ninguna notificación adicional al
  comensal o al cajero más allá de que la promoción deja de aplicar descuento mientras está
  `Pausada` (comportamiento ya vigente desde la spec 063, sin cambio).
- El botón "Agregar visibles" se conserva como acción de alta masiva sobre los resultados de
  búsqueda filtrados, en vez de eliminarse, porque sigue siendo útil para conjuntos amplios
  (p. ej. "toda la categoría Granizados") una vez que la búsqueda ya no lista el catálogo
  completo por defecto.
