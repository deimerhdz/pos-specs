# Feature Specification: Corrección de bugs y mejoras — Menú QR

**Feature Branch**: `041-correccion-bugs-menu-qr`

**Created**: 2026-08-27

**Status**: Draft

**Naturaleza de esta spec**: **spec de corrección de bugs y mejoras**, no una funcionalidad nueva
desde cero. Igual que las specs [019](../019-correccion-cuenta-mesas-fusionadas/spec.md),
[020](../020-correccion-validacion-opciones-mesero/spec.md) y
[021](../021-correccion-orden-borrado-imagen-r2/spec.md), cita nombres de archivo y línea del
código actual (`pos-heladeria` y `pos-backend`) porque son el contrato observable que se está
corrigiendo, no una fuga de detalles de implementación. Agrupa cuatro correcciones/mejoras
independientes del módulo de Menú QR y del formulario de productos, cada una verificable por
separado.

**Autorización de negocio (Principio I de la [Constitución](../../.specify/memory/constitution.md))**:
solicitado directamente por el dueño/desarrollador del proyecto el 2026-08-27, mediante el listado
detallado de bugs y mejoras que da origen a esta spec (problema, comportamiento esperado y
criterios de aceptación explícitos para cada uno de los cuatro puntos).

**Input**: User description: "Corrección de bugs y mejoras del módulo de Menú QR y del formulario
de productos: (1) bloquear la creación de una nueva sesión de comensal después de cerrar sesión —
logout + refresh, back, forward y reingreso a una URL previa no deben permitir un nuevo acceso sin
volver a escanear el QR de la mesa; (2) el PNG del QR de una mesa descargado desde el panel de
gestión debe identificar visualmente la mesa, y debe poder descargarse en dos formatos — 'Mostrador'
(mayor tamaño) y 'Sticker' (compacto) — sin cambiar la información codificada en el QR; (3)
reemplazar el placeholder de helado que se muestra en el catálogo del Menú QR para productos sin
imagen por uno genérico de 'imagen no disponible'; (4) mostrar el botón 'Copiar insumos de la
variante a otros tamaños' en el formulario de productos únicamente cuando el producto tenga
activado el manejo de inventario, reaccionando de inmediato al activar/desactivar el switch."

## Clarifications

### Session 2026-08-27

- Q: ¿Cuándo debe un navegador que cerró sesión poder volver a iniciar una sesión nueva en esa
  misma mesa? → A: El acceso al Menú QR debe estar vinculado a un flujo válido de escaneo del
  código QR; la URL del menú no debe constituir por sí misma una credencial de acceso. Tras cerrar
  sesión, todo estado/credencial que permita reutilizar el acceso DEBE invalidarse/eliminarse.
  Acceder mediante una URL almacenada, el historial del navegador, una recarga o una navegación
  directa DEBE rechazarse, exigiendo un nuevo escaneo del QR — sin excepción automática por cambio
  de `table_session` de la mesa ni por tiempo transcurrido desde el cierre de sesión.

## User Scenarios & Testing *(mandatory)*

<!--
  Cada historia corresponde a uno de los cuatro puntos del reporte de bugs, numerados igual que en
  el reporte original para trazabilidad. La prioridad refleja impacto: seguridad de sesión (P1),
  identificación física de mesas en operación real (P1), consistencia visual del catálogo (P3), e
  integridad de una acción de formulario sin efecto en base de datos hasta guardar (P3).
-->

### User Story 1 - Cerrar sesión del Menú QR invalida el acceso de verdad (Priority: P1) — Bug 1

Un comensal escanea el QR de su mesa, escribe su nombre y navega el menú. Cuando termina, pulsa
"Salir"/"Cerrar sesión". A partir de ese momento, ese mismo navegador no debe poder volver a entrar
al menú de esa mesa —ni recargando la página, ni con "Atrás"/"Adelante", ni reabriendo la misma URL
más tarde— sin que el comensal vuelva a pasar por el flujo de acceso del QR desde cero.

**Why this priority**: es el problema de seguridad/integridad de sesión más grave del lote —hoy
cualquiera que reutilice el navegador de un comensal que ya cerró sesión puede seguir generando
pedidos a nombre de una "sesión nueva" en esa mesa sin haber vuelto a escanear nada físicamente.

**Independent Test**: se puede probar abriendo el menú vía QR, creando una sesión, cerrando sesión,
y verificando en cada uno de estos casos que NO aparece la posibilidad de crear una sesión nueva sin
pasar de nuevo por el flujo de acceso: recargar la página, pulsar "Atrás", pulsar "Adelante" después
de "Atrás", y volver a abrir la misma URL desde el historial del navegador.

**Acceptance Scenarios**:

1. **Given** un comensal que escanea el QR de su mesa, **When** escribe su nombre, **Then** el
   sistema crea su sesión y le da acceso al menú (comportamiento actual sin cambios,
   `POST /cart/sessions` → `open_session`, `pos-backend/app/api/v1/cart/service.py:109-166`).
2. **Given** un comensal con sesión activa, **When** pulsa "Salir"/"Cerrar sesión", **Then** su
   token de sesión (`session_token`) queda invalidado tanto en el navegador (se borra de
   `localStorage`, clave `pos.diner.session_token`) como en el servidor (`SessionParticipant.status`
   pasa a `closed`; cualquier petición posterior con ese token responde `401`) — este comportamiento
   ya funciona correctamente hoy (`leave_session`,
   `pos-backend/app/api/v1/cart/service.py:507-515`; `close_participant`,
   `pos-backend/app/core/qr_context.py:72-93`; validación de estado en `open_session_context`,
   `pos-backend/app/core/qr_context.py:149-208`) y esta spec lo protege como característica que no
   debe regresar.
3. **Given** un comensal que ya cerró sesión, **When** recarga el navegador en la misma URL del
   menú, **Then** el sistema NO debe ofrecerle de inmediato la pantalla de "escribe tu nombre" para
   iniciar una sesión nueva — debe mostrarle un estado explícito de "acceso finalizado, vuelve a
   escanear el código QR de tu mesa", sin ruta directa desde ahí hacia la creación de una sesión.
4. **Given** un comensal que ya cerró sesión, **When** pulsa el botón "Atrás" del navegador, **Then**
   el sistema NO debe restaurar ninguna vista del menú/carrito autenticado que existiera antes del
   cierre de sesión — cualquier estado en memoria de la vista previa a "Salir" queda descartado, no
   solo el token.
5. **Given** el escenario anterior, **When** el comensal pulsa "Adelante" después de "Atrás",
   **Then** tampoco se restaura el acceso — el resultado es el mismo estado explícito de "acceso
   finalizado" del escenario 3.
6. **Given** un comensal que ya cerró sesión, **When** abre de nuevo, más tarde, la misma URL que ya
   había visitado (historial, marcador, o el enlace que quedó abierto en otra pestaña), **Then**
   obtiene el mismo estado de "acceso finalizado" — no la pantalla de captura de nombre.
7. **Given** el estado de "acceso finalizado" en un navegador, **When** el comensal quiere volver a
   pedir, **Then** la única vía prevista es repetir el flujo de acceso por QR (escanear físicamente
   el código de la mesa con la cámara del celular) — la URL del menú, por sí sola, nunca vuelve a
   funcionar como credencial de acceso para ese intento ya cerrado, sin importar cuánto tiempo pase
   ni si la mesa cambia de ocupación mientras tanto.

---

### User Story 2 - El QR descargado de una mesa se identifica y se ajusta al uso físico (Priority: P1) — Bug 2

Un administrador entra al panel de gestión de mesas/QR, descarga el PNG del código QR de una mesa
específica (o genera el lote completo) y lo imprime para colocarlo físicamente. El PNG debe dejar
claro a qué mesa pertenece, y debe poder elegir entre un formato grande para mostrador y uno
compacto para sticker, sin que cambie el destino/contenido que el QR codifica.

**Why this priority**: un QR indistinguible entre mesas es un problema operativo real —el personal
puede imprimir/colocar el QR equivocado en una mesa y desviar pedidos a la mesa incorrecta; es tan
crítico como el bug de sesión, aunque no sea de seguridad.

**Independent Test**: se puede probar descargando el PNG de la mesa 1 y de la mesa 2 desde el panel,
confirmando que cada imagen muestra un texto distinto ("Mesa 1", "Mesa 2") correspondiente al
`number`/`name` real de cada mesa, que el QR decodifica exactamente a la misma URL que hoy
(`menuUrlForToken`, `pos-heladeria/src/app/modules/tables/services/table-qr.util.ts:11-13`), y que
existen dos opciones de descarga con dimensiones distintas para la misma mesa.

**Acceptance Scenarios**:

1. **Given** el modal de QR de una mesa en el panel de gestión
   (`pos-heladeria/src/app/modules/tables/components/table-qr.component.ts`), **When** el
   administrador descarga el PNG, **Then** la imagen resultante incluye visualmente el identificador
   real de la mesa (`Mesa {{ table.number }}`, tomado de `DiningTable.number`
   —`pos-backend/app/models/dining_table.py:15-17`—, el mismo dato ya usado para el encabezado del
   modal y para el nombre del archivo descargado), no un índice de posición en una lista.
2. **Given** ese mismo modal, **When** el administrador elige descargar, **Then** puede escoger entre
   dos opciones claramente diferenciadas — "Mostrador" y "Sticker" — en vez de un único botón
   "Descargar PNG" sin variantes.
3. **Given** la opción "Mostrador", **When** se descarga, **Then** el PNG resultante tiene mayor
   tamaño físico/resolución, pensado para verse bien sobre una superficie visible (mostrador/atril),
   con el identificador de mesa en texto grande y legible, y una zona de seguridad (quiet zone)
   alrededor del QR.
4. **Given** la opción "Sticker", **When** se descarga, **Then** el PNG resultante es más compacto,
   conserva el QR perfectamente escaneable (sin reducir su resolución al punto de afectar la
   lectura), y también muestra el identificador de mesa.
5. **Given** ambas variantes de la misma mesa, **When** se decodifica cada QR, **Then** ambas
   apuntan exactamente al mismo destino —la URL firmada con el token QR de esa mesa
   (`GET /orders/tables/{table_id}/qr-token` → `mint_qr_token`,
   `pos-backend/app/core/qr_token.py:77-80`)— sin ninguna diferencia en la información codificada.
6. **Given** la generación masiva de QR de todas las mesas
   (`pos-heladeria/src/app/modules/tables/pages/table-qr-sheet.component.ts`), **When** se genera el
   lote, **Then** cada tarjeta conserva el identificador correcto de su propia mesa (`t.number`,
   opcionalmente `t.name`), igual que ya ocurre hoy en la hoja imprimible.

---

### User Story 3 - El catálogo del Menú QR usa un placeholder neutro para productos sin imagen (Priority: P3) — Bug 3

Un comensal navega el catálogo del Menú QR. Para un producto sin imagen configurada, en lugar de un
emoji de helado (🍦), ve un ícono genérico que comunica claramente "sin imagen", sin insinuar que el
producto es un helado.

**Why this priority**: es un defecto puramente visual/de percepción de marca —el sistema puede
vender cualquier tipo de producto, no solo helados— sin impacto funcional ni de seguridad; se
prioriza último dentro del lote.

**Independent Test**: se puede probar consultando el catálogo del Menú QR con un producto sin
`image_url` y verificando que se muestra el nuevo ícono genérico en vez del emoji 🍦, mientras que un
producto con `image_url` sigue mostrando su imagen real sin cambios.

**Acceptance Scenarios**:

1. **Given** un producto del catálogo del Menú QR sin `image_url`, **When** el comensal lo ve en la
   grilla, **Then** el sistema muestra un ícono genérico de "imagen no disponible" en el lugar donde
   hoy se renderiza el emoji `🍦`
   (`pos-heladeria/src/app/modules/tables/pages/public-menu.component.ts:346-350`), sin representar
   ningún tipo de producto específico (helado, bebida, comida).
2. **Given** un producto del catálogo del Menú QR con `image_url` configurada, **When** el comensal
   lo ve, **Then** sigue mostrando su imagen real, sin ningún cambio de comportamiento.
3. **Given** el nuevo ícono, **When** se implementa, **Then** reutiliza el sistema de íconos
   compartido ya existente del proyecto (`pos-heladeria/src/app/shared/icon/icon.component.ts`,
   íconos SVG de trazo único estilo Lucide) en vez de introducir una librería de íconos externa
   nueva o un archivo de imagen adicional.

---

### User Story 4 - "Copiar insumos" solo aparece cuando el producto maneja inventario (Priority: P3) — Bug 4

Un administrador crea o edita un producto con varios tamaños/variantes. El botón "Copiar insumos y
sabores de [variante] a los otros tamaños" solo debe estar visible cuando ese producto tiene
activado el switch "maneja inventario"; si el switch está apagado, la acción no tiene insumos que
copiar y no debe ofrecerse. El botón debe aparecer o desaparecer de inmediato al mover el switch, sin
recargar la pantalla.

**Why this priority**: es una corrección de consistencia de UI sobre una acción que hoy no persiste
nada hasta que se guarda el formulario (`copyConfigToOthers`,
`pos-heladeria/src/app/modules/products/pages/product-form.component.ts:858-881`, solo modifica el
`draft` en memoria) — bajo riesgo, prioridad más baja del lote.

**Independent Test**: se puede probar abriendo el formulario de un producto con varios tamaños,
apagando el switch "maneja inventario" y verificando que el botón "Copiar insumos..." desaparece de
inmediato; encendiéndolo de nuevo y verificando que reaparece; repitiendo ambas verificaciones tanto
en creación como en edición de un producto existente que ya tenga insumos guardados.

**Acceptance Scenarios**:

1. **Given** el formulario de creación o edición de un producto con más de un tamaño/variante
   (`draft().hasSizes && draft().variants.length > 1`), **When** el switch "maneja inventario"
   (`draft().tracks_inventory`) está apagado, **Then** el botón "Copiar insumos y sabores de
   «...» a los otros tamaños"
   (`pos-heladeria/src/app/modules/products/pages/product-form.component.ts:384-389`) NO se muestra.
2. **Given** el mismo formulario, **When** el switch "maneja inventario" está encendido, **Then** el
   botón sí se muestra, igual que hoy.
3. **Given** el formulario abierto con el switch apagado, **When** el administrador lo activa
   (`toggleTracksInventory()`, `product-form.component.ts:823-837`), **Then** el botón aparece de
   inmediato, sin recargar la página.
4. **Given** el formulario abierto con el switch encendido y el botón visible, **When** el
   administrador lo desactiva, **Then** el botón desaparece de inmediato.
5. **Given** un producto ya existente que tiene insumos guardados de antes, **When** se abre su
   formulario de edición con el switch apagado, **Then** el botón permanece oculto —igual que la
   sección de insumos, que ya queda deshabilitada hoy cuando el switch está apagado
   (`product-form.component.ts:244`, mensaje alternativo `:378-382`)— sin perder los insumos ya
   guardados (comportamiento de spec 027, sin cambios).
6. **Given** un producto con el switch apagado, **When** se intenta ejecutar la copia de insumos por
   cualquier otra vía visible en el formulario (no solo el botón principal), **Then** no existe
   ninguna otra ruta en la interfaz para disparar `copyConfigToOthers()` sin pasar por ese botón —
   ocultarlo es suficiente para bloquear la acción en el formulario, dado que la operación no llama a
   ningún endpoint propio (no existe un endpoint de "copiar insumos" en el backend; los datos
   copiados solo se persisten al guardar el producto completo con `save()`,
   `product-form.component.ts:1067`).

---

### Edge Cases

- **Bug 1 — el token QR físico de la mesa es permanente por diseño protegido**: la regla protegida
  A-24 (`RN-CART-24`, spec [007](../007-menu-carrito-qr/spec.md)) establece que el token QR impreso
  de la mesa no expira nunca y solo se invalida rotando el secreto de firma del servidor —esta spec
  NO reabre ni modifica esa regla. En consecuencia, un escaneo físico nuevo y una recarga de la misma
  URL producen técnicamente la misma petición al servidor; ver Assumptions para cómo esta spec
  resuelve esa tensión sin tocar la permanencia del token QR.
- **Bug 1 — otro comensal se une a la misma mesa después de que uno cerró sesión**: el cierre de
  sesión de un comensal es individual (`SessionParticipant`); si la `TableSession` de la mesa sigue
  activa porque otros comensales continúan, un tercero que escanea el mismo QR debe poder seguir
  uniéndose a esa sesión activa con normalidad (`RN-CART-01`, sin cambios) — el bloqueo de esta spec
  aplica solo al navegador/participante que cerró sesión explícitamente, no a la mesa completa.
- **Bug 1 — expiración natural del token de sesión (sin logout explícito)**: fuera de alcance; el
  comportamiento de expiración por inactividad (`RN-CART-17` a `RN-CART-20`, spec 007) no cambia,
  esta spec corrige únicamente el caso de cierre de sesión **explícito**.
- **Bug 1 — la mesa se libera y se abre una `table_session` nueva mientras el navegador sigue
  bloqueado** (aclarado en Clarifications, sesión 2026-08-27): el bloqueo del navegador que cerró
  sesión NO se levanta automáticamente por ese cambio — sigue rechazando la reutilización de esa URL
  ya cerrada; solo un intento de acceso genuinamente nuevo (repetir el flujo del QR) la destraba.
- **Bug 2 — mesa sin `name` configurado, solo `number`**: el identificador visible en el PNG usa
  `number` (siempre presente y único, `DiningTable.number`) como dato principal; `name` es opcional y
  secundario, igual que ya lo trata hoy la hoja de generación masiva
  (`table-qr-sheet.component.ts:79-88`).
- **Bug 2 — impresión a baja calidad**: la variante "Sticker", al ser más compacta, es la más
  sensible a perder legibilidad si se imprime muy pequeña; su resolución debe mantenerse suficiente
  para que el módulo QR siga siendo escaneable al tamaño mínimo de sticker previsto (ver Assumptions).
- **Bug 3 — producto con `image_url` que apunta a un recurso roto/caído**: fuera de alcance de esta
  spec; el comportamiento ante una URL de imagen inválida (a diferencia de ausente) no cambia.
- **Bug 3 — el mismo emoji 🍦 se usa también en pantallas de administración/staff** (`products-page.component.ts`,
  `product-form.component.ts`, `product-select.component.ts`): fuera de alcance; esta spec corrige
  únicamente el catálogo del Menú QR (`public-menu.component.ts`), no las pantallas internas de
  staff/admin, que no comparten ningún componente con el catálogo del comensal.
- **Bug 4 — producto de un solo tamaño (sin variantes múltiples)**: el botón ya no se muestra hoy en
  ese caso (`draft().hasSizes && draft().variants.length > 1`); esta spec no modifica esa condición
  existente, solo le agrega la condición del switch de inventario.
- **Bug 4 — cambiar el switch varias veces sin guardar**: solo importa el estado del switch en cada
  instante para decidir la visibilidad del botón; no hay persistencia intermedia (comportamiento ya
  documentado en spec 027, Edge Cases).

## Requirements *(mandatory)*

### Functional Requirements

**Bug 1 — Invalidación real del acceso al Menú QR tras cerrar sesión**

- **FR-001**: Al cerrar sesión, el sistema DEBE seguir invalidando el `session_token` del comensal
  tanto en el navegador (borrado de `localStorage`, clave `pos.diner.session_token`) como en el
  servidor (`SessionParticipant.status = "closed"`, rechazado con `401` en cualquier petición
  posterior) — comportamiento ya correcto hoy, que esta spec protege sin modificar.
- **FR-002**: Tras un cierre de sesión explícito, recargar el navegador en la misma URL del menú NO
  DEBE presentar directamente la pantalla de captura de nombre como paso hacia una sesión nueva —
  DEBE mostrar un estado explícito de "acceso finalizado", distinto de la pantalla de primer acceso.
- **FR-003**: Tras un cierre de sesión explícito, usar el botón "Atrás" del navegador NO DEBE
  restaurar ninguna vista de menú/carrito autenticada que existiera antes del cierre de sesión, ni
  ningún estado en memoria previo a "Salir".
- **FR-004**: Tras un cierre de sesión explícito, usar el botón "Adelante" (después de "Atrás") NO
  DEBE restaurar el acceso — debe producir el mismo estado de "acceso finalizado" que FR-002/FR-003.
- **FR-005**: Reabrir, en el mismo navegador, una URL del Menú QR que ya fue objeto de un cierre de
  sesión explícito (por historial, marcador, o pestaña que quedó abierta) NO DEBE permitir crear una
  sesión nueva sin pasar de nuevo por el flujo de acceso — mismo estado de "acceso finalizado". Este
  rechazo NO DEBE tener una condición automática de vencimiento: ni el paso del tiempo desde el
  cierre de sesión, ni un cambio en la ocupación/`table_session` de la mesa, debe por sí solo
  restaurar la posibilidad de crear sesión en ese acceso ya cerrado — la URL del menú, por sí misma,
  NO es una credencial de acceso válida una vez cerrado ese acceso.
- **FR-006**: La única vía prevista para que un comensal que cerró sesión vuelva a acceder al menú de
  su mesa DEBE ser repetir el flujo de acceso del código QR (nueva lectura del código físico de la
  mesa) mediante un intento de acceso genuinamente nuevo, sin atajos desde el estado de "acceso
  finalizado" ni desde ningún estado/credencial que haya quedado almacenado en el navegador del
  intento anterior.
- **FR-007**: La corrección de FR-002 a FR-005 NO DEBE modificar el contenido codificado del token QR
  físico de la mesa (`mint_qr_token`/`verify_qr_token`,
  `pos-backend/app/core/qr_token.py:77-93`) ni la regla protegida A-24/`RN-CART-24` (permanencia sin
  `exp`) — la corrección se resuelve del lado del navegador/aplicación sobre ese mismo token, no
  cambiando su emisión o verificación.

**Bug 2 — QR de mesas identificable y en el tamaño adecuado**

- **FR-008**: El PNG descargado del QR de una mesa (individual, `table-qr.component.ts`) DEBE incluir
  visualmente el identificador real de la mesa (`DiningTable.number`, con `name` como dato
  secundario si existe), generado dinámicamente a partir del dato de la mesa — nunca a partir de la
  posición de la mesa en una lista.
- **FR-009**: El panel de descarga del QR de una mesa DEBE ofrecer dos opciones claramente
  diferenciadas — "Mostrador" y "Sticker" — en lugar de un único botón de descarga sin variantes.
- **FR-010**: La variante "Mostrador" DEBE tener mayor tamaño/resolución que la variante "Sticker",
  con el identificador de mesa en texto grande y legible, apropiada para colocarse sobre una
  superficie visible.
- **FR-011**: La variante "Sticker" DEBE ser más compacta que "Mostrador", conservando una resolución
  de QR suficiente para que siga siendo escaneable, y mostrando también el identificador de mesa.
- **FR-012**: Ambas variantes DEBEN codificar exactamente el mismo destino/token que el QR actual
  (`menuUrlForToken`, `table-qr.util.ts:11-13`) — esta spec no modifica la información funcional
  codificada en el QR, solo su presentación visual y tamaño de salida.
- **FR-013**: Ambas variantes DEBEN mantener una zona de seguridad (quiet zone) alrededor del módulo
  QR y contraste suficiente para no degradar su lectura.
- **FR-014**: Ambas variantes DEBEN poder descargarse como PNG, igual que la descarga actual.
- **FR-015**: La generación masiva de QR de todas las mesas (`table-qr-sheet.component.ts`) DEBE
  seguir conservando el identificador correcto de cada mesa individual, sin regresión respecto al
  comportamiento actual.

**Bug 3 — Placeholder genérico para productos sin imagen en el catálogo del Menú QR**

- **FR-016**: El catálogo del Menú QR (`public-menu.component.ts:346-350`) DEBE mostrar un ícono
  genérico de "imagen no disponible" para todo producto sin `image_url`, en reemplazo del emoji de
  helado (`🍦`) actual.
- **FR-017**: Ese ícono genérico NO DEBE representar visualmente ningún tipo de producto específico
  (helado, granizado, bebida, comida).
- **FR-018**: Un producto del catálogo del Menú QR con `image_url` configurada DEBE seguir mostrando
  su imagen real, sin cambios.
- **FR-019**: El ícono genérico DEBE implementarse reutilizando el componente de íconos compartido ya
  existente del proyecto (`shared/icon/icon.component.ts`), siguiendo el mismo estilo SVG de trazo
  único ya usado por los íconos ahí definidos, sin introducir una librería de íconos externa nueva.
- **FR-020**: Este cambio DEBE aplicarse únicamente al catálogo del Menú QR (comensal); las pantallas
  internas de administración/staff que hoy usan el mismo emoji de forma independiente
  (`products-page.component.ts`, `product-form.component.ts`) quedan fuera de alcance de esta spec.

**Bug 4 — "Copiar insumos" solo visible con manejo de inventario activo**

- **FR-021**: El botón "Copiar insumos y sabores de «...» a los otros tamaños"
  (`product-form.component.ts:384-389`) DEBE mostrarse únicamente cuando el producto tenga el switch
  "maneja inventario" (`draft().tracks_inventory`) activado, además de la condición ya existente de
  tener más de un tamaño/variante (`draft().hasSizes && draft().variants.length > 1`).
- **FR-022**: Al activar el switch "maneja inventario" sobre un producto con más de un
  tamaño/variante, el botón DEBE aparecer de inmediato, sin recargar la pantalla.
- **FR-023**: Al desactivar el switch "maneja inventario", el botón DEBE desaparecer de inmediato,
  sin recargar la pantalla.
- **FR-024**: Esta corrección DEBE aplicar tanto en la creación como en la edición de productos,
  incluyendo productos existentes que ya tengan insumos guardados de antes.
- **FR-025**: Esta corrección NO DEBE afectar el comportamiento ya existente de gestión de variantes,
  tamaños, ni la persistencia/visibilidad de insumos ya guardados al apagar/encender el switch (spec
  [027](../027-control-inventario-productos/spec.md), sin cambios).
- **FR-026**: Dado que la operación de copiar insumos no llama a ningún endpoint propio y solo
  modifica el `draft` en memoria hasta que se guarda el formulario completo (`save()`,
  `product-form.component.ts:1067`), ocultar el botón según FR-021 es suficiente para impedir la
  acción — no existe otra ruta de UI para dispararla; si en el futuro se agrega un endpoint dedicado
  para esta operación, DEBE validar de forma independiente que el producto tenga el manejo de
  inventario activado antes de ejecutarla.

### Key Entities *(include if feature involves data)*

- **SessionParticipant / TableSession**: identidad y estado (`open`/`closed`) de un comensal dentro
  de la sesión de una mesa (spec 007). El cierre de sesión de esta spec actúa sobre el
  `SessionParticipant` del navegador que cierra sesión, sin afectar a otros comensales activos en la
  misma `TableSession`.
- **Token QR / Token de sesión**: credenciales del flujo de comensal (spec 007). El token QR
  (permanente, sin `exp`, regla protegida A-24) no se modifica; el token de sesión sigue
  invalidándose al cerrar sesión (comportamiento protegido por esta spec, no nuevo).
- **DiningTable**: mesa física, con `number` (identificador único siempre presente) y `name`
  (opcional). Es la fuente del identificador visual que debe aparecer en el PNG del QR (Bug 2).
- **Product**: producto del catálogo; `image_url` determina si se muestra imagen real o el nuevo
  placeholder genérico (Bug 3); `tracks_inventory` determina si el producto maneja inventario y, con
  esta spec, también si se ofrece la acción de copiar insumos entre tamaños (Bug 4).
- **ProductVariant / Insumos (receta fija, grupos de opciones que descuentan)**: configuración de
  consumo de inventario por presentación (spec 003/027); la acción de copiarlos entre tamaños del
  mismo producto es la que esta spec condiciona a `tracks_inventory`.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los intentos de reingresar al Menú QR después de un cierre de sesión
  explícito —por recarga, "Atrás", "Adelante" o reapertura de una URL ya visitada, sin importar
  cuánto tiempo haya pasado ni si la mesa cambió de ocupación— resulta en el estado de "acceso
  finalizado", nunca en la creación de una sesión nueva ni en la restauración de la sesión cerrada.
- **SC-002**: El 100% de los comensales que cierran sesión y realizan un nuevo intento de acceso
  genuino (repitiendo el flujo de escaneo del QR) completan exitosamente ese nuevo flujo y pueden
  volver a pedir — la restricción de SC-001 aplica únicamente a la reutilización del acceso ya
  cerrado, nunca a un intento de acceso nuevo y válido.
- **SC-003**: El 100% de los PNG descargados (individuales o del lote masivo) de códigos QR de mesa
  muestran, a simple vista, el identificador correcto de su propia mesa.
- **SC-004**: El 100% de los códigos QR descargados —en ambas variantes, Mostrador y Sticker— se
  leen correctamente con una cámara de celular estándar, sin degradación de lectura frente al
  comportamiento actual.
- **SC-005**: El 100% del contenido codificado en los QR descargados permanece idéntico al que se
  codifica hoy, verificable decodificando el QR y comparando la URL resultante.
- **SC-006**: El 100% de los productos sin imagen en el catálogo del Menú QR muestran el ícono
  genérico de "imagen no disponible"; ninguno muestra el emoji de helado.
- **SC-007**: El 100% de los productos con imagen configurada en el catálogo del Menú QR siguen
  mostrando su imagen real, sin regresión.
- **SC-008**: El estado de visibilidad del botón "Copiar insumos..." coincide, en el 100% de las
  veces, con el estado actual del switch "maneja inventario" en el mismo instante, tanto en
  creación como en edición de productos, sin necesidad de recargar la pantalla.

## Out of Scope

- **Bug 1**: rotar o modificar el secreto de firma, el formato o la permanencia del token QR físico
  de la mesa (regla protegida A-24/`RN-CART-24`, spec 007) — cualquier cambio a esa regla exige una
  nueva decisión de negocio/seguridad explícita, fuera de esta spec. Tampoco cambia la expiración por
  inactividad del token de sesión (`RN-CART-17` a `RN-CART-20`) ni el mecanismo de unión de un nuevo
  comensal a una `TableSession` que sigue activa (`RN-CART-01`).
- **Bug 2**: cambiar el destino/información que el QR codifica, el mecanismo de firma del token QR, o
  agregar más de dos formatos de descarga. Tampoco define el diseño gráfico exacto (tipografía,
  colores, disposición) más allá de los criterios de identificación y tamaño ya especificados —eso
  se resuelve en la fase de planeación.
- **Bug 3**: el placeholder de las pantallas internas de administración/staff
  (`products-page.component.ts`, `product-form.component.ts`, `product-select.component.ts`), que
  usan el mismo emoji de forma independiente y no comparten componente con el catálogo del comensal.
  Tampoco cubre productos con `image_url` inválida/caída (a diferencia de ausente).
- **Bug 4**: cualquier cambio a las reglas ya vigentes de manejo de inventario, receta fija, grupos
  de opciones o persistencia de insumos al apagar/encender el switch (spec 003/027) — se reutilizan
  sin modificación. Tampoco agrega un endpoint de backend dedicado a "copiar insumos" (no existe hoy
  y esta spec no lo requiere, ver FR-026).

## Assumptions

- **Bug 1 — cómo se opera "requerir un nuevo escaneo físico" sin tocar la permanencia del token
  QR** (resuelto en Clarifications, sesión 2026-08-27): dado que el token QR de la mesa es permanente
  por diseño protegido (A-24/`RN-CART-24`) y que, a nivel de red, un escaneo físico nuevo puede
  producir una petición indistinguible de recargar la misma URL, esta spec asume que "la URL no es
  una credencial" se implementa mediante una marca persistente de "acceso cerrado" que vive en el
  contexto de navegación (pestaña/historial) del intento que cerró sesión, no en el token QR en sí.
  Esa marca NO se limpia automáticamente por el paso del tiempo ni por un cambio de `table_session`
  de la mesa (decisión explícita del negocio, ver Clarifications) — solo un intento de acceso
  genuinamente nuevo, que no herede esa marca (por ejemplo, un contexto de navegación distinto al que
  cerró sesión, como el que típicamente abre un escaneo físico nuevo desde la cámara del celular), la
  deja sin efecto. El mecanismo exacto de almacenamiento de esa marca se define en la fase de
  planeación, no en esta spec. Se documenta como riesgo residual conocido y aceptado: un usuario que
  borre manualmente los datos del sitio en su navegador, o que use otro navegador/perfil/pestaña
  nueva, alcanza un contexto de navegación sin esa marca y por lo tanto sí puede llegar a la pantalla
  de nombre usando la misma URL —esto es inherente a que el token QR en sí no cambia (fuera de
  alcance, ver Out of Scope) y a que ningún mecanismo del lado del navegador puede demostrar de forma
  criptográfica que una petición se originó en un escaneo físico real; cerrar esa brecha por completo
  requeriría una nueva decisión de negocio sobre la regla protegida A-24 (por ejemplo, rotar el
  token por mesa bajo demanda).
- **Bug 2 — dimensiones de referencia**: esta spec no fija píxeles exactos (se definen en
  planeación), pero asume como referencia razonable que "Mostrador" apunta a un tamaño de impresión
  tipo atril/tarjeta de mesa (más grande, mayor resolución) y "Sticker" a un tamaño de adhesivo
  compacto (menor, pero con el módulo QR sobre el mínimo necesario para lectura confiable a corta
  distancia) — ambos generados con la misma librería `qrcode` ya usada hoy en el frontend
  (`QRCode.toDataURL`, que ya soporta parámetros de ancho y margen).
- **Bug 2 — identificador visible**: se usa `DiningTable.number` (siempre presente y único) como
  identificador principal ("Mesa {number}"), y `DiningTable.name` como dato secundario opcional si
  existe, replicando el patrón que ya usa la hoja de generación masiva actual.
- **Bug 3 — alcance del ícono nuevo**: se agrega como un caso nuevo dentro del componente de íconos
  compartido existente (`shared/icon/icon.component.ts`), no como una librería externa ni un archivo
  de imagen — coherente con que el proyecto no usa hoy ninguna librería de íconos de terceros.
- **Bug 4 — alcance del switch**: `tracks_inventory` es un atributo a nivel de producto (no por
  variante, spec 027 FR-007); esta spec reutiliza esa misma señal ya presente en el formulario
  (`draft().tracks_inventory`) sin crear ningún atributo nuevo.
- **Referencias de archivo y línea**: citadas tal como están en `pos-heladeria` y `pos-backend` al
  momento de esta spec (2026-08-27); si el código cambia antes de implementarse, la spec debe
  re-verificarse contra el código vigente antes de usarse como contrato de corrección.
