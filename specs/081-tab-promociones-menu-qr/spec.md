# Feature Specification: Pestaña dedicada de Promociones en el menú QR

**Feature Branch**: `081-tab-promociones-menu-qr`

**Created**: 2026-09-12

**Status**: Draft

> **Revisión 2026-09-12 (tras probar la implementación en un entorno real)**: el dueño reportó
> dos hallazgos al usar la pestaña recién construida: (1) un defecto — la pestaña "Promociones" y
> la primera categoría aparecían resaltadas como activas al mismo tiempo; (2) una regla de negocio
> que faltaba especificar — si hay promociones vigentes al entrar al menú, deben ser lo primero
> que ve el comensal (pestaña "Promociones" seleccionada por defecto), no la primera categoría.
> El defecto (1) se corrige sin cambiar el spec (era un error de la vista, no de comportamiento
> deseado). La regla (2) se agrega como **FR-014**, nueva.
>
> **Revisión 2026-09-12 (segunda ronda de prueba)**: el dueño reportó que, al abrir el modal de
> selección desde "Promociones" para un producto con una sola presentación, el encabezado seguía
> mostrando el precio de lista ($15.000) mientras el botón "Agregar" ya mostraba correctamente el
> total con descuento ($7.000) — un mismo modal con dos precios distintos para la misma
> operación. Se agrega **FR-015** para que el encabezado, dentro de este flujo, muestre siempre
> el mismo total que se va a cobrar.
>
> El dueño pidió además, en la misma revisión, que la fila de la presentación deje de mostrar el
> precio por unidad ("$3.500 c/u") y en su lugar tache el precio de lista junto al precio del
> paquete — se agrega **FR-016**.
>
> **Revisión 2026-09-12 (tercera ronda)**: el dueño encontró el mismo producto por el buscador
> (no por la pestaña "Promociones"), subió la cantidad manualmente hasta 2 (el mínimo de la
> regla) y el encabezado/la fila seguían sin mostrar el descuento — la primera versión de FR-015/
> FR-016 solo activaba el precio con descuento cuando el modal se abría **desde la pestaña**, no
> cuando la cantidad configurada por cualquier otra vía ya cumplía el mínimo. Se corrige FR-015 y
> FR-016 para que dependan únicamente de si la cantidad ya satisface la regla — nunca de por
> dónde se llegó al modal. El paso forzado de cantidad de 2 en 2 (FR-004 a FR-006) **no cambia**:
> sigue exclusivo de la pestaña "Promociones", tal como se decidió en la sesión de clarificación
> — esta corrección es solo de precio mostrado, no de cómo se mueve la cantidad.

**Input**: User description: "actualmente en el menu qr, las promociones se habilitan en el mismo card de las tarjetas de los procductos, es confunde al usuario porque primero no sabe que unidades debe comprar para aplicar a la promocion y segundo, debe buscar entre varios productos para encontrar los productos que tienen promocion, lo que me gustaria implementar es una nueva opcion en el menu de navegacion que diga promociones y que cuando el usuario precione ahi entonces se muestren unicamente los productos en promocion, y que al agregar el producto al carrito, se agregue de acuerdo a la promocion, en caso de promociones por precio de paquete, el producto se debe agregar de acuerdo a las unidades minimas establecidas, por ejemplo 2 x 17 mil entonces al agregar el producto al carrito siempre se agregan de 2 en 2 aplicando las reglas de descuento"

## Clarifications

### Session 2026-09-12

- Q: Cuando una regla de promoción cubre varios productos o variantes distintos que se pueden combinar entre sí (como "Llevando 2 entre Mediana y Pequeña pagas $12.000"), ¿el incremento forzado en la pestaña "Promociones" debe aplicarse por producto individual, o debe permitir combinar distintos productos del mismo grupo para completar el mínimo? → A: Incremento forzado solo por producto individual; las combinaciones mixtas entre productos distintos quedan fuera de alcance de esta pestaña (se documenta como limitación conocida; esa mezcla sigue disponible agregando cada producto desde su categoría original, sin cambio de comportamiento ahí).
- Q: ¿El incremento forzado por cantidad mínima (FR-004) debe aplicarse también a reglas de tipo "porcentaje" con cantidad mínima mayor a 1, o únicamente a las de "precio de paquete" del ejemplo? → A: Aplicar el incremento a cualquier regla vigente con cantidad mínima > 1, sin importar el tipo (porcentaje o precio de paquete); el problema reportado (no saber cuántas unidades comprar) no depende del tipo de descuento.
- Q: Cuando la cantidad en el carrito es un múltiplo mayor a la cantidad mínima (ej. 4 con mínima 2) y el comensal disminuye una vez, ¿baja un paso completo (4→2) o se retira el producto por completo sin importar la cantidad? → A: Disminuye un paso completo (múltiplo de la cantidad mínima) en cada pulsación, simétrico al incremento; solo se retira del carrito al llegar a 0.
- Q: Si el producto ya está en el carrito con una cantidad que no es múltiplo de la cantidad mínima (agregada antes desde su categoría normal) y el comensal abre esa tarjeta desde "Promociones", ¿el control debe ajustar automáticamente la cantidad al múltiplo más cercano hacia arriba, o dejarla como está hasta la siguiente pulsación? → A: Ajustar automáticamente al múltiplo más cercano hacia arriba en cuanto se abre la tarjeta desde "Promociones".

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Encontrar todos los productos en promoción sin buscar por categoría (Priority: P1)

Un comensal abre el menú QR de la mesa y quiere saber qué productos tienen promoción vigente en este momento, sin tener que revisar categoría por categoría buscando la insignia de promoción en cada tarjeta.

**Why this priority**: Es el problema principal reportado: hoy las promociones están dispersas entre las tarjetas de producto de distintas categorías y el comensal debe buscarlas una por una. Sin esta pestaña, el resto de la funcionalidad (agregar respetando las reglas) no tiene dónde vivir.

**Independent Test**: Puede probarse abriendo el menú QR con al menos una promoción vigente sobre productos de distintas categorías, presionando la nueva opción "Promociones" en la navegación, y verificando que se listan únicamente los productos con al menos una variante cubierta por una promoción vigente, sin importar su categoría original.

**Acceptance Scenarios**:

1. **Given** el menú QR tiene productos en varias categorías y solo algunos de ellos tienen una promoción vigente, **When** el comensal presiona la opción "Promociones" en la navegación, **Then** el listado muestra únicamente esos productos, independientemente de su categoría.
2. **Given** el comensal está viendo la pestaña "Promociones", **When** revisa cualquiera de las tarjetas listadas, **Then** puede ver la condición de la promoción (tipo de descuento y cantidad mínima) igual que la vería en la tarjeta del producto dentro de su categoría original.
3. **Given** ninguna promoción está vigente en este momento, **When** el comensal presiona "Promociones", **Then** el sistema le indica claramente que no hay promociones activas, en lugar de mostrar una lista vacía sin explicación.
4. **Given** al menos un producto tiene una promoción vigente, **When** el comensal ingresa al menú QR (o reanuda su sesión), **Then** la pestaña "Promociones" aparece seleccionada de entrada, sin que el comensal tenga que presionarla (FR-014).
5. **Given** ninguna promoción está vigente en este momento, **When** el comensal ingresa al menú QR, **Then** la primera categoría aparece seleccionada de entrada, igual que antes de esta spec (FR-014).

---

### User Story 2 - Agregar al carrito respetando las unidades mínimas de la promoción (Priority: P1)

Un comensal encuentra un producto en la pestaña "Promociones" con una regla de "precio de paquete" (por ejemplo, "2 x $17.000") y quiere agregarlo al carrito sin tener que saber de antemano cuántas unidades necesita comprar para que el descuento aplique.

**Why this priority**: Es el segundo problema reportado: el comensal no sabe cuántas unidades debe comprar para que la promoción se active, y termina agregando una cantidad que no alcanza el mínimo, sin recibir el descuento esperado. Resolverlo es tan crítico como poder encontrar el producto (User Story 1), porque de lo contrario encontrar el producto no evita la confusión original.

**Independent Test**: Puede probarse agregando, desde la pestaña "Promociones", un producto cuya única variante elegible está cubierta por una regla de precio de paquete con cantidad mínima 2, y verificando que la cantidad en el carrito queda en 2 (no en 1) con el precio de paquete ya aplicado.

**Acceptance Scenarios**:

1. **Given** un producto en la pestaña "Promociones" tiene una regla de precio de paquete "2 x $17.000" sobre su única variante, **When** el comensal presiona agregar al carrito por primera vez, **Then** la línea del carrito queda con cantidad 2 y el precio de paquete ya aplicado, no con cantidad 1.
2. **Given** ese mismo producto ya está en el carrito con cantidad 2, **When** el comensal presiona nuevamente el control para aumentar la cantidad, **Then** la cantidad pasa a 4 (incremento de 2 en 2), nunca a 3.
3. **Given** ese mismo producto está en el carrito con cantidad 2, **When** el comensal presiona el control para disminuir la cantidad, **Then** el producto se retira por completo del carrito (no queda en 1 unidad, que no calificaría para el paquete completo).
4. **Given** ese mismo producto está en el carrito con cantidad 4, **When** el comensal presiona una vez el control para disminuir la cantidad, **Then** la cantidad pasa a 2 (un paso completo menos), no a 3 ni se retira el producto del carrito.
5. **Given** un producto en la pestaña "Promociones" tiene una regla vigente con cantidad mínima 1 (de cualquier tipo), **When** el comensal lo agrega al carrito, **Then** se agrega una unidad a la vez, igual que el comportamiento actual, porque una sola unidad ya cumple el mínimo.

---

### User Story 3 - Las categorías normales del menú no cambian su comportamiento (Priority: P2)

Un comensal que navega el menú por categoría (como hace hoy) y agrega un producto que también tiene una promoción, sin haber pasado por la pestaña "Promociones", espera que agregar al carrito funcione exactamente igual que hoy.

**Why this priority**: Protege el comportamiento existente en el resto del menú (categorías normales) mientras se introduce la nueva pestaña, evitando que el cambio de reglas de cantidad se filtre a flujos que el comensal ya conoce y que no fueron parte del problema reportado.

**Independent Test**: Puede probarse agregando al carrito, desde la categoría original de un producto (no desde "Promociones"), un producto con una regla de precio de paquete de cantidad mínima 2, y verificando que se agrega de a una unidad como hoy, mostrando el descuento solo cuando la cantidad acumulada alcance el mínimo.

**Acceptance Scenarios**:

1. **Given** un producto con una regla de precio de paquete de cantidad mínima 2 está visible en su categoría original, **When** el comensal lo agrega al carrito desde esa categoría (no desde "Promociones"), **Then** se agrega una unidad, igual que el comportamiento actual del menú.

---

### Edge Cases

- ¿Qué pasa si una promoción deja de estar vigente (por vencimiento de fecha, horario o cambio de estado) mientras el comensal tiene la pestaña "Promociones" abierta o productos de esa promoción en el carrito? El sistema sigue las mismas reglas de vigencia y recálculo ya existentes para el resto del menú y del carrito; esta funcionalidad no introduce una regla de vigencia distinta.
- ¿Qué pasa si un producto tiene varias variantes y solo algunas están cubiertas por una regla vigente? El producto aparece en "Promociones", y el incremento por cantidad mínima aplica únicamente a la variante (y opciones, si el producto las requiere) que efectivamente está cubierta por la regla; otras variantes del mismo producto se agregan de forma normal.
- ¿Qué pasa si el comensal ya tenía el producto en el carrito con una cantidad que no es múltiplo de la cantidad mínima (por haberlo agregado antes desde su categoría) y luego interactúa con ese mismo producto desde la pestaña "Promociones"? El control de cantidad en la vista "Promociones" ajusta la cantidad al múltiplo de la cantidad mínima más cercano hacia arriba antes de seguir incrementando o decrementando en pasos de esa cantidad mínima.
- ¿Qué pasa si no hay conexión con el catálogo o la promoción se elimina justo cuando el comensal intenta agregarla? Se aplican los mismos mensajes de error y reintentos que hoy existen para cualquier producto no disponible.
- ¿Qué pasa si la regla vigente que cubre un producto también cubre otros productos o variantes distintas combinables entre sí (por ejemplo, "2 entre Mediana y Pequeña")? Dentro de la pestaña "Promociones", el incremento por cantidad mínima se aplica sumando únicamente unidades del mismo producto en esa tarjeta; combinar productos distintos del mismo grupo para completar el mínimo queda fuera del alcance de esta pestaña y sigue disponible agregando cada producto desde su categoría original, sin cambio de comportamiento ahí.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El menú QR MUST incluir una nueva opción "Promociones" en la navegación, junto a las categorías existentes.
- **FR-002**: Al seleccionar "Promociones", el sistema MUST mostrar únicamente los productos que tengan al menos una variante cubierta por una regla de promoción vigente en ese momento, sin importar a qué categoría pertenezcan.
- **FR-003**: Las tarjetas de producto mostradas en "Promociones" MUST conservar la misma información de condición de la promoción (tipo de descuento y cantidad mínima) que ya se muestra hoy en la tarjeta del producto dentro de su categoría original.
- **FR-004**: Cuando el comensal agrega al carrito un producto desde la pestaña "Promociones" cuya variante elegible está cubierta por una regla vigente con cantidad mínima mayor a 1, el sistema MUST agregar o aumentar la cantidad en pasos iguales a esa cantidad mínima (por ejemplo, de 2 en 2 para una regla "2 x $17.000"), sin importar si la regla es de tipo porcentaje o precio de paquete.
- **FR-005**: Cuando la regla vigente que cubre la variante tiene cantidad mínima igual a 1, el sistema MUST agregar el producto de a una unidad, igual que el comportamiento actual.
- **FR-006**: Cuando el comensal disminuye la cantidad de un producto agregado desde "Promociones", el sistema MUST reducirla en pasos completos iguales a la cantidad mínima de su regla vigente (simétrico al incremento de FR-004), retirando el producto del carrito por completo únicamente cuando la cantidad llega a 0 (es decir, al disminuir estando exactamente en la cantidad mínima).
- **FR-007**: El carrito MUST reflejar el precio con el descuento de la promoción aplicado desde el momento en que se agrega el producto desde "Promociones", ya que la cantidad agregada satisface la cantidad mínima requerida.
- **FR-008**: Cuando ninguna promoción está vigente al momento de que el comensal abra la pestaña "Promociones", el sistema MUST mostrarla igualmente en la navegación, pero indicando que no hay promociones activas en este momento, en lugar de una lista vacía sin explicación.
- **FR-009**: Navegar por una categoría existente (fuera de "Promociones") MUST seguir mostrando todos los productos de esa categoría sin filtrar por si tienen o no promoción vigente, igual que hoy.
- **FR-010**: Agregar al carrito un producto desde una categoría existente (no desde "Promociones") MUST conservar el comportamiento actual de cantidad libre, unidad por unidad, sin el incremento por cantidad mínima definido en FR-004.
- **FR-011**: Cuando un producto tiene varias variantes y solo algunas están cubiertas por una regla vigente, el incremento por cantidad mínima (FR-004) MUST aplicar únicamente a la combinación de variante (y opciones, si aplica) efectivamente cubierta por esa regla.
- **FR-012**: Si la cantidad ya presente en el carrito para un producto no es múltiplo de la cantidad mínima de su regla vigente (por haberse agregado antes desde una categoría normal), el control de cantidad en la vista "Promociones" MUST ajustarla al múltiplo de la cantidad mínima más cercano hacia arriba antes de seguir incrementando o decrementando en esos pasos.
- **FR-013**: Cuando la regla vigente que cubre un producto también cubre otros productos o variantes distintas combinables entre sí, el incremento por cantidad mínima definido en FR-004 MUST aplicarse sumando únicamente unidades del mismo producto en esa tarjeta; el sistema MUST NOT forzar ni bloquear la combinación de productos distintos del mismo grupo para completar el mínimo dentro de la pestaña "Promociones" — esa combinación sigue disponible agregando cada producto desde su categoría original.
- **FR-014**: Al ingresar al menú QR (o reanudar sesión), si existe al menos un producto en promoción vigente, el sistema MUST seleccionar automáticamente la pestaña "Promociones" como vista inicial, en vez de la primera categoría — es lo primero que debe ver el comensal si hay algo en promoción. Si no hay ninguna promoción vigente en ese momento, el sistema MUST conservar el comportamiento actual (primera categoría seleccionada). Esta selección inicial no se repite en refrescos posteriores del menú dentro de la misma sesión, solo al entrar.
- **FR-015**: Dentro del modal de selección, cuando la cantidad configurada para la presentación elegida ya satisface la cantidad mínima de su regla vigente (sin importar si el modal se abrió desde "Promociones", desde una categoría o desde el buscador), el precio destacado del encabezado MUST reflejar el total que efectivamente se cobrará (con el descuento aplicado), no el precio de lista de una sola unidad — evita que el encabezado muestre un precio distinto (y más alto) que el del botón de confirmar, justo debajo de la misma pantalla. Mientras la cantidad no alcance el mínimo, el encabezado MUST seguir mostrando el precio de lista, sin cambio.
- **FR-016**: Dentro del modal de selección, en la fila de la presentación actualmente elegida, si la cantidad configurada ya satisface la cantidad mínima de su regla vigente, el sistema MUST mostrar el precio de lista tachado junto al precio del paquete completo, y la condición corta sin el desglose "por unidad" — igual que FR-015, sin importar cómo se llegó al modal. Mientras no se alcance el mínimo (o para cualquier otra presentación no elegida), la fila MUST seguir mostrando el precio y la condición completa de siempre, sin cambio.

### Key Entities

- **Producto en promoción**: producto del catálogo que tiene al menos una variante cubierta por una regla de promoción vigente (según el estado, fechas, días y horario ya definidos para las promociones existentes); es el criterio que determina qué aparece en la pestaña "Promociones".
- **Pestaña "Promociones"**: nueva opción de navegación del menú QR que filtra el catálogo mostrado a únicamente los productos en promoción, en lugar de agruparlos por categoría.
- **Cantidad mínima de la regla**: valor ya existente en cada regla de promoción (definido en la administración de promociones) que esta funcionalidad usa como el paso de incremento/decremento al agregar el producto desde la pestaña "Promociones".

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un comensal puede ver todos los productos con promoción vigente con una sola selección en la navegación (o sin ninguna, si ya hay promociones vigentes al entrar — FR-014), sin necesidad de revisar cada categoría del menú.
- **SC-002**: El 100% de las líneas de carrito creadas desde la pestaña "Promociones" para reglas con cantidad mínima mayor a 1 quedan en una cantidad que es múltiplo exacto de esa cantidad mínima.
- **SC-003**: El 100% de los productos agregados desde la pestaña "Promociones" muestran el precio promocional aplicado en el carrito desde el primer agregado, sin requerir que el comensal ajuste manualmente la cantidad para alcanzar el descuento.
- **SC-004**: Agregar productos desde categorías existentes (fuera de "Promociones") mantiene el mismo comportamiento de cantidad y el mismo resultado de descuento que antes de esta funcionalidad, sin regresiones reportadas.

## Assumptions

- Se reutiliza sin cambios el criterio y la información de vigencia, tipo de regla y cantidad mínima ya definidos por el modelo de promociones existente (promoción → regla → conjunto de variantes); esta funcionalidad no modifica el motor de cálculo de descuentos, solo agrega una vista de navegación filtrada y una regla de incremento de cantidad al agregar desde esa vista.
- El incremento/decremento por cantidad mínima descrito en FR-004 a FR-006 aplica únicamente cuando el producto se agrega o se ajusta desde la pestaña "Promociones". Agregar el mismo producto desde su categoría original conserva el comportamiento libre de cantidad ya existente (FR-010), para no introducir un cambio de comportamiento no solicitado en el resto del menú.
- La pestaña "Promociones" se ubica junto a las categorías existentes en la navegación del menú (no las reemplaza), y permanece visible aunque no haya promociones vigentes en el momento (FR-008).
- Si un producto requiere selección de variante u opciones antes de agregarse (como ya ocurre hoy en el menú), esa selección ocurre igual desde la pestaña "Promociones"; el incremento por cantidad mínima se aplica después de esa selección, sobre la combinación específica cubierta por la regla vigente.
- No está en el alcance de esta funcionalidad: cambiar cómo se crean o administran las promociones, ni el motor que calcula los descuentos (ya cubiertos por especificaciones anteriores); tampoco cambiar el orden o diseño visual de las tarjetas de producto, más allá de mostrarlas dentro de esta nueva vista filtrada.
- Cuando una regla vigente cubre varios productos o variantes combinables entre sí, esta funcionalidad no ofrece dentro de la pestaña "Promociones" una forma de combinarlos para completar la cantidad mínima compartida; esa combinación sigue funcionando como hoy, agregando cada producto desde su categoría original (FR-013).
