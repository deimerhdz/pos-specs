# Feature Specification: Rediseño responsive de la Terminal de Mesas (escritorio/tablet/móvil)

**Feature Branch**: `076-terminal-mesas-rediseno-responsive`

**Created**: 2026-09-04

**Status**: Draft

**Naturaleza de esta spec**: spec de **rediseño de interfaz con funcionalidad nueva acotada**. A
diferencia de spec 036 (que definió el layout actual: franja superior con pestañas + carrusel
horizontal de una sola fila con flechas, `pos-tables-panel.component.ts:57-102`) y de spec 059 (que
corrigió cuándo se piden datos y agregó tarjetas para pedidos de Domicilio/Para llevar), esta spec
reemplaza el carrusel por una cuadrícula responsive de 3 variantes (escritorio/tablet/móvil, según
las 3 imágenes de referencia en `terminal-mesas/screen.png`, `screen-tablet.png`,
`screen-mobile.png`) y agrega tres piezas de funcionalidad nueva, acotadas explícitamente por el
negocio (ver Clarifications): contadores de ocupación, una barra superior operativa (turno de caja,
reloj, estado de sincronización), y visibilidad de quién atiende cada pedido de mesa ("Atendido
por", sin importar si es Cajero o Mesero) (reutilizando un dato que el backend ya captura hoy pero
que ninguna pantalla expone,
`app/models/customer_order.py:115` en `pos-backend`, campo `user_id`, referencia blanda a
`shared.users.id`, nulo solo cuando el pedido lo envía el cliente por QR).

El resto del comportamiento ya implementado en `table-sessions.component.ts` (paneles central y de
cobro una vez hay una mesa/pedido seleccionado, atajos F2/F3/ESC/Ctrl+P, diálogo de éxito) no
cambia — esta spec solo reordena el contenedor visual alrededor de ese comportamiento y lo hace
responsive.

**Autorización de negocio (Principio I y Principio II de la
[Constitución](../../.specify/memory/constitution.md))**: solicitado directamente por el
dueño/desarrollador del proyecto el 2026-09-04, adjuntando 3 imágenes de diseño de referencia
(escritorio, tablet y móvil) para actualizar la Terminal de Mesas ya implementada, con instrucción
explícita de implementar el diseño tal cual sin cambiar la paleta de colores actual, y de consultar
antes de implementar cualquier funcionalidad nueva detectada en las imágenes. Como resultado de esa
consulta (sesión de aclaración 2026-09-04, ver abajo), el negocio decidió el alcance exacto de las
tres piezas de funcionalidad nueva mencionadas arriba.

**Input**: descripción del usuario (verbatim): "en la carpeta @terminal-mesas hay un diseño que
contempla 3 pantallas, desktop, table y mobile para actualizar en la pantalla de la terminal de
mesas que existe actualmente en el sistema, quiero que implementes el diseño tal cual sin cambiar
la paleta de colores actualmente posible mente incluya nueva funcionalidad, si las detectas me
preguntas para determinar si se implementan o no".

## Clarifications

### Session 2026-09-04

- Q: El diseño muestra "Mozo #2" como dato de la tarjeta de una mesa (además de productos y
  tiempo); hoy el sistema no guarda qué mesero específico gestiona cada pedido. ¿Se implementa? →
  A: **Sí, completo** — se implementa mostrando el mesero asociado a cada pedido de mesa.
- Q: El diseño agrega una barra superior nueva con turno de caja, estado de sincronización, reloj,
  insignia "POS", ícono de candado y botón "Abrir turno de caja" (atajo F1) — nada de esto existe
  hoy en esta pantalla. ¿Qué alcance se implementa? → A: **Todo el alcance, con los botones
  funcionando** de verdad (turno de caja real, reloj real, atajo F1 operativo).
- Q: Dentro de esa barra, ¿qué debe hacer el ícono de candado (no existe ningún concepto de
  "bloquear la terminal" hoy)? → A: **Se quita del alcance** — no se implementa.
- Q: ¿Qué debe hacer/representar la insignia "POS"? → A: **Etiqueta fija, no clickeable** —
  solo indica que la pantalla opera en modo POS/mostrador.
- Q: El mockup móvil muestra el buscador con placeholder "Buscar mesa, mozo o comanda..." — más
  amplio que el buscador actual (solo por mesa). ¿Se amplía? → A: **No** — se mantiene la búsqueda
  solo por mesa, sin cambio funcional del buscador.
- Q: La referencia de "mesero" en la tarjeta (Historia 4), ¿debe mostrarse para cualquier miembro
  del staff que creó el pedido (Cajero o Mesero), o solo cuando el creador tiene específicamente el
  rol "Mesero"? → A: **Siempre** — se muestra quién creó/gestiona el pedido sin importar si es
  Cajero o Mesero, con una etiqueta genérica ("Atendido por"), no exclusiva del rol Mesero.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ver y elegir cualquier mesa sin importar el dispositivo (Priority: P1)

El cajero/mesero abre la Terminal de Mesas desde un computador de escritorio, una tablet o un
celular. Hoy, sin importar el ancho de pantalla, las tarjetas de mesa se muestran siempre en una
sola fila horizontal con flechas de desplazamiento (`pos-tables-panel.component.ts:57-102`, spec
036) — en pantallas angostas esto obliga a desplazarse tarjeta por tarjeta para ver todas las mesas.
Con esta mejora, la grilla se organiza en varias filas que se ajustan al ancho disponible (más
columnas en escritorio, menos en tablet, menos aún en móvil), con scroll vertical natural en vez de
un carrusel horizontal.

**Why this priority**: es el pedido explícito y central de la spec — sin esto, ninguna otra mejora
tiene sentido, porque el resto del rediseño (barra superior, contadores, mesero) se apoya en el
mismo contenedor de grilla que esta historia reemplaza.

**Independent Test**: se puede probar completamente abriendo la Terminal de Mesas en tres anchos de
pantalla distintos (escritorio, tablet, móvil) y verificando que en cada uno se ven varias filas de
tarjetas de mesa sin flechas de desplazamiento horizontal, con la cantidad de columnas que
corresponde a cada uno.

**Acceptance Scenarios**:

1. **Given** la Terminal de Mesas abierta en un ancho de escritorio (≥ 1024px, mismo umbral `lg` ya
   usado hoy en esta pantalla para el hint de atajos, `table-sessions.component.ts:73`), **When** el
   cajero mira la grilla, **Then** ve 4 tarjetas de mesa por fila, envolviendo en varias filas con
   scroll vertical si hay más de 4 mesas.
2. **Given** la Terminal de Mesas abierta en un ancho de tablet (768–1023px, umbral `md` ya usado
   hoy en esta pantalla, `table-sessions.component.ts:54`), **When** el cajero mira la grilla,
   **Then** ve 3 tarjetas de mesa por fila.
3. **Given** la Terminal de Mesas abierta en un ancho de móvil (< 768px), **When** el cajero mira la
   grilla, **Then** ve 2 tarjetas de mesa por fila.
4. **Given** cualquiera de los tres anchos, **When** el cajero selecciona una tarjeta, **Then** el
   comportamiento de selección es exactamente el mismo ya implementado hoy (resalta la tarjeta,
   muestra su detalle) — solo cambia cómo se organizan visualmente las tarjetas, no qué hace
   seleccionarlas.
5. **Given** las pestañas "Domicilios"/"Para llevar" con pedidos pendientes de cobro (spec 059),
   **When** el cajero las abre en cualquier ancho, **Then** esas tarjetas usan la misma grilla
   responsive (misma cantidad de columnas por ancho) en vez del acomodo en fila que usan hoy
   (`pos-tables-panel.component.ts:107-122`).

---

### User Story 2 - Ver de un vistazo cuántas mesas están libres, ocupadas o pendientes (Priority: P1)

El cajero abre la Terminal de Mesas y quiere saber, sin contar tarjeta por tarjeta, cuántas mesas
hay en total y cuántas están libres, ocupadas o pendientes de algo. Hoy esa cuenta no se muestra en
ningún lado de esta pantalla — ni en la franja superior ni en los filtros de ocupación
(`pos-tables-panel.component.ts:43-53`, que solo muestran la etiqueta sin ningún número).

**Why this priority**: mejora directa de orientación operativa sin ninguna dependencia de datos
nuevos — todo el conteo ya existe en la grilla actual, solo no se resume en ningún lado.

**Independent Test**: se puede probar completamente contando manualmente las tarjetas visibles por
estado y verificando que los contadores mostrados (tanto el resumen superior como los filtros)
coinciden exactamente con esa cuenta manual.

**Acceptance Scenarios**:

1. **Given** la pestaña "Mesas" abierta, **When** el cajero mira la parte superior de la pantalla,
   **Then** ve un resumen con el total de mesas y cuántas están ocupadas, libres y pendientes.
2. **Given** ese mismo resumen, **When** el estado de una mesa cambia (por ejemplo, se libera al
   cobrarse), **Then** el resumen se actualiza para reflejar el nuevo conteo, sin necesitar recargar
   la pantalla — mismo mecanismo reactivo que ya actualiza hoy cada tarjeta individual.
3. **Given** los filtros de ocupación ("Todas"/"Libres"/"Ocupadas"/"Pendientes"), **When** el cajero
   los mira, **Then** cada uno muestra junto a su etiqueta la cantidad de mesas que corresponde a
   ese filtro.
4. **Given** el resumen superior y los contadores de los filtros, **When** se comparan entre sí,
   **Then** siempre son consistentes entre ellos (mismo origen de datos) — nunca muestran números
   distintos para el mismo estado.

---

### User Story 3 - Ver y abrir el turno de caja sin salir de la Terminal de Mesas (Priority: P2)

El cajero abre la Terminal de Mesas y quiere saber si tiene un turno de caja abierto, quién es el
cajero de turno, y si no hay turno abierto, poder abrirlo sin ir manualmente al módulo de Caja. Hoy
esta pantalla no muestra ningún dato de turno de caja (el store sí carga `cashShiftId()`, pero solo
lo usa internamente para el cobro, nunca lo presenta) y no ofrece ningún acceso directo para abrir
un turno — abrir un turno hoy solo es posible desde el módulo de Caja
(`cash-session.store.ts:445`, `openShift()` en `cash.service.ts:105`).

**Why this priority**: depende de que exista contenido para mostrar en la barra superior (Historia
1 ya rediseña el contenedor), pero es funcionalidad nueva real, no solo visual — expone datos que
antes eran invisibles y un acceso directo a una acción que antes requería salir de la pantalla.

**Independent Test**: se puede probar completamente abriendo la Terminal de Mesas sin turno de caja
abierto, verificando que aparece la invitación a abrir turno, pulsándola (o con F1) y comprobando
que se abre el mismo flujo de apertura de turno ya implementado en el módulo de Caja; luego,
repitiendo con un turno ya abierto, verificando que en su lugar se muestra el cajero y el turno
activos.

**Acceptance Scenarios**:

1. **Given** ningún turno de caja abierto en la caja registradora de esta terminal, **When** el
   cajero abre la Terminal de Mesas, **Then** la barra superior muestra un botón para abrir turno de
   caja, con el atajo `F1` documentado junto a él.
2. **Given** ese estado, **When** el cajero pulsa ese botón (o presiona `F1`), **Then** se dispara
   el mismo flujo de apertura de turno ya implementado en el módulo de Caja (`openShift()`), sin
   duplicar esa lógica.
3. **Given** un turno de caja ya abierto, **When** el cajero abre la Terminal de Mesas, **Then** la
   barra superior muestra en su lugar el nombre del cajero de turno (mismo dato ya expuesto por
   `CashShift.user_name`) — sin ofrecer el botón de abrir turno mientras ya hay uno activo.
4. **Given** cualquiera de los dos estados anteriores, **When** el cajero mira la barra superior,
   **Then** también ve un reloj con la hora y fecha actuales, y un indicador de si la pantalla está
   en línea y sincronizada con el servidor.
5. **Given** la barra superior, **When** el cajero la mira, **Then** también ve una etiqueta fija
   "POS" (sin acción al pulsarla) que indica que esta pantalla opera en modo POS/mostrador — no hay
   ningún ícono de bloqueo/candado (removido del alcance).

---

### User Story 4 - Ver quién atiende cada pedido de mesa (Priority: P2)

El cajero (o un mesero, spec 075) mira la grilla de mesas y quiere saber, sin abrir el detalle de
cada una, qué compañero está atendiendo el pedido de una mesa ocupada — sin importar si ese
compañero tiene rol Cajero o Mesero. Hoy el pedido de una mesa ya guarda internamente qué usuario lo
creó (`user_id` en `DiningOrder`, `pos-backend`, `app/models/customer_order.py:115` — nulo solo para
pedidos enviados por el cliente vía QR), pero ninguna pantalla del frontend lo expone ni lo muestra.

**Why this priority**: cierra una brecha de visibilidad operativa real entre compañeros de turno,
reutilizando un dato ya capturado — no requiere ninguna migración de base de datos ni campo nuevo,
solo exponerlo y presentarlo.

**Independent Test**: se puede probar completamente creando un pedido de mesa con un usuario de
staff autenticado (Cajero o Mesero), abriendo la Terminal de Mesas (con cualquier usuario con
acceso) y verificando que la tarjeta de esa mesa muestra una referencia a quien lo creó.

**Acceptance Scenarios**:

1. **Given** un pedido de mesa creado por un usuario de staff autenticado (Cajero o Mesero),
   **When** el cajero mira la tarjeta de esa mesa en la grilla, **Then** ve una referencia corta
   ("Atendido por") a ese usuario, sin importar cuál de los dos roles tenga.
2. **Given** un pedido creado por un cliente desde el menú QR (sin usuario de staff asociado,
   `user_id` nulo), **When** el cajero mira su tarjeta, **Then** no se muestra ninguna referencia de
   "Atendido por" (la tarjeta se ve igual que hoy para este caso, sin espacio vacío ni error).
3. **Given** una mesa libre sin ningún pedido, **When** el cajero mira su tarjeta, **Then** tampoco
   se muestra ninguna referencia de "Atendido por" (no aplica).
4. **Given** una mesa con más de un pedido activo (fusión de mesas u otro escenario ya soportado),
   **When** el cajero mira su tarjeta, **Then** se muestra "Atendido por" del pedido principal que
   ya determina hoy el resto de la información resumida de la tarjeta (mismo criterio ya usado para
   elegir qué pedido resume el estado/total de la tarjeta).

---

### User Story 5 - Saber qué hacer cuando no hay ninguna mesa seleccionada (Priority: P3)

El cajero abre la Terminal de Mesas sin haber seleccionado todavía ninguna mesa ni pedido. Hoy, en
ese estado, el panel central muestra el mensaje ya existente de "Mesa libre" o el placeholder del
panel de pedido (según spec 045), y el panel derecho de cobro muestra "Pedido de mostrador" con un
botón para crear un pedido nuevo. Con el rediseño, ese estado inicial se presenta de forma más
completa: un panel de detalle con mensaje de bienvenida, una referencia visible de los atajos de
teclado disponibles, y el botón de crear pedido nuevo siempre visible al final de ese panel.

**Why this priority**: es la mejora de menor impacto funcional de las cinco (no cambia ninguna
regla de negocio, solo la claridad del estado vacío) — puede implementarse de último sin bloquear
ninguna de las anteriores.

**Independent Test**: se puede probar completamente abriendo la Terminal de Mesas sin seleccionar
nada y verificando que el panel de la derecha muestra el mensaje de bienvenida, la lista de atajos
de teclado disponibles y el botón fijo de crear pedido nuevo.

**Acceptance Scenarios**:

1. **Given** la Terminal de Mesas recién abierta sin ninguna mesa ni pedido seleccionado, **When**
   el cajero mira el panel derecho, **Then** ve un mensaje indicando que debe seleccionar una mesa o
   un pedido para ver su detalle.
2. **Given** ese mismo estado, **When** el cajero mira ese panel, **Then** también ve una referencia
   de los atajos de teclado disponibles en esta pantalla (buscar mesa, crear pedido, abrir turno de
   caja — con sus teclas respectivas).
3. **Given** ese mismo estado, **When** el cajero mira la parte inferior del panel, **Then** ve un
   botón fijo "+ Crear pedido nuevo" con su atajo, con el mismo comportamiento ya implementado hoy
   (navega a la creación manual de pedido).
4. **Given** el cajero selecciona una mesa o un pedido, **When** mira ese mismo panel, **Then** dejan
   de mostrarse el mensaje de bienvenida y la lista de atajos, reemplazados por el comportamiento ya
   existente (detalle del pedido y flujo de cobro) — sin cambios de contenido respecto a hoy.

---

### Edge Cases

- **Ancho de pantalla justo en el límite entre dos variantes** (768px o 1024px exactos): se resuelve
  con el mismo criterio de breakpoint ya usado hoy en esta pantalla (`md`/`lg`), sin zona muerta ni
  salto brusco de columnas.
- **Menos mesas que columnas en la fila** (por ejemplo, 2 mesas en una grilla de 4 columnas en
  escritorio): las tarjetas se alinean al inicio de la fila, sin estirarse para llenar el espacio
  sobrante ni dejar huecos con aspecto de error.
- **Pestaña "Domicilios"/"Para llevar" sin ningún pedido pendiente**: sigue mostrando el mismo
  mensaje informativo de listado vacío ya existente (spec 036 FR-003, spec 059 FR-009) — sin
  regresión, y sin contadores (el resumen de ocupación de Historia 2 es exclusivo de la pestaña
  "Mesas", que es la única con el concepto de "libre/ocupada/pendiente").
- **El indicador de sincronización pierde conexión de red a mitad de sesión**: cambia a su estado
  de "sin conexión" sin bloquear la pantalla ni perder la selección actual; vuelve a "en línea"
  automáticamente al recuperar conexión.
- **El cajero pulsa `F1` mientras ya existe un turno de caja abierto**: no ocurre nada (mismo
  criterio que hoy ya aplica a otros atajos que solo actúan si su condición aplica, por ejemplo `F3`
  sin mesa seleccionada).
- **El cajero pulsa `F1` mientras está escribiendo en el buscador u otro campo de texto**: no se
  interpreta como atajo, mismo criterio ya aplicado hoy a los demás atajos de esta pantalla
  (`table-sessions.component.ts:272-273`, chequeo de `typing`).
- **Un pedido de mesa fue creado por un usuario que luego fue eliminado o cambiado de rol**: la
  tarjeta muestra la mejor referencia disponible sin fallar (mismo criterio ya usado hoy para otros
  nombres de usuario que pueden faltar, como `closed_by_user_name`).
- **Dos miembros del staff distintos (Cajero y/o Mesero) agregan productos al mismo pedido de mesa a
  lo largo de su ciclo de vida**: la referencia "Atendido por" mostrada corresponde a quien creó el
  pedido originalmente (mismo dato que ya existe, capturado una sola vez al crear el pedido) — no
  cambia dinámicamente con cada producto agregado después.
- **El resumen de ocupación y los contadores de los filtros deben coincidir siempre**: ambos se
  calculan a partir del mismo conjunto de datos ya cargado (`store.tablesView()`), nunca de fuentes
  separadas que puedan desincronizarse entre sí.
- **Interacción con la carga diferida de spec 059 (Historia 1 de esa spec)**: spec 059 difirió la
  petición del estado del turno de caja hasta que el cajero selecciona un pedido real, precisamente
  porque hasta entonces nadie la necesitaba. Esta spec sí la necesita desde el primer instante (la
  barra superior debe mostrar el turno actual o el botón de abrirlo sin que el cajero tenga que
  seleccionar nada primero, FR-015/FR-017) — la barra superior nueva dispara esa consulta por su
  propia cuenta al montarse (mismo criterio de caché ya usado: solo si `CashService.shift()` todavía
  no tiene valor), sin modificar `PosTerminalStore.init()`/`ensureCheckoutDataLoaded()`. El catálogo
  de métodos de pago (la otra petición que difirió spec 059) sigue diferido sin cambios — la barra
  superior no lo necesita.
- **Un usuario con rol Mesero mira la barra superior**: spec 075 excluye el módulo de Caja del
  alcance del rol Mesero (mapa de acceso). El botón de abrir turno (FR-015) NO se muestra para ese
  rol —solo para Admin/Cajero, que sí tienen acceso a Caja— para no ofrecer una acción que el
  `roleGuard` ya existente rebotaría de inmediato; un Mesero sí ve el nombre del cajero de turno
  cuando ya hay uno abierto (FR-017), igual que cualquier otro rol.

## Requirements *(mandatory)*

### Functional Requirements — Grilla responsive (reemplaza el carrusel)

- **FR-001**: El sistema DEBE reemplazar el carrusel horizontal de una sola fila (con flechas de
  desplazamiento) usado hoy para las tarjetas de mesa por una cuadrícula que envuelve en varias
  filas con scroll vertical.
- **FR-002**: En ancho de escritorio (≥ 1024px), la cuadrícula DEBE mostrar 4 tarjetas por fila.
- **FR-003**: En ancho de tablet (768px–1023px), la cuadrícula DEBE mostrar 3 tarjetas por fila.
- **FR-004**: En ancho de móvil (< 768px), la cuadrícula DEBE mostrar 2 tarjetas por fila.
- **FR-005**: La misma cuadrícula responsive (FR-002 a FR-004) DEBE aplicar también a las tarjetas
  de pedidos de Domicilio y Para llevar (spec 059), en vez de su acomodo actual en una sola franja.
- **FR-006**: El sistema NO DEBE remover ninguna información hoy visible en una tarjeta de mesa o de
  pedido (estado, cantidad de productos, tiempo transcurrido, total, badge de "N pedidos" cuando
  aplica) al adoptar la nueva cuadrícula — mismo criterio de no regresión ya exigido por spec 036
  FR-013.
- **FR-007**: En móvil, el texto secundario y el estado de cada tarjeta DEBEN mostrarse en su forma
  abreviada, siguiendo el patrón de la imagen de referencia móvil (por ejemplo "4 prod · 14m" en vez
  de "4 productos · Hace 14 min"; "En prep." en vez de "En preparación"; "Por conf." en vez de "Por
  confirmar") — en escritorio y tablet el texto se mantiene completo, sin abreviar.
- **FR-008**: En móvil, la pestaña "Para llevar" DEBE mostrarse abreviada como "Llevar"; en
  escritorio y tablet se mantiene "Para llevar" completo.

### Functional Requirements — Contadores de ocupación (Historia 2)

- **FR-009**: Con la pestaña "Mesas" activa, el sistema DEBE mostrar un resumen con el total de
  mesas y cuántas están ocupadas, libres y pendientes de algo.
- **FR-010**: Cada filtro de ocupación ("Todas"/"Libres"/"Ocupadas"/"Pendientes") DEBE mostrar junto
  a su etiqueta la cantidad de mesas que le corresponde.
- **FR-011**: El resumen (FR-009) y los contadores de los filtros (FR-010) DEBEN calcularse siempre
  a partir del mismo conjunto de datos, de forma que nunca puedan mostrar cifras distintas entre sí
  para el mismo estado.
- **FR-012**: Los contadores (FR-009, FR-010) DEBEN actualizarse automáticamente cuando cambia el
  estado de cualquier mesa, sin requerir recargar la pantalla — mismo mecanismo reactivo ya usado
  hoy para actualizar cada tarjeta individual.

### Functional Requirements — Barra superior operativa (Historia 3)

- **FR-013**: El sistema DEBE mostrar en la barra superior de la Terminal de Mesas un reloj con la
  hora y la fecha actuales.
- **FR-014**: El sistema DEBE mostrar un indicador de si la pantalla está en línea/sincronizada con
  el servidor.
- **FR-015**: Cuando no exista un turno de caja abierto para la caja registradora de esta terminal,
  la barra superior DEBE mostrar un botón para abrir turno de caja, con el atajo de teclado `F1`
  documentado junto a él.
- **FR-016**: Activar ese botón o el atajo `F1` (FR-015) DEBE disparar el mismo flujo de apertura de
  turno ya implementado en el módulo de Caja, sin duplicar su lógica de negocio.
- **FR-017**: Cuando exista un turno de caja abierto, la barra superior DEBE mostrar en su lugar el
  nombre del cajero de turno — y NO DEBE ofrecer el botón de abrir turno mientras ya hay uno activo.
- **FR-018**: La barra superior DEBE mostrar una etiqueta fija "POS" sin ninguna acción al pulsarla.
- **FR-019**: La barra superior NO DEBE incluir ningún ícono ni acción de "bloquear pantalla" —
  queda explícitamente fuera de alcance (ver Clarifications).
- **FR-020**: En móvil, los elementos de la barra superior DEBEN condensarse (por ejemplo, la hora
  sin segundos, la fecha abreviada, el botón de turno con una etiqueta corta) priorizando que quepan
  las acciones funcionales (turno de caja, búsqueda, filtros); la etiqueta "POS" puede omitirse en
  este ancho sin que eso reduzca ninguna acción disponible.

### Functional Requirements — Usuario de staff asociado al pedido (Historia 4)

- **FR-021**: El sistema DEBE exponer, para cada pedido de mesa que tenga un usuario de staff
  asociado (`user_id` no nulo, sin importar si su rol es Cajero o Mesero), una referencia a ese
  usuario hacia el frontend — reutilizando el dato ya capturado al crear el pedido, sin agregar
  ningún campo ni migración nueva.
- **FR-022**: La tarjeta de una mesa con un pedido activo creado por un usuario de staff DEBE
  mostrar una referencia corta a ese usuario ("Atendido por"), sin condicionarla a que su rol sea
  específicamente Mesero — un pedido creado por un Cajero se muestra exactamente igual.
- **FR-023**: Una mesa sin pedido, o con un pedido creado sin usuario de staff asociado (pedido de
  cliente por QR, `user_id` nulo), NO DEBE mostrar ninguna referencia de "Atendido por" en su
  tarjeta.
- **FR-024**: Cuando una mesa tenga más de un pedido activo, la referencia "Atendido por" mostrada
  DEBE corresponder al mismo pedido que ya determina hoy el resto de la información resumida de la
  tarjeta (mismo criterio de pedido principal ya usado por el sistema).

### Functional Requirements — Panel de detalle sin selección (Historia 5)

- **FR-025**: Sin ninguna mesa ni pedido seleccionado, el panel derecho DEBE mostrar un mensaje
  indicando que debe seleccionarse una mesa o un pedido para ver su detalle.
- **FR-026**: Ese mismo panel DEBE mostrar una referencia de los atajos de teclado disponibles en la
  pantalla (buscar mesa, crear pedido nuevo, abrir turno de caja) junto con su tecla respectiva.
- **FR-027**: Ese mismo panel DEBE mostrar, fijo en su parte inferior, el botón "+ Crear pedido
  nuevo" con el mismo comportamiento ya implementado (navega a la creación manual de pedido).
- **FR-028**: Al seleccionar una mesa o un pedido, el mensaje de bienvenida y la referencia de
  atajos (FR-025, FR-026) DEBEN dejar de mostrarse, reemplazados por el comportamiento ya existente
  del panel central y del panel de cobro — sin cambios de contenido ni de flujo respecto a hoy.

### Functional Requirements — Preservación de vocabulario y paleta existentes

- **FR-029**: El sistema NO DEBE introducir ninguna palabra nueva de estado — las tarjetas de mesa y
  de pedido DEBEN seguir usando exactamente el mismo vocabulario ya existente ("Libre", "Ocupada",
  "En preparación", "Listo", "Pago pendiente", "Por confirmar", "Reservada" — `pos-terminal.store.ts:
  115-123`), incluso donde la imagen de referencia sugiere un texto distinto (por ejemplo "Listo
  para servir" o "Pago").
- **FR-030**: El sistema NO DEBE cambiar los colores ya asociados a cada estado (`STATUS_META`,
  `pos-terminal.store.ts:115-123`) ni ningún otro color ya usado en la pantalla (botones primarios,
  bordes, fondos) — la imagen de referencia se usa como guía de estructura y disposición visual, no
  como paleta de colores literal a copiar.
- **FR-031**: Las pestañas de tipo de orden ("Mesas"/"Domicilios"/"Para llevar") y los filtros de
  ocupación ("Todas"/"Libres"/"Ocupadas"/"Pendientes") DEBEN conservar exactamente las mismas
  etiquetas y el mismo comportamiento ya existente — esta spec solo les agrega contadores (FR-010) y
  los reubica visualmente.

### Key Entities *(include if feature involves data)*

- **Pedido (`DiningOrder`)**: entidad ya existente. Esta spec no agrega ningún campo nuevo: expone
  hacia el frontend un campo ya capturado (`user_id`, referencia blanda al usuario de staff —
  Cajero o Mesero— que creó el pedido) que hoy no llega a ninguna pantalla, para presentarlo como
  "Atendido por" en la tarjeta de mesa.
- **Turno de caja (`CashShift`)**: entidad ya existente (módulo de Caja). Esta spec no cambia su
  modelo de datos ni sus reglas de apertura/cierre — solo agrega, desde la Terminal de Mesas, un
  punto de lectura de su estado actual y un acceso directo a su flujo de apertura ya implementado.
- **Mesa (`Table`) y su estado derivado (`TableDisplayStatus`)**: sin cambios en su modelo ni en su
  lógica de derivación (`deriveTableStatus()`); esta spec solo cambia cómo se organizan
  visualmente sus tarjetas y agrega el resumen de conteo por estado.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: En cualquiera de los tres anchos soportados (escritorio, tablet, móvil), el cajero
  puede ver todas las mesas disponibles usando únicamente scroll vertical, sin necesitar ningún
  desplazamiento horizontal ni flechas de navegación.
- **SC-002**: El resumen de ocupación (total/ocupadas/libres/pendientes) coincide, en el 100% de los
  casos verificados, con el conteo manual de las tarjetas visibles en la grilla.
- **SC-003**: El cajero puede saber si tiene un turno de caja abierto y, si no lo tiene, abrirlo, sin
  salir de la Terminal de Mesas, en una sola interacción (pulsar el botón, o `F1`).
- **SC-004**: El 100% de los pedidos de mesa creados por un usuario de staff (Cajero o Mesero)
  muestran una referencia "Atendido por" visible en su tarjeta, sin necesitar abrir el detalle del
  pedido.
- **SC-005**: Ninguna acción hoy disponible en la Terminal de Mesas (buscar, filtrar, seleccionar,
  crear pedido, cobrar) deja de estar disponible en ninguno de los tres anchos soportados.
- **SC-006**: El tiempo hasta que el cajero identifica el estado general de ocupación del salón (sin
  contar tarjetas manualmente) se reduce a una sola mirada a la parte superior de la pantalla.

## Out of Scope

- **Cualquier cambio de backend distinto a exponer el campo `user_id` ya existente del pedido**:
  ninguna migración, columna ni entidad nueva — el dato ya se captura hoy al crear el pedido.
- **Reasignación manual de "Atendido por" en un pedido ya creado**: la referencia mostrada es
  siempre quien creó el pedido originalmente (Cajero o Mesero); cambiarla manualmente después queda
  fuera de esta spec.
- **Función real para la insignia "POS"**: queda como etiqueta fija sin acción, por decisión
  explícita del negocio (ver Clarifications).
- **Bloqueo de pantalla / ícono de candado**: removido explícitamente del alcance (ver
  Clarifications) — no se implementa ningún mecanismo de bloqueo ni de reingreso de PIN.
- **Ampliar el buscador a mesero o número de comanda**: se mantiene exactamente la búsqueda actual,
  solo por mesa (ver Clarifications).
- **Comportamiento del panel central y del panel de cobro una vez hay una mesa/pedido
  seleccionado**: sin cambios — siguen rigiendo las reglas ya definidas por specs 036/045/048/049;
  esta spec solo restylea el contenedor alrededor de ese comportamiento y el estado "sin selección"
  (Historia 5).
- **Un verdadero motor de sincronización offline-first**: el sistema no tiene hoy persistencia local
  ni cola de sincronización; el indicador de sincronización (FR-014) refleja conectividad de red y
  el éxito de la última carga de datos, no un mecanismo nuevo de trabajo sin conexión.
- **Cambiar el vocabulario de estados o la paleta de colores ya establecidos**: se preservan tal
  cual (FR-029, FR-030) — la imagen de referencia se usa solo como guía de estructura y disposición.

## Assumptions

- **Los breakpoints de escritorio/tablet/móvil (FR-002 a FR-004) reutilizan los mismos umbrales ya
  usados hoy en esta pantalla** (`md` ≈ 768px, `lg` ≈ 1024px, `table-sessions.component.ts:54,73`)
  en vez de introducir un sistema de breakpoints nuevo — más consistente con el resto de la
  interfaz que definir uno específico para esta spec.
- **El indicador de sincronización (FR-014) se deriva de la conectividad de red del navegador y del
  éxito/fracaso de la última carga de datos ya expuesta por el store** (`store.error()`), no de un
  mecanismo de sincronización nuevo — el sistema siempre ha dependido de una conexión activa al
  servidor, sin ningún modo de trabajo desconectado hoy.
- **La etiqueta de turno mostrada junto al cajero de turno (por ejemplo "Turno Mañana") se deriva de
  la hora en que se abrió el turno (`CashShift.opened_at`)** — el modelo de datos actual no tiene un
  campo de nombre de turno (`cash.interface.ts:12-23` no incluye ninguno); se deriva del horario en
  vez de agregar un campo nuevo, para no expandir el alcance de datos de esta spec.
- **En anchos de móvil, el flujo de detalle de una mesa/pedido seleccionado (Historia 5, punto 4)
  reemplaza la vista de la grilla en vez de mostrarse lado a lado** — un ancho de móvil no tiene
  espacio para ambos paneles simultáneamente; al volver de ese detalle, el cajero regresa a la
  grilla en el mismo estado (filtro, pestaña, scroll) en que la dejó. Ninguna de las 3 imágenes de
  referencia muestra explícitamente este estado seleccionado en móvil — se asume el patrón estándar
  de "maestro-detalle" que colapsa a una sola vista por vez en anchos angostos, ya sin alternativa
  razonable distinta dado el espacio disponible.
- **La referencia corta "Atendido por" (FR-022) reutiliza el mismo patrón ya usado para mostrar
  personas en esta pantalla** (nombre corto o inicial, igual que ya se usa para el cajero de turno)
  — no se introduce un sistema de numeración de meseros nuevo; el formato exacto ("Mozo #2" de la
  imagen de referencia era solo un dato de ejemplo, no un identificador real que el sistema deba
  generar). Tampoco se condiciona a ningún rol específico (Clarifications, sesión 2026-09-04):
  se muestra igual para un pedido creado por un Cajero que por un Mesero.
- **Los números de ejemplo en las imágenes de referencia (por ejemplo "16 mesas totales · 9
  ocupadas" en la barra superior frente a "Ocupadas 7" en los filtros) son datos de muestra
  inconsistentes entre sí, no un requisito de que difieran** — FR-011 exige que ambos contadores
  siempre coincidan en la implementación real, porque se calculan del mismo origen de datos.
- **Las tres imágenes de referencia representan únicamente el estado sin ninguna mesa/pedido
  seleccionado** — el estado con selección activa (paneles central y de cobro) no tiene una imagen
  de referencia nueva y por eso conserva su comportamiento y disposición ya implementados, solo
  encajados dentro del nuevo panel derecho unificado (Historia 5, punto 4).
