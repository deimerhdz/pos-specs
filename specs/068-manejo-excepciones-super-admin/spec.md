# Feature Specification: Manejo de excepciones y respuestas de error consistentes en el módulo super-admin

**Feature Branch**: `068-manejo-excepciones-super-admin`

**Created**: 2026-09-02

**Status**: Draft

**Input**: User description: "Implementar/refactorizar el manejo de excepciones únicamente en el módulo `super-admin` de un backend FastAPI multi-tenant, sin migrar ni modificar masivamente otros módulos. Establecer un patrón correcto, reutilizable y consistente: separar errores de dominio/aplicación (sin depender de FastAPI), de infraestructura (PostgreSQL, Redis, servicios externos) y de traducción HTTP; reutilizar excepciones y patrones ya existentes en vez de duplicarlos; garantizar aislamiento multi-tenant usando el `tenant_id` del contexto autenticado y no uno enviado sin validar; responder siempre con una estructura de error consistente (`success`, `error.code`, `error.message`, `error.details`, `request_id`) sin exponer trazas, SQL, credenciales ni tokens; integrar Sentry respetando la configuración existente, activo únicamente en producción, sin enviar excepciones de dominio esperadas, y con contexto de `request_id`/`tenant_id`/`user_id`/módulo/operación cuando sea posible; y documentar el patrón resultante para poder migrar después otros módulos."

## Clarifications

### Session 2026-09-02

- Q: El panel de super-admin en pos-heladeria (frontend Angular; servicios de tenant, plan, catálogo de métodos de pago y usuarios de super-admin) hoy lee el error de cada respuesta 4xx como `err.error.detail` (texto plano). El formato de error objetivo (`success`/`error.{code,message,details}`/`request_id`) no incluye ese campo `detail`. Este spec es solo de `pos-backend` y no toca `pos-heladeria`: ¿cómo se maneja esa compatibilidad? → A: El nuevo envelope conserva, además de `success`/`error`/`request_id`, un campo `detail` de nivel superior con el mismo texto de `error.message`, para que las pantallas actuales del panel de super-admin sigan mostrando el mensaje real sin que este spec necesite tocar `pos-heladeria`. La migración del frontend a los campos estructurados queda para un trabajo posterior, fuera de este spec.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Errores de negocio claros y sin fugas de información técnica (Priority: P1)

Como persona que opera el panel de super-admin (asignar planes, crear tenants, administrar el
catálogo de métodos de pago y planes), cuando una operación falla por una razón de negocio
—tenant o plan inexistente, ciclo de facturación sin precio definido, dato duplicado, estado
inválido—, quiero recibir un mensaje claro y específico de por qué falló la operación, sin que
la respuesta exponga detalles técnicos internos del sistema.

**Why this priority**: Es el comportamiento observable mínimo que hace útil todo el resto del
feature. Sin mensajes claros y consistentes, ni la clasificación interna de errores ni la
integración con observabilidad tienen ningún efecto perceptible para quien usa o mantiene el
módulo.

**Independent Test**: Se puede probar por completo invocando cada endpoint de super-admin con
datos que disparan cada motivo de fallo conocido (recurso inexistente, conflicto, estado
inválido, sin permisos) y verificando que cada respuesta trae un mensaje específico a esa causa,
el código HTTP correcto, y ningún fragmento de SQL, ruta de archivo, traza de pila, credencial o
token en el cuerpo de la respuesta.

**Acceptance Scenarios**:

1. **Given** un super-admin autenticado, **When** intenta asignar un plan a un `tenant_id` que no
   existe, **Then** recibe un error de "tenant no encontrado" específico, con el código HTTP de
   recurso no encontrado, y sin ninguna pista sobre la estructura interna de la base de datos.
2. **Given** un super-admin autenticado, **When** intenta asignar un plan con un ciclo de
   facturación que no tiene precio definido para ese plan, **Then** recibe un error de conflicto
   específico a esa causa, con el código HTTP correspondiente.
3. **Given** un super-admin autenticado, **When** ocurre una falla técnica inesperada (por
   ejemplo, la base de datos no responde) al procesar cualquier operación del módulo, **Then**
   recibe una respuesta de error genérica y segura (sin trazas ni detalles internos), con el
   código HTTP de error del servidor.
4. **Given** un usuario que no tiene rol de super-admin (o no está autenticado), **When** intenta
   invocar cualquier endpoint del módulo, **Then** la solicitud se rechaza con el mismo código y
   mensaje que hoy, sin revelar ningún dato sobre tenants, usuarios o planes existentes.

---

### User Story 2 - Observabilidad en producción sin ruido de errores esperados (Priority: P2)

Como integrante del equipo técnico responsable de operar el backend, quiero que las fallas
técnicas inesperadas del módulo super-admin en producción queden registradas automáticamente
con el contexto necesario para diagnosticarlas (identificador de la solicitud, quién la originó,
qué módulo y operación fallaron), y quiero que los errores de negocio esperados —que ocurren
todo el tiempo durante el uso normal— no generen ruido en esa misma herramienta.

**Why this priority**: Depende de que la clasificación de errores de la Historia 1 ya distinga
entre "error de negocio esperado" y "falla técnica inesperada"; sin esa base no hay manera
confiable de decidir qué reportar. Es la segunda prioridad porque agrega capacidad de diagnóstico,
pero el módulo sigue siendo funcional y seguro sin ella.

**Independent Test**: Se puede probar provocando, en un entorno configurado como producción, una
falla técnica real (ej. una dependencia de infraestructura caída) y verificando que aparece un
evento en la plataforma de monitoreo con el contexto esperado; y provocando, en el mismo entorno,
varios errores de negocio esperados (recurso no encontrado, conflicto) y verificando que ninguno
de ellos genera un evento allí. Se puede confirmar además que, fuera de producción, ninguna de las
dos situaciones genera un evento.

**Acceptance Scenarios**:

1. **Given** el sistema configurado como entorno de producción, **When** ocurre una falla técnica
   inesperada en cualquier operación del módulo super-admin, **Then** se registra un evento en la
   plataforma de monitoreo de errores que incluye el identificador de la solicitud, el
   identificador del super-admin autenticado, el módulo y la operación afectada.
2. **Given** el sistema configurado como entorno de producción, **When** ocurre un error de
   negocio esperado (recurso no encontrado, conflicto, estado inválido, sin autorización),
   **Then** no se genera ningún evento en la plataforma de monitoreo de errores para esa solicitud.
3. **Given** el sistema configurado como un entorno distinto de producción (desarrollo, pruebas
   automatizadas, staging), **When** ocurre cualquier tipo de error en el módulo super-admin —de
   negocio o técnico—, **Then** no se genera ningún evento en la plataforma de monitoreo de
   errores.
4. **Given** la integración con la plataforma de monitoreo de errores deshabilitada o no
   disponible, **When** ocurre cualquier error en el módulo super-admin, **Then** la respuesta al
   llamador se comporta exactamente igual que si la integración estuviera activa.

---

### User Story 3 - Continuidad para el panel de super-admin existente (Priority: P3)

Como persona que ya usa hoy el panel de super-admin del frontend, quiero seguir viendo el mismo
mensaje de error real que veo actualmente cuando algo falla, aunque el backend cambie internamente
la forma en que construye y clasifica sus errores.

**Why this priority**: Es la prioridad más baja porque no cambia ningún comportamiento nuevo — su
único propósito es no introducir una regresión visible mientras se entregan las Historias 1 y 2.
Es independiente porque se puede verificar sin haber completado la integración con observabilidad.

**Independent Test**: Se puede probar reproduciendo, contra el backend ya cambiado, los mismos
escenarios de error que hoy muestra el panel de super-admin (tenant no encontrado, plan no
encontrado, conflicto de ciclo de facturación, datos duplicados) y confirmando que el panel
sigue mostrando el mismo texto que mostraba antes del cambio, sin modificar el código del panel.

**Acceptance Scenarios**:

1. **Given** una de las pantallas actuales del panel de super-admin que hoy muestra el mensaje de
   error tal cual lo envía el backend, **When** ocurre el mismo error de negocio después del
   cambio, **Then** la pantalla muestra el mismo mensaje que mostraba antes, sin ningún cambio en
   el código del frontend.

---

### Edge Cases

- ¿Qué pasa cuando una operación de creación de tenant recibe todos los datos correctos pero el
  envío del correo de bienvenida falla porque el servicio de mensajería (cola/worker) no está
  disponible? El tenant debe seguir reportándose como creado exitosamente (comportamiento actual
  protegido); en producción, esa falla técnica de infraestructura debe quedar visible para el
  equipo técnico sin afectar la respuesta que recibe quien llama.
- ¿Qué pasa cuando el identificador de tenant, plan o usuario llega por la ruta o el cuerpo de la
  solicitud pero no corresponde a ningún registro real? Debe tratarse como "recurso no
  encontrado", nunca usarse para inferir ni confirmar la existencia de otro registro relacionado.
- ¿Qué pasa cuando alguien sin rol de super-admin (incluido un usuario válido de un tenant)
  intenta usar cualquier endpoint del módulo? Se rechaza sin revelar si el recurso solicitado
  existe o no.
- ¿Qué pasa cuando dos operaciones intentan crear o modificar el mismo recurso de forma que
  produce un valor duplicado (por ejemplo, un host de tenant ya usado)? Debe reportarse como
  conflicto, con un mensaje que identifique el campo en conflicto sin exponer detalles del motor
  de base de datos.
- ¿Qué pasa si la plataforma de monitoreo de errores no está configurada o falla al recibir un
  evento? El módulo super-admin debe seguir respondiendo con normalidad a quien llama.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE responder toda solicitud fallida a un endpoint del módulo
  super-admin (incluidos sus sub-recursos: catálogo de métodos de pago y catálogo de planes) con
  una estructura de error consistente que incluya, como mínimo: un indicador de éxito, un código
  de error estable, un mensaje legible para quien llama, un campo de detalles opcional, un
  identificador de la solicitud, y —por compatibilidad con los consumidores existentes— un campo
  de mensaje plano de nivel superior igual al mensaje legible.
- **FR-002**: El sistema DEBE devolver el código de estado HTTP semánticamente correcto según la
  categoría del error: recurso no encontrado, conflicto, estado inválido, no autenticado, y sin
  permiso de super-admin.
- **FR-003**: Ninguna respuesta de error del módulo super-admin DEBE contener información técnica
  interna (consultas o fragmentos de SQL, trazas de pila, nombres internos de esquema, rutas de
  archivo, credenciales o tokens), sin importar si el error se originó por una regla de negocio o
  por una falla técnica.
- **FR-004**: El sistema DEBE seguir rechazando, con el mismo código y sin revelar información
  sobre tenants, usuarios o planes existentes, cualquier solicitud a un endpoint del módulo que no
  provenga de un usuario autenticado con rol de super-admin (comportamiento ya vigente, que este
  cambio no puede debilitar).
- **FR-005**: Cuando una operación del módulo haga referencia a un tenant, plan, usuario o método
  de pago que no exista, el sistema DEBE responder con un error de "recurso no encontrado"
  específico a ese tipo de entidad.
- **FR-006**: El sistema DEBE clasificar internamente los errores del módulo super-admin en
  categorías de negocio reutilizables (por ejemplo: recurso no encontrado, regla de negocio
  violada, conflicto, estado inválido, sin autorización), de forma que esa clasificación sea
  verificable sin necesidad de construir ni simular una solicitud HTTP.
- **FR-007**: El sistema DEBE preservar, sin cambios observables por quien llama, todo flujo que
  hoy tolera una falla parcial no crítica (por ejemplo: la creación de un tenant se sigue
  reportando exitosa aunque el envío del correo de bienvenida falle).
- **FR-008**: En el entorno de producción, el sistema DEBE reportar a la plataforma de monitoreo
  de errores toda falla técnica inesperada (no un error de negocio esperado) ocurrida dentro del
  módulo super-admin, incluyendo al menos: el identificador de la solicitud, el identificador del
  super-admin autenticado que la originó, el módulo y la operación afectada.
- **FR-009**: El sistema NO DEBE reportar a la plataforma de monitoreo de errores los errores de
  negocio esperados (recurso no encontrado, conflicto, estado inválido, sin autorización)
  ocurridos durante el uso normal del módulo.
- **FR-010**: Fuera del entorno de producción (desarrollo, pruebas automatizadas, staging), el
  sistema NO DEBE enviar ningún evento a la plataforma de monitoreo de errores, incluso si la
  integración está configurada.
- **FR-011**: El sistema DEBE seguir respondiendo con normalidad a toda solicitud del módulo
  super-admin cuando la integración con la plataforma de monitoreo de errores está deshabilitada,
  mal configurada o no disponible.
- **FR-012**: Ningún dato sensible (contraseñas, tokens de autenticación, cuerpos completos de
  solicitud) DEBE incluirse en la información que se envía a la plataforma de monitoreo de
  errores.
- **FR-013**: La identidad y el rol de super-admin usados para autorizar cada operación DEBEN
  obtenerse siempre de la sesión o token autenticado, nunca de un valor recibido sin validar en el
  cuerpo, la ruta o los parámetros de la solicitud. Cuando una operación reciba un identificador de
  tenant, plan o usuario en la ruta o el cuerpo, ese identificador se usa únicamente para ubicar el
  registro correspondiente, validando su existencia antes de operar sobre él.
- **FR-014**: El sistema DEBE mantener las reglas de negocio ya definidas para cada endpoint
  existente del módulo (listar usuarios, listar tenants, asignar/cambiar/renovar plan, crear
  tenant, catálogo de métodos de pago, catálogo de planes); este cambio reorganiza cómo se
  detectan, clasifican y comunican los errores, no las reglas de negocio en sí.
- **FR-015**: Todo error devuelto por el módulo super-admin DEBE incluir un identificador de
  solicitud que permita correlacionar lo que ve quien llama con lo que el equipo técnico observa
  en los registros y, en producción, en la plataforma de monitoreo de errores.

### Key Entities

- **Respuesta de error estándar**: estructura consistente que devuelve cualquier endpoint del
  módulo super-admin ante un fallo. Incluye indicador de éxito, código de error, mensaje legible,
  detalles opcionales, identificador de la solicitud, y el campo de mensaje plano de compatibilidad
  descrito en la sección de Clarifications.
- **Categoría de error de negocio**: clasificación interna y reutilizable del motivo de un fallo
  (recurso no encontrado, regla de negocio violada, conflicto, estado inválido, sin autorización),
  independiente de si el error se comunica luego por HTTP.
- **Evento de monitoreo de errores**: registro que se envía a la plataforma de observabilidad
  únicamente en producción y únicamente ante fallas técnicas inesperadas, enriquecido con el
  contexto de la solicitud y la operación que falló.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las respuestas de error de los endpoints del módulo super-admin sigue la
  misma estructura, verificable sin conocer los detalles internos de la implementación.
- **SC-002**: El 0% de las respuestas de error del módulo super-admin expone detalles técnicos
  internos (consultas, trazas, credenciales, tokens), evaluado sobre una muestra representativa de
  escenarios de error que incluye fallas técnicas no anticipadas.
- **SC-003**: Un integrante del equipo técnico puede localizar, en la plataforma de monitoreo de
  errores y usando solo el identificador de solicitud recibido en una respuesta de error de
  producción, el evento correspondiente en menos de 1 minuto.
- **SC-004**: Durante el uso normal del módulo en producción, el 0% de los errores de negocio
  esperados genera un evento en la plataforma de monitoreo, mientras que el 100% de las fallas
  técnicas inesperadas sí queda registrado allí.
- **SC-005**: Fuera de producción, cero eventos llegan a la plataforma de monitoreo de errores,
  sin importar cuántos errores —de negocio o técnicos— ocurran durante pruebas o desarrollo.
- **SC-006**: Las pantallas existentes del panel de super-admin que hoy muestran el mensaje de
  error real lo siguen mostrando después del cambio, sin ninguna modificación en el código de ese
  panel.
- **SC-007**: El módulo super-admin sigue respondiendo correctamente al 100% de sus solicitudes
  cuando la integración con la plataforma de monitoreo de errores está completamente deshabilitada.

## Assumptions

- El módulo super-admin es una superficie de administración global de la plataforma, no un módulo
  con alcance a un único tenant: quien lo usa se autentica como super-admin global (sin tenant
  propio) y se autoriza por su rol, no por pertenecer a un tenant. Por eso, el "aislamiento
  multi-tenant" de este spec consiste en (a) impedir que cualquier usuario que no sea super-admin
  global llegue a estos endpoints —control que ya existe hoy y este cambio no debe debilitar
  (FR-004)— y (b) que ningún identificador de tenant recibido en una ruta o cuerpo se use para nada
  distinto de ubicar el registro correspondiente (FR-013); no consiste en ocultarle tenants entre
  sí a un super-admin, cuyo trabajo es precisamente administrarlos a todos.
- Hoy no existe un manejador de errores global en el backend ni una estructura de respuesta de
  error estandarizada: cada endpoint construye su propio error y quien llama recibe únicamente un
  mensaje plano, sin identificador de solicitud. Este spec asume que hace falta crear la
  infraestructura mínima y reutilizable para ambas cosas, tal como autoriza el pedido original
  cuando esa infraestructura no existe todavía.
- El modelo de datos actual de tenant no tiene ningún campo de estado activo/inactivo; por lo
  tanto, "tenant inactivo" no es un caso manejable hoy y queda fuera del alcance de este spec
  (agregarlo sería una funcionalidad nueva, no una reorganización del manejo de errores).
- Este spec reutiliza el mismo criterio de entorno de producción que ya se usa en el resto del
  backend, en vez de introducir nuevos valores de configuración de entorno.
- El panel de super-admin del frontend consume hoy varios servicios que leen el mensaje de error
  como texto plano de nivel superior. Por la decisión registrada en Clarifications, la nueva
  estructura de error conserva ese mismo campo de mensaje plano junto con los campos nuevos, para
  no romper esas pantallas sin necesidad de modificar el repositorio del frontend.
- Este spec cubre únicamente el backend: el módulo super-admin y la infraestructura mínima y
  reutilizable de manejo de errores y observabilidad que ese módulo necesita. No incluye migrar
  otros módulos del backend a este mismo patrón, ni ningún cambio en el frontend — ambas cosas
  quedan como trabajo futuro, guiado por el patrón que este spec documenta.
