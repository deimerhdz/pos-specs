# Feature Specification: Notificaciones en Tiempo Real Multi-Tenant

**Feature Branch**: `077-notificaciones-tiempo-real`

**Created**: 2026-09-07

**Status**: Draft

**Input**: User description: "Cuando llega un pedido nuevo desde el menú QR de una mesa, el backend emite el evento por SSE, pero solo lo recibe la vista "Terminal de Mesas". Si el cajero está en otra sección del POS (por ejemplo, Ventas), no se entera del pedido nuevo. Si el cajero sale de la pestaña o la cierra, no hay ninguna notificación.

Objetivo

Reemplazar la notificación acoplada a una sola vista por un sistema de notificaciones en tiempo real, multi-tenant, que:

Llegue al cajero sin importar en qué página del POS esté.
Llegue también como notificación push del navegador cuando la pestaña no está enfocada o está cerrada.
Informe al comensal anónimo (QR) del estado de su propio pedido y de la confirmación de pago, en tiempo real.
Quede diseñado para poder incorporar canales de entrega adicionales (email, SMS, WhatsApp) más adelante, sin rehacer el despachador de eventos.

Actores
Cajero / staff del POS, autenticado, perteneciente a un tenant.
Comensal anónimo, identificado solo por su sesión/pedido en el menú QR.
(Fuera de alcance de este spec) Administrador del tenant configurando canales de alerta para reportes o stock bajo.

Escenarios de usuario
Dado un cajero con el POS abierto en la sección "Ventas", cuando un comensal confirma un pedido desde el menú QR de una mesa, entonces el cajero ve la notificación de inmediato, sin necesidad de estar en "Terminal de Mesas".
Dado un cajero con el POS abierto pero la pestaña sin foco (o el navegador minimizado), cuando llega un pedido nuevo, entonces recibe una notificación push del navegador.
Dado un comensal anónimo con el menú QR de su pedido abierto, cuando el cajero confirma el pago, entonces el comensal ve la confirmación en tiempo real sin recargar la página.
Dados dos tenants distintos con cajeros conectados simultáneamente, cuando llega un pedido para el tenant A, entonces ningún usuario del tenant B recibe ni puede suscribirse a esa notificación.
Dado un cajero que pierde la conexión (red inestable, cierre accidental de pestaña) y vuelve a abrir el POS, entonces puede ver qué notificaciones se perdió mientras estuvo desconectado.

Requisitos funcionales
RF-001: El sistema debe emitir una notificación cuando se crea un pedido nuevo desde el menú QR de una mesa.
RF-002: El sistema debe emitir una notificación cuando cambia el estado de un pedido, incluyendo como mínimo la confirmación de pago.
RF-003: Toda notificación dirigida al staff debe llegar a todos los usuarios conectados del tenant correspondiente, sin importar la página/sección del POS en la que se encuentren.
RF-004: Cuando el navegador del destinatario no tiene la pestaña del POS enfocada, el sistema debe entregar además una notificación push del navegador equivalente al evento.
RF-005: El comensal anónimo debe recibir en tiempo real el estado de su propio pedido y la confirmación de pago, sin recargar la página, limitado a su propia sesión/pedido.
RF-006: Cada notificación debe quedar persistida (tenant, tipo de evento, payload, entidad relacionada, timestamp, estado de entrega/lectura por canal) — no debe existir únicamente como mensaje en tránsito.
RF-007: Un tenant no debe poder recibir, ver ni suscribirse, bajo ninguna circunstancia, a notificaciones de otro tenant.
RF-008: Al reconectarse (tras pérdida de red o reapertura del POS), el cliente debe poder recuperar las notificaciones relevantes que no vio mientras estuvo desconectado.
RF-009: El mecanismo que genera y distribuye las notificaciones debe estar desacoplado del mecanismo de entrega, de forma que agregar un nuevo canal no requiera modificar la lógica de negocio que origina el evento.
RF-010 (modelo de datos, no implementación): el modelo debe permitir asociar, por tenant, qué canales de entrega están habilitados por tipo de evento — aunque hoy solo existan los canales "en la aplicación" y "push del navegador".

Requisitos no funcionales
RNF-001: La solución no debe depender de polling contra PostgreSQL para detectar eventos nuevos.
RNF-002: La arquitectura debe soportar múltiples instancias del backend corriendo en paralelo sin perder notificaciones ni duplicarlas.
RNF-003: El aislamiento por tenant debe validarse del lado del servidor, no únicamente filtrarse en el cliente.
RNF-004: La latencia entre la creación del evento y su entrega al destinatario conectado debe ser, en condiciones normales, casi inmediata (segundos, no minutos).
RNF-005: Cada tenant debe tener una política de retención de 90 días sobre sus notificaciones persistidas (NotificationEvent); pasado ese período, deben purgarse automáticamente y dejar de ser accesibles vía API. El valor de retención debe guardarse como dato de configuración (no como constante fija en código), con 90 días como valor por defecto, para poder ajustarlo por tenant o por plan más adelante sin tocar código.

Fuera de alcance (explícito)
Implementación de los canales de email, SMS o WhatsApp — solo se deja el punto de extensión (RF-009, RF-010).
Configuración por el administrador del tenant de qué canal usar para alertas de stock bajo o reportes de ventas.
Integración real con proveedores externos de email/SMS/WhatsApp.

Entidades clave
NotificationEvent: evento persistido (tenant, tipo, payload, entidad relacionada, timestamps, estado de entrega/lectura por canal, fecha de purga según la política de retención del tenant).
NotificationChannel: contrato de canal de entrega; implementaciones actuales: en-aplicación (tiempo real) y push del navegador; futuras: email, SMS, WhatsApp.
PushSubscription: suscripción push de un usuario/dispositivo (endpoint, claves), asociada a un tenant.

Checklist de aceptación
Un pedido nuevo en cualquier mesa notifica al cajero sin importar la página activa del POS.
Con la pestaña sin foco, la notificación también llega como push del navegador.
El comensal ve el estado de su pedido y la confirmación de pago en tiempo real.
Existe una prueba automatizada que demuestra que un evento del tenant A nunca llega a un suscriptor del tenant B.
Tras una desconexión, el cliente puede recuperar las notificaciones perdidas.
Agregar un canal nuevo (ej. email) no requiere tocar el código que genera los eventos de negocio.
Una notificación con más de 90 días de antigüedad para un tenant se purga automáticamente y deja de ser accesible vía API.

Contexto técnico sugerido (no vinculante — para /speckit.plan, no para /speckit.specify): registro de eventos persistido dentro del schema del tenant, Redis Pub/Sub o Streams como backbone de distribución, conexión SSE/WebSocket mantenida a nivel de app-shell en Angular, Web Push con Service Worker y claves VAPID, patrón de canal (interfaz NotificationChannel + implementaciones), job programado que purgue eventos vencidos por tenant."

## Clarifications

### Session 2026-09-07

- Q: Cuando varios cajeros del mismo tenant están conectados, ¿el estado de "leída/atendida" de una notificación es individual por cajero o compartido para todo el tenant? → A: Compartido por tenant — en cuanto un cajero la marca/atiende, se considera atendida para todo el tenant (bandeja de equipo).
- Q: ¿Qué transiciones de estado del pedido deben disparar una notificación, además de la creación del pedido y la confirmación de pago? → A: Solo confirmación de pago — no se agregan otras transiciones de estado (por ejemplo "en preparación" o "listo") en el alcance de esta funcionalidad.
- Q: Cuando un cajero tiene la pestaña del POS enfocada y visible, ¿la notificación en la aplicación debe incluir una alerta sonora además del indicador visual? → A: Sí — indicador visual (insignia/contador y aviso en pantalla) más un sonido audible, para que un pedido no pase inadvertido aunque el cajero no esté mirando la pantalla en ese momento.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El cajero ve el pedido nuevo sin importar en qué página del POS esté (Priority: P1)

Como cajero con el POS abierto en cualquier sección (Ventas, Inventario, Reportes, etc.), quiero enterarme de inmediato cuando un comensal confirma un pedido desde el menú QR de una mesa, para poder atenderlo sin depender de tener abierta la pantalla de Terminal de Mesas.

**Why this priority**: Es el problema que origina esta funcionalidad — hoy la notificación está acoplada a una sola vista y el pedido puede pasar inadvertido. Sin esto, el resto de canales de entrega no tienen ningún valor.

**Independent Test**: Con un cajero autenticado y con el POS abierto en la sección "Ventas", confirmar un pedido desde el menú QR de una mesa de ese tenant y verificar que la notificación aparece de inmediato en el POS del cajero, sin necesidad de navegar a Terminal de Mesas ni de recargar la página.

**Acceptance Scenarios**:

1. **Given** un cajero con el POS abierto en la sección "Ventas", **When** un comensal confirma un pedido desde el menú QR de una mesa de ese mismo tenant, **Then** el cajero ve la notificación de inmediato (indicador visual y sonido audible), sin necesidad de estar en "Terminal de Mesas".
2. **Given** un cajero con el POS abierto en cualquier sección distinta a Terminal de Mesas, **When** se confirma el pago de un pedido existente, **Then** el cajero recibe la notificación de esa confirmación de pago sin importar la sección en la que se encuentre.
3. **Given** varios cajeros del mismo tenant conectados simultáneamente en distintas secciones del POS, **When** llega un pedido nuevo, **Then** todos los cajeros conectados de ese tenant reciben la notificación.

---

### User Story 2 - Aislamiento estricto de notificaciones entre tenants (Priority: P1)

Como negocio que opera una plataforma multi-tenant, quiero que las notificaciones de un tenant nunca lleguen ni sean accesibles para usuarios de otro tenant, para proteger la confidencialidad de los pedidos y ventas de cada negocio.

**Why this priority**: Es una condición de seguridad no negociable para una plataforma multi-tenant; una fuga de notificaciones entre tenants expone información comercial sensible (pedidos, montos, mesas) de un negocio a otro.

**Independent Test**: Con dos tenants distintos y cajeros conectados simultáneamente en cada uno, generar un pedido nuevo para el tenant A y verificar —incluyendo con una prueba automatizada— que ningún usuario conectado del tenant B recibe la notificación ni puede suscribirse al canal de distribución del tenant A.

**Acceptance Scenarios**:

1. **Given** dos tenants distintos con cajeros conectados simultáneamente, **When** llega un pedido nuevo para el tenant A, **Then** ningún usuario del tenant B recibe la notificación.
2. **Given** un usuario autenticado del tenant B, **When** intenta suscribirse por cualquier medio al canal de notificaciones del tenant A, **Then** el servidor rechaza la suscripción, independientemente de lo que el cliente muestre u oculte en la interfaz.
3. **Given** el sistema en operación normal, **When** se ejecuta la prueba automatizada de aislamiento entre tenants, **Then** la prueba confirma que ningún evento del tenant A fue entregado a un suscriptor del tenant B.

---

### User Story 3 - El cajero recibe notificación push cuando la pestaña no tiene foco (Priority: P2)

Como cajero con el POS abierto pero con la pestaña sin foco o el navegador minimizado, quiero recibir una notificación push del navegador equivalente a la notificación en la aplicación, para no perderme un pedido nuevo o un cambio de estado mientras estoy usando otra ventana o aplicación.

**Why this priority**: Extiende la cobertura de la Historia 1 al caso en que el cajero ni siquiera tiene la pestaña visible; es la segunda causa raíz mencionada en el problema original ("si el cajero sale de la pestaña... no hay ninguna notificación").

**Independent Test**: Con un cajero que ha concedido permiso de notificaciones push del navegador y tiene la pestaña del POS sin foco (u otra ventana activa), generar un pedido nuevo o un cambio de estado de pedido y verificar que llega una notificación push del sistema operativo/navegador equivalente al evento.

**Acceptance Scenarios**:

1. **Given** un cajero con el POS abierto pero la pestaña sin foco (o el navegador minimizado), **When** llega un pedido nuevo, **Then** recibe una notificación push del navegador.
2. **Given** un cajero con la pestaña del POS enfocada y visible, **When** llega un pedido nuevo, **Then** no se le duplica la notificación como push del navegador (ya la ve en la aplicación).
3. **Given** un cajero que no ha concedido permiso de notificaciones push del navegador, **When** llega un pedido nuevo con la pestaña sin foco, **Then** igual recibe la notificación en la aplicación en cuanto vuelve a la pestaña, sin que la ausencia de permiso push rompa el resto del sistema.

---

### User Story 4 - El comensal anónimo ve el estado de su pedido y la confirmación de pago en tiempo real (Priority: P2)

Como comensal anónimo con el menú QR de mi pedido abierto, quiero ver en tiempo real la confirmación de mi pago, para saber qué está pasando sin tener que recargar la página ni preguntar en caja.

**Why this priority**: Es el otro extremo del mismo problema de notificación en tiempo real, ahora del lado del comensal; mejora directamente la experiencia de pedido por QR ya existente en el sistema.

**Independent Test**: Con un comensal anónimo con el menú QR de su pedido abierto, hacer que el cajero confirme el pago desde el POS y verificar que la vista del comensal se actualiza en tiempo real sin recargar la página, mostrando la confirmación de pago.

**Acceptance Scenarios**:

1. **Given** un comensal anónimo con el menú QR de su pedido abierto, **When** el cajero confirma el pago, **Then** el comensal ve la confirmación en tiempo real sin recargar la página.
2. **Given** dos comensales distintos con pedidos distintos en la misma mesa, **When** se confirma el pago del pedido de uno, **Then** solo ese comensal ve la actualización; el otro comensal no ve cambios en su propia vista a causa del pedido ajeno.

---

### User Story 5 - El cajero recupera las notificaciones perdidas tras una desconexión (Priority: P3)

Como cajero que perdió la conexión (red inestable o cierre accidental de la pestaña) y vuelvo a abrir el POS, quiero poder ver qué notificaciones me perdí mientras estuve desconectado, para no dejar pedidos sin atender por un corte de conexión.

**Why this priority**: Es una red de seguridad sobre las Historias 1 y 3; sin ella, una desconexión temporal puede hacer que un pedido pase completamente inadvertido, pero el sistema ya es funcional y valioso sin este refinamiento.

**Independent Test**: Con un cajero conectado, simular una pérdida de conexión, generar uno o más pedidos/cambios de estado para su tenant durante la desconexión, reconectar (recargando o reabriendo el POS) y verificar que el cajero puede ver las notificaciones generadas mientras estuvo desconectado.

**Acceptance Scenarios**:

1. **Given** un cajero que pierde la conexión y vuelve a abrir el POS, **When** se reconecta, **Then** puede ver qué notificaciones se perdió mientras estuvo desconectado.
2. **Given** un cajero que se reconecta tras una desconexión, **When** revisa las notificaciones recuperadas, **Then** solo ve las de su propio tenant, en el mismo orden en que ocurrieron.
3. **Given** un cajero que nunca se desconectó, **When** ocurre cualquier evento, **Then** no se le presentan notificaciones duplicadas por efecto del mecanismo de recuperación.

---

### Edge Cases

- ¿Qué pasa si el mismo cajero tiene el POS abierto en varias pestañas o dispositivos a la vez? Debe recibir la notificación en todas sus sesiones activas, sin que eso se interprete como un fallo de duplicación entre tenants.
- ¿Qué pasa si el comensal cierra la pestaña del menú QR antes de que el pago se confirme? El sistema no debe fallar; al reabrir el enlace de su pedido, el comensal debe ver el estado actual (no depende de haber estado conectado en el momento del cambio).
- ¿Qué pasa si el navegador del cajero deniega o no soporta el permiso de notificaciones push? El resto del sistema (notificación en la aplicación) debe seguir funcionando con normalidad.
- ¿Qué pasa si el estado de un pedido cambia varias veces en pocos segundos? Todas las notificaciones deben llegar, en el orden correcto, sin perderse ni intercalarse entre pedidos distintos.
- ¿Qué pasa si el backend está corriendo en varias instancias en paralelo? Ninguna notificación debe perderse ni entregarse duplicada por esa causa.
- ¿Qué pasa con una notificación que aún no ha sido vista por ningún cajero cuando se cumple su plazo de retención de 90 días? Se purga igual, según la política del tenant; el sistema no garantiza disponibilidad indefinida de notificaciones no vistas.
- ¿Qué pasa si un tenant nuevo no tiene configurado explícitamente su valor de retención? Debe aplicarse el valor por defecto de 90 días.
- ¿Qué pasa si el dispositivo del cajero tiene el sonido silenciado o el sistema operativo bloquea la reproducción automática de audio? El indicador visual debe seguir funcionando con normalidad; la ausencia de sonido no debe impedir que la notificación sea visible.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE emitir una notificación cuando se crea un pedido nuevo desde el menú QR de una mesa.
- **FR-002**: El sistema DEBE emitir una notificación cuando se confirma el pago de un pedido. Otras transiciones de estado del pedido (por ejemplo "en preparación" o "listo") quedan fuera del alcance de esta funcionalidad.
- **FR-003**: Toda notificación dirigida al personal (staff) DEBE llegar a todos los usuarios conectados del tenant correspondiente, sin importar la página o sección del POS en la que se encuentren. Mientras la pestaña del POS esté enfocada y visible, la notificación en la aplicación DEBE incluir tanto un indicador visual (insignia/contador y aviso en pantalla) como una alerta sonora audible.
- **FR-004**: Cuando la pestaña del POS del destinatario no está enfocada, el sistema DEBE entregar además una notificación push del navegador equivalente al evento.
- **FR-005**: El comensal anónimo DEBE recibir en tiempo real la confirmación de pago de su propio pedido, sin recargar la página, limitado estrictamente a su propia sesión/pedido.
- **FR-006**: Cada notificación DEBE quedar persistida, registrando como mínimo: tenant, tipo de evento, contenido (payload), entidad relacionada, marca de tiempo, y estado de entrega/lectura por canal — no debe existir únicamente como mensaje en tránsito. Para notificaciones dirigidas al staff, el estado de lectura/atención es compartido por tenant (no por usuario individual): en cuanto cualquier cajero conectado la marca como atendida, deja de mostrarse como pendiente para el resto del equipo de ese tenant.
- **FR-007**: Un tenant NO DEBE, bajo ninguna circunstancia, poder recibir, ver ni suscribirse a notificaciones de otro tenant.
- **FR-008**: Al reconectarse (tras pérdida de red o reapertura del POS), el personal conectado DEBE poder recuperar las notificaciones relevantes de su tenant que no vio mientras estuvo desconectado.
- **FR-009**: El mecanismo que genera y distribuye las notificaciones DEBE estar desacoplado del mecanismo de entrega, de forma que agregar un nuevo canal de entrega no requiera modificar la lógica de negocio que origina el evento.
- **FR-010**: El modelo de datos DEBE permitir asociar, por tenant, qué canales de entrega están habilitados para cada tipo de evento — aunque hoy solo existan los canales "en la aplicación" y "push del navegador", y aunque la configuración de esa asociación por parte del administrador del tenant quede fuera de esta funcionalidad.
- **FR-011**: El sistema DEBE permitir configurar, por tenant, el permiso de notificaciones push del navegador de forma independiente por usuario/dispositivo, de modo que la ausencia de ese permiso no impida recibir notificaciones en la aplicación.

### Non-Functional Requirements

- **NFR-001**: La solución NO DEBE depender de sondeo periódico (polling) contra la base de datos para detectar eventos nuevos.
- **NFR-002**: La arquitectura DEBE soportar múltiples instancias del backend corriendo en paralelo sin perder notificaciones ni entregarlas duplicadas.
- **NFR-003**: El aislamiento por tenant DEBE validarse del lado del servidor; no es suficiente filtrarlo únicamente en el cliente.
- **NFR-004**: La latencia entre la creación del evento y su entrega al destinatario conectado DEBE ser, en condiciones normales, casi inmediata (del orden de segundos, nunca minutos).
- **NFR-005**: Cada tenant DEBE tener una política de retención sobre sus notificaciones persistidas, con 90 días como valor por defecto, guardada como dato de configuración (no como valor fijo en código) para poder ajustarse por tenant o por plan en el futuro sin modificar código. Las notificaciones que superen su período de retención DEBEN purgarse automáticamente y dejar de estar disponibles vía API.

### Key Entities *(include if feature involves data)*

- **NotificationEvent**: Evento de notificación persistido. Representa un hecho de negocio ya ocurrido (pedido nuevo, cambio de estado) del que se debe informar. Incluye: tenant al que pertenece, tipo de evento, contenido/payload, entidad relacionada (pedido, mesa, sesión de comensal), marca de tiempo de creación, estado de entrega/lectura por canal, y fecha de purga calculada según la política de retención vigente del tenant. Para notificaciones de staff, el estado de lectura/atención es único por notificación y compartido por todo el tenant (no se rastrea por usuario individual).
- **NotificationChannel**: Contrato/abstracción de un canal de entrega de notificaciones. Implementaciones actuales dentro de esta funcionalidad: notificación en la aplicación (tiempo real) y notificación push del navegador. Implementaciones futuras (fuera de alcance de esta funcionalidad): email, SMS, WhatsApp.
- **PushSubscription**: Suscripción push de un usuario/dispositivo del staff (identificador de endpoint y claves de cifrado), asociada a un tenant y a un usuario, usada para entregar notificaciones push del navegador cuando su pestaña del POS no tiene foco.
- **Configuración de canales por tipo de evento**: Asociación, por tenant, de qué canales de entrega están habilitados para cada tipo de evento de notificación. Hoy cubre únicamente "en la aplicación" y "push del navegador"; el modelo debe permitir sumar canales futuros sin cambiar su forma.
- **Política de retención de notificaciones**: Dato de configuración por tenant que define, en días, cuánto tiempo se conservan sus notificaciones persistidas antes de purgarse automáticamente (90 días por defecto).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los pedidos nuevos creados desde el menú QR de una mesa generan una notificación visible para el cajero, sin importar la sección del POS en la que se encuentre en ese momento.
- **SC-002**: En condiciones normales de red, el cajero conectado ve la notificación de un pedido nuevo o de un cambio de estado en menos de 5 segundos desde que ocurre el evento.
- **SC-003**: Con la pestaña del POS sin foco, el cajero recibe la notificación equivalente como push del navegador dentro del mismo margen de tiempo que la notificación en la aplicación (segundos, no minutos).
- **SC-004**: El comensal anónimo ve la confirmación de pago de su pedido actualizada en tiempo real, sin recargar la página, en menos de 5 segundos desde que el cajero confirma el pago.
- **SC-005**: Una prueba automatizada demuestra, de forma repetible, que ningún evento de notificación de un tenant llega a un suscriptor de otro tenant (0 fugas detectadas).
- **SC-006**: Tras una reconexión, el cajero recupera el 100% de las notificaciones de su tenant emitidas mientras estuvo desconectado (dentro de la ventana de retención vigente), sin duplicados.
- **SC-007**: Incorporar un canal de entrega nuevo (por ejemplo, email) no requiere ninguna modificación en el código que genera los eventos de negocio de pedidos, solo la adición de una nueva implementación de canal.
- **SC-008**: Las notificaciones con más de 90 días de antigüedad (o el valor de retención configurado para el tenant) dejan de estar disponibles vía API de forma automática, sin intervención manual.
- **SC-009**: El sistema mantiene cero pérdidas y cero duplicados de notificaciones al operar con múltiples instancias del backend en paralelo, verificado mediante prueba automatizada o de carga.

## Assumptions

- El comensal anónimo se identifica mediante el mismo mecanismo de sesión/pedido que ya usa el menú QR existente; esta funcionalidad no introduce un nuevo esquema de autenticación para el comensal.
- La notificación push del navegador (RF-004) aplica al personal autenticado del POS (cajero/staff). No se solicita ni se requiere permiso de notificaciones push del navegador al comensal anónimo del menú QR; su experiencia en tiempo real se resuelve mientras su pestaña está abierta (Historia 4).
- La recuperación de notificaciones perdidas (RF-008) aplica al personal del POS a través de un listado de notificaciones accesible dentro de la aplicación. El comensal anónimo, al reabrir el enlace de su pedido, ve el estado vigente de su pedido en ese momento, sin necesidad de un historial de eventos perdidos.
- El valor de retención de notificaciones se agrega como un dato de configuración por tenant con 90 días por defecto; la posibilidad de que un administrador lo edite desde una interfaz queda fuera de esta funcionalidad y podrá exponerse más adelante sin cambios de código, según NFR-005.
- Los tipos de evento cubiertos por esta funcionalidad son: pedido nuevo originado en el menú QR de mesa, y confirmación de pago de ese pedido. Otras transiciones de estado del pedido (por ejemplo "en preparación" o "listo") y otros tipos de evento de negocio (alertas de stock bajo, reportes) quedan explícitamente fuera de alcance.
- Los pedidos tomados directamente por el personal (por ejemplo, venta mostrador) no son el disparador de esta funcionalidad; el disparador específico es el pedido originado desde el menú QR de una mesa, tal como lo define RF-001.
- Un mismo cajero puede tener más de una sesión activa (varias pestañas o dispositivos); se espera que reciba la notificación en cada sesión activa sin que ello se considere una fuga entre tenants ni una duplicación indebida.
