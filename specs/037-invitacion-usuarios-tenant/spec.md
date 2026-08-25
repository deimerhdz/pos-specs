# Feature Specification: Alta de usuarios internos por invitación

**Feature Branch**: `037-invitacion-usuarios-tenant`

**Created**: 2026-08-25

**Status**: Draft

**Input**: User description: "Hoy, un usuario ADMIN de un tenant crea usuarios internos desde el
dashboard escribiendo a mano el correo y la contraseña... Objetivo: que dar de alta a un usuario
interno se haga exclusivamente por invitación al correo, y que la persona invitada recorra el
mismo primer ingreso que ya recorre el usuario inicial de un tenant." (especificación completa
provista por el usuario, ver historial de la conversación).

## Clarifications

### Session 2026-08-25

- Q: ¿Qué debe pasar si un ADMIN invita un correo que ya perteneció a un usuario del tenant que
  fue desactivado (dado de baja) previamente, en vez de a un usuario activo? → A: Se bloquea igual
  que un correo de usuario activo, con el mismo error explícito, sin importar si la cuenta
  existente está activa o inactiva. Reactivar el acceso de un ex-usuario sigue el mecanismo ya
  existente de activar/desactivar cuentas (fuera del alcance de este spec), no pasa por una nueva
  invitación.
- Q: ¿Las acciones sobre invitaciones (crear, reenviar, cancelar) deben quedar registradas en el
  log de auditoría del sistema, indicando qué ADMIN hizo la acción y cuándo? → A: No es necesario
  para esta versión; queda fuera de alcance.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El ADMIN invita a alguien por correo y rol (Priority: P1)

Un ADMIN del tenant abre el listado de usuarios, pulsa "Agregar usuario", escribe únicamente el
correo de la persona y elige su rol (ADMIN o CASHIER). No existe ningún campo de contraseña que
llenar ni que inventar.

**Why this priority**: es el punto de entrada de todo el feature — sin poder crear la invitación
no hay nada más que probar, y es el cambio que elimina el problema de negocio central (contraseñas
conocidas por un tercero).

**Independent Test**: se puede probar completamente abriendo el formulario "Agregar usuario",
verificando que no contiene campo de contraseña, y enviándolo con un correo y rol válidos para un
tenant sin invitación previa para ese correo.

**Acceptance Scenarios**:

1. **Given** un ADMIN autenticado en su tenant, **When** abre el formulario "Agregar usuario",
   **Then** ve exactamente dos controles — correo electrónico y selector de rol (ADMIN/CASHIER) —
   y ningún campo de contraseña.
2. **Given** un correo que no tiene invitación pendiente ni cuenta activa en ese tenant, **When**
   el ADMIN envía el formulario, **Then** se crea una invitación pendiente para ese correo y rol,
   no se crea ninguna cuenta de usuario, y no existe ninguna forma de que el ADMIN vea o conozca
   una contraseña para esa invitación.
3. **Given** una invitación creada, **When** se revisa cualquier respuesta de la API o los
   registros (logs) del sistema asociados a esa creación, **Then** en ningún lugar aparece la
   contraseña temporal generada.
4. **Given** un usuario con rol CASHIER autenticado, **When** visita el listado de usuarios del
   tenant, **Then** no ve el botón "Agregar usuario".
5. **Given** un correo que ya corresponde a un usuario activo del tenant, **When** el ADMIN
   intenta invitarlo, **Then** el sistema muestra un error explícito y no envía ningún correo.
6. **Given** un tenant marcado como inactivo o suspendido, **When** su ADMIN intenta enviar una
   invitación, **Then** el sistema rechaza el envío con un error explícito y no se envía ningún
   correo.

---

### User Story 2 - La persona invitada activa su cuenta en su primer ingreso (Priority: P1)

La persona invitada recibe un correo con el enlace de acceso del negocio, su nombre de usuario (su
propio correo) y una contraseña temporal. Entra por primera vez con esos datos y el sistema la
obliga a fijar su propia contraseña antes de dejarla continuar.

**Why this priority**: sin este paso la invitación no sirve de nada — es la otra mitad
imprescindible del flujo, y es la que garantiza que la contraseña final solo la conoce la propia
persona.

**Independent Test**: se puede probar completamente creando una invitación, tomando el correo y la
contraseña temporal recibidos, autenticándose con ellos, y verificando que el sistema exige el
cambio de contraseña antes de permitir cualquier otra acción.

**Acceptance Scenarios**:

1. **Given** una invitación pendiente con su contraseña temporal vigente, **When** la persona
   invitada se autentica por primera vez con su correo y esa contraseña, **Then** el sistema crea
   la cuenta de usuario con el rol indicado en la invitación, activa la marca de cambio
   obligatorio de contraseña, y marca la invitación como consumida.
2. **Given** una cuenta recién creada por consumo de invitación, **When** la persona intenta usar
   cualquier función del sistema sin cambiar antes su contraseña, **Then** el sistema se lo impide
   y la obliga a cambiarla primero (comportamiento ya existente, sin modificar).
3. **Given** que la persona cambia su contraseña exitosamente, **When** se completa esa operación,
   **Then** la marca de cambio obligatorio se desactiva (comportamiento ya existente, sin
   modificar) y, al revisar el listado de usuarios del tenant, ese correo aparece como usuario
   activo y ya no como invitación pendiente.
4. **Given** una invitación ya cancelada o reenviada (con contraseña temporal distinta a la que
   tiene la persona), **When** intenta autenticarse con la contraseña temporal que recibió
   originalmente, **Then** el sistema rechaza el intento.

---

### User Story 3 - El ADMIN distingue pendientes de activos en el listado (Priority: P2)

El ADMIN abre el listado de usuarios del tenant y ve, además de las cuentas activas, las
invitaciones que todavía no se han usado, con su correo, su rol y su fecha de envío, marcadas de
forma clara para no confundirlas con una cuenta real.

**Why this priority**: sin visibilidad el ADMIN no puede saber a quién perseguir para que active
su cuenta, ni verificar que una invitación se consumió correctamente.

**Independent Test**: se puede probar creando invitaciones y consumiendo una de ellas, y
verificando en el listado que las pendientes y las activas se muestran diferenciadas con los datos
correctos.

**Acceptance Scenarios**:

1. **Given** un tenant con usuarios activos e invitaciones pendientes, **When** el ADMIN abre el
   listado de usuarios, **Then** ve ambos grupos claramente diferenciados, y cada invitación
   pendiente muestra su correo, su rol y su fecha de envío.
2. **Given** una invitación pendiente que se consume, **When** el ADMIN refresca el listado,
   **Then** ese correo ya no aparece como pendiente y sí aparece como usuario activo.

---

### User Story 4 - El ADMIN reenvía o cancela una invitación pendiente (Priority: P2)

El ADMIN necesita corregir un correo mal escrito (cancelando esa invitación e invitando de nuevo
con el correo correcto) o reenviar una invitación cuya contraseña temporal se perdió, así como
cortar el acceso de una invitación que ya no debería usarse.

**Why this priority**: sin esta capacidad, un error de tipeo o una invitación que ya no aplica
queda sin remedio salvo esperar o intervenir manualmente en el sistema.

**Independent Test**: se puede probar creando una invitación, reenviándola y verificando que la
contraseña anterior deja de servir y la nueva sí funciona; y por separado, creando otra invitación,
cancelándola y verificando que su contraseña temporal deja de servir.

**Acceptance Scenarios**:

1. **Given** una invitación pendiente, **When** el ADMIN la reenvía, **Then** el sistema genera
   una contraseña temporal nueva, invalida de inmediato la anterior, actualiza la fecha de envío
   mostrada en el listado, y envía un nuevo correo con los mismos tres datos (enlace, usuario,
   contraseña temporal).
2. **Given** la contraseña temporal original de una invitación ya reenviada, **When** alguien
   intenta autenticarse con ella, **Then** el sistema la rechaza; solo la contraseña más reciente
   funciona.
3. **Given** una invitación pendiente, **When** el ADMIN la cancela, **Then** su contraseña
   temporal deja de servir de inmediato y la invitación deja de aparecer como pendiente.

---

### Edge Cases

- El mismo correo puede tener invitaciones o cuentas independientes en dos tenants distintos al
  mismo tiempo, cada una con su propia contraseña temporal vigente — no hay colisión entre
  tenants.
- El correo escrito con mayúsculas o espacios adicionales se normaliza (recorte de espacios y
  comparación insensible a mayúsculas/minúsculas) antes de comparar unicidad o de intentar el
  login.
- Si dos ADMIN intentan invitar el mismo correo casi al mismo tiempo, solo una invitación queda
  creada; el segundo intento recibe el error de "ya existe una invitación pendiente para ese
  correo".
- Si se cancela una invitación mientras la persona está en medio de su primer ingreso, la
  cancelación gana: cualquier intento de autenticación posterior con esa contraseña temporal falla.
- Un correo con formato válido pero dominio inexistente puede aceptar el envío en el momento (el
  sistema no verifica rebotes asíncronos); solo se considera "fallo de envío" (y por tanto
  invitación no utilizable) un error inmediato al despachar el mensaje.
- Invitar el propio correo del ADMIN que invita se comporta igual que invitar cualquier correo que
  ya es usuario activo del tenant: se rechaza con error explícito, porque el ADMIN ya tiene cuenta
  en ese tenant.
- Invitar mientras el tenant está inactivo o suspendido se rechaza explícitamente, sin enviar
  ningún correo.
- Invitar un correo que ya corresponde a un usuario del tenant que fue desactivado (dado de baja)
  se rechaza con el mismo error explícito que un usuario activo; el correo de una cuenta
  desactivada sigue reservado dentro de ese tenant y no queda libre para una nueva invitación.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El dashboard del tenant DEBE reemplazar el formulario de alta de usuario por un
  botón "Agregar usuario" que abre un formulario con exactamente dos controles — correo
  electrónico y selector de rol (ADMIN/CASHIER) — sin ningún campo de contraseña.
- **FR-002**: Solo un usuario con rol ADMIN del tenant DEBE poder ver el botón "Agregar usuario" y
  enviar invitaciones; un usuario con rol CASHIER no debe ver ese control.
- **FR-003**: Al enviar el formulario, el sistema DEBE crear una invitación pendiente asociada al
  tenant, el correo y el rol indicados, sin crear ninguna cuenta de usuario en ese momento.
- **FR-004**: El sistema DEBE eliminar por completo — en la interfaz y en cualquier endpoint del
  backend — el mecanismo que permite a un ADMIN crear un usuario proporcionando directamente su
  contraseña; toda creación de usuario interno ocurre exclusivamente a través del flujo de
  invitación descrito en esta especificación.
- **FR-005**: Al crear (o reenviar) una invitación, el sistema DEBE enviar al correo indicado un
  mensaje con el enlace de acceso del tenant, el nombre de usuario (el propio correo) y una
  contraseña temporal generada con el mismo mecanismo criptográficamente seguro que usan hoy las
  contraseñas temporales del sistema (12 caracteres aleatorios de letras mayúsculas, minúsculas,
  dígitos y los símbolos `!@#$%*?`).
- **FR-006**: La contraseña temporal generada NUNCA debe mostrarse en pantalla, no debe incluirse
  en ninguna respuesta de la API, y no debe registrarse en ningún log del sistema; quien invita no
  tiene ningún medio dentro del producto para conocerla.
- **FR-007**: Cuando la persona invitada se autentica por primera vez con su correo y la
  contraseña temporal vigente, el sistema DEBE crear la cuenta de usuario con el rol indicado en
  la invitación, activar la marca de cambio obligatorio de contraseña y marcar la invitación como
  consumida, como una sola operación.
- **FR-008**: A partir del primer ingreso con contraseña temporal, el flujo de cambio obligatorio
  de contraseña y la desactivación de esa marca al completarlo son los que el sistema ya
  implementa hoy; esta especificación no los modifica.
- **FR-009**: El listado de usuarios del tenant DEBE mostrar, junto a los usuarios activos, las
  invitaciones pendientes con su correo, su rol y su fecha de envío, visualmente diferenciadas de
  una cuenta activa.
- **FR-010**: El ADMIN DEBE poder reenviar una invitación pendiente; el reenvío genera una nueva
  contraseña temporal, invalida de inmediato la anterior, actualiza la fecha de envío mostrada en
  el listado, y dispara un nuevo correo con los mismos datos de FR-005.
- **FR-011**: El ADMIN DEBE poder cancelar una invitación pendiente; al cancelarla, su contraseña
  temporal deja de servir de inmediato y la invitación deja de aparecer como pendiente.
- **FR-012**: Si el envío del correo de invitación (inicial o de reenvío) falla, la invitación no
  debe quedar en un estado utilizable (ninguna contraseña temporal vigente asociada) y el ADMIN
  debe ver un mensaje de error explícito; el sistema nunca debe confirmar un envío exitoso cuando
  el correo no salió.
- **FR-013**: Ni las invitaciones pendientes ni los usuarios DEBEN poder crearse, listarse,
  reenviarse o cancelarse fuera del tenant de quien realiza la acción.
- **FR-014**: El correo DEBE ser único por invitación/usuario dentro de un tenant, no
  globalmente; la misma persona puede tener invitaciones o cuentas independientes en tenants
  distintos, cada una con su propio ciclo de vida y su propia contraseña temporal.
- **FR-015**: El sistema DEBE rechazar la creación de una invitación si ya existe una invitación
  pendiente para el mismo correo en el mismo tenant, o si ese correo ya corresponde a un usuario de
  ese tenant — activo o inactivo/desactivado —; en todos los casos se muestra un error explícito y
  no se envía ningún correo. Reactivar el acceso de una cuenta inactiva no pasa por una invitación;
  usa el mecanismo de activación/desactivación de usuarios ya existente en el sistema.
- **FR-016**: El sistema DEBE normalizar el correo (recorte de espacios y comparación insensible a
  mayúsculas/minúsculas) al crear, buscar y comparar invitaciones y usuarios.
- **FR-017**: La contraseña temporal de una invitación no caduca por tiempo; deja de servir
  únicamente cuando la invitación se consume (FR-007), se reenvía (FR-010) o se cancela (FR-011).
- **FR-018**: El sistema DEBE rechazar el envío de una invitación cuando el tenant del ADMIN que
  invita está inactivo o suspendido, mostrando un error explícito y sin enviar ningún correo.
- **FR-019**: El sistema NO impone un límite máximo de invitaciones enviadas por tenant por unidad
  de tiempo en esta versión; la única protección contra invitaciones repetidas es la regla de
  unicidad de FR-015.

### Key Entities *(include if feature involves data)*

- **Invitación pendiente**: correo (normalizado), rol asignado (ADMIN/CASHIER), tenant al que
  pertenece, fecha de envío, estado (pendiente / consumida / cancelada) y la contraseña temporal
  vigente asociada (nunca expuesta fuera del correo enviado). Al consumirse da origen a un
  **Usuario**; al cancelarse o reenviarse, la contraseña temporal previa deja de ser válida.
- **Usuario (personal)**: cuenta de ADMIN o CASHIER del tenant — ya existente en el sistema.
  Relevante aquí: se origina a partir de una invitación consumida, hereda su rol de esa
  invitación, y nace con la marca de cambio obligatorio de contraseña activada.
- **Tenant**: negocio dueño de sus propias invitaciones y usuarios; su estado (activo / inactivo o
  suspendido) condiciona si puede enviar nuevas invitaciones.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un ADMIN puede invitar a una persona completando únicamente correo y rol, sin
  inventar ni comunicar ninguna contraseña.
- **SC-002**: El 100% de las cuentas de usuario interno nuevas se originan a partir de una
  invitación consumida; cero cuentas nuevas se crean con una contraseña elegida por quien las da
  de alta.
- **SC-003**: Una persona invitada completa su primer ingreso (autenticarse con la contraseña
  temporal y fijar su propia contraseña) en un único intento, sin pasos de soporte adicionales.
- **SC-004**: El ADMIN distingue, de un vistazo en el listado de usuarios, el 100% de las
  invitaciones pendientes frente a las cuentas activas.
- **SC-005**: Tras reenviar o cancelar una invitación, la contraseña temporal anterior deja de
  funcionar desde el siguiente intento de inicio de sesión, sin ninguna ventana de uso posterior.
- **SC-006**: Ningún intento de invitar un correo ya usado (como usuario activo o invitación
  pendiente) de ese mismo tenant resulta en el envío de un correo.

## Assumptions

- Se elimina por completo (interfaz y backend) el mecanismo que permitía a un ADMIN crear
  usuarios con una contraseña elegida por él; no queda ninguna vía alterna, ni siquiera interna o
  de soporte, para ese flujo (decisión confirmada con el usuario).
- Invitar mientras el tenant está inactivo o suspendido se rechaza explícitamente, sin distinguir
  entre "inactivo" y "suspendido" a estos efectos (decisión confirmada con el usuario).
- Esta versión no impone un límite de tasa de invitaciones enviadas por tenant por hora; la única
  salvaguarda contra invitaciones repetidas es la regla de unicidad por correo+tenant (decisión
  confirmada con el usuario, que revierte el límite originalmente propuesto en el pedido inicial).
- El sistema no verifica activamente rebotes de correo (dominio inexistente); "envío fallido" se
  refiere solo a errores síncronos al despachar el mensaje, no a rebotes reportados después de
  forma asíncrona por el proveedor de correo.
- Corregir un correo mal escrito en una invitación pendiente se hace cancelándola y creando una
  nueva con el correo correcto; "reenviar" siempre reenvía al mismo correo ya registrado en la
  invitación, no permite editarlo.
- El enlace de acceso enviado en el correo es el mismo enlace de inicio de sesión del tenant que ya
  usa el alta del primer usuario — no un enlace de activación de un solo uso con token propio.
- Los roles admitidos en la invitación son exactamente ADMIN y CASHIER, los mismos roles ya
  existentes del sistema de personal.
- Esta versión no exige registrar en el log de auditoría del sistema las acciones de crear,
  reenviar o cancelar una invitación (decisión confirmada con el usuario); puede añadirse después
  si el negocio lo pide explícitamente.
