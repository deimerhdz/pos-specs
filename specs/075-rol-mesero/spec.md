# Feature Specification: Rol Mesero con Acceso Restringido a Terminal de Mesas y Órdenes

**Feature Branch**: `075-rol-mesero`

**Created**: 2026-09-04

**Status**: Draft

**Input**: User description: "actualmente hay varios roles en el sistema super_admin admin y cashier, quiero incorporar un nuevo rol para los meseros, este nuevo rol debera estar disponible desde el panel de admin del tenant y solo tendra acceso para acceder a la terminal de mesas y para consultar las ordenes, la implementacion de este nuevo rol debe incluirse en el backend y frontend"

## Clarifications

### Session 2026-09-04

- Q: Dentro de "terminal de mesas" (pantalla donde hoy se toman los pedidos y también se cobra la cuenta de la mesa), ¿el Mesero debe poder cobrar/cerrar cuentas, o esa acción debe quedar reservada a Cajero/Admin? → A: El Mesero puede cobrar — tiene acceso completo a la Terminal de Mesas tal como existe hoy, incluida la acción de cobro.
- Q: Hoy el servidor no restringe por rol de negocio qué funciones puede invocar un Cajero (solo exige ser Admin para gestionar usuarios); el resto de pantallas solo se ocultan en la interfaz. ¿El bloqueo para Mesero debe ser también real del lado del servidor? → A: Sí — el servidor debe denegar cualquier solicitud fuera del alcance del Mesero (Terminal de Mesas y consulta de Órdenes), no solo ocultarla en la interfaz. Este mecanismo de bloqueo por rol se construye como parte de esta funcionalidad (hoy no existe un equivalente para ningún otro rol restringido).
- Q: ¿Se revisa también el alcance actual de Admin y Cajero al mismo tiempo? → A: No — el acceso actual de Admin y Cajero a todas las pantallas y funciones permanece exactamente igual; esta funcionalidad únicamente agrega el rol Mesero (y el mecanismo de bloqueo por servidor que ese nuevo rol requiere).
- Mapa de acceso por módulo confirmado por el negocio (ver tabla en Requisitos Funcionales): Mesero solo accede a Terminal de Mesas (con cobro incluido) y a Órdenes en modo consulta; todo lo demás (Dashboard, Caja, Ajustes, Ventas, Categorías, Productos, Promociones, configuración de Mesas, Inventario, Proveedores, Usuarios, Reportes) queda fuera de su alcance. "Mi cuenta" y "Mi plan" siguen disponibles para cualquier usuario autenticado, sin cambio.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Un Admin asigna el rol Mesero a un miembro de su equipo (Priority: P1)

Como Admin de un tenant, quiero poder asignar el rol "Mesero" a un miembro de mi equipo (al invitarlo por primera vez o cambiando el rol de un usuario ya existente) desde el mismo panel donde hoy asigno "Admin" o "Cajero", para poder dar acceso al sistema a mis meseros sin darles acceso a funciones de caja, inventario, reportes u otras áreas administrativas.

**Why this priority**: Sin la capacidad de asignar el rol, ninguna otra parte de esta funcionalidad tiene forma de activarse. Es el punto de entrada de todo lo demás.

**Independent Test**: Un Admin abre la pantalla de gestión de usuarios de su tenant, invita a un nuevo usuario (o edita uno existente) seleccionando "Mesero" como rol, y verifica que el usuario queda creado/actualizado con ese rol y aparece etiquetado como "Mesero" en el listado.

**Acceptance Scenarios**:

1. **Given** un Admin está en la pantalla de gestión de usuarios de su tenant, **When** invita a un nuevo usuario y selecciona el rol "Mesero", **Then** el usuario invitado queda registrado con rol Mesero una vez acepta la invitación.
2. **Given** un Admin está viendo un usuario existente con rol Cajero o Admin, **When** cambia su rol a "Mesero", **Then** el cambio se aplica de inmediato, sin que el usuario afectado tenga que cerrar sesión para que tome efecto.
3. **Given** un Admin abre el listado de usuarios de su tenant, **When** revisa un usuario con rol Mesero, **Then** el rol se muestra etiquetado como "Mesero" (no como un código técnico) igual que "Admin" y "Cajero" ya se muestran hoy.

---

### User Story 2 - Un Mesero solo ve y usa Terminal de Mesas y Órdenes (Priority: P1)

Como usuario con rol Mesero, al iniciar sesión quiero ver únicamente la Terminal de Mesas y la consulta de Órdenes en mi navegación, para poder hacer mi trabajo (tomar pedidos, gestionar mesas, cobrar cuentas, consultar el estado de una orden) sin que se me muestren pantallas administrativas que no me corresponden ni que puedan confundirme.

**Why this priority**: Es el valor central que motiva la funcionalidad: un rol de acceso reducido y predecible para el personal de mesero, visible y usable de inmediato tras el login.

**Independent Test**: Con un usuario de rol Mesero, iniciar sesión y verificar que el menú de navegación solo muestra Terminal de Mesas y Órdenes (además de las opciones de cuenta personal disponibles para cualquier usuario), que la Terminal de Mesas funciona igual que para un Cajero (tomar pedidos, gestionar sesiones de mesa, cobrar), y que Órdenes solo permite consultar sin acciones que cambien el estado de una orden.

**Acceptance Scenarios**:

1. **Given** un usuario con rol Mesero inicia sesión, **When** revisa el menú de navegación, **Then** solo aparecen las opciones "Terminal de mesas" y "Órdenes" (además de "Mi cuenta" y "Mi plan", disponibles para cualquier usuario autenticado).
2. **Given** un usuario con rol Mesero está en la Terminal de Mesas, **When** toma un pedido, gestiona una sesión de mesa o cobra la cuenta de una mesa, **Then** todas esas acciones funcionan exactamente igual que hoy funcionan para un usuario con rol Cajero.
3. **Given** un usuario con rol Mesero abre la pantalla de Órdenes, **When** consulta el listado o el detalle de una orden, **Then** puede ver la información pero no se le muestran acciones que modifiquen el estado de esa orden.
4. **Given** un usuario con rol Mesero intenta entrar por URL directa (enlace guardado, marcador) a una pantalla fuera de su alcance (por ejemplo Inventario, Reportes, Usuarios, Caja, Ajustes), **When** la solicitud se procesa, **Then** el sistema lo redirige a la Terminal de Mesas en lugar de mostrar esa pantalla o un error técnico.

---

### User Story 3 - El bloqueo de acceso del Mesero también se aplica del lado del servidor (Priority: P1)

Como negocio, quiero que la restricción de acceso del rol Mesero no dependa únicamente de que la interfaz oculte botones y pantallas, para que un usuario con ese rol no pueda alcanzar datos o funciones fuera de su alcance (inventario, reportes, cifras de ventas, gestión de usuarios, etc.) por ninguna otra vía distinta a la interfaz estándar.

**Why this priority**: Ocultar una opción en pantalla no es una restricción real si la función subyacente sigue respondiendo a cualquiera que la solicite; sin este bloqueo, el rol Mesero no cumple la promesa de "solo tendrá acceso a...".

**Independent Test**: Con credenciales de un usuario Mesero, intentar solicitar directamente (sin pasar por la interfaz estándar) una función fuera de su alcance —por ejemplo, datos de inventario, reportes o gestión de usuarios— y verificar que la solicitud es rechazada, mientras que una solicitud sobre Terminal de Mesas o consulta de Órdenes sí se procesa con normalidad.

**Acceptance Scenarios**:

1. **Given** un usuario con rol Mesero, **When** solicita directamente una función fuera de su alcance (inventario, reportes, caja, ventas, productos, categorías, promociones, configuración de mesas, proveedores, usuarios, ajustes), **Then** el sistema rechaza la solicitud, sin importar si se originó desde la interfaz estándar o no.
2. **Given** un usuario con rol Mesero, **When** solicita una función dentro de Terminal de Mesas o de consulta de Órdenes, **Then** el sistema la procesa con normalidad.
3. **Given** un usuario con rol Admin o Cajero, **When** solicita cualquier función que hoy ya puede usar, **Then** el comportamiento es exactamente el mismo que antes de esta funcionalidad — este bloqueo nuevo no le agrega ninguna restricción adicional a esos dos roles.

---

### Edge Cases

- Un usuario que tenía rol Cajero es cambiado a Mesero: pierde de inmediato el acceso a Caja y a Ventas (que sí tenía como Cajero) y conserva el acceso a Terminal de Mesas y Órdenes que ya tenía.
- Un tenant no tiene ningún usuario con rol Mesero: el sistema funciona exactamente igual que hoy; el rol es opcional, no obligatorio.
- Un usuario con rol Mesero tiene una sesión abierta en el momento en que un Admin le cambia el rol (hacia o desde Mesero): la restricción o ampliación de acceso aplica de inmediato, sin requerir que cierre sesión.
- El rol Super Admin (panel de plataforma, distinto al panel de administración del tenant) no se ve afectado por esta funcionalidad.
- Un usuario con rol Mesero intenta acceder a la sub-pantalla de "orden manual" dentro de Terminal de Mesas: se le permite, porque es parte de la misma Terminal de Mesas a la que ya tiene acceso completo.

## Requirements *(mandatory)*

### Mapa de acceso por módulo

| Módulo | Admin | Cajero (sin cambio) | Mesero (nuevo) |
|---|---|---|---|
| Dashboard | Sí | No | No |
| Caja | Sí | Sí | No |
| Ajustes (información del negocio, métodos de pago, unidades de medida, grupos de opciones) | Sí | No | No |
| Ventas / punto de venta | Sí | Sí | No |
| Categorías | Sí | No | No |
| Productos | Sí | No | No |
| Promociones | Sí | No | No |
| Mesas — configuración y código QR | Sí | No | No |
| **Terminal de mesas** (tomar pedidos, gestionar sesiones de mesa, cobrar) | Sí | Sí | **Sí** |
| **Órdenes** (listado y detalle) | Sí (lectura y acciones) | Sí (lectura y acciones) | **Sí, solo consulta** |
| Inventario | Sí | No | No |
| Proveedores | Sí | No | No |
| Usuarios | Sí | No | No |
| Reportes | Sí | No | No |
| Mi cuenta / Mi plan | Sí | Sí | Sí |

### Functional Requirements

- **FR-001**: El sistema MUST permitir que un Admin de un tenant asigne el rol "Mesero" a un usuario de su mismo tenant, tanto al invitar a un nuevo usuario como al cambiar el rol de uno ya existente, usando el mismo mecanismo de gestión de usuarios ya disponible hoy para asignar "Admin" y "Cajero".
- **FR-002**: El sistema MUST mostrar "Mesero" como una opción de rol seleccionable en la interfaz de gestión de usuarios del tenant (formulario de invitación y edición de rol de un usuario existente), etiquetada en español igual que "Admin" y "Cajero".
- **FR-003**: El sistema MUST restringir la navegación disponible para un usuario con rol Mesero a únicamente Terminal de Mesas y Órdenes, además de las pantallas de cuenta personal (Mi cuenta, Mi plan) ya disponibles para cualquier usuario autenticado del tenant.
- **FR-004**: El sistema MUST permitir a un usuario con rol Mesero usar la Terminal de Mesas con exactamente la misma funcionalidad que hoy tiene disponible un usuario con rol Cajero, incluyendo tomar pedidos, gestionar sesiones de mesa y cobrar/cerrar la cuenta de una mesa.
- **FR-005**: El sistema MUST permitir a un usuario con rol Mesero consultar el listado y el detalle de Órdenes, sin exponerle en esa pantalla ninguna acción que modifique el estado de una orden.
- **FR-006**: El sistema MUST impedir, a nivel de interfaz, que un usuario con rol Mesero llegue a cualquier pantalla fuera de su alcance (Dashboard, Caja, Ajustes, Ventas, Categorías, Productos, Promociones, configuración de Mesas, Inventario, Proveedores, Usuarios, Reportes) — incluyendo el intento de entrar por URL directa, enlace guardado o marcador — redirigiéndolo en su lugar a la Terminal de Mesas.
- **FR-007**: El sistema MUST denegar, también del lado del servidor y no solo en la interfaz, cualquier solicitud que un usuario con rol Mesero haga sobre una función o dato fuera del alcance definido en FR-004/FR-005, sin importar por qué vía llegue esa solicitud.
- **FR-008**: El sistema MUST mantener sin ningún cambio el alcance de acceso actual de los roles Admin y Cajero a todas las pantallas y funciones existentes — esta funcionalidad únicamente agrega el rol Mesero y el mecanismo de bloqueo por servidor que ese nuevo rol requiere.
- **FR-009**: El sistema MUST aplicar de inmediato cualquier cambio de rol hacia o desde Mesero, sin requerir que el usuario afectado cierre sesión para que el nuevo alcance de acceso tenga efecto.
- **FR-010**: El sistema MUST permitir que un tenant tenga cero, uno o varios usuarios con rol Mesero, sin ningún límite adicional al que ya aplique, si existe, al total de usuarios del tenant.
- **FR-011**: El sistema MUST seguir excluyendo el rol Super Admin de las opciones asignables desde el panel de administración del tenant, igual que hoy (el rol Mesero se suma a las opciones ya asignables: Admin y Cajero).

### Key Entities *(include if feature involves data)*

- **Rol**: catálogo de roles asignables a un usuario del tenant. Se agrega "Mesero" como un tercer valor asignable desde el panel del tenant, junto a "Admin" y "Cajero" (Super Admin sigue sin ser asignable desde ese panel).
- **Usuario**: miembro del equipo de un tenant; su rol determina qué pantallas y funciones puede usar. Un usuario puede tener el rol Mesero asignado igual que hoy puede tener Admin o Cajero.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los usuarios con rol Mesero, al iniciar sesión, solo pueden ver y usar Terminal de Mesas y Órdenes (en modo consulta) en la interfaz, sin ninguna excepción.
- **SC-002**: El 100% de los intentos de un usuario con rol Mesero de alcanzar una función fuera de su alcance —ya sea por la interfaz o por cualquier otra vía— son rechazados por el sistema.
- **SC-003**: Un Admin puede asignar el rol Mesero a un miembro de su equipo (invitación o cambio de rol) en menos de 1 minuto, sin pasos adicionales a los que ya usa hoy para asignar Admin o Cajero.
- **SC-004**: El comportamiento de los roles Admin y Cajero permanece exactamente igual al actual después de esta funcionalidad, sin ninguna restricción ni ampliación no solicitada.

## Assumptions

- El alcance de "Terminal de Mesas" para el rol Mesero es idéntico al que hoy tiene el rol Cajero, incluyendo la acción de cobrar/cerrar la cuenta de una mesa dentro de esa misma pantalla (confirmado explícitamente por el negocio).
- "Consultar las órdenes" significa acceso de solo lectura al listado y al detalle de Órdenes, sin acciones que cambien el estado de una orden desde esa pantalla.
- El acceso actual de los roles Admin y Cajero a todas las pantallas y funciones no se modifica con esta funcionalidad; solo se agrega el rol Mesero y el mecanismo de bloqueo por servidor que ese nuevo rol requiere (confirmado por el negocio).
- La pantalla de destino por defecto para un usuario Mesero —tanto al iniciar sesión como al ser redirigido tras intentar entrar a algo fuera de su alcance— es la Terminal de Mesas, siguiendo el mismo criterio que hoy usa el rol Cajero con la pantalla de Caja.
- El rol Super Admin, al pertenecer a un panel de plataforma distinto del panel de administración del tenant, queda fuera del alcance de esta funcionalidad.
- El mecanismo que hoy deniega el acceso a un módulo cuando el plan del tenant no lo incluye (por ejemplo Inventario o Promociones) es independiente de esta funcionalidad: un Mesero no tiene acceso a esos módulos por su rol, sin importar si el plan del tenant los incluye o no.
