# Feature Specification: Auditoría del ciclo de vida de una orden

**Feature Branch**: `074-auditoria-ordenes`

**Created**: 2026-09-03

**Status**: Draft

**Input**: User description: "Backend FastAPI multi-tenant (aislamiento por schema de PostgreSQL vía schema_translate_map), SQLAlchemy + Alembic. Logging actual: basado en request_id, enviado a Sentry. El módulo de creación/gestión de órdenes no tiene log de auditoría. [...] Implementa el logging de auditoría del ciclo de vida de una orden, con correlación por order_id, actor explícito (comensal/cajero/sistema), tenant explícito, datos sensibles fuera de Sentry, y un sistema de registro definido explícitamente."

## Mapeo del flujo actual (hallazgos previos a este spec)

Antes de definir requisitos se verificó el ciclo de vida real de una orden contra el mapeo asumido en la solicitud original. Diferencias relevantes encontradas y su efecto sobre el alcance de este spec:

- **Facturación dividida por persona**: contrario a lo asumido, está viva y activa (no deprecada). Sin embargo, por decisión explícita para este spec, el cierre de mesa y la división de cuenta **quedan fuera de alcance** — este spec cubre únicamente el ciclo de vida de la orden individual (creación → confirmación → pago → cancelación), no el cierre de sesión de mesa.
- **Movimiento de caja como efecto colateral**: no existe hoy como evento automático (la caja se concilia al cierre de turno contra los pagos ya registrados). Por decisión explícita, este spec audita la confirmación del pago en sí misma como el hecho relevante, sin introducir un evento de "movimiento de caja" que el sistema no genera hoy.
- **Confirmación de la orden (recepción en Terminal)**: en la práctica ya no es un paso manual independiente en la mayoría de los casos — ocurre como efecto automático de la confirmación del pago (efectivo o transferencia aprobada). El diseño de auditoría debe cubrir ambas rutas (confirmación manual de recuperación y confirmación automática ligada al pago) como el mismo tipo de evento.
- **Logging operativo actual (request_id → Sentry)**: hoy no está aplicado al módulo de órdenes (solo a un módulo administrativo distinto). Este spec no construye esa capa operativa — solo agrega la capa de auditoría descrita aquí, que es independiente.
- El resto del mapeo (una orden sin pagar por comensal, métodos de pago, cancelación iniciada por comensal o staff, y el campo que distingue origen caja/QR) se confirmó tal como se esperaba.
- **Adenda post-implementación**: durante la Fase 3 (`/speckit-implement`) se detectó un cuarto camino de pago no contemplado en el mapeo original: `checkout_and_send` (`POST /orders/{order_id}/checkout-and-send`), que cobra y envía a cocina una comanda `hold_for_payment` en un solo paso, llevándola de `recibida` a `pagada` directamente, sin pasar por `_confirm_order_impl` ni por el flujo de intento de pago pendiente que usan los demás caminos. Se decidió extender el alcance de este spec para cubrirlo (FR-001, FR-014) en vez de dejarlo como limitación conocida, porque implica dinero cobrado — el mismo riesgo que motiva todo el feature.

## Clarifications

### Session 2026-09-03

- Q: Si falla el intento de guardar un evento de auditoría, ¿la transición de negocio de la orden debe fallar también, o debe completarse igual dejando el evento para reintentarlo aparte? → A: Desacoplado — la transición de negocio se completa igual; el evento de auditoría se reintenta o se recupera aparte, sin bloquear al actor que ejecuta la acción.
- Q: ¿Qué nivel de staff debe poder consultar el historial de auditoría de una orden — cualquier miembro del staff, o solo roles administrativos? → A: Solo rol administrativo, igual que el mecanismo de auditoría admin-only ya existente en el sistema.
- Q: ¿Qué retención mínima debe garantizar el sistema para el historial de auditoría de una orden? → A: Mínimo 5 años, alineado con la retención contable/de soporte documental en Colombia. **[Reemplazado en esta misma sesión, ver más abajo — el alcance cambió de tabla propia a solo Sentry.]**
- Q: Cambio de alcance solicitado por el usuario: el log de auditoría ya no se almacena en una tabla propia; el objetivo pasa a ser auditar las peticiones HTTP que llegan al servidor y enviar ese log a Sentry como único destino. ¿Qué debe pasar con el nombre del comensal y el comprobante de pago, dado que ya no existe un registro interno donde guardarlos sin exponerlos? → A: Se permiten enviar a Sentry, transformados de forma no reversible (hash/ofuscados) — nunca en texto plano.
- Q: Dado que ya no hay tabla propia, y Sentry retiene los logs solo 7-30 días según el plan, ¿qué retención debe prometer el spec para el historial de auditoría? → A: La que dé Sentry según el plan contratado (7-30 días); se retira el requisito de 5 años.
- Q: Sin tabla propia, ¿cómo debe quedar la capacidad de un administrador de consultar el historial de auditoría de una orden? → A: Se consulta directamente en el dashboard de Sentry, filtrando por el identificador de la orden; este spec no introduce una API ni pantalla propia de consulta.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Reconstruir el historial completo de una orden para resolver disputas (Priority: P1)

Un administrador del negocio necesita, ante un reclamo ("el cliente dice que pagó pero la caja no lo tiene registrado", o una inconsistencia de inventario/caja detectada después del hecho), reconstruir en orden cronológico todo lo que le pasó a una orden específica: quién la creó, quién la confirmó, cómo se pagó y quién canceló qué, aunque esos pasos hayan ocurrido en momentos distintos, iniciados por personas distintas (el comensal desde su celular, el cajero desde la Terminal, o el propio sistema).

**Why this priority**: Sin esta capacidad, una disputa de pago o una pérdida de inventario/dinero no tiene forma confiable de resolverse — es el motivo original de este trabajo.

**Independent Test**: Se puede probar de forma aislada tomando una orden que pasó por creación, confirmación, pago y (opcionalmente) cancelación, y verificando que, dentro de la ventana de retención vigente en Sentry, filtrar por el identificador de esa orden devuelve un evento por cada transición, todos asociados a ese mismo identificador, en el orden correcto.

**Acceptance Scenarios**:

1. **Given** una orden creada vía QR, confirmada y pagada en efectivo, **When** un administrador filtra en Sentry por el identificador de esa orden, **Then** ve un evento por cada paso (creación, confirmación, pago) en orden cronológico, todos vinculados al mismo identificador de orden.
2. **Given** una orden creada manualmente por un cajero y luego cancelada por otro cajero, **When** se filtra en Sentry por el identificador de esa orden, **Then** ambos eventos (creación y cancelación) aparecen asociados a la misma orden aunque hayan sido ejecutados por personas distintas en momentos distintos.

---

### User Story 2 - Identificar con certeza quién ejecutó cada acción sensible (Priority: P1)

Cuando una orden se cancela o un pago se confirma manualmente, el administrador necesita saber con certeza si la acción la inició el propio comensal (sin cuenta, identificado solo por su sesión de mesa) o un miembro del staff específico (con su usuario y rol) — o si fue el sistema el que la ejecutó automáticamente como efecto de otra acción. Esta distinción es crítica porque cancelaciones y confirmaciones de pago son los puntos más comunes de error operativo o fraude interno.

**Why this priority**: Sin un actor explícito y verificable, la auditoría no sirve para rendición de cuentas — cualquier evento podría atribuirse incorrectamente o quedar ambiguo.

**Independent Test**: Se puede probar generando un evento desde cada tipo de actor (comensal cancelando su propia orden, cajero cancelando una orden, y una confirmación automática disparada por el sistema al aprobarse un pago) y verificando que cada evento resultante identifica correctamente el tipo de actor y su identificador, sin ambigüedad ni con una referencia genérica a "usuario".

**Acceptance Scenarios**:

1. **Given** un comensal cancela su propio pedido antes de que la cocina lo prepare, **When** se registra el evento, **Then** el actor queda identificado como comensal, referenciado por su identificador de sesión (nunca por su nombre).
2. **Given** un cajero cancela una orden confirmada, **When** se registra el evento, **Then** el actor queda identificado como ese cajero específico, con su identificador de usuario y su rol.
3. **Given** la confirmación de un pago dispara automáticamente el paso de una orden a "confirmada" sin que nadie llame ese paso directamente, **When** se registra el evento de confirmación, **Then** el actor queda identificado de forma que se distinga si fue una acción automática del sistema o una atribuible al cajero que confirmó el pago.

---

### User Story 3 - Mantener los datos sensibles del comensal fuera de texto plano en la herramienta de monitoreo externa (Priority: P2)

El nombre del comensal (dato personal) y el comprobante de pago enviado por transferencia (dato financiero) no deben aparecer en texto plano en lo que el sistema envía a la herramienta externa de monitoreo de errores (Sentry) — que es, por decisión de este spec, el único destino del log de auditoría. Cuando el evento de auditoría necesita referenciar alguno de esos dos datos, se envía una versión transformada de forma no reversible (p. ej. un hash), nunca el valor original.

**Why this priority**: Es un requisito de protección de datos personales y financieros frente a un proveedor externo — no bloquea la operación diaria, pero es una condición no negociable de cómo se implementa todo lo demás, más aún ahora que Sentry es el único lugar donde ese evento queda registrado.

**Independent Test**: Se puede probar generando eventos que involucren un comensal con nombre y una orden con comprobante de transferencia adjunto, y verificando que ninguno de esos dos valores aparece en texto plano en los datos enviados a Sentry — solo su forma transformada/no reversible, cuando aplique.

**Acceptance Scenarios**:

1. **Given** un comensal llamado con nombre real completa un pedido, **When** se generan los eventos de auditoría asociados, **Then** el nombre no aparece en texto plano en ningún dato enviado a Sentry; a lo sumo aparece su forma transformada de manera no reversible.
2. **Given** un cajero aprueba un comprobante de transferencia, **When** se registra el evento de auditoría de esa aprobación, **Then** el contenido/URL del comprobante no aparece en texto plano en ningún dato enviado a Sentry; a lo sumo aparece su forma transformada de manera no reversible.

---

### Edge Cases

- ¿Qué actor se registra cuando una transición ocurre como efecto automático de otra acción (p. ej. la orden pasa de "recibida" a "confirmada" como consecuencia de que el cajero confirmó el pago, sin llamar el paso de confirmación por separado)? El evento debe distinguir esto de una acción manual directa, sin perder la trazabilidad de qué acción humana lo originó.
- ¿Qué pasa si la transición de negocio falla (por ejemplo, la orden no se pudo confirmar) después de haberse intentado? No debe quedar un evento de auditoría afirmando una transición que en realidad no se completó.
- ¿Qué pasa cuando la cancelación de una orden causa pérdida de inventario (ítems que la cocina ya había empezado a preparar)? El evento de cancelación debe permitir distinguir este caso del de una reversión completa sin pérdida.
- ¿Qué pasa con una orden creada manualmente por un cajero, sin origen QR y sin ningún comensal involucrado? Los eventos de esa orden no deben requerir un actor tipo comensal en ningún momento de su historial.
- ¿Qué pasa cuando un comprobante de transferencia es rechazado (no aprobado)? El evento debe registrar el motivo de rechazo sin exponer el contenido del comprobante en el destino externo.
- ¿Qué pasa si dos eventos de la misma orden ocurren casi al mismo tiempo (p. ej. el sistema confirma el pago y la orden en la misma operación)? El orden cronológico de los eventos debe seguir siendo reconstruible de forma inequívoca.
- ¿Qué pasa si falla o se demora el envío de un evento de auditoría hacia Sentry (p. ej. la herramienta externa no responde) justo cuando la transición de negocio correspondiente sí se completó? La transición de negocio no se bloquea ni se revierte por esa falla; el envío se reintenta de forma separada, sin afectar al actor que ejecutó la acción, y sin garantía de que el evento llegue a existir si el reintento también falla.
- ¿Qué pasa si un administrador necesita investigar una orden cuyos eventos ya superaron la ventana de retención de Sentry (por ejemplo, una disputa que surge meses después)? El log de auditoría de este spec ya no está disponible para ese caso; queda fuera del alcance de esta capacidad, dado que no existe almacenamiento propio más allá de Sentry.
- ¿Qué pasa cuando el staff cobra y envía a cocina, en un solo paso, una comanda que estaba en espera de pago (sin pasar por un intento de pago pendiente ni por una confirmación manual separada)? Este camino, identificado durante la implementación (no estaba en el mapeo original del Paso 1), también involucra dinero cobrado — la misma clase de riesgo que motiva este spec — así que también se audita: ver FR-001 y FR-014.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE registrar un evento de auditoría por cada transición del ciclo de vida de una orden: creación (vía QR o manual por staff), confirmación (manual o automática por efecto del pago), registro de un intento de pago, confirmación de pago en efectivo, aprobación de comprobante de transferencia, rechazo de comprobante de transferencia, cancelación, y cobro-y-envío en un solo paso de una comanda en espera de pago (ver FR-014).
- **FR-002**: Cada evento de auditoría DEBE incluir el identificador de la orden a la que pertenece, de forma que todos los eventos de una misma orden puedan recuperarse juntos aunque hayan ocurrido en distintas solicitudes, distintos momentos y con distintos actores.
- **FR-003**: Cada evento de auditoría DEBE identificar explícitamente al actor que lo originó como exactamente uno de: comensal anónimo (referenciado por su identificador de sesión de mesa, nunca por su nombre), miembro del staff (referenciado por su identificador de usuario y su rol), o sistema (para transiciones automáticas no atribuibles directamente a una acción humana). El actor nunca se modela como una referencia directa a la tabla de usuarios, dado que el comensal no tiene una cuenta de usuario.
- **FR-004**: Cada evento de auditoría DEBE registrar explícitamente a qué negocio/tenant pertenece en el momento en que ocurre, sin depender de que ese dato se pueda inferir después a partir de otra información de la solicitud.
- **FR-005**: El nombre del comensal y el contenido o URL del comprobante de pago NO DEBEN aparecer en texto plano en los datos que el sistema envía a la herramienta externa de monitoreo de errores; cuando esa información deba referenciarse en un evento de auditoría, DEBE transformarse mediante un mecanismo no reversible (p. ej. un hash) antes de enviarse — nunca en su forma original.
- **FR-006**: La herramienta externa de monitoreo de errores (Sentry) DEBE ser el único destino y mecanismo de registro de los eventos de auditoría; este spec NO introduce una tabla ni almacenamiento interno dedicado para este propósito. Esta decisión debe quedar documentada explícitamente como parte de este trabajo, no dejarse implícita, incluyendo la consecuencia de que el historial de auditoría queda sujeto a la ventana de retención de esa herramienta (ver FR-013).
- **FR-007**: El registro informativo y de errores operativos que ya existe hoy hacia la herramienta externa de monitoreo (vía identificador de solicitud) DEBE seguir funcionando sin cambios. Los eventos de auditoría descritos en este spec viajan al mismo destino, pero DEBEN quedar identificables como una categoría propia y distinguible dentro de esos datos (p. ej. un tipo/nombre de evento reconocible), sin mezclarse de forma indistinguible con el registro operativo existente.
- **FR-008**: El staff con rol administrativo DEBE poder ubicar el historial de eventos de auditoría de una orden específica directamente en el panel de la herramienta externa de monitoreo, filtrando por el identificador de la orden. Este spec NO introduce una API ni pantalla propia de consulta; el acceso a ese panel externo queda gobernado por los controles de acceso de esa herramienta, no por un control de acceso propio de este sistema.
- **FR-009**: El evento de cancelación DEBE registrar si fue iniciado por el comensal o por el staff, y DEBE permitir distinguir si la cancelación produjo pérdida de inventario (ítems ya en preparación) frente a una reversión completa.
- **FR-010**: Un evento de auditoría solo DEBE emitirse cuando la transición de negocio correspondiente se completó efectivamente; no debe emitirse un evento afirmando una transición que falló o se revirtió.
- **FR-011**: El sistema NO DEBE bloquear ni revertir una transición de negocio de la orden (creación, confirmación, pago, cancelación) debido a una falla al enviar su evento de auditoría hacia la herramienta externa de monitoreo; ese envío se maneja de forma desacoplada de la transición (por ejemplo, mediante reintento), sin afectar la experiencia del actor que ejecuta la acción.
- **FR-012**: La transformación no reversible aplicada al nombre del comensal y al comprobante de pago (FR-005) DEBE ser consistente para un mismo valor de entrada, de forma que un administrador pueda reconocer que dos eventos distintos se refieren al mismo comensal o al mismo comprobante sin necesidad de conocer el valor original.
- **FR-013**: El sistema NO DEBE prometer, para el historial de auditoría, una disponibilidad mayor a la ventana de retención vigente del plan contratado de la herramienta externa de monitoreo (7 a 30 días según el plan); este spec no introduce ningún mecanismo adicional de exportación, respaldo o almacenamiento de más largo plazo para ese historial.
- **FR-014**: Cuando el staff cobra y envía a cocina una comanda en espera de pago en una sola operación (sin un intento de pago pendiente previo ni una confirmación manual separada — la orden pasa de `recibida` a `pagada` en un único paso), el sistema DEBE registrar tanto la confirmación de la orden (actor `sistema`, igual que cualquier otra confirmación automática por efecto del pago) como el cobro en sí, con el o los métodos de pago usados y el monto total — sin necesidad de que ese cobro pase por el flujo de intento de pago pendiente que sí usan los demás caminos de pago.

### Key Entities *(include if feature involves data)*

- **Evento de auditoría de orden**: entrada de log estructurada, enviada a la herramienta externa de monitoreo, que representa un hecho ocurrido en el ciclo de vida de una orden. Incluye el identificador de la orden, el negocio/tenant al que pertenece, el tipo de evento (creación, confirmación, intento de pago, confirmación de pago, aprobación/rechazo de comprobante, cancelación), el actor que lo originó, el momento en que ocurrió, y los detalles propios del evento — con cualquier dato sensible transformado de forma no reversible, nunca en texto plano. No existe una copia persistida de este evento fuera de la herramienta externa de monitoreo.
- **Actor**: quien origina un evento de auditoría. Es exactamente uno de: comensal anónimo (identificador de sesión de mesa), miembro del staff (identificador de usuario + rol), o sistema (transición automática).
- **Orden**: la orden de cliente cuyo ciclo de vida se audita; entidad ya existente en el sistema, no introducida por este spec.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Para cualquier orden cuyos eventos sigan dentro de la ventana de retención vigente, el staff con rol administrativo puede ubicar, filtrando por el identificador de la orden en el panel de la herramienta externa de monitoreo, el 100% de los eventos de su ciclo de vida (creación, confirmación, pago, cancelación), en orden cronológico correcto.
- **SC-002**: El 100% de los eventos de auditoría generados durante pruebas de aceptación tienen un actor identificado sin ambigüedad (tipo + identificador), sin ningún evento con actor nulo o genérico.
- **SC-003**: El 0% de los eventos de auditoría enviados hacia la herramienta externa de monitoreo contienen el nombre del comensal o el comprobante de pago en texto plano (sin transformar).
- **SC-004**: El historial de auditoría de una orden está disponible durante la ventana de retención vigente del plan contratado de la herramienta externa de monitoreo (7 a 30 días); este spec no promete disponibilidad más allá de esa ventana.
- **SC-005**: Ante una orden cancelada cuyos eventos sigan dentro de la ventana de retención, el administrador puede determinar quién la canceló (comensal o staff específico) y si hubo pérdida de inventario, directamente desde el panel de la herramienta externa de monitoreo, sin tener que cruzar información con otros sistemas.

## Assumptions

- El alcance de este spec es el ciclo de vida de la orden individual (creación, confirmación, intentos y confirmación de pago, cancelación). El cierre de sesión de mesa y la división de cuenta por comensal (facturación dividida) quedan explícitamente fuera de alcance, aunque el sistema los soporte hoy — se decidió así porque ese cierre pertenece a un flujo distinto (el de la sesión de mesa, no el de una orden individual) y podría tratarse como una extensión futura.
- No se audita un "movimiento de caja" como efecto colateral separado del pago, porque el sistema no genera hoy ese movimiento de forma automática por orden (la caja se concilia al cierre de turno). Se audita la confirmación del pago en sí misma como el hecho relevante.
- Las transiciones de estado a nivel de ítem individual dentro de una orden (p. ej. el seguimiento de cocina de cada plato) quedan fuera de alcance de este spec; solo se audita el estado de la orden como conjunto.
- Se asume que ya existe un mecanismo para conocer, en el momento de cada transición, el negocio/tenant activo y — cuando aplica — el usuario de staff autenticado; este spec no introduce esos mecanismos, solo exige que su valor quede capturado en cada evento.
- Se asume que el volumen de eventos de auditoría es proporcional al volumen de órdenes del negocio, sin picos que requieran un tratamiento de escala distinto al del resto del sistema.
- El envío del evento de auditoría hacia la herramienta externa de monitoreo es desacoplado de la transición de negocio que audita (ver Clarifications, sesión 2026-09-03): una falla al enviarlo no bloquea ni revierte la operación de la orden. Esto prioriza no afectar la operación diaria del negocio sobre una garantía estricta de "cero eventos perdidos"; se asume que existe un mecanismo de reintento que minimiza — sin garantizar en el 100% de los casos de falla — la pérdida del evento.
- Decisión de alcance (ver Clarifications, sesión 2026-09-03): este spec NO introduce una tabla ni almacenamiento interno propio para el log de auditoría. La herramienta externa de monitoreo (Sentry) es el único destino. Esto implica renunciar a una retención propia de largo plazo a cambio de no construir ni mantener infraestructura de almacenamiento adicional; el historial de auditoría queda acotado a la ventana de retención del plan contratado (7 a 30 días).
- El nombre del comensal y el comprobante de pago, cuando se referencian en un evento de auditoría, se envían transformados de forma no reversible (p. ej. hash), nunca en texto plano — dado que Sentry es un proveedor externo y es el único lugar donde ese evento queda registrado, no existe ya un registro interno alternativo donde conservar el valor original sin exponerlo.
- Una investigación o disputa que involucre eventos ya fuera de la ventana de retención de la herramienta externa de monitoreo no puede resolverse con este mecanismo; se asume que ese escenario, si llega a ser relevante para el negocio, se atiende como una necesidad futura distinta (por ejemplo, con un almacenamiento propio), no como parte de este spec.
