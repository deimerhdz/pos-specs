# Feature Specification: Planes de Suscripción por Tenant (Límites y Accesos a Módulos)

**Feature Branch**: `033-planes-suscripcion-tenant`

**Created**: 2026-08-24

**Status**: Draft

**Input**: User description: "Cada tenant necesita estar asociado a un plan que determina qué módulos del sistema puede usar y con qué límites. En esta fase NO se incluye pasarela de pago para cobrar la suscripción: la asignación y gestión del plan es administrativa, a cargo del Super Admin. El Super Admin define planes como un conjunto de características —límites numéricos (con la opción de 'ilimitado') y accesos on/off a módulos— y asocia cada tenant a un único plan vigente. El sistema valida en tiempo real los límites del plan al crear recursos (mesas, cajas, usuarios, productos, métodos de pago activos) y el acceso a módulos (inventario, compras, promociones), bloqueando la acción con un mensaje explicativo cuando corresponda."

> **Nota de alcance ampliado (2026-08-24)**: el Input original excluía duración, precio y
> renovación. Tras la sesión de ampliación registrada en Clarifications más abajo, esta spec sí
> los incluye — como datos administrativos y una fecha de vencimiento que bloquea
> automáticamente, nunca como cobro real ni pasarela de pago (ver Historia de Usuario 5 y
> Supuestos).

## Clarifications

### Session 2026-08-24

- Q: ¿Cómo se determina el plan inicial de un tenant nuevo, para que nunca quede sin plan
  vigente (RF-3)? → A: Elección obligatoria al crear el tenant — quien crea el tenant (el
  Super Admin) debe elegir explícitamente un plan como parte del propio formulario de alta;
  no existe un plan por defecto implícito, y la creación del tenant no se completa sin elegir
  uno.
- Q: Cuando el Super Admin baja a un tenant a un plan con un límite numérico menor a lo que ya
  tiene en uso (ej. 8 mesas, nuevo plan permite 5), ¿qué pasa con los recursos existentes que
  superan el nuevo límite? → A: Se conservan sin cambios; el tenant solo queda bloqueado para
  crear recursos nuevos de ese tipo hasta bajar del límite por su propia acción.
- Q: Cuando se le retira a un tenant el acceso a un módulo (ej. inventario) que ya tenía datos
  cargados, ¿qué pasa con esos datos existentes? → A: El módulo queda completamente
  inaccesible — desaparece de la navegación y no se puede ni siquiera consultar; los datos
  siguen existiendo en la base, pero el tenant no tiene forma de verlos hasta que se le
  reasigne el acceso.
- Q: Para los tipos de recurso que en el sistema actual pueden marcarse como activo/inactivo
  (usuarios, cajas, productos), ¿el límite del plan cuenta solo los registros activos, o
  todos los registros existentes sin importar su estado? → A: Todos los registros existentes
  cuentan, estén activos o inactivos; desactivar un recurso no libera cupo del límite. Esto es
  distinto de la regla ya establecida para métodos de pago activos, que sí se cuentan solo
  sobre los actualmente activos.
- Q: Cuando dos solicitudes de creación del mismo tipo de recurso llegan casi al mismo tiempo
  y ambas verían el límite como no alcanzado, ¿el sistema debe garantizar de forma estricta que
  nunca se supere el límite, o es aceptable que en casos raros de carrera se cree un recurso de
  más? → A: Garantía estricta — el límite nunca se supera, ni siquiera bajo solicitudes
  simultáneas; como máximo una de las solicitudes concurrentes tiene éxito.
- Q: ¿Un plan debe definir obligatoriamente las 8 características (límites de mesas, cajas,
  usuarios, productos, métodos de pago, y accesos a inventario, compras, promociones) al
  crearse, o puede quedar incompleto y dejar alguna sin configurar? → A: Pueden omitirse; toda
  característica no configurada explícitamente en un plan se trata por defecto como bloqueada
  — límite numérico en 0 (no se puede crear ninguno de ese recurso) o acceso de módulo
  denegado, según corresponda.

### Session 2026-08-24 (ampliación — precio, duración y renovación)

- Q: ¿Qué debe pasar cuando el período de un plan (mensual/anual) de un tenant termina sin que
  el Super Admin lo renueve? → A: El sistema bloquea automáticamente al tenant desde ese
  momento — mismo mecanismo y mensajes ya definidos para límites (FR-006) y módulos (FR-009),
  sin borrar ningún dato existente.
- Q: ¿Cómo se relacionan el precio y la duración con el Plan? → A: Un plan define dos precios
  independientes (mensual y anual); al asignar o cambiar el plan de un tenant, el Super Admin
  elige además el ciclo de facturación (mensual o anual) para esa asignación específica —no es
  un atributo fijo del plan, sino de la asignación.
- Q: ¿Esto amplía la spec 033 ya existente o requiere una spec nueva? → A: Se amplía la spec 033
  en el mismo lugar — nada de su implementación había arrancado todavía, así que no hay
  artefactos huérfanos que dejar atrás.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El Super Admin define planes con límites y accesos (Priority: P1)

Como Super Admin, quiero crear y editar planes indicando, para cada uno, un conjunto de
características —límites numéricos (con la opción de "ilimitado") y accesos on/off a
módulos—, para tener un catálogo de planes que luego pueda ofrecer a los tenants.

**Why this priority**: Es la base de toda la funcionalidad. Sin planes definidos no hay nada
que asignar a un tenant ni nada contra qué validar límites o accesos.

**Independent Test**: Puede probarse por completo sin que ningún tenant participe: el Super
Admin crea un plan "Básico" con nombre, descripción, un límite numérico (ej. 5 mesas) y un
acceso de módulo (ej. inventario = no), lo ve listado, y puede editarlo. Todo esto es
verificable en una pantalla de administración de plataforma, sin tocar código.

**Acceptance Scenarios**:

1. **Given** el Super Admin está en la administración de planes, **When** crea un plan nuevo
   indicando nombre, descripción y sus características (límites numéricos y accesos de
   módulo), **Then** el plan queda registrado y disponible para asignarse a cualquier tenant.
2. **Given** un plan existe, **When** el Super Admin edita el valor de una de sus
   características (ej. sube el límite de mesas de 5 a 8), **Then** el cambio se refleja de
   inmediato para todos los tenants que tengan ese plan asignado.
3. **Given** un plan existe, **When** el Super Admin marca una característica numérica como
   "ilimitado", **Then** ningún tenant con ese plan vuelve a ser bloqueado por esa
   característica.
4. **Given** el Super Admin está creando o editando un plan, **When** define un precio mensual,
   un precio anual, o ambos, **Then** el plan queda registrado con esos precios, cada uno
   independiente del otro (Clarifications, sesión de ampliación).

---

### User Story 2 - El Super Admin asigna y cambia el plan de un tenant (Priority: P2)

Como Super Admin, quiero asignar el plan vigente de un tenant y cambiarlo en cualquier
momento, para que cada negocio opere bajo el plan que le corresponde.

**Why this priority**: Sin esta historia, el catálogo de planes de la Historia 1 no tiene
ningún efecto sobre ningún tenant real — es el punto donde un plan empieza a regir de verdad.

**Independent Test**: Al crear un tenant de prueba, el formulario de alta exige elegir un plan
entre los ya creados (ej. Básico y Pro) antes de completarse — no hay forma de crear un tenant
sin plan. Luego el Super Admin lo cambia a "Pro", confirmando que el cambio de plan vigente se
refleja de inmediato.

**Acceptance Scenarios**:

1. **Given** el Super Admin está creando un tenant nuevo, **When** intenta completar la
   creación sin haber elegido un plan, **Then** el sistema no permite completar la creación
   hasta que elija uno de los planes existentes (Clarifications).
2. **Given** un tenant tiene un plan vigente, **When** el Super Admin le asigna un plan
   distinto, **Then** el nuevo plan pasa a regir de inmediato y el anterior deja de aplicar.
3. **Given** un tenant es cambiado a un plan con un límite numérico menor al que ya tiene en
   uso (ej. 8 mesas, nuevo plan permite 5), **When** se confirma el cambio de plan, **Then**
   los recursos existentes que superan el nuevo límite se conservan sin cambios, y el tenant
   solo queda bloqueado para crear recursos nuevos de ese tipo hasta bajar del límite
   (Clarifications).
4. **Given** un tenant es cambiado a un plan que ya no incluye un módulo que tenía activo con
   datos cargados, **When** se confirma el cambio de plan, **Then** ese módulo queda
   completamente inaccesible para el tenant —desaparece de la navegación y no se puede
   consultar— hasta que se le reasigne el acceso; los datos no se borran (Clarifications).
5. **Given** el Super Admin está asignando o cambiando el plan de un tenant, **When** elige el
   plan, **Then** el sistema exige además elegir un ciclo de facturación (mensual o anual) para
   esa asignación, y calcula la fecha de vencimiento a partir de ese momento y ese ciclo
   (Clarifications, sesión de ampliación).
6. **Given** el Super Admin intenta elegir un ciclo de facturación (ej. anual) para el que el
   plan elegido no tiene precio definido, **When** intenta confirmar la asignación, **Then** el
   sistema rechaza esa combinación hasta que elija un ciclo con precio definido en ese plan
   (Clarifications, sesión de ampliación).
7. **Given** un tenant tiene una asignación de plan vigente (vencida o no), **When** el Super
   Admin la renueva (mismo plan u otro, con su ciclo de facturación), **Then** el período se
   reinicia desde ese momento y cualquier bloqueo por vencimiento se levanta de inmediato
   (Clarifications, sesión de ampliación).

---

### User Story 3 - El sistema bloquea la creación de recursos que exceden el límite del plan (Priority: P3)

Como Tenant Admin, cuando intento crear un recurso gobernado por un límite numérico de mi
plan (mesa, caja, usuario, producto, método de pago activo) y ya alcancé ese límite, quiero
que el sistema bloquee la creación y me explique el motivo, para entender por qué no puedo
continuar y qué debo hacer.

**Why this priority**: Es el mecanismo que realmente protege el modelo de negocio de límites
por plan — sin él, los límites definidos en las Historias 1 y 2 son solo información, no una
restricción real.

**Independent Test**: Con un tenant en un plan con límite de 5 mesas y exactamente 5 mesas ya
creadas, un Tenant Admin intenta crear una sexta mesa y verifica que la creación se rechaza
con un mensaje que menciona el límite (5) y que no queda ninguna mesa nueva creada.

**Acceptance Scenarios**:

1. **Given** un tenant tiene un límite de 5 mesas en su plan y ya tiene 5 mesas creadas,
   **When** el Tenant Admin intenta crear una sexta mesa, **Then** el sistema rechaza la
   creación, no crea el registro, y muestra un mensaje que indica el límite alcanzado (5) y
   que debe actualizar su plan para crear más.
2. **Given** un tenant tiene un límite de 5 mesas y solo 4 creadas, **When** el Tenant Admin
   crea una quinta mesa, **Then** la creación se permite sin ningún aviso de límite.
3. **Given** un tenant tiene la característica de mesas marcada como "ilimitado", **When** el
   Tenant Admin crea cualquier cantidad de mesas, **Then** nunca se bloquea la creación por
   ese motivo.
4. **Given** un límite numérico gobierna varios tipos de recurso (mesas, cajas, usuarios,
   productos, métodos de pago activos), **When** el tenant alcanza el límite de cualquiera de
   ellos, **Then** el mismo mecanismo de bloqueo y mensaje explicativo aplica de forma
   consistente para ese recurso específico.

---

### User Story 4 - El sistema deniega el acceso a módulos que el plan no incluye (Priority: P3)

Como Tenant Admin, cuando intento acceder a un módulo que mi plan no incluye (ej.
inventario, compras, promociones), quiero ver un mensaje claro de que necesito otro plan, en
vez de un error técnico o un módulo que simplemente no responde.

**Why this priority**: Junto con la Historia 3, es lo que hace cumplir de verdad la
diferenciación entre planes — sin esto, un tenant en un plan básico podría seguir usando
módulos que no pagó/no le fueron habilitados.

**Independent Test**: Con un tenant en un plan sin acceso a compras, el Tenant Admin intenta
entrar al módulo de compras y verifica que ve un mensaje explicando que su plan no lo
incluye, no un error técnico ni una pantalla en blanco.

**Acceptance Scenarios**:

1. **Given** el plan de un tenant no incluye acceso a un módulo (ej. compras), **When** el
   Tenant Admin intenta acceder a ese módulo, **Then** el sistema le niega el acceso y
   muestra un mensaje claro indicando que su plan actual no lo incluye.
2. **Given** el plan de un tenant sí incluye acceso a un módulo, **When** el Tenant Admin
   accede a ese módulo, **Then** el acceso se concede con normalidad, sin ninguna
   restricción adicional derivada del plan.

---

### User Story 5 - El sistema bloquea automáticamente a un tenant cuyo plan vence sin renovarse (Priority: P3)

Como Tenant Admin, cuando el período (mensual/anual) del plan de mi negocio vence sin que el
Super Admin lo haya renovado, quiero que el sistema me bloquee de la misma forma clara que ya
usa para límites y módulos, para saber que necesito que mi plan se renueve, sin perder ningún
dato en el proceso.

**Why this priority**: Es lo que hace que la duración y el precio de un plan (Historias 1 y 2)
tengan un efecto real — sin este bloqueo automático, un tenant seguiría operando indefinidamente
sobre un plan cuyo período ya terminó, igual que sin las Historias 3 y 4 los límites y accesos
de un plan serían solo información.

**Independent Test**: Con un tenant cuya asignación de plan tiene fecha de vencimiento en el
pasado (y el Super Admin no la renovó), verificar que cualquier intento de crear un recurso
limitado o acceder a un módulo se bloquea con el mismo tipo de mensaje que un límite alcanzado o
un módulo no incluido — sin que el Super Admin tenga que hacer nada para que el bloqueo empiece
a aplicar.

**Acceptance Scenarios**:

1. **Given** la asignación de plan de un tenant venció sin renovarse, **When** el Tenant Admin
   intenta crear cualquier recurso gobernado por límite (mesa, caja, usuario, producto, método
   de pago activo), **Then** el sistema rechaza la creación con un mensaje que indica que el
   plan venció y debe renovarse — mismo mecanismo que FR-006 (Clarifications, sesión de
   ampliación).
2. **Given** la asignación de plan de un tenant venció sin renovarse, **When** el Tenant Admin
   intenta acceder a cualquier módulo gobernado por el plan (inventario, compras,
   promociones), **Then** el sistema deniega el acceso con un mensaje que indica que el plan
   venció — mismo mecanismo que FR-009 (Clarificaciones, sesión de ampliación).
3. **Given** un tenant queda bloqueado por vencimiento, **When** se consultan sus recursos ya
   creados o los datos de sus módulos, **Then** nada se pierde ni se borra — el bloqueo solo
   impide creaciones nuevas y accesos nuevos, igual que el resto de bloqueos por plan
   (Clarifications, sesión de ampliación).
4. **Given** un tenant tiene una asignación de plan sin ciclo de facturación ni fecha de
   vencimiento (ej. un plan de transición o de uso interno), **When** pasa cualquier cantidad
   de tiempo, **Then** ese tenant nunca se bloquea automáticamente por vencimiento
   (Clarifications, sesión de ampliación).
5. **Given** un tenant está bloqueado por vencimiento, **When** el Super Admin lo renueva
   (Historia de Usuario 2, Acceptance Scenario 7), **Then** el bloqueo se levanta de inmediato,
   sin ninguna acción adicional del Tenant Admin.

---

### User Story 6 - El Tenant Admin consulta su plan y su consumo actual (Priority: P4)

Como Tenant Admin, quiero ver mi plan actual, cuánto llevo consumido de cada límite (ej. "4
de 5 mesas") y cuándo vence mi período actual, en una sola pantalla, para saber si necesito un
plan superior o una renovación antes de toparme con un bloqueo.

**Why this priority**: Es una historia de visibilidad, no de protección del negocio — mejora
la experiencia y reduce sorpresas, pero las Historias 3, 4 y 5 ya garantizan el cumplimiento del
plan (incluyendo el bloqueo por vencimiento) sin que esta pantalla exista.

**Independent Test**: Con un tenant en un plan con límite de 5 mesas y 4 ya creadas, el
Tenant Admin abre la pantalla de su plan y verifica que ve "4 de 5" para mesas, junto con el
estado (sí/no) de cada acceso de módulo de su plan y la fecha de vencimiento de su período
actual, todo en una sola vista.

**Acceptance Scenarios**:

1. **Given** un tenant tiene un plan con varias características numéricas y de módulo,
   **When** el Tenant Admin abre la pantalla de su plan, **Then** ve, en una sola pantalla,
   el nombre de su plan, y para cada característica numérica cuánto lleva usado sobre el
   límite (o "ilimitado"), y para cada acceso de módulo si está incluido o no.
2. **Given** una característica numérica está marcada como "ilimitado", **When** el Tenant
   Admin la consulta en esta pantalla, **Then** ve "ilimitado" en vez de un número de límite.
3. **Given** la asignación de plan de un tenant tiene un ciclo de facturación y una fecha de
   vencimiento, **When** el Tenant Admin abre la pantalla de su plan, **Then** ve esa fecha de
   vencimiento junto con el resto de la información (Clarifications, sesión de ampliación).
4. **Given** la asignación de plan de un tenant no tiene fecha de vencimiento (plan de
   transición/interno), **When** el Tenant Admin abre la pantalla de su plan, **Then** ve un
   indicador de "sin vencimiento" en vez de una fecha (Clarifications, sesión de ampliación).

---

### Edge Cases

- Un tenant es cambiado a un plan con un límite numérico menor a lo que ya tiene en uso: los
  recursos existentes se conservan sin cambios; el tenant solo queda bloqueado para crear
  recursos nuevos de ese tipo hasta bajar del límite (Clarifications).
- A un tenant se le retira el acceso a un módulo que ya tenía datos cargados: el módulo queda
  completamente inaccesible (no de solo lectura); los datos no se borran, pero el tenant no
  puede verlos hasta que se le reasigne el acceso (Clarifications).
- El Super Admin intenta crear un tenant sin elegir un plan: el sistema no permite completar
  la creación — no existe un tenant sin plan, ni siquiera transitoriamente (Clarifications).
- El Super Admin intenta desactivar/eliminar un plan que tiene tenants asignados actualmente:
  el sistema no debe dejar a esos tenants sin plan vigente como efecto colateral.
- El Super Admin edita el valor de una característica de un plan (ej. sube el límite de
  mesas): el cambio aplica de inmediato a todos los tenants que tengan ese plan, sin que el
  Super Admin tenga que tocar cada tenant uno por uno.
- Un tenant está justo en su límite (ej. 5 de 5 mesas) y el Super Admin le sube el límite del
  plan (a 8): el tenant puede volver a crear recursos de inmediato, sin ninguna acción
  adicional de su parte.
- Dos solicitudes de creación del mismo recurso llegan casi al mismo tiempo cuando queda
  exactamente un cupo disponible (ej. 4 de 5 mesas): el sistema garantiza que como máximo una
  de las dos tenga éxito; la otra se rechaza con el mismo mensaje de límite alcanzado, sin
  importar el orden exacto de llegada (Clarifications).
- Una asignación de plan no tiene ciclo de facturación ni fecha de vencimiento (plan de
  transición o de uso interno): nunca se bloquea automáticamente, sin importar cuánto tiempo
  pase (Clarifications, sesión de ampliación).
- El Super Admin renueva el plan de un tenant que ya venció y está bloqueado: el bloqueo se
  levanta de inmediato, sin ninguna acción adicional del Tenant Admin (Clarifications, sesión de
  ampliación).
- El Super Admin intenta asignar un ciclo de facturación (ej. anual) para el que el plan
  elegido no tiene precio definido: el sistema rechaza esa combinación hasta que elija un ciclo
  con precio definido en ese plan (Clarifications, sesión de ampliación).
- Un tenant queda bloqueado por vencimiento: igual que el retiro manual de un módulo o la baja
  de un límite, los recursos y datos existentes se conservan sin cambios — el vencimiento nunca
  borra nada (Clarifications, sesión de ampliación).

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST permitir al Super Admin crear y editar planes, cada uno con
  nombre, descripción y un conjunto de características. Un plan no está obligado a definir
  explícitamente las 8 características posibles; puede dejar algunas sin configurar.
- **FR-002**: Cada característica de un plan MUST ser de uno de dos tipos: límite numérico
  (con "ilimitado" como valor válido) o acceso a un módulo (sí/no). Toda característica que un
  plan no defina explícitamente MUST tratarse por defecto como bloqueada: límite numérico en 0
  para los tipos de recurso limitado, o acceso denegado para los módulos (Clarifications).
- **FR-003**: El sistema MUST garantizar que todo tenant tenga, en todo momento, exactamente
  un plan vigente — nunca queda sin plan asignado.
- **FR-004**: El sistema MUST exigir que se elija un plan existente como parte del propio
  proceso de creación de un tenant — la creación no se completa sin esa elección, y no existe
  ningún mecanismo de plan por defecto implícito. Esa elección MUST incluir además el ciclo de
  facturación de la asignación inicial (FR-017).
- **FR-005**: El sistema MUST validar en tiempo real, al intentar crear un recurso gobernado
  por un límite numérico (mesa, caja, usuario, producto, método de pago activo), si el tenant
  ya alcanzó el límite de su plan vigente para ese recurso. Para cajas, usuarios y productos,
  el conteo de uso incluye todos los registros existentes de ese tipo, estén activos o
  inactivos — desactivar uno de estos recursos no libera cupo del límite. Para métodos de
  pago, el conteo se limita a los que el tenant tiene actualmente activos (ver Supuestos). Las
  mesas no tienen estado activo/inactivo en el sistema actual, por lo que su conteo es
  simplemente el total de mesas existentes.
- **FR-006**: Cuando un tenant alcanzó el límite numérico de su plan para un recurso, el
  sistema MUST bloquear la creación de ese recurso (no crear el registro) y mostrar un
  mensaje que indique el límite alcanzado y que debe actualizar su plan para crear más.
- **FR-007**: El sistema MUST permitir que una característica numérica esté marcada como
  "ilimitado", en cuyo caso nunca bloquea la creación de ese recurso por motivo de límite.
- **FR-008**: El sistema MUST validar, al intentar acceder a un módulo gobernado por un
  acceso booleano (inventario, compras, promociones), que el plan vigente del tenant lo
  incluya antes de permitir el acceso.
- **FR-009**: Cuando el plan vigente de un tenant no incluye un módulo, el sistema MUST
  denegar el acceso y mostrar un mensaje claro, no técnico, indicando que su plan actual no
  lo incluye.
- **FR-010**: El sistema MUST permitir al Super Admin cambiar el plan vigente de un tenant en
  cualquier momento.
- **FR-011**: Cuando el Super Admin cambia el plan de un tenant a uno con un límite numérico
  menor a lo que el tenant ya tiene en uso, el sistema MUST conservar los recursos existentes
  sin cambios y solo bloquear la creación de recursos nuevos de ese tipo mientras el tenant
  siga por encima del nuevo límite.
- **FR-012**: Cuando el Super Admin cambia el plan de un tenant a uno que ya no incluye un
  módulo con datos existentes, el sistema MUST volver ese módulo completamente inaccesible
  para el tenant (mismo comportamiento de FR-008/FR-009 — no un modo de solo lectura
  especial), sin borrar los datos ya cargados, hasta que se le reasigne el acceso.
- **FR-013**: El sistema MUST permitir al Tenant Admin consultar, en una sola pantalla, el
  nombre de su plan vigente, la fecha de vencimiento de su asignación (o un indicador de "sin
  vencimiento" si no tiene, FR-021) y, para cada característica numérica, cuánto lleva
  consumido frente al límite (o "ilimitado" si aplica), y para cada acceso de módulo si está
  incluido o no.
- **FR-014**: Cuando el Super Admin edita el valor de una característica de un plan, el
  sistema MUST aplicar el cambio de inmediato a todos los tenants que tengan ese plan
  vigente, sin requerir ninguna acción adicional sobre cada tenant.
- **FR-015**: Cuando dos o más solicitudes de creación del mismo tipo de recurso limitado
  para un mismo tenant llegan de forma simultánea o casi simultánea, el sistema MUST
  garantizar que el límite del plan nunca se supere como resultado de esa concurrencia — como
  máximo tantas solicitudes tienen éxito como cupo quedaba disponible al momento de resolverse
  la carrera; el resto se rechaza con el mismo mensaje de límite alcanzado (Clarifications).
- **FR-016**: El sistema MUST permitir al Super Admin definir, para cada plan, un precio
  mensual y un precio anual, cada uno opcional de forma independiente del otro (Clarifications,
  sesión de ampliación).
- **FR-017**: El sistema MUST exigir un ciclo de facturación (mensual o anual) como parte de
  toda asignación o cambio de plan de un tenant, y MUST rechazar la elección de un ciclo para
  el cual el plan elegido no tiene un precio definido (FR-016) (Clarifications, sesión de
  ampliación).
- **FR-018**: El sistema MUST calcular la fecha de vencimiento de una asignación de plan como
  la fecha en que se asignó o renovó más un período (un mes para el ciclo mensual, un año para
  el anual) (Clarifications, sesión de ampliación).
- **FR-019**: Cuando la fecha de vencimiento de la asignación de plan de un tenant se cumple
  sin que el Super Admin la haya renovado, el sistema MUST bloquear automáticamente, desde ese
  momento, la creación de todo recurso gobernado por límite (mismo mecanismo y mensaje de
  FR-006) y el acceso a todo módulo (mismo mecanismo y mensaje de FR-009), sin borrar ningún
  dato existente (Clarifications, sesión de ampliación).
- **FR-020**: El sistema MUST permitir al Super Admin renovar la asignación de plan vigente de
  un tenant (el mismo plan u otro, con su ciclo de facturación) en cualquier momento, antes o
  después del vencimiento — la renovación MUST reiniciar el período desde ese momento y
  levantar de inmediato cualquier bloqueo por vencimiento, sin requerir ninguna acción del
  Tenant Admin (mismo criterio de aplicación inmediata que FR-010/FR-014) (Clarifications,
  sesión de ampliación).
- **FR-021**: Una asignación de plan sin ciclo de facturación ni fecha de vencimiento MUST no
  bloquearse nunca automáticamente por vencimiento (Clarifications, sesión de ampliación).

### Key Entities

- **Plan**: Representa un nivel de suscripción administrado por el Super Admin (ej. Básico,
  Pro, Premium). Define su nombre, descripción, el conjunto de características que lo
  componen, y un precio mensual y un precio anual (cada uno opcional de forma independiente,
  Clarifications sesión de ampliación). Un plan puede estar asignado a cero, uno o varios
  tenants a la vez.
- **Característica de Plan**: Una regla dentro de un plan. Tiene un tipo (límite numérico o
  acceso de módulo), una clave que identifica a qué gobierna (mesas, cajas, usuarios,
  productos, métodos de pago activos, inventario, compras, promociones), y un valor: un
  número o "ilimitado" para los límites numéricos, sí/no para los accesos de módulo. Un plan
  no necesita definir las 8 claves posibles; cualquiera no definida explícitamente se
  comporta como bloqueada (límite 0 o acceso denegado, según el tipo) (Clarifications).
- **Asignación de Plan por Tenant**: La relación entre un tenant y su plan vigente en un
  momento dado. Determina qué características aplican al validar límites y accesos de ese
  tenant. Cada tenant tiene exactamente una asignación vigente a la vez. Además del plan,
  incluye el ciclo de facturación elegido (mensual o anual), la fecha en que se asignó o
  renovó por última vez, y la fecha de vencimiento calculada a partir de esas dos (ausente =
  sin vencimiento, nunca bloquea — Clarifications, sesión de ampliación). Renovar reinicia
  estas dos fechas sin cambiar necesariamente el plan.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El Super Admin puede crear un plan, asignarlo a un tenant de prueba, y
  verificar —usando la aplicación, sin leer código— que ese tenant no puede superar ninguno
  de los límites definidos, en menos de 10 minutos.
- **SC-002**: El 100% de los intentos de crear un recurso que superaría el límite numérico
  del plan del tenant son bloqueados, sin excepción, con un mensaje explicativo que menciona
  el límite.
- **SC-003**: El 100% de los intentos de acceder a un módulo no incluido en el plan del
  tenant son denegados con un mensaje claro, nunca con un error técnico.
- **SC-004**: Un Tenant Admin puede ver, en una sola pantalla y sin ayuda de soporte, cuánto
  lleva consumido de cada límite de su plan.
- **SC-005**: Un cambio de plan (asignación nueva, o edición de una característica de un
  plan ya asignado) aplica en el siguiente intento de creación o acceso del tenant, sin
  requerir que el tenant cierre sesión o espere un proceso posterior.
- **SC-006**: El 100% de los tenants, en todo momento, tiene exactamente un plan vigente
  asignado — ninguno queda sin plan ni con más de uno vigente a la vez.
- **SC-007**: El 100% de los tenants cuya asignación de plan venció sin renovarse quedan
  bloqueados (mismos criterios que SC-002/SC-003) desde el primer intento de creación o acceso
  posterior al vencimiento, sin que el Super Admin tenga que hacer nada para que el bloqueo
  empiece a aplicar (Clarifications, sesión de ampliación).

## Assumptions

- El módulo de Promociones ya existe en el sistema (specs anteriores de este mismo
  repositorio ya lo cubren) — esta funcionalidad solo agrega la validación de acceso sobre él,
  no lo crea.
- El límite de "métodos de pago activos simultáneamente" se cuenta sobre los métodos de pago
  que el tenant tiene actualmente activos (no sobre el catálogo completo de la plataforma).
- Para cajas, usuarios y productos —recursos que en el sistema actual pueden marcarse como
  activos o inactivos—, el límite del plan cuenta todos los registros existentes sin importar
  su estado; desactivar uno de estos recursos no libera cupo (Clarifications). Esto contrasta
  deliberadamente con la regla de métodos de pago activos del punto anterior.
- Los ocho tipos de característica listados (mesas, cajas, usuarios, productos, métodos de
  pago activos, inventario, compras, promociones) son el conjunto inicial para esta fase;
  agregar un tipo de característica nuevo en el futuro es una extensión posterior, no parte
  de esta spec.
- No se requiere un historial auditable de cambios de plan en esta fase — alcanza con saber
  cuál es el plan vigente de cada tenant en cada momento.
- Un plan no puede eliminarse mientras tenga al menos un tenant asignado; el Super Admin debe
  reasignar esos tenants a otro plan antes de poder eliminarlo (o el sistema simplemente no
  permite eliminar planes en uso).
- Esta funcionalidad sigue siendo completamente administrativa en cuanto al cobro: el precio de
  un plan (FR-016) es un dato de referencia, no dispara ningún proceso de pago ni de
  facturación — no hay pasarela de pago ni cobro automático en ningún punto de esta spec. A
  diferencia de la versión inicial de esta spec, sí incluye duración (ciclo de facturación
  mensual/anual, FR-017/FR-018) y vencimiento: cuando el período de un tenant vence sin que el
  Super Admin lo renueve manualmente, el sistema lo bloquea automáticamente (FR-019, Historia de
  Usuario 5); la renovación (FR-020) sigue siendo una acción manual del Super Admin, igual que
  la asignación inicial — nunca un cobro ni un proceso automático de facturación.
- Los roles Super Admin y Tenant Admin corresponden a los roles y permisos ya existentes en
  el sistema; esta funcionalidad no introduce roles nuevos.
