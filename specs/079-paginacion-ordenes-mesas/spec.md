# Feature Specification: Paginación y filtros en Órdenes y Mesas

**Feature Branch**: `079-paginacion-ordenes-mesas`

**Created**: 2026-09-07

**Status**: Draft

**Input**: User description: "implementar paginacion en los siguientes modulos (revisar tanto el backend como el frontend): ordenes, modulo de mesas. Adicional a eso en el modulo de ordenes eliminar la opcion de filtrado por Bloqueadas, tambien permitir filtrar por estado y tipo de orden y que el tipo de orden sea visible en la lista de ordenes."

## Aclaraciones

### Sesión 2026-09-07

- P: El listado de órdenes lo consumen la pantalla "Órdenes", el Dashboard y la Terminal de Mesas (esta en tiempo real y necesita todas las órdenes activas). ¿A qué afecta la paginación? → R: Paginación "opt-in": solo la pantalla "Órdenes" pide resultados paginados; el Dashboard y la Terminal de Mesas siguen recibiendo la lista completa sin cambios.
- P: ¿Qué pantallas cubre "módulo de mesas"? → R: Solo la pantalla "Mesas" (`/dashboard/mesas`), el listado y la gestión de mesas y sus QR. La "Terminal de mesas" (tablero de sesiones) queda fuera.
- P: ¿Cómo se comportan los filtros de estado y de tipo de orden? → R: Dos filtros independientes, cada uno de selección única (con opción "Todos") y combinables entre sí; se resuelven en el servidor junto con la paginación.
- P: Al quitar el filtro "Bloqueadas", ¿qué pasa con las órdenes en ese estado? → R: Se quita solo el botón de acceso rápido; las órdenes "bloqueada" siguen visibles en el listado sin filtro y se pueden ver eligiendo "Bloqueada" en el filtro de estado.
- P: Cuando la página solicitada ya no existe (un filtro redujo el total, o se cancelaron/borraron órdenes mientras se navegaba), ¿a qué página válida debe saltar la pantalla? → R: Se recoloca en la última página con resultados (clamp al máximo válido); si el conjunto quedó vacío, la página 1. El cambio de filtro o de tamaño de página sigue llevando a la página 1 (FR-004).
- P: Cuando varias órdenes comparten exactamente la misma fecha y hora de creación, ¿en qué orden secundario deben quedar para que las páginas sean estables? → R: Desempate por el identificador interno de la orden en descendente. Ese identificador es estable pero no cronológico, así que dentro de un mismo instante de creación el orden entre esas pocas filas es arbitrario; lo que se garantiza es que, para un conjunto fijo, la secuencia total es determinista y verificable página a página, sin huecos ni repeticiones (SC-008).
- P: ¿Los seis valores del filtro de estado (Todos, Por confirmar, Abierta, Bloqueada, Pagada, Cancelada) son exactamente los que debe poder elegir la persona? → R: Sí; esos seis son el conjunto completo del filtro de estado, sin añadir ni quitar ninguno.
- P: Además de la Terminal de Mesas y el Dashboard, ¿hay otras pantallas (p. ej. "Pagos por confirmar") que consuman el listado de órdenes y deban seguir recibiéndolo completo? → R: No. Solo la Terminal de Mesas y el Dashboard consumen ese listado; "Pagos por confirmar" y las demás pantallas obtienen sus datos por otra vía y no se ven afectadas por la paginación.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Navegar las órdenes por páginas (Priority: P1)

Una persona del personal (cajero, administrador) abre la pantalla "Órdenes" para revisar la operación. Hoy la pantalla carga y pinta **todas** las órdenes históricas de una sola vez; con miles de órdenes la carga es lenta y el listado es inmanejable. Con esta funcionalidad la pantalla muestra un bloque acotado de órdenes (una "página"), con controles para elegir cuántas ver por página y para avanzar o retroceder, y un indicador de en qué página está y cuántos resultados hay en total.

**Why this priority**: Es el problema central que motiva la solicitud. Sin paginación la pantalla se degrada de forma continua a medida que el negocio acumula órdenes y ya es el listado que más crece del sistema. Entrega valor por sí sola aunque no se toquen los filtros.

**Independent Test**: Con varias decenas de órdenes en el sistema, abrir "Órdenes" y verificar que solo se muestra el tamaño de página elegido, que "Anterior/Siguiente" recorren todo el conjunto, que el total y el número de páginas son correctos, y que la carga inicial ya no depende del total de órdenes.

**Acceptance Scenarios**:

1. **Given** 130 órdenes en el sistema y el tamaño de página por defecto (20), **When** abro la pantalla "Órdenes", **Then** veo las 20 órdenes más recientes, "Página 1 de 7" y el total (130).
2. **Given** que estoy en la página 1, **When** pulso "Siguiente", **Then** veo las 20 órdenes siguientes en orden de fecha de creación descendente y el indicador pasa a "Página 2 de 7".
3. **Given** que estoy en la página 3, **When** cambio el tamaño de página a 50, **Then** vuelvo a la página 1 y veo 50 órdenes.
4. **Given** que estoy en la última página con 10 resultados y el resto de páginas llenas, **When** la abro, **Then** veo solo esas 10 órdenes y "Siguiente" queda deshabilitado.
5. **Given** una pantalla "Órdenes" recién abierta sin filtros, **When** se carga, **Then** el conjunto y el orden (fecha de creación, más recientes primero) son los mismos que hoy, solo que segmentados en páginas.

---

### User Story 2 - Filtrar las órdenes por estado y por tipo, con el tipo visible (Priority: P2)

La misma persona necesita acotar el listado: "ver solo las órdenes para llevar", "ver solo las canceladas", "ver las abiertas a domicilio". Hoy solo existe un grupo de botones de estado (incluido uno de "Bloqueadas") y el tipo de orden no aparece en la lista: hay que abrir cada orden para saber si es en mesa, para llevar o a domicilio. Con esta funcionalidad hay dos filtros —estado y tipo de orden—, cada fila muestra el tipo, y el acceso rápido "Bloqueadas" desaparece.

**Why this priority**: Amplifica el valor de la paginación (encontrar un subconjunto en vez de pasar páginas) y responde a peticiones explícitas del negocio, pero la pantalla ya es utilizable con solo la Historia 1.

**Independent Test**: Con órdenes de varios estados y tipos, aplicar cada filtro por separado y combinados, y comprobar que el listado, el total y las páginas reflejan exactamente el subconjunto; comprobar que cada fila muestra el tipo de orden y que ya no existe el botón "Bloqueadas".

**Acceptance Scenarios**:

1. **Given** órdenes de todos los tipos, **When** selecciono el tipo "Para llevar", **Then** el listado muestra solo órdenes para llevar y el total/páginas se recalculan para ese subconjunto.
2. **Given** el filtro de tipo en "Para llevar", **When** selecciono además el estado "Abierta", **Then** el listado muestra solo órdenes para llevar que además están abiertas.
3. **Given** cualquier filtro aplicado en la página 4, **When** cambio el filtro, **Then** vuelvo a la página 1 del nuevo resultado.
4. **Given** la lista de órdenes, **When** la observo, **Then** cada fila indica el tipo de orden (En mesa / Para llevar / Domicilio) sin necesidad de abrir el detalle.
5. **Given** la pantalla "Órdenes", **When** miro los filtros de estado, **Then** ya no existe el botón "Bloqueadas", pero "Bloqueada" sigue disponible como opción del filtro de estado y las órdenes bloqueadas siguen apareciendo cuando no hay filtro de estado.
6. **Given** el filtro de estado en "Pagada", **When** se aplica, **Then** aparecen las órdenes que la persona ve como "Pagada" en la lista —las que ya tienen venta emitida aunque su estado interno siga en "abierta"/"bloqueada", y las que ya están en estado interno "pagada"—, y nunca las canceladas.
7. **Given** una orden histórica sin tipo registrado, **When** filtro por un tipo concreto, **Then** esa orden no aparece; **When** el filtro de tipo está en "Todos", **Then** sí aparece y su fila muestra "Sin especificar".

---

### User Story 3 - Navegar las mesas por páginas (Priority: P3)

Un administrador abre la pantalla "Mesas" para gestionar las mesas del local y sus códigos QR. En locales con muchas mesas el listado completo es largo de recorrer. Con esta funcionalidad la pantalla "Mesas" muestra las mesas por páginas, con el mismo control de tamaño y navegación que "Órdenes".

**Why this priority**: El volumen de mesas crece mucho menos que el de órdenes y está acotado por el plan del tenant, así que el problema es menor y menos urgente. Aun así aporta consistencia y alivia los locales grandes.

**Independent Test**: Con más mesas que el tamaño de página, abrir "Mesas" y verificar que solo se muestra una página, que la navegación recorre todas las mesas en orden de número ascendente, y que crear/editar/activar/cambiar estado de una mesa sigue funcionando y deja al usuario en una vista coherente.

**Acceptance Scenarios**:

1. **Given** 64 mesas y el tamaño de página por defecto (20), **When** abro "Mesas", **Then** veo 20 mesas ordenadas por número ascendente y "Página 1 de 4".
2. **Given** que estoy en la página 2 de "Mesas", **When** cambio el estado operativo de una mesa, **Then** la operación se aplica y sigo viendo una página válida del listado.
3. **Given** que estoy en "Mesas", **When** creo una mesa nueva, **Then** la lista se actualiza y la mesa aparece en la página que le corresponde por su número.
4. **Given** la pantalla "Mesas" recién abierta, **When** se carga, **Then** el conjunto y el orden (número ascendente) son los mismos que hoy, solo que segmentados en páginas.

---

### Edge Cases

- **Página fuera de rango**: si la página solicitada ya no existe (se aplicó un filtro que reduce el total, se borraron/cancelaron datos, o el estado apunta a una página vieja), el sistema recoloca al usuario en la última página con resultados (o la página 1 si el conjunto quedó vacío) sin error.
- **Combinación de filtros sin resultados**: el listado muestra un estado vacío claro ("No hay órdenes con estos filtros") y los controles de página reflejan 0 resultados / 0 páginas.
- **Tamaño de página grande con pocos resultados**: el selector de "por página" sigue siendo accesible aunque solo haya una página (comportamiento ya resuelto por el control de paginación compartido).
- **Órdenes sin tipo de orden (históricas)**: se muestran como "Sin especificar" y quedan fuera al filtrar por un tipo concreto.
- **Discrepancia entre estado interno y estado mostrado**: una orden con venta emitida pero estado interno "abierta" debe aparecer bajo el filtro "Pagada" (lo que ve la persona), no bajo "Abierta".
- **Órdenes en estado "bloqueada"**: nunca desaparecen de la pantalla "Órdenes" respecto al comportamiento actual; solo deja de existir el botón de acceso rápido.
- **Otros consumidores del listado**: la Terminal de Mesas (tiempo real) y el Dashboard siguen recibiendo todas las órdenes / todas las mesas activas; la paginación no los afecta.
- **Cambio de tamaño de página mientras se navega**: recalcula el número de páginas y reposiciona al usuario en la primera página.

## Requirements *(mandatory)*

### Functional Requirements

#### Paginación de Órdenes

- **FR-001**: La pantalla "Órdenes" MUST presentar las órdenes segmentadas en páginas, con un tamaño de página por defecto de 20 y opciones seleccionables de 10, 20, 50 y 100.
- **FR-002**: La pantalla MUST ofrecer navegación "Anterior/Siguiente", un indicador "Página X de Y" y el número total de resultados del conjunto mostrado.
- **FR-003**: El orden de las órdenes MUST ser por fecha de creación descendente (más recientes primero) y, ante fechas de creación iguales, por el identificador interno de la orden descendente como desempate estable (no cronológico), de modo que la paginación en servidor sea determinista y verificable página a página. El conjunto y el orden principal (fecha de creación) se conservan respecto a la pantalla actual.
- **FR-004**: Cambiar cualquier filtro o el tamaño de página MUST devolver al usuario a la primera página del nuevo resultado.
- **FR-005**: Si la página solicitada queda fuera de rango, el sistema MUST recolocar al usuario en la última página con resultados (clamp al máximo válido) y mostrarla sin error; si el conjunto quedó vacío, MUST mostrar la página 1 con el estado vacío. El cambio de filtro o de tamaño de página sigue llevando a la página 1 (FR-004).
- **FR-006**: La segmentación en páginas MUST resolverse en el servidor: la pantalla "Órdenes" no descarga el conjunto completo de órdenes para paginar en el cliente.
- **FR-007**: El total de resultados y el número de páginas MUST corresponder al conjunto ya filtrado (no al total sin filtrar).

#### Filtros de Órdenes

- **FR-008**: La pantalla "Órdenes" MUST permitir filtrar por estado de la orden, con exactamente estas seis opciones (conjunto completo, no se añaden ni se quitan otras): Todos, Por confirmar (recibida), Abierta, Bloqueada, Pagada, Cancelada.
- **FR-009**: La pantalla "Órdenes" MUST permitir filtrar por tipo de orden, con las opciones: Todos, En mesa, Para llevar, Domicilio.
- **FR-010**: Cada filtro (estado y tipo) MUST ser de selección única e incluir la opción "Todos" (sin filtrar por ese eje).
- **FR-011**: Los dos filtros MUST poder combinarse: el resultado cumple simultáneamente el estado y el tipo seleccionados.
- **FR-012**: Los filtros MUST aplicarse en el servidor y combinarse con la paginación.
- **FR-013**: El estado por defecto de la pantalla al abrirla MUST ser sin filtrar (Todos / Todos), mostrando el mismo conjunto que hoy (segmentado en páginas).
- **FR-014**: El grupo de botones de acceso rápido de estado MUST eliminarse de la pantalla "Órdenes": el acceso rápido "Bloqueadas" no se reintroduce en ninguna forma y los demás estados pasan al filtro desplegable de estado (FR-008). Entre el despliegue de la Historia 1 y el de la Historia 2 la pantalla puede quedar sin filtro de estado; ese estado transitorio se registra en la decisión de negocio (`A-72`).
- **FR-015**: Las órdenes en estado "bloqueada" MUST seguir apareciendo en el listado cuando no hay filtro de estado, y MUST poder consultarse seleccionando "Bloqueada" en el filtro de estado.
- **FR-016**: El filtro por estado MUST coincidir con el estado que la persona ve en cada fila: "Pagada" selecciona las órdenes que la persona ve como pagadas —las que tienen venta emitida y las que ya están en estado interno "pagada"—, salvo las canceladas; "Cancelada" selecciona las canceladas; los demás valores seleccionan órdenes en ese estado que aún no tienen venta emitida.

#### Tipo de orden visible

- **FR-017**: Cada fila de la lista de "Órdenes" MUST mostrar el tipo de orden (En mesa / Para llevar / Domicilio).
- **FR-018**: Una orden sin tipo de orden registrado MUST mostrarse con una etiqueta explícita ("Sin especificar") y MUST quedar excluida de los resultados al filtrar por un tipo concreto.

#### Paginación de Mesas

- **FR-019**: La pantalla "Mesas" (`/dashboard/mesas`) MUST presentar las mesas segmentadas en páginas, con el mismo control de tamaño y navegación que "Órdenes" y el mismo tamaño por defecto (20).
- **FR-020**: El orden de las mesas MUST ser por número de mesa ascendente, conservando el orden actual de la pantalla.
- **FR-021**: La segmentación en páginas de "Mesas" MUST resolverse en el servidor.
- **FR-022**: Las operaciones de la pantalla "Mesas" (crear, editar, activar/desactivar, cambiar estado operativo, ver/imprimir QR) MUST seguir funcionando y, al completarse, MUST dejar al usuario en una vista coherente del listado (la misma página cuando sea posible).

#### Compatibilidad y no regresión

- **FR-023**: La paginación MUST ser "opt-in": solo las pantallas "Órdenes" y "Mesas" solicitan resultados paginados.
- **FR-024**: Los únicos consumidores del listado compartido de órdenes que hoy necesitan todas las órdenes son la Terminal de Mesas en tiempo real y el Dashboard; ambos MUST seguir recibiendo el conjunto completo con el mismo comportamiento actual, incluida la semántica del filtro de "órdenes de sesiones activas". Ninguna otra pantalla consume ese listado compartido: "Pagos por confirmar" y las demás vistas de revisión de cobro obtienen sus datos por otra vía y MUST NOT verse afectadas por la paginación.
- **FR-025**: Los demás consumidores del listado de mesas que hoy necesitan todas las mesas —Terminal de Mesas, Dashboard, resolución de etiqueta de mesa en "Órdenes" y en el detalle de orden, hoja de impresión de códigos QR— MUST seguir recibiendo el conjunto completo con el mismo comportamiento actual.
- **FR-026**: Esta funcionalidad MUST NOT cambiar el ciclo de vida de la orden, el significado del estado "bloqueada", ni el modelo de datos de órdenes o mesas.

### Key Entities *(include if feature involves data)*

- **Orden**: comanda de la operación. Atributos relevantes para esta funcionalidad: estado del ciclo de facturación, tipo de orden (en mesa / para llevar / domicilio, puede faltar en órdenes históricas), fecha de creación (criterio de orden), y si ya tiene una venta emitida o su estado interno ya es "pagada" (cualquiera de las dos hace que la persona la vea como "Pagada").
- **Mesa**: mesa física del local. Atributos relevantes: número (criterio de orden), nombre, estado operativo, si está activa.
- **Página de resultados**: bloque acotado de un listado. Atributos: elementos de la página, total de resultados del conjunto, página actual, tamaño de página, número de páginas.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: En la pantalla "Órdenes", el tiempo hasta ver la primera página se mantiene por debajo de 2 segundos aunque el sistema acumule 5.000 o más órdenes (hoy ese tiempo crece con el total).
- **SC-002**: La pantalla "Órdenes" nunca transfiere ni pinta más órdenes que el tamaño de página seleccionado.
- **SC-003**: Una persona puede aislar un subconjunto concreto (p. ej. "órdenes para llevar canceladas") en 3 interacciones o menos y ver de inmediato cuántas hay.
- **SC-004**: El tipo de orden es visible en el 100 % de las filas de la lista sin abrir el detalle.
- **SC-005**: La pantalla "Mesas" carga en menos de 2 segundos con 500 mesas registradas.
- **SC-006**: Cero regresiones en la Terminal de Mesas y en el Dashboard: siguen mostrando todas las órdenes y mesas activas que muestran hoy.
- **SC-007**: Ninguna orden en estado "bloqueada" deja de ser accesible desde la pantalla "Órdenes" respecto al comportamiento actual.
- **SC-008**: Los resultados y el orden de ambas pantallas, sin filtros aplicados, son idénticos a los actuales salvo por la segmentación en páginas (verificable comparando el conjunto completo página a página).

## Assumptions

- Se reutiliza el patrón de paginación ya existente en el sistema (respuesta paginada con total y número de páginas en el backend; barra de paginación reutilizable y consulta reactiva en el frontend; tamaños 10/20/50/100). No se introduce un patrón nuevo.
- No se añade búsqueda por texto en "Órdenes" ni en "Mesas" como parte de esta funcionalidad.
- No se añaden filtros nuevos a la pantalla "Mesas" (solo paginación); su presentación actual (número, nombre, estado, acciones) no cambia.
- No se persisten entre visitas los filtros ni la página elegida; cada vez que se abre la pantalla arranca sin filtros en la página 1. (Se puede evaluar como mejora aparte.)
- El filtro por tipo de orden se refiere a cómo se atiende la orden (en mesa / para llevar / domicilio) y no distingue el canal de origen (mostrador, QR, WhatsApp, API).
- La "Terminal de mesas" (tablero de sesiones, `/dashboard/mesas-sesiones`) queda fuera del alcance.
- Todos los textos visibles nuevos se redactan en español de Colombia.
- El listado de "Órdenes" no es en tiempo real hoy; se refresca al recargar la pantalla, y ese comportamiento se mantiene.

## Impacto sobre el Sistema Existente

### Impacto sobre funcionalidades existentes

- **Pantalla "Órdenes"**: cambia de mostrar todas las órdenes a mostrarlas por páginas; el filtrado pasa a resolverse en el servidor; se retira el grupo de botones de acceso rápido de estado (los estados migran al filtro desplegable de la Historia 2; el acceso rápido "Bloqueadas" no se reintroduce); se añade un filtro por tipo de orden y la columna/etiqueta de tipo en cada fila. Es un cambio de comportamiento deliberado y solicitado por el negocio; debe quedar registrado como decisión de negocio en `specs/000-reconocimiento/registro-de-anomalias.md` con quién y cuándo, antes de implementarse, incluida la ausencia temporal del filtrado por estado si la Historia 1 se despliega antes que la Historia 2.
- **Pantalla "Mesas"**: cambia de mostrar todas las mesas a mostrarlas por páginas. Las operaciones de gestión de mesas no cambian.
- **Terminal de Mesas y Dashboard**: sin cambios de comportamiento. Deben seguir recibiendo el conjunto completo de órdenes/mesas; la paginación es opcional y solo la usan las dos pantallas afectadas.
- **Resolución de etiqueta de mesa** en "Órdenes" y en el detalle de orden, y **hoja de impresión de QR**: sin cambios; siguen consumiendo el listado completo de mesas.

### Impacto sobre datos existentes

- Ninguno. No se crean ni modifican entidades, campos ni relaciones. No hay migración de datos.
- Las órdenes históricas sin tipo de orden registrado se tratan como "Sin especificar" en la presentación y el filtrado; no se rellenan ni se modifican.

### Decisiones de compatibilidad

- El contrato del listado de órdenes y del listado de mesas se mantiene compatible: sin los parámetros de paginación, la respuesta y el comportamiento son los actuales.
- El filtro de estado ya existente en el listado de órdenes conserva su semántica para cualquier consumidor que ya lo use.

## Dependencies

- Requiere que el listado de órdenes exponga (como ya lo hace) el tipo de orden y la señal de "tiene venta emitida" por cada orden.
- Requiere el control de paginación reutilizable y el patrón de listado paginado ya presentes en el sistema (usados hoy en Ventas, Inventario y Auditoría).
