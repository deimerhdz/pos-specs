# Feature Specification: Acceso de Super Admin en `admin.skeilopos.com` y Nombre Completo en Invitaciones

**Feature Branch** (la spec vive en la carpeta `specs/091-admin-subdominio-nombre-completo/` de `pos-specs`, rama `main`). Las ramas de código de `pos-backend` y `pos-heladeria` siguen el Principio XIV y se definen en `tasks.md`

**Created**: 2026-10-02

**Status**: Ready for implementation

**Input**: User description: "Resolver el enrutamiento y la autenticación global aislando el acceso de los Super Administradores en el subdominio `admin.skeilopos.com`, y mejorar la gestión de personal permitiendo registrar y visualizar el nombre completo al invitar usuarios a un tenant. (Brief completo con 13 escenarios BDD: 4 de Super Admin y 9 de invitaciones.)"

Usuarios:

- **Super Admin**: usuario de plataforma, sin negocio (tenant) asignado, que administra globalmente el sistema.
- **Administrador de Tenant**: dueño o gestor de una heladería que invita a su personal (cajeros, meseros, otros administradores).

## Clarifications

### Session 2026-10-02

- Q: ¿Los 13 escenarios del brief están todos cubiertos? → A: **Sí**: 4 escenarios "Admin" (Historias 1 y 2) + 9 escenarios de invitaciones (Historias 3 a 5). La tabla de trazabilidad al final de los requisitos mapea cada uno a sus FR y SC.
- Q: ¿Qué nombre se muestra a quien hoy no tiene nombre capturado? → A: **su correo**, tanto en el texto como en la inicial del avatar (sin migración forzada de datos).

- Q: ¿Además de `admin`, qué otros nombres de subdominio se reservan? → A: **Opción B (`admin`, `assets`, `api`) más `docs`** por decisión del usuario, además de las ya reservadas `www` y `app` (lista final: `www`, `app`, `admin`, `assets`, `api`, `docs`). `assets` protege el dominio de imágenes `assets.skeilopos.com`.
- Q: ¿Qué caracteres cuentan como "letras" válidas en el nombre completo? → A: **cualquier letra del alfabeto latino, incluidas las extendidas** (tildes, ñ, ü, ë, ç, ø, ß, etc.); no se aceptan otros alfabetos (cirílico, árabe, chino, etc.).
- Q: ¿Qué pasa con el acceso del Super Admin por el dominio raíz `skeilopos.com`? → A: **deja de existir**: `skeilopos.com` ya sirve otra página (landing), así que no lleva a ningún login ni redirige. El único acceso de plataforma es `admin.skeilopos.com` (FR-006). Hoy la aplicación trata el dominio raíz como contexto de Super Admin; esa conducta se retira.
- Q: ¿`http://localhost:4200` (sin `admin`) debe seguir llevando al login de Super Admin como "modo local"? → A: **No** (aclaración del usuario tras probar la implementación, 2026-10-02). Al login de plataforma solo se llega con la URL con `admin` (`admin.localhost:4200` en desarrollo, `admin.skeilopos.com` en producción). `localhost` y `127.0.0.1` se tratan como cualquier host no reconocido: aviso "Esta dirección no corresponde a ningún acceso", sin formulario ni llamadas al API. Se elimina el "modo local" que la spec y el plan habían supuesto.

### Hallazgos del reconocimiento (contexto, no decisiones)

- El inicio de sesión resuelve el negocio por el host desde el que se entra. Si el host **no corresponde a ningún negocio registrado**, hoy el sistema busca la cuenta entre los **Super Admin**. Consecuencia: un Super Admin podría iniciar sesión desde un subdominio inexistente o mal escrito (por ejemplo `cualquier-cosa.skeilopos.com`), lo que contradice el aislamiento que pide este brief (Escenario Admin 3 cubre solo un subdominio de negocio existente). Esta spec exige que **solo el subdominio de plataforma** admita el acceso global.
- En el frontend, el subdominio `admin` hoy **no** figura entre los reservados (`www`, `app`): se interpretaría como un negocio llamado "admin". El alta de negocios tampoco rechaza hoy ese nombre.
- Las cuentas ya guardan un nombre obligatorio. Al aceptar una invitación hoy se les pone **el correo como nombre**, porque la invitación no captura nombre alguno. Es el origen de los "usuarios legacy" del Escenario 8: su nombre guardado es su propio correo (o está vacío, en cuentas creadas por el Super Admin con el nombre opcional).
- El dashboard del Super Admin ya tiene un formulario de usuarios con el campo "Nombre" marcado como **opcional**; el del Administrador de Tenant (página "Usuarios") no tiene campo de nombre.
- La causa exacta del error 404 en `admin.skeilopos.com/login` **no está diagnosticada**: puede estar en el enrutamiento del hosting, en el reconocimiento del subdominio por parte de la aplicación, o en ambos. La fase de plan debe diagnosticarla antes de corregirla; esta spec define solo el comportamiento esperado.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El Super Admin entra por `admin.skeilopos.com/login` (Priority: P1)

Como Super Admin, quiero abrir `admin.skeilopos.com/login`, iniciar sesión con mi cuenta de plataforma y llegar al panel central de administración, sin errores de página no encontrada.

**Why this priority**: hoy ese punto de entrada responde 404; el Super Admin no tiene una puerta oficial y dedicada a la administración global. Es el bloqueo principal del brief.

**Independent Test**: abrir `admin.skeilopos.com/login` con una cuenta de Super Admin válida y comprobar que se llega al panel central; recargar el navegador en una ruta interna del panel y comprobar que tampoco da 404.

**Acceptance Scenarios**:

1. **Given** soy un Super Admin con cuenta activa, **When** abro `admin.skeilopos.com/login` y envío mis credenciales correctas, **Then** el sistema me autentica contra las cuentas de plataforma y me da acceso al panel central. *(Escenario Admin 1)*
2. **Given** estoy dentro del panel en `admin.skeilopos.com`, **When** recargo la página o abro directamente un enlace interno del panel, **Then** la página carga (no hay 404) y conservo mi sesión.
3. **Given** envío credenciales incorrectas en `admin.skeilopos.com/login`, **When** el sistema las evalúa, **Then** veo el mismo mensaje de credenciales inválidas que en cualquier otro inicio de sesión, sin pista de si la cuenta existe.
4. **Given** soy un Super Admin con la cuenta desactivada, **When** intento entrar, **Then** se me rechaza con el mismo aviso de cuenta inactiva que ya existe.

---

### User Story 2 - Aislamiento estricto entre plataforma y negocios (Priority: P1)

Como dueño de la plataforma, quiero que las cuentas de plataforma solo puedan iniciar sesión en el subdominio de plataforma y las cuentas de negocio solo en el subdominio de su propio negocio, y que ningún negocio pueda apropiarse del nombre `admin`.

**Why this priority**: es una frontera de seguridad. Si se rompe, una cuenta de negocio podría llegar a la administración global, o un Super Admin operar con la identidad de un negocio.

**Independent Test**: intentar los cuatro cruces (negocio→admin, Super Admin→negocio, Super Admin→subdominio inexistente, alta de negocio con `admin`) y comprobar que todos se rechazan, y que el acceso legítimo de cada tipo de cuenta sigue funcionando.

**Acceptance Scenarios**:

1. **Given** soy un usuario de un negocio, **When** intento iniciar sesión en `admin.skeilopos.com/login` con mis credenciales correctas, **Then** el sistema rechaza mi acceso (no se busca en ningún negocio ni se me concede sesión). *(Escenario Admin 2)*
2. **Given** soy un Super Admin, **When** intento iniciar sesión en `tenant-prueba.skeilopos.com/login` (un negocio existente), **Then** el sistema rechaza mi acceso. *(Escenario Admin 3)*
3. **Given** soy un Super Admin, **When** intento iniciar sesión desde un subdominio que no corresponde a ningún negocio registrado ni al de plataforma, **Then** el sistema rechaza mi acceso (el aislamiento no depende de que el negocio exista).
4. **Given** soy Super Admin y tengo sesión abierta en `admin.skeilopos.com`, **When** abro un subdominio de negocio, **Then** esa sesión no me da acceso allí; y a la inversa, una sesión de negocio no da acceso a `admin.skeilopos.com`.
5. **Given** intento registrar un negocio nuevo, **When** uso `admin`, `assets`, `api` o `docs` como su subdominio (en cualquier combinación de mayúsculas/minúsculas o con espacios alrededor), **Then** el sistema lo bloquea indicando que es una palabra reservada. *(Escenario Admin 4)*
6. **Given** un negocio ya existente, **When** inicia sesión en su propio subdominio como siempre, **Then** el comportamiento no cambia.

---

### User Story 3 - Invitar a una persona registrando su nombre completo (Priority: P1)

Como Administrador de Tenant, en la página "Usuarios", quiero capturar el "Nombre completo" junto con el correo y el rol al invitar a alguien, para que el correo de invitación lo salude por su nombre y la lista de usuarios lo muestre identificado.

**Why this priority**: es el valor de negocio central de la segunda mitad del brief: hoy el personal aparece en la lista solo con su correo.

**Independent Test**: invitar a "María Pérez" con un correo y un rol, comprobar que el correo recibido la saluda por su nombre, que al aceptar la invitación la lista de usuarios muestra "María Pérez" con una "M" en el avatar.

**Acceptance Scenarios**:

1. **Given** soy Administrador de Tenant en "Usuarios", **When** completo "Nombre completo" con "María Pérez", añado un correo, selecciono un rol y presiono "Enviar", **Then** se crea la invitación, el correo saluda a "María Pérez", y cuando la persona acepta, la lista de usuarios muestra "María Pérez" con una "M" en el avatar. *(Escenario 1)*
2. **Given** escribo "José Ñañez O'Brien-Díaz", **When** envío la invitación, **Then** se procesa correctamente y el nombre se muestra tal cual, con tildes, ñ, apóstrofe y guion. *(Escenario 5)*
3. **Given** la invitación está pendiente de aceptar, **When** veo la lista de invitaciones pendientes, **Then** cada una muestra el nombre capturado junto al correo (o solo el correo si es una invitación anterior a esta spec).
4. **Given** el correo de invitación, **When** el nombre contiene caracteres que en un correo HTML serían especiales (por ejemplo un apóstrofe), **Then** el saludo los muestra literalmente, sin romper el formato del correo.

---

### User Story 4 - Validación del nombre (Priority: P1)

Como Administrador de Tenant, quiero que el formulario me avise de inmediato si el nombre no es válido, y quiero que el servidor rechace nombres inválidos aunque alguien se salte el formulario.

**Why this priority**: sin validación en servidor, el campo es una vía de inyección de contenido (HTML/scripts) hacia la pantalla del personal y hacia los correos.

**Independent Test**: probar vacío, solo espacios, 1 carácter, 101 caracteres, contenido con etiquetas HTML y una petición directa al servicio sin nombre; en todos se bloquea el envío y se muestra el mensaje correcto.

**Acceptance Scenarios**:

1. **Given** dejo "Nombre completo" vacío, **When** envío, **Then** veo "El nombre es obligatorio" y no se envía la invitación. *(Escenario 2)*
2. **Given** escribo únicamente espacios, **When** envío, **Then** se trata como vacío y veo "El nombre es obligatorio". *(Escenario 3)*
3. **Given** escribo un nombre de 1 carácter, o de más de 100, **When** envío, **Then** veo un mensaje que indica que debe tener entre 2 y 100 caracteres y se bloquea el envío. *(Escenario 4)*
4. **Given** escribo un nombre con etiquetas HTML o scripts (por ejemplo `<script>alert(1)</script>`), **When** envío, **Then** se rechaza por formato inválido y no se guarda ni se envía nada. *(Escenario 6)*
5. **Given** se envía una petición directa al servicio de invitaciones sin nombre, **When** el servidor la evalúa, **Then** responde con error de validación (HTTP 422) indicando que el nombre es obligatorio. *(Escenario 7)*
6. **Given** escribo " Ana " (espacios al inicio y al final), **When** envío, **Then** el nombre se guarda como "Ana" y la validación de longitud se hizo sobre el texto ya recortado.
7. **Given** escribo un nombre con dígitos o símbolos (por ejemplo "Ana3" o "Ana@"), **When** envío, **Then** se rechaza por formato inválido con un mensaje que explique los caracteres permitidos.
8. **Given** dos invitaciones con exactamente el mismo nombre y correos distintos, **When** se envían, **Then** ambas se aceptan (el nombre no es único).

---

### User Story 5 - Lista de usuarios con nombre, avatar y compatibilidad con cuentas anteriores (Priority: P2)

Como Administrador de Tenant, quiero que la lista de usuarios muestre el nombre de cada persona y la inicial en su avatar, y que las cuentas anteriores, sin nombre, sigan viéndose bien mostrando su correo.

**Why this priority**: da el beneficio visible de la captura del nombre y evita que el cambio rompa la lista de las cuentas existentes; depende de la Historia 3 pero es verificable por separado con datos de prueba.

**Independent Test**: abrir "Usuarios" en un negocio con cuentas antiguas (sin nombre propio) y cuentas nuevas con nombre; comprobar que las antiguas muestran el correo y la inicial del correo, las nuevas el nombre y su inicial, y que no hay errores.

**Acceptance Scenarios**:

1. **Given** existen usuarios antiguos sin nombre configurado, **When** abro "Usuarios", **Then** cada uno muestra su correo como nombre, la inicial del correo en el avatar, y la lista funciona sin errores. *(Escenario 8)*
2. **Given** un usuario con nombre "María Pérez", **When** abro "Usuarios", **Then** el avatar muestra "M" en mayúscula y el nombre completo aparece como título de la fila.
3. **Given** el formulario de invitación abierto con un nombre escrito, **When** presiono "Cancelar" y lo vuelvo a abrir, **Then** el campo "Nombre completo" está completamente vacío (y también los demás campos y mensajes de error). *(Escenario 9)*
4. **Given** un nombre largo (100 caracteres), **When** se muestra en la lista, **Then** no rompe el diseño de la fila.

---

### User Story 6 - Nombre completo al crear usuarios desde el dashboard del Super Admin (Priority: P2)

Como Super Admin, quiero que el formulario con el que doy de alta usuarios de un negocio exija el mismo "Nombre completo" con las mismas reglas, para que el personal quede identificado igual sin importar quién lo creó.

**Why this priority**: el brief pide el campo en ambos dashboards; se separa porque el formulario del Super Admin es distinto (hoy tiene el nombre opcional y llama a endpoints que no existen en el servicio, ver A-102) y puede entregarse después de la Historia 3.

**Independent Test**: en el dashboard del Super Admin, probar el formulario con nombre vacío y con nombres inválidos y comprobar las mismas reglas y mensajes que en la Historia 4; y comprobar que la tabla muestra nombre e inicial de los usuarios ya existentes.

**Acceptance Scenarios**:

1. **Given** estoy creando un usuario desde el dashboard del Super Admin, **When** dejo el nombre vacío o solo espacios, **Then** veo "El nombre es obligatorio" y no se crea.
2. **Given** el mismo formulario, **When** escribo un nombre fuera de 2–100 caracteres o con caracteres no permitidos, **Then** veo los mismos mensajes que en la Historia 4.
3. **Given** usuarios existentes en la tabla del Super Admin, **When** la abro, **Then** cada uno se muestra con su nombre e inicial (o con su correo e inicial del correo si no tiene nombre propio).
4. **Given** edito un usuario antiguo sin nombre, **When** guardo cambios sin tocar el nombre, **Then** el guardado no se bloquea por el nombre vacío; si escribo un nombre, debe cumplir las reglas.

---

### Edge Cases

- ¿Qué pasa con invitaciones **ya enviadas y pendientes** antes de esta spec (no tienen nombre)? Siguen siendo válidas y se aceptan como hoy; su cuenta queda con el correo como nombre (comportamiento legacy) y la lista de pendientes muestra solo el correo. No se invalidan ni se exige reenviarlas.
- ¿Qué pasa si se **reenvía** una invitación? Conserva el nombre capturado originalmente; el correo reenviado también saluda por ese nombre.
- ¿Y si se reenvía una invitación legacy sin nombre? El saludo del correo usa una fórmula genérica sin nombre (no el correo en el saludo) y sigue funcionando.
- ¿Qué pasa con nombres con **apóstrofe tipográfico** (’) o con mayúsculas/minúsculas mezcladas? Se aceptan tal cual; no se cambia el uso de mayúsculas.
- ¿Y con nombres con **solo guiones o apóstrofes** ("---", "''")? Se rechazan: debe haber al menos dos letras.
- ¿Y con **emojis, dígitos o puntuación** ("Ana 😀", "Ana2", "Ana.")? Se rechazan por formato inválido.
- ¿Y con **letras latinas extendidas** ("François", "Zoë", "Müller", "Søren")? Se aceptan. ¿Y con otros alfabetos ("Иван", "李雷")? Se rechazan por formato inválido.
- ¿Qué pasa si el nombre tiene **espacios internos repetidos** ("María   Pérez")? Se guarda sin alterar el contenido interno, solo recortado en los extremos; la validación de formato los admite.
- ¿Qué pasa si ya existe un negocio cuyo subdominio es `admin`, `assets`, `api` o `docs` (o con otras mayúsculas)? La spec no lo migra ni lo borra; la verificación previa debe confirmar que **no existe** ninguno y, si existiera, se resuelve como decisión aparte antes de desplegar.
- ¿Qué pasa con los subdominios reservados que ya existen (`www`, `app`) y con `localhost`/entornos de desarrollo? `www` y `app` dejan de dar acceso (host no reconocido). `localhost` y `127.0.0.1` tampoco dan acceso: en desarrollo la plataforma se prueba en `admin.localhost:4200` y los negocios en `<slug>.localhost:4200`.
- ¿Qué pasa si un Super Admin abre un subdominio de negocio ya con sesión de plataforma? No obtiene acceso (Historia 2, escenario 4); se le dirige al inicio de sesión.
- ¿Qué pasa con las **sesiones** de Super Admin ya abiertas desde el dominio raíz o desde un entorno anterior al desplegar? La aplicación deja de reconocerlas como sesión de plataforma fuera de `admin.skeilopos.com` (la dirección se trata como no reconocida) y el Super Admin vuelve a entrar por `admin.skeilopos.com/login`; no se pierde ningún dato. El token no se revoca en el servidor: sigue sirviendo solo para las rutas de plataforma, nunca para las de un negocio (FR-007).
- ¿Qué pasa con quien abra `skeilopos.com/login` o guarde esa dirección como favorito? El dominio raíz sirve su propia página (fuera del alcance de esta spec); no se redirige ni se ofrece login ahí (FR-006).
- ¿Qué pasa con las **respuestas de error** del inicio de sesión rechazado por aislamiento? Deben ser indistinguibles de credenciales incorrectas, para no revelar si la cuenta existe en el otro ámbito.

## Requirements *(mandatory)*

### Functional Requirements

**Acceso de plataforma (Super Admin)**

- **FR-001**: `admin.skeilopos.com/login` MUST cargar la pantalla de inicio de sesión de plataforma sin error 404, y toda ruta interna del panel central MUST poder abrirse directamente o recargarse en ese subdominio.
- **FR-002**: En el subdominio de plataforma, el inicio de sesión MUST autenticar únicamente cuentas de plataforma (sin negocio asignado); MUST NOT buscar ni aceptar cuentas de ningún negocio.
- **FR-003**: Una cuenta de negocio que intente iniciar sesión en el subdominio de plataforma MUST ser rechazada, con una respuesta indistinguible de "credenciales inválidas".
- **FR-004**: Una cuenta de plataforma que intente iniciar sesión en el subdominio de un negocio MUST ser rechazada, con una respuesta indistinguible de "credenciales inválidas".
- **FR-005**: Una cuenta de plataforma que intente iniciar sesión desde un subdominio que **no es el de plataforma ni corresponde a un negocio registrado** MUST ser rechazada: el acceso global solo existe en el subdominio de plataforma.
- **FR-006**: El dominio raíz `skeilopos.com` MUST NOT ofrecer ningún inicio de sesión, ni de plataforma ni de negocio: ya sirve otra página (decisión del negocio, 2026-10-02) y su autenticación no se resuelve desde esta aplicación. Cualquier intento de iniciar sesión que llegue al servicio de autenticación desde el dominio raíz MUST ser rechazado como cualquier otro host no autorizado (FR-005). El único punto de entrada de plataforma es `admin.skeilopos.com`.
- **FR-007**: Una sesión emitida para plataforma MUST NOT dar acceso a las pantallas ni datos de un negocio desde un subdominio de negocio, y una sesión de negocio MUST NOT dar acceso al panel central.
- **FR-008**: Los nombres `admin`, `assets`, `api` y `docs` MUST quedar reservados a nivel de plataforma: el alta de un negocio con cualquiera de ellos como subdominio MUST ser bloqueada, comparando sin distinguir mayúsculas/minúsculas ni espacios alrededor, con un mensaje que indique que es una palabra reservada. La reserva debe valer tanto en la pantalla de alta como en el servicio (que no se pueda saltar con una petición directa).
- **FR-009**: Las palabras ya reservadas hoy (`www`, `app`) MUST seguir reservadas; la lista final de reservadas (`www`, `app`, `admin`, `assets`, `api`, `docs`) MUST ser única y coherente entre lo que ve la pantalla y lo que valida el servicio.
- **FR-010**: Para el inicio de sesión de un negocio existente en su propio subdominio, el comportamiento MUST permanecer idéntico al actual.

**Nombre completo en invitaciones y usuarios**

- **FR-011**: El formulario "Invitar usuario" de la página "Usuarios" del Administrador de Tenant MUST incluir el campo obligatorio "Nombre completo", además del correo y el rol.
- **FR-012**: El formulario de usuarios del dashboard del Super Admin MUST tratar "Nombre completo" como obligatorio al **crear** un usuario, con las mismas reglas (hoy es opcional). Este requisito cubre la validación en pantalla; el guardado depende de la decisión registrada en A-102.
- **FR-013**: Antes de validar, el nombre MUST recortarse (espacios al inicio y al final); un nombre que queda vacío tras recortar MUST tratarse como no informado.
- **FR-014**: Reglas del nombre: obligatorio; entre 2 y 100 caracteres (tras recortar); solo letras del alfabeto latino (incluidas tildes, ñ y letras extendidas como ü, ë, ç, ø o ß; no otros alfabetos), espacios, apóstrofes y guiones; MUST contener al menos dos letras; sin requisito de unicidad.
- **FR-015**: Mensajes de validación en español: "El nombre es obligatorio" (vacío o solo espacios); "El nombre debe tener entre 2 y 100 caracteres"; y un mensaje de formato inválido que indique los caracteres permitidos. Si varias reglas fallan a la vez, se muestra la primera en ese orden.
- **FR-016**: El servidor MUST validar las mismas reglas con independencia de la pantalla: una petición sin nombre, con nombre vacío/solo espacios o con formato inválido MUST responder error de validación (HTTP 422) con el motivo, y MUST NOT crear invitación ni enviar correo.
- **FR-017**: Un nombre con etiquetas HTML, scripts u otros caracteres fuera de los permitidos MUST ser rechazado por formato inválido; MUST NOT guardarse ni mostrarse como contenido interpretable en ninguna pantalla ni correo.
- **FR-018**: La invitación MUST conservar el nombre capturado hasta que se acepte, y la cuenta resultante MUST quedar con ese nombre. Reenviar la invitación MUST conservar el nombre.
- **FR-019**: El correo de invitación MUST saludar a la persona por su nombre completo ("Hola, María Pérez") y MUST mostrar el nombre sin alterar el formato del correo. Cuando la invitación no tenga nombre (anterior a esta spec), el correo MUST usar un saludo genérico.
- **FR-020**: La lista de usuarios (tanto en la página "Usuarios" del negocio como en el dashboard del Super Admin) MUST mostrar el nombre de cada persona y, en el avatar, la inicial del nombre en mayúscula.
- **FR-021**: La lista de invitaciones pendientes MUST mostrar el nombre capturado junto al correo cuando exista.
- **FR-022**: Para usuarios sin nombre propio (nombre vacío o igual a su correo) la lista MUST mostrar el correo como nombre y su inicial en el avatar, sin errores, sin migración forzada de datos y sin alterar los datos existentes.
- **FR-023**: Al presionar "Cancelar" en el formulario de invitación (o cerrarlo), todos sus campos, incluido "Nombre completo", y sus mensajes de error MUST quedar vacíos al volver a abrirlo. Lo mismo tras un envío exitoso.
- **FR-024**: Las invitaciones pendientes anteriores a esta spec MUST seguir aceptándose sin cambios en su comportamiento.
- **FR-025**: Un nombre válido MUST mostrarse exactamente como se escribió (salvo el recorte de extremos), con tildes, ñ, apóstrofes y guiones intactos.

### Key Entities

- **Cuenta de plataforma (Super Admin)**: persona sin negocio asignado que administra todo el sistema. Solo puede autenticarse en el subdominio de plataforma.
- **Cuenta de negocio**: persona perteneciente a un negocio, con rol (Administrador, Cajero, Mesero). Solo puede autenticarse en el subdominio de su negocio.
- **Subdominio de negocio**: identificador único con que se registra un negocio y que forma su dirección. Existe una lista de palabras reservadas que ningún negocio puede usar; esta spec añade `admin`, `assets`, `api` y `docs`.
- **Invitación**: oferta pendiente de alta de una persona en un negocio, con correo, rol y —desde esta spec— nombre completo; al aceptarse da origen a una cuenta de negocio con ese nombre.
- **Nombre completo**: texto identificador de la persona (2–100 caracteres, letras latinas incluidas las extendidas, espacios, apóstrofes y guiones), no único, mostrado en listas, avatar y correos.

### Trazabilidad de los 13 escenarios del brief

| Escenario del brief | Cubierto por |
|---|---|
| Admin 1 (éxito) | Historia 1; FR-001, FR-002 |
| Admin 2 (negocio → admin) | Historia 2.1; FR-003, FR-007 |
| Admin 3 (Super Admin → negocio) | Historia 2.2–2.3; FR-004, FR-005, FR-006 |
| Admin 4 (slug reservado) | Historia 2.5; FR-008, FR-009 |
| 1 (invitación exitosa) | Historia 3.1; FR-011, FR-018–FR-020 |
| 2 (nombre vacío) | Historia 4.1; FR-013–FR-015 |
| 3 (solo espacios) | Historia 4.2; FR-013 |
| 4 (límites de longitud) | Historia 4.3; FR-014, FR-015 |
| 5 (caracteres legítimos) | Historia 3.2; FR-014, FR-025 |
| 6 (anti-XSS) | Historia 4.4; FR-016, FR-017 |
| 7 (validación de backend) | Historia 4.5; FR-016 |
| 8 (usuarios legacy) | Historia 5.1; FR-022 |
| 9 (limpieza de estado) | Historia 5.3; FR-023 |

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un Super Admin llega al panel central desde `admin.skeilopos.com/login` en un solo intento con credenciales correctas, y el 100 % de las rutas internas del panel se pueden recargar sin error de página no encontrada.
- **SC-002**: En las 4 combinaciones de cruce (cuenta de negocio en plataforma, Super Admin en negocio existente, Super Admin en subdominio inexistente, alta de negocio con `admin`, `assets`, `api` o `docs`), el 100 % de los intentos se rechaza, y el mensaje mostrado no permite distinguir si la cuenta existe.
- **SC-003**: El 100 % de los negocios existentes conserva su inicio de sesión y su operación sin cambios tras el despliegue (cero reportes de acceso perdido).
- **SC-004**: Un Administrador de Tenant invita a una persona con nombre, correo y rol en menos de 30 segundos, y el correo recibido la saluda por su nombre completo.
- **SC-005**: El 100 % de los nombres inválidos probados (vacío, solo espacios, 1 carácter, 101 caracteres, con dígitos, con etiquetas HTML) se rechaza tanto en la pantalla como en una petición directa, con el mensaje correcto y sin enviar correo ni guardar datos.
- **SC-006**: El 100 % de los usuarios existentes sin nombre propio sigue apareciendo en la lista, con su correo y la inicial de su correo, sin errores visibles.
- **SC-007**: Ninguna invitación pendiente anterior a esta spec queda inutilizable: el 100 % se puede aceptar como antes.
- **SC-008**: Un nombre válido con caracteres especiales (tildes, ñ, apóstrofe, guion) se muestra idéntico al escrito en la lista, el avatar y el correo, en el 100 % de los casos probados.

## Assumptions

- **Roles invitables y alcance de pantallas**: el campo "Nombre completo" se añade al formulario de invitación del Administrador de Tenant (página "Usuarios") y al formulario de alta de usuarios del dashboard del Super Admin. El formulario del Super Admin hoy llama a `POST/PATCH /super-admin/users`, que **no existen en el servicio** (la spec 037 retiró la creación directa; verificado con HTTP 405). Esta spec solo aplica a ese formulario la validación del nombre y la presentación de nombre e inicial en su tabla; **no** crea endpoints ni restituye la creación directa. Restituirla, retirar el botón o cualquier otra salida se decide en una spec aparte (anomalía A-102; decisión O-1 con la opción A adoptada por defecto, ver `research.md` D11).
- **Edición de cuentas antiguas**: al editar una cuenta anterior sin nombre propio desde el dashboard del Super Admin, el nombre no es obligatorio para guardar otros cambios; si se escribe, debe cumplir las reglas. No se fuerza ningún nombre a cuentas existentes.
- **"Sin nombre" significa**: nombre vacío o igual al correo de la cuenta. Es la forma real en que hoy existen las cuentas sin nombre.
- **Apóstrofes admitidos**: se aceptan el apóstrofe recto (') y el tipográfico (’).
- **Mayúsculas**: la inicial del avatar se muestra en mayúscula; el nombre guardado no se transforma.
- **Mensajes**: los textos de error de la pantalla y del servicio están en español de Colombia, coherentes con el resto de la aplicación.
- **Dominio raíz**: `skeilopos.com` pertenece a otra página del negocio (decisión del negocio, 2026-10-02) y queda fuera de esta aplicación y de esta spec; solo se exige que el servicio de autenticación no acepte inicios de sesión provenientes de él.
- **Desarrollo local**: se conserva un modo de pruebas locales del acceso de plataforma equivalente al actual (sin depender del dominio de producción).
- **Subdominio de plataforma**: es `admin.skeilopos.com`; su nombre `admin` queda reservado a nivel de plataforma sin posibilidad de reasignarlo a un negocio.
- **Datos existentes**: no hay migración forzada de datos; las invitaciones pendientes y las cuentas actuales conservan su estado. Antes de desplegar se verifica que ningún negocio existente use los subdominios `admin`, `assets`, `api` ni `docs`.
- **Fuera de alcance**: cambiar el nombre de una cuenta ya aceptada desde la página "Usuarios" del negocio, permitir que una persona edite su propio nombre, crear Super Admins desde la pantalla, internacionalizar los mensajes, y cualquier cambio a facturas ya emitidas (Principio VII — esta spec no las toca).
- **Impacto sobre funcionalidades existentes**: cambia un comportamiento protegido (el servicio de invitaciones deja de aceptar peticiones sin nombre, y el inicio de sesión deja de aceptar Super Admin desde cualquier subdominio no registrado); por ello los tests de caracterización afectados se actualizarán de forma explícita y las decisiones de negocio se registrarán en el registro de anomalías **antes** de implementar (Principios II, III y XI).
- **Dependencia de despliegue**: requiere que el subdominio `admin.skeilopos.com` esté apuntado al mismo frontend en el hosting; esa configuración es parte del trabajo de despliegue, no de esta spec funcional.
