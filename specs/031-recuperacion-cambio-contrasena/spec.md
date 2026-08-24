# Feature Specification: Recuperación y Cambio de Contraseña (Personal)

**Feature Branch**: `031-recuperacion-cambio-contrasena`

**Created**: 2026-08-24

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad **nueva** (fase de evolución funcional, Principio I de
la [Constitución](../../.specify/memory/constitution.md)). Se construye sobre
[spec 001](../001-auth-personal/spec.md) — que ya define cuenta, login y sesiones
(`access_token`/`refresh_token`) para el personal del local (cajero/admin) — y agrega dos
capacidades sobre ese modelo: un flujo de recuperación por correo que **hoy no existe** ("no hay
'reset' por correo dentro de este módulo", spec 001, User Story 3), y un endurecimiento del cambio
de contraseña autenticado que spec 001 ya documenta (equivalente a `POST /auth/change-password`,
`RN-AUTH-01`, `RN-AUTH-02`). **Cambia explícitamente** tres comportamientos ya definidos en spec
001, autorizados aquí como decisión de negocio (Principio II) a partir del detalle numérico dado
en esta misma solicitud: (1) la longitud de una contraseña nueva pasa de 6-128 caracteres
(`RN-AUTH-01`, `RN-AUTH-09`) a 8-12 caracteres; (2) un cambio de contraseña exitoso ahora cierra
otras sesiones activas de la cuenta, algo que hoy no ocurre; (3) un cambio exitoso ahora dispara un
correo de aviso al titular, algo que hoy tampoco ocurre. **No** reabre ni modifica la verificación
de `current_password` ni el flag `must_change_password` (`RN-AUTH-01`/`RN-AUTH-02`), que siguen
aplicando sin cambios en el flujo autenticado, ni la anomalía A-22 (truncamiento bcrypt a 72
bytes), que queda sin efecto práctico porque el nuevo máximo de 12 caracteres nunca la alcanza.

**Input**: User description: implementar dos flujos de cambio de contraseña para el personal
(cajero/admin) del POS: (A) recuperación no autenticada desde el login, mediante enlace de un solo
uso enviado por correo, válido 30 minutos exactos desde su emisión, con mensaje de confirmación
idéntico exista o no la cuenta (para no revelar qué correos están registrados) y un límite de 3
solicitudes por cuenta cada 15 minutos en ventana deslizante; y (B) cambio voluntario autenticado
desde Ajustes de cuenta, verificando la contraseña actual, sin afectar ningún otro dato del
perfil. Ambos flujos exigen una contraseña nueva de 8 a 12 caracteres (sin combinaciones
obligatorias de mayúsculas/números/símbolos), distinta de la actual, con doble confirmación
exacta, envían un correo de aviso tras cualquier cambio exitoso, y cierran las demás sesiones
activas de la cuenta (todas, en el flujo A; todas menos la actual, en el flujo B). Incluye reglas
numéricas de expiración e invalidación de enlaces, límite de solicitudes con ventana deslizante, y
casos límite de concurrencia, dispositivos distintos, doble envío y manejo de espacios/tildes en la
contraseña.

## Clarifications

### Session 2026-08-24

- Q: Cuando una cuenta supera el límite de 3 solicitudes de enlace en 15 minutos, ¿qué debe ver el
  usuario en pantalla — el mensaje distinto de bloqueo, o el mismo mensaje genérico de siempre? →
  A: El límite y su mensaje de bloqueo ("Has pedido demasiados enlaces...") aplican igual exista o
  no la cuenta detrás del correo ingresado — se cuenta por correo, no por cuenta confirmada. El
  mensaje genérico de no-revelación (FR-003) solo se muestra cuando la solicitud **no** está
  bloqueada por ese límite.
- Q: El sistema es multi-tenant (`x-tenant-host` resuelve el tenant en spec 001) y el mismo correo
  puede existir en cuentas de distintos tenants. Cuando alguien pide un enlace desde la pantalla de
  login de un tenant, ¿la búsqueda de la cuenta y el envío del enlace deben limitarse solo a ese
  tenant? → A: Sí — igual que el login (`RN-AUTH-05`, spec 001), la búsqueda de la cuenta, el envío
  del enlace y el conteo del límite de FR-010 se resuelven exclusivamente dentro del tenant
  identificado por `x-tenant-host` en la pantalla de login desde la que se originó la solicitud. Si
  el mismo correo tiene cuenta en otro tenant, esa cuenta no se ve afectada ni recibe correo.
- Q: Si alguien pide un enlace para un correo cuya cuenta existe pero está desactivada
  (`active=False`, spec 001), ¿se le envía el enlace igual, o se trata como si la cuenta no
  existiera? → A: Se trata igual que un correo sin cuenta — mensaje genérico de FR-003, sin enlace
  ni correo enviado. Preserva el mismo criterio que ya usa el login para negar acceso a cuentas
  desactivadas (`RN-AUTH-04`, spec 001).
- Q: Si la cuenta tenía activo el flag "debe cambiar contraseña" (`must_change_password`,
  `RN-AUTH-02` de spec 001), ¿fijar una contraseña nueva por el Flujo A también lo limpia, igual
  que ya hace el Flujo B? → A: Sí — el Flujo A también deja `must_change_password=False` al guardar
  con éxito, con el mismo criterio de `RN-AUTH-02`: fijar una contraseña nueva elegida por el
  usuario cumple el propósito del flag sin importar por cuál de los dos flujos se hizo.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Recuperar el acceso cuando se olvida la contraseña (Priority: P1)

Un usuario que olvidó su contraseña y no puede iniciar sesión pulsa "Restablecer contraseña" desde
la pantalla de login, escribe su correo, recibe un enlace de un solo uso, lo abre y define una
contraseña nueva escribiéndola dos veces. Al terminar, el sistema lo lleva a la pantalla de login,
donde entra con la contraseña nueva.

**Why this priority**: sin este flujo, una contraseña olvidada deja la cuenta bloqueada de forma
permanente — spec 001 no ofrece ningún mecanismo de recuperación por correo. Es el problema
central que motiva esta spec y el único camino de acceso para un usuario que no puede autenticarse
por ningún otro medio.

**Independent Test**: se puede probar completamente pidiendo un enlace con un correo registrado,
abriéndolo, definiendo una contraseña nueva, y confirmando que el login funciona con la contraseña
nueva y falla con la anterior — sin depender de que el flujo B (User Story 2) esté implementado.

**Acceptance Scenarios**:

1. **Given** la pantalla de inicio de sesión, **When** el usuario la abre en un móvil de 375 px de
   ancho, **Then** ve el enlace "Restablecer contraseña" sin necesidad de hacer scroll (FR-001).
2. **Given** el usuario pulsa "Restablecer contraseña", **When** llega a la pantalla siguiente,
   **Then** ve un único campo "Correo electrónico" y un botón "Enviar enlace" (FR-002).
3. **Given** un correo que sí pertenece a una cuenta, **When** el usuario lo escribe y pulsa
   "Enviar enlace", **Then** ve el mensaje "Si existe una cuenta con ese correo, te enviamos un
   enlace para restablecer tu contraseña. Revisa tu bandeja de entrada y la carpeta de spam." y
   recibe, en menos de 2 minutos, un correo con un enlace de un solo uso (FR-003, FR-004).
4. **Given** un correo que **no** pertenece a ninguna cuenta, **When** el usuario lo escribe y
   pulsa "Enviar enlace", **Then** ve exactamente el mismo mensaje del escenario anterior y no
   recibe ningún correo (FR-003).
5. **Given** un enlace emitido a las 14:00:00, **When** se abre a las 14:29:59, **Then** es válido
   y se muestran los campos "Nueva contraseña" y "Confirmar nueva contraseña"; **When** en cambio
   se abre a las 14:30:01, **Then** se muestra un mensaje de enlace caducado con un botón "Pedir un
   enlace nuevo", sin mostrar los campos (FR-007, FR-011).
6. **Given** el usuario completa "Nueva contraseña" y "Confirmar nueva contraseña" con la misma
   contraseña válida y pulsa "Guardar contraseña", **When** el guardado tiene éxito, **Then** el
   enlace queda consumido, todas las sesiones activas de la cuenta se cierran, y el usuario es
   enviado a la pantalla de login con un mensaje de confirmación, donde entra con la contraseña
   nueva y ya no con la anterior (FR-008, FR-009).
7. **Given** un enlace ya usado con éxito, **When** se abre de nuevo, **Then** se muestra el mismo
   mensaje de enlace inválido con el botón "Pedir un enlace nuevo" (FR-007, FR-008).
8. **Given** el usuario pide un enlace a las 14:00 (válido hasta las 14:30) y pide otro a las 14:10
   (válido hasta las 14:40), **When** intenta abrir a las 14:15 el enlace de las 14:00, **Then**
   ese enlace ya no funciona — solo el emitido a las 14:10 sigue vigente (FR-005).
9. **Given** una misma cuenta pide enlaces a las 10:00, 10:02 y 10:05 (3 solicitudes), **When**
   pide una 4ª a las 10:07, **Then** la solicitud queda bloqueada con el mensaje "Has pedido
   demasiados enlaces. Vuelve a intentarlo en unos minutos." y no se envía correo; **When** pide
   una 5ª a las 10:15:01, **Then** se envía normalmente, porque la solicitud de las 10:00 ya salió
   de la ventana de 15 minutos (FR-010).

---

### User Story 2 - Cambiar la contraseña voluntariamente desde Ajustes de cuenta (Priority: P2)

Un usuario con sesión activa entra a Ajustes de cuenta, ve la sección "Cambiar contraseña", escribe
su contraseña actual y la nueva dos veces, pulsa "Actualizar contraseña" y ve un mensaje de éxito.
El resto de los datos de su perfil (nombre, correo, etc.) queda exactamente igual.

**Why this priority**: endurece un mecanismo que ya existe hoy (spec 001, User Story 3) con
verificación de sesiones y aviso por correo, pero no es la vía de acceso urgente que resuelve una
cuenta bloqueada — por eso queda después de la recuperación (User Story 1).

**Independent Test**: con una sesión ya iniciada, se puede probar completamente entrando a Ajustes
de cuenta, escribiendo la contraseña actual y una nueva válida dos veces, pulsando "Actualizar
contraseña", y confirmando el mensaje de éxito y que el login posterior funciona con la contraseña
nueva — sin depender del flujo A (User Story 1).

**Acceptance Scenarios**:

1. **Given** un usuario en Ajustes de cuenta, **When** revisa la pantalla, **Then** ve una sección
   titulada "Cambiar contraseña" con los campos "Contraseña actual", "Nueva contraseña",
   "Confirmar nueva contraseña" y el botón "Actualizar contraseña" (FR-013, FR-014).
2. **Given** una contraseña actual correcta y una nueva válida escrita dos veces de forma idéntica,
   **When** el usuario pulsa "Actualizar contraseña", **Then** ve un mensaje de éxito en la misma
   pantalla, los tres campos quedan vacíos, y su sesión actual sigue iniciada mientras el resto de
   sesiones de esa cuenta se cierran (FR-017, FR-018).
3. **Given** una contraseña actual incorrecta, **When** el usuario pulsa "Actualizar contraseña",
   **Then** ve un error señalado en ese campo específico y la contraseña de la cuenta no cambia —
   verificable cerrando sesión e intentando entrar con la contraseña antigua (FR-016).
4. **Given** el usuario edita el nombre de su perfil en la misma pantalla sin guardarlo, **When**
   pulsa "Actualizar contraseña" con datos válidos, **Then** solo la contraseña cambia; al recargar
   la página, el nombre sigue siendo el que estaba guardado antes, no el editado (FR-015).

---

### User Story 3 - Recibir aviso de seguridad tras un cambio de contraseña (Priority: P3)

Tras cualquier cambio de contraseña exitoso, por cualquiera de los dos flujos, el titular de la
cuenta recibe un correo informándole del cambio, con fecha y hora, para que pueda reaccionar si no
fue él quien lo hizo.

**Why this priority**: es una capa de seguridad adicional, no un requisito para completar ninguno
de los dos flujos — puede probarse y entregar valor de forma independiente aunque no añade una
pantalla nueva.

**Independent Test**: se puede probar disparando un cambio de contraseña exitoso por cualquiera de
los dos flujos y confirmando que llega un correo de aviso a la cuenta, de forma independiente a
qué flujo lo originó.

**Acceptance Scenarios**:

1. **Given** un cambio de contraseña exitoso por el flujo de recuperación (User Story 1), **When**
   se completa, **Then** llega un correo a la cuenta con la fecha y hora del cambio y qué hacer si
   no fue el titular (FR-022).
2. **Given** un cambio de contraseña exitoso desde Ajustes de cuenta (User Story 2), **When** se
   completa, **Then** llega el mismo tipo de correo de aviso (FR-022).

---

### Edge Cases

- **Enlace abierto en un navegador o dispositivo distinto de donde se pidió**: funciona igual — no
  está atado al dispositivo ni al navegador que lo solicitó (FR-004).
- **Enlace abierto por un usuario que ya tiene sesión iniciada en otra cuenta**: esa sesión
  existente se cierra y el restablecimiento continúa con normalidad (FR-006, FR-007).
- **Doble clic en "Guardar contraseña", o reintento por mala conexión**: solo se aplica un cambio;
  el segundo intento se comporta como un enlace ya usado, no como un error confuso (FR-008).
- **El mismo enlace abierto en dos pestañas, guardado en ambas**: la segunda pestaña muestra el
  error de enlace ya usado (FR-007, FR-008).
- **La cuenta cambia de correo electrónico mientras hay un enlace vigente**: ese enlace queda
  invalidado (FR-012).
- **Contraseña nueva con espacios al inicio o al final**: se conservan tal cual, no se recortan
  (FR-025).
- **Contraseña nueva con tildes o eñes**: se acepta sin alterarla (FR-026).
- **Falla el envío del correo en el proveedor** (enlace o aviso de cambio): el usuario ve igual el
  mensaje genérico de éxito; el fallo queda registrado para diagnóstico interno, sin exponerse al
  usuario (FR-028).
- **Contraseña pegada desde un gestor de contraseñas**: los campos deben permitir pegar (FR-027).
- **Solicitud de enlace para una cuenta desactivada (`active=False`)**: se trata igual que un
  correo sin cuenta — mensaje genérico, sin enlace ni correo enviado (FR-004).
- **Solicitud de enlace para un correo que sí existe pero en otro tenant**: no se envía enlace ni
  correo para esa cuenta; la búsqueda y el límite de solicitudes se resuelven solo dentro del
  tenant de la pantalla de login de origen (FR-004, FR-010).

## Requirements *(mandatory)*

### Functional Requirements

**Flujo A — Recuperación desde el inicio de sesión**

- **FR-001**: El formulario de inicio de sesión DEBE incluir un enlace visible con el texto
  "Restablecer contraseña", visible sin scroll en un ancho de 375 px.
- **FR-002**: Ese enlace DEBE llevar a una pantalla con un único campo "Correo electrónico" y un
  botón "Enviar enlace".
- **FR-003**: Al enviar, si la solicitud no está bloqueada por el límite de FR-010, el sistema
  DEBE mostrar siempre el mismo mensaje, exista o no la cuenta: "Si existe una cuenta con ese
  correo, te enviamos un enlace para restablecer tu contraseña. Revisa tu bandeja de entrada y la
  carpeta de spam."
- **FR-004**: Si la cuenta existe **dentro del tenant identificado por `x-tenant-host` en la
  pantalla de login de origen** (spec 001, `RN-AUTH-05`) **y está activa** (`active=True`, spec
  001, `RN-AUTH-04`), el sistema DEBE enviar un correo con un enlace de un solo uso que caduca a
  los 30 minutos exactos de su emisión, en menos de 2 minutos desde la solicitud. Una cuenta con el
  mismo correo en otro tenant, o una cuenta desactivada (`active=False`), no cuenta como existente
  para esta solicitud: se comporta igual que un correo sin cuenta (FR-003), sin enlace ni correo.
- **FR-005**: Pedir un enlace nuevo DEBE invalidar de inmediato cualquier enlace anterior de esa
  misma cuenta que siga vigente.
- **FR-006**: El enlace DEBE abrir una pantalla con los campos "Nueva contraseña" y "Confirmar
  nueva contraseña" y un botón "Guardar contraseña". Si quien lo abre ya tiene una sesión iniciada
  en otra cuenta, esa sesión se cierra antes de continuar.
- **FR-007**: El sistema DEBE validar el enlace antes de mostrar los campos. Si el enlace es
  inválido, caducado o ya usado, DEBE mostrar en su lugar un mensaje de error con un botón "Pedir
  un enlace nuevo".
- **FR-008**: Al guardar con éxito, el enlace DEBE quedar consumido y no puede volver a usarse; una
  segunda confirmación sobre el mismo enlace (doble clic, reintento de red, o una segunda pestaña)
  DEBE tratarse como enlace ya usado, sin aplicar un segundo cambio ni mostrar un error distinto.
- **FR-009**: Tras guardar con éxito, el sistema DEBE cerrar todas las sesiones activas de esa
  cuenta, dejar `must_change_password=False` si estaba activo (`RN-AUTH-02`, spec 001), y enviar al
  usuario a la pantalla de inicio de sesión con un mensaje de confirmación.
- **FR-010**: El sistema DEBE limitar las solicitudes de enlace a 3 por correo ingresado cada 15
  minutos, en ventana deslizante, **exista o no una cuenta con ese correo, y dentro del tenant
  identificado por `x-tenant-host` en la pantalla de origen** (el conteo es por correo ingresado en
  ese tenant, no por cuenta confirmada, para no delatar por comportamiento diferido si la cuenta
  existe). Al superar el límite, DEBE mostrar el mensaje "Has pedido demasiados enlaces. Vuelve a
  intentarlo en unos minutos." sin enviar correo, sea o no una cuenta real. El mismo correo en otro
  tenant tiene su propio conteo independiente.
- **FR-011**: La vigencia del enlace DEBE contarse desde el instante en que se emite, no desde que
  se abre el correo.
- **FR-012**: Si la cuenta cambia de correo electrónico mientras un enlace sigue vigente, ese
  enlace DEBE quedar invalidado.

**Flujo B — Cambio desde Ajustes de cuenta**

- **FR-013**: En Ajustes de cuenta DEBE existir una sección titulada "Cambiar contraseña".
- **FR-014**: Esa sección DEBE contener los campos "Contraseña actual", "Nueva contraseña" y
  "Confirmar nueva contraseña", y un botón "Actualizar contraseña".
- **FR-015**: El botón "Actualizar contraseña" DEBE actualizar únicamente la contraseña — no debe
  guardar, modificar ni enviar ningún otro dato del perfil, aunque el usuario haya editado otros
  campos de la misma pantalla sin guardarlos.
- **FR-016**: Si la contraseña actual es incorrecta, el sistema DEBE señalarlo en ese campo y no
  cambiar nada. La verificación reutiliza, sin cambios, la de spec 001 (`RN-AUTH-01`).
- **FR-017**: Tras un cambio con éxito, la sesión desde la que se hizo el cambio DEBE seguir activa;
  el resto de sesiones de esa cuenta DEBEN cerrarse.
- **FR-018**: Tras un cambio con éxito, los tres campos DEBEN vaciarse y mostrarse un mensaje de
  confirmación en la misma pantalla.

**Comunes a ambos flujos**

- **FR-019**: La contraseña nueva DEBE tener entre 8 y 12 caracteres, sin exigir combinaciones
  obligatorias de mayúsculas, números ni símbolos.
- **FR-020**: Los campos "Nueva contraseña" y "Confirmar nueva contraseña" DEBEN coincidir
  exactamente; si no coinciden, el sistema DEBE señalarlo antes de enviar nada y no cambiar la
  contraseña.
- **FR-021**: La contraseña nueva NO DEBE poder ser igual a la contraseña que la cuenta tiene en
  ese momento (solo se compara contra la actual, no contra un historial).
- **FR-022**: Tras cualquier cambio de contraseña exitoso (por cualquiera de los dos flujos), el
  sistema DEBE enviar un correo de aviso a la dirección de la cuenta, indicando fecha y hora del
  cambio y qué hacer si no fue el titular.
- **FR-023**: Los campos de contraseña DEBEN ocultar el texto por defecto y ofrecer un control para
  mostrarlo.
- **FR-024**: Todas las pantallas de esta spec (login, solicitud de enlace, definición de
  contraseña nueva, sección de Ajustes) DEBEN funcionar en móvil.
- **FR-025**: Los espacios al inicio o al final de la contraseña nueva DEBEN conservarse tal cual,
  sin recortarse.
- **FR-026**: Tildes y eñes en la contraseña nueva DEBEN aceptarse sin alterarse.
- **FR-027**: Los campos de contraseña DEBEN permitir pegar texto (por ejemplo, desde un gestor de
  contraseñas).
- **FR-028**: Si el envío del correo falla en el proveedor (enlace o aviso de cambio), el usuario
  DEBE ver igualmente el mensaje genérico de éxito; el fallo DEBE quedar registrado para
  diagnóstico interno.

### Key Entities *(include if feature involves data)*

- **Cuenta**: la misma entidad ya definida en spec 001 (personal cajero/admin, con su contraseña
  actual, incluido su `tenant_id`). Esta spec no agrega atributos nuevos a la cuenta, salvo la
  relación con las solicitudes de restablecimiento. La búsqueda de la cuenta por correo para el
  Flujo A se hace dentro del tenant resuelto por `x-tenant-host` en la pantalla de origen, igual
  que el login (spec 001, `RN-AUTH-05`); el mismo correo en otro tenant es una cuenta distinta.
- **Solicitud de restablecimiento (enlace)**: token de un solo uso asociado a una cuenta (y por lo
  tanto a su tenant), con momento de emisión y momento de expiración (emisión + 30 minutos). Su
  estado es vigente, usado, o invalidado (por una solicitud posterior de la misma cuenta, o por un
  cambio de correo de la cuenta). Solo se crea cuando el correo ingresado sí pertenece a una cuenta
  dentro del tenant de origen.
- **Conteo de solicitudes (límite de frecuencia)**: contador de intentos por correo ingresado
  dentro de un tenant, en una ventana deslizante de 15 minutos (FR-010), independiente de si ese
  correo pertenece a una cuenta real en ese tenant. Es un concepto separado del token: existe
  incluso para correos sin cuenta, para que el límite se aplique de forma idéntica y no revele por
  su comportamiento si la cuenta existe. El mismo correo en otro tenant tiene su propio contador.
- **Sesión**: la misma entidad ya definida en spec 001 (`access_token`/`refresh_token`). Lo nuevo
  aquí es que un cambio de contraseña exitoso cierra sesiones de la cuenta — todas, en el flujo A;
  todas menos la que originó el cambio, en el flujo B.
- **Aviso de cambio de contraseña**: correo enviado al titular de la cuenta tras cualquier cambio
  exitoso, con fecha y hora del cambio. No se persiste como historial visible dentro del producto.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un usuario que olvidó su contraseña, con acceso a su correo, recupera el acceso a su
  cuenta sin intervención de soporte en menos de 5 minutos, contando desde que pide el enlace hasta
  que entra con la contraseña nueva.
- **SC-002**: El correo con el enlace de restablecimiento llega a la bandeja del usuario en menos
  de 2 minutos en el 100% de las solicitudes con cuenta registrada y no bloqueada por el límite de
  solicitudes.
- **SC-003**: El 0% de las solicitudes con un correo no registrado revela, mediante el mensaje en
  pantalla o el envío de un correo, si esa cuenta existe.
- **SC-004**: El 100% de los enlaces abiertos después de sus 30 minutos de vigencia, o ya usados,
  muestran el mensaje de error correspondiente en vez de los campos de contraseña.
- **SC-005**: El 100% de los cambios de contraseña exitosos cierran las sesiones que corresponde
  cerrar según el flujo (todas, en recuperación; todas menos la de origen, en el cambio
  autenticado), verificable recargando una sesión abierta en otro navegador.
- **SC-006**: El 100% de los cambios de contraseña exitosos, por cualquiera de los dos flujos,
  generan un correo de aviso al titular de la cuenta.
- **SC-007**: El 100% de los cambios de contraseña realizados desde Ajustes de cuenta dejan el
  resto de los datos del perfil (nombre, correo, etc.) exactamente igual a como estaban guardados
  antes del cambio, incluso si el usuario había editado otros campos sin guardarlos.
- **SC-008**: Tras el lanzamiento, las solicitudes de soporte para restablecer manualmente una
  contraseña olvidada de un usuario con acceso a su correo se reducen a cero.

## Out of Scope

- Autenticación en dos pasos (2FA/MFA) de cualquier tipo.
- Recuperación de cuenta cuando el usuario ya no tiene acceso a su correo.
- Preguntas de seguridad.
- Recuperación por SMS o teléfono.
- Historial de contraseñas (impedir reutilizar las N anteriores) — solo se compara contra la
  contraseña actual (FR-021).
- Medidor visual de fortaleza de contraseña.
- Comprobación contra listas de contraseñas filtradas.
- Inicio de sesión sin contraseña (enlace mágico).
- Restablecimiento forzado por un administrador sobre la cuenta de otro usuario.
- Caducidad periódica obligatoria de contraseñas.
- Registro visible para el usuario, dentro del producto, de sus cambios de contraseña o de sus
  sesiones activas — el único rastro es el correo de aviso (FR-022).

## Assumptions

- **"Usuario" en esta spec es el mismo personal del local (cajero/admin) de spec 001** — no un
  comensal, que en este sistema no tiene cuenta con contraseña (spec 007).
- **Cambio de comportamiento explícito #1** (Principio II): la longitud de una contraseña nueva
  pasa de 6-128 caracteres (spec 001, `RN-AUTH-01`, `RN-AUTH-09`) a 8-12 caracteres, para cualquier
  contraseña fijada de aquí en adelante por cualquiera de los dos flujos. Contraseñas ya existentes
  con más de 12 caracteres siguen siendo válidas para iniciar sesión hasta que se cambien.
- **Cambio de comportamiento explícito #2**: hoy ningún cambio de contraseña cierra otras sesiones;
  a partir de esta spec, todo cambio exitoso cierra las sesiones que correspondan según el flujo
  (FR-009, FR-017).
- **Cambio de comportamiento explícito #3**: hoy ningún cambio de contraseña notifica por correo; a
  partir de esta spec, todo cambio exitoso dispara un correo de aviso al titular (FR-022).
- **El flujo B endurece, no reemplaza, el mecanismo ya definido en spec 001 (User Story 3)**: la
  verificación de la contraseña actual y la limpieza del flag `must_change_password` al guardar con
  éxito (`RN-AUTH-01`, `RN-AUTH-02`) siguen aplicando sin cambios; lo nuevo es la longitud
  (FR-019), el cierre de otras sesiones (FR-017) y el correo de aviso (FR-022). El flujo A, al ser
  un mecanismo nuevo, aplica ese mismo criterio de `RN-AUTH-02` por primera vez a un cambio no
  autenticado: también limpia `must_change_password` al guardar con éxito (FR-009).
- **La anomalía A-22 (truncamiento bcrypt a 72 bytes) queda sin efecto práctico** para contraseñas
  nuevas: el máximo de 12 caracteres de esta spec nunca alcanza ese límite, ni siquiera con
  caracteres multibyte (tildes, eñes).
- **El límite de 3 solicitudes de enlace cada 15 minutos se cuenta por correo ingresado dentro de
  un mismo tenant** (exista o no cuenta detrás de ese correo, para no delatar su existencia por el
  comportamiento del límite — ver Clarifications), no por IP ni por dispositivo.
- Los textos de pantalla y de correo se redactan en español, consistentes con el resto del
  producto.
