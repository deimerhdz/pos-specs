# Feature Specification: Catálogo de Métodos de Pago Administrado por el Super Admin

**Feature Branch**: `032-catalogo-metodos-pago`

**Created**: 2026-08-24

**Status**: Draft

**Input**: User description: "Catálogo de métodos de pago administrado por el Super Admin: hoy cada tenant define y gestiona su propio conjunto de métodos de pago sin catálogo central. Se busca que el Super Admin administre un catálogo de métodos de pago a nivel de plataforma, que los tenants solo activen métodos de ese catálogo y completen sus propios datos de integración (número de cuenta, código QR, número de celular, etc.), sin poder crear métodos nuevos por fuera del catálogo. Incluye migración de la configuración actual de los tenants (efectivo, Nequi, Bancolombia) al nuevo modelo."

## Clarifications

### Session 2026-08-24

- Q: Cuando el cajero abre la pantalla de cobro y selecciona un método como Nequi o Transferencia Bancolombia, ¿debe ver los datos de integración del tenant (número de cuenta, celular, código QR) para poder mostrárselos al cliente, o la pantalla solo confirma el nombre del método sin mostrar esos datos? → A: El cajero solo ve el nombre del método; los datos de integración son solo administrativos (visibles únicamente para el Tenant Admin).
- Q: Si hoy un tenant tiene configurado un método de pago propio que no corresponde a ninguno de los tres conocidos (efectivo, Nequi, Bancolombia), ¿qué debe pasar con ese método durante la migración al nuevo catálogo? → A: Antes de ejecutar la migración, el Super Admin revisa los métodos personalizados existentes en los tenants y los agrega al catálogo cuando sean válidos para el negocio; la migración los mapea igual que a efectivo/Nequi/Bancolombia, preservando la configuración del tenant.
- Q: Para que un campo de integración obligatorio (como el número de celular de Nequi a 10 dígitos) cuente como "diligenciado", ¿el sistema debe validar que el valor cumpla un formato específico, o basta con que el tenant haya escrito algo en el campo? → A: Cada campo de integración del catálogo define un formato esperado (ej. numérico, longitud exacta) y el sistema lo valida antes de marcar el campo —y la configuración— como completo.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El Super Admin administra el catálogo de métodos de pago de la plataforma (Priority: P1)

Como Super Admin, quiero crear, editar y activar/desactivar métodos de pago en un catálogo único de la plataforma —indicando qué datos de integración requiere cada uno—, para tener control centralizado sobre qué métodos de pago pueden existir en el sistema y garantizar consistencia entre todos los tenants.

**Why this priority**: Es la base de toda la funcionalidad. Sin un catálogo administrado por el Super Admin no existe nada que un tenant pueda activar; es la pieza que separa "qué tipo de método de pago es" de "cómo lo usa un tenant específico".

**Independent Test**: Puede probarse por completo sin que ningún tenant participe: el Super Admin crea un método de pago (ej. "Daviplata") definiendo sus campos de integración requeridos y opcionales, lo ve listado en el catálogo, lo edita, y puede desactivarlo. Todo esto es verificable en una pantalla de administración de plataforma, sin tocar código.

**Acceptance Scenarios**:

1. **Given** el Super Admin está en la administración del catálogo de métodos de pago, **When** crea un nuevo método de pago indicando su nombre y los campos de integración que requiere, **Then** el método queda registrado en el catálogo con estado activo y disponible para que cualquier tenant lo active.
2. **Given** un método de pago existe en el catálogo, **When** el Super Admin edita su nombre o sus campos de integración, **Then** los cambios se reflejan en el catálogo para futuras activaciones, sin alterar las configuraciones que los tenants ya completaron previamente.
3. **Given** un método de pago del catálogo está activo, **When** el Super Admin lo desactiva a nivel de plataforma, **Then** ningún tenant nuevo puede activarlo, y los tenants que ya lo tenían activo dejan de poder usarlo para cobros nuevos a partir de ese momento.
4. **Given** un método de pago fue desactivado a nivel de plataforma y tiene ventas históricas asociadas en uno o más tenants, **When** se consulta el histórico o los reportes de esas ventas, **Then** la información aparece sin cambios, tal como se registró en su momento.

---

### User Story 2 - El Tenant Admin activa y configura métodos de pago para su negocio (Priority: P2)

Como Tenant Admin, quiero ver el catálogo de métodos de pago disponibles en la plataforma, activar los que uso en mi negocio, completar sus datos de integración (número de cuenta, número de celular, código QR, etc.) y desactivar los que ya no uso, para que mi negocio cobre únicamente con los métodos que realmente ofrezco.

**Why this priority**: Es el flujo que le da valor directo al tenant y depende de que el catálogo (Historia 1) ya exista. Sin esta historia, el catálogo del Super Admin no tiene ningún efecto operativo para el negocio.

**Independent Test**: Con un catálogo ya poblado (ej. Efectivo, Nequi, Transferencia Bancolombia), un Tenant Admin puede activar "Nequi", completar el número de celular requerido, y verificar que la activación queda registrada con estado "configurado", sin necesitar que ningún cajero intervenga. También puede desactivar un método que había activado.

**Acceptance Scenarios**:

1. **Given** el catálogo tiene métodos de pago activos a nivel de plataforma, **When** el Tenant Admin consulta el catálogo desde su panel, **Then** ve únicamente los métodos que el Super Admin tiene activos (no ve métodos desactivados por el Super Admin).
2. **Given** el Tenant Admin activa un método de pago del catálogo que requiere datos de integración (ej. Nequi), **When** completa todos los campos requeridos (ej. número de celular a 10 dígitos), **Then** el método queda marcado como activo y completamente configurado para su tenant.
3. **Given** el Tenant Admin activa un método de pago que requiere datos de integración, **When** dejar campos requeridos sin completar, **Then** el método queda activo pero marcado como incompleto, y el sistema se lo indica claramente al Tenant Admin.
4. **Given** un método de pago no requiere datos de integración adicionales (ej. Efectivo), **When** el Tenant Admin lo activa, **Then** queda disponible de inmediato sin pedir ningún campo adicional.
5. **Given** un método de pago está activo y configurado para el tenant, **When** el Tenant Admin lo desactiva, **Then** deja de estar disponible para cobros nuevos en ese tenant, sin afectar las ventas ya registradas con ese método.

---

### User Story 3 - El Cajero cobra usando solo métodos de pago completamente disponibles (Priority: P3)

Como Cajero, quiero que la pantalla de cobro me muestre únicamente los métodos de pago que mi tenant activó y configuró por completo —identificados por su nombre, sin mostrar los datos de integración— para confirmar el cobro correcto sin confundirme con métodos incompletos o no habilitados.

**Why this priority**: Es el punto donde el catálogo y la configuración del tenant se materializan en la operación diaria, pero depende por completo de que las Historias 1 y 2 ya funcionen. El flujo del cajero en sí (confirmar un cobro) no cambia; solo cambia el origen de la lista de métodos que ve.

**Independent Test**: Con un tenant que tiene "Efectivo" y "Nequi" activos y completos, y "Transferencia Bancolombia" activado pero incompleto, un cajero abre la pantalla de cobro y verifica que solo ve "Efectivo" y "Nequi" como opciones, sin necesitar revisar configuración alguna.

**Acceptance Scenarios**:

1. **Given** un tenant tiene varios métodos de pago activados, algunos completamente configurados y otros no, **When** el cajero abre la pantalla de cobro, **Then** solo ve los métodos activados y completamente configurados.
2. **Given** un tenant activa y termina de configurar un nuevo método de pago, **When** el cajero vuelve a abrir la pantalla de cobro, **Then** el nuevo método aparece disponible de inmediato, sin pasos adicionales.
3. **Given** el Super Admin desactiva un método de pago a nivel de plataforma que el tenant tenía activo, **When** el cajero abre la pantalla de cobro después de la desactivación, **Then** ese método ya no aparece como opción.

---

### Edge Cases

- Un tenant activa un método de pago pero no termina de llenar los datos de integración requeridos: el método no debe aparecer en la pantalla de cobro de caja hasta quedar completo, y el Tenant Admin debe poder identificar fácilmente cuáles de sus métodos activados están incompletos.
- El Super Admin desactiva un método de pago que tiene ventas históricas asociadas: el histórico de ventas y los reportes no deben verse afectados ni deben mostrar el método como "eliminado" o "inválido" retroactivamente.
- El Super Admin desactiva un método de pago a nivel de plataforma que un tenant tiene activo y configurado: el tenant deja de verlo en caja, pero conserva los datos de integración que ya había capturado (por si el Super Admin lo reactiva más adelante, el tenant no tiene que volver a capturarlos).
- El Super Admin edita los campos de integración requeridos de un método de pago después de que algunos tenants ya lo activaron con la definición anterior: las configuraciones ya completadas por esos tenants no deben invalidarse automáticamente por el cambio.
- **Migración de datos existentes**: los métodos de pago que hoy ya usan los tenants (efectivo, Nequi, Bancolombia) deben pasar al nuevo modelo del catálogo sin que ningún tenant pierda su configuración actual ni tenga que volver a capturarla desde cero; después de la migración, cada tenant debe seguir viendo en caja exactamente los mismos métodos que veía antes.
- **Métodos de pago personalizados fuera de los tres conocidos**: si algún tenant tiene hoy un método de pago propio que no corresponde a efectivo, Nequi o Bancolombia, el Super Admin debe revisarlo y agregarlo al catálogo antes de ejecutar la migración, para que también se mapee preservando la configuración del tenant; ningún método personalizado existente debe perderse ni quedar sin poder migrarse por no tener equivalente previo en el catálogo.
- Un tenant intenta registrar o solicitar un método de pago que no existe en el catálogo del Super Admin: el sistema no debe permitirlo; solo puede activar métodos ya existentes en el catálogo.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST permitir al Super Admin crear métodos de pago en un catálogo único de la plataforma, indicando su nombre y qué campos de datos de integración requiere.
- **FR-002**: El sistema MUST permitir al Super Admin editar los métodos de pago existentes en el catálogo (nombre y campos de integración requeridos/opcionales).
- **FR-003**: El sistema MUST permitir al Super Admin activar y desactivar métodos de pago a nivel de plataforma.
- **FR-004**: Cada método de pago del catálogo MUST definir qué campo o campos de integración necesita un tenant para poder usarlo (por ejemplo: sin campos adicionales, número de celular, número de cuenta, tipo de cuenta, imagen de código QR), cuáles de esos campos son obligatorios y cuáles opcionales, y el formato esperado de cada campo (ej. numérico, longitud exacta) cuando aplique.
- **FR-005**: El sistema MUST permitir al Tenant Admin consultar el catálogo de métodos de pago que el Super Admin tiene activos a nivel de plataforma.
- **FR-006**: El sistema MUST ocultar del Tenant Admin los métodos de pago que el Super Admin desactivó a nivel de plataforma, salvo cuando el propio tenant ya los tenía activados previamente (en cuyo caso se le debe indicar que ya no están disponibles para cobrar).
- **FR-007**: El sistema MUST permitir al Tenant Admin activar, para su propio tenant, cualquier método de pago que esté activo en el catálogo de la plataforma.
- **FR-008**: El sistema MUST permitir al Tenant Admin completar los valores de los campos de integración requeridos por el método de pago que activó.
- **FR-009**: El sistema MUST marcar la activación de un método de pago por parte de un tenant como "incompleta" mientras falten campos obligatorios por diligenciar o alguno de los diligenciados no cumpla el formato esperado definido por el catálogo (ej. longitud, tipo de dato), y como "completa" solo una vez todos los campos obligatorios estén diligenciados y validados.
- **FR-010**: El sistema MUST permitir al Tenant Admin desactivar, para su propio tenant, un método de pago que previamente había activado.
- **FR-011**: El sistema MUST impedir que un tenant cree, registre o solicite un método de pago que no exista en el catálogo administrado por el Super Admin.
- **FR-012**: La pantalla de cobro en caja MUST mostrar únicamente los métodos de pago que el tenant tiene activados Y completamente configurados, y que además siguen activos en el catálogo de la plataforma.
- **FR-012a**: La pantalla de cobro en caja MUST mostrar cada método de pago disponible únicamente por su nombre, sin exponer al cajero los datos de integración capturados (número de cuenta, celular, imagen de código QR); esos datos son de uso administrativo y solo deben ser visibles para el Tenant Admin.
- **FR-013**: Cuando el Super Admin desactiva un método de pago a nivel de plataforma, el sistema MUST dejar de mostrarlo como disponible para cobros nuevos en todos los tenants que lo tenían activo, sin eliminar ni modificar las ventas ya registradas con ese método.
- **FR-014**: El sistema MUST preservar sin cambios el método de pago registrado en ventas históricas, independientemente de cambios posteriores al catálogo (edición, desactivación) o a la configuración del tenant (desactivación, eliminación).
- **FR-015**: El sistema MUST migrar los métodos de pago que los tenants usan actualmente (efectivo, Nequi, Bancolombia, y cualquier método personalizado que el Super Admin haya incorporado previamente al catálogo) al nuevo modelo de catálogo, preservando la configuración de integración que cada tenant ya tenía capturada, sin requerir que el tenant vuelva a capturarla.
- **FR-015a**: Antes de ejecutar la migración, el Super Admin MUST poder revisar los métodos de pago personalizados que los tenants tengan configurados fuera de efectivo, Nequi y Bancolombia, y crearlos en el catálogo cuando correspondan a métodos válidos para el negocio, de forma que la migración pueda mapearlos sin pérdida de configuración.
- **FR-016**: Después de la migración, cada tenant MUST seguir viendo en la pantalla de cobro exactamente los métodos de pago que tenía disponibles antes del cambio, sin interrupciones operativas.
- **FR-017**: El sistema MUST permitir a un tenant tener, como máximo, una configuración activa por método de pago del catálogo (ej. una sola cuenta Nequi activa a la vez); para usar datos de integración distintos para el mismo método, el tenant debe primero desactivar la configuración activa antes de activar una nueva.

### Key Entities

- **Método de Pago (Catálogo)**: Representa un tipo de método de pago administrado por el Super Admin a nivel de plataforma (ej. Efectivo, Nequi, Transferencia Bancolombia, Daviplata). Define su nombre, su estado (activo/inactivo a nivel plataforma) y qué campos de integración requiere un tenant para poder usarlo, incluyendo cuáles son obligatorios, cuáles opcionales, y el formato esperado de cada uno (ej. numérico, longitud exacta) cuando aplique.
- **Configuración de Método de Pago por Tenant**: Representa la activación de un método del catálogo por parte de un tenant específico. Incluye a qué método del catálogo hace referencia, los valores capturados para los campos de integración requeridos (ej. número de celular, número de cuenta, tipo de cuenta, imagen de código QR), su estado de completitud (completa/incompleta) y su estado de disponibilidad para ese tenant (activa/inactiva).
- **Venta / Registro de Cobro Histórico**: Registro existente de una venta que referencia el método de pago usado al momento del cobro. Su relación con el método de pago (catálogo y configuración del tenant) es únicamente de referencia histórica y no debe verse afectada por cambios posteriores al catálogo o a la configuración del tenant.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El Super Admin puede crear un nuevo método de pago en el catálogo y verificar, sin intervención técnica, que queda disponible para activación en un tenant de prueba en menos de 5 minutos.
- **SC-002**: Un Tenant Admin puede activar y terminar de configurar un método de pago (ej. completar el número de celular de Nequi) en menos de 2 minutos, y el método aparece disponible en la pantalla de cobro de caja de inmediato tras completarse.
- **SC-003**: El 100% de las ventas históricas conservan el método de pago exactamente como se registró en su momento, incluso después de que ese método sea editado o desactivado en el catálogo o en la configuración del tenant.
- **SC-004**: El 100% de los tenants que ya usaban efectivo, Nequi o Bancolombia antes del cambio conservan esos métodos disponibles en caja después de la migración, sin necesidad de volver a capturar ningún dato.
- **SC-005**: El 0% de los métodos de pago activados pero incompletos aparece en la pantalla de cobro de caja.
- **SC-006**: Al desactivar un método de pago a nivel de plataforma, el 100% de los tenants que lo tenían activo deja de verlo disponible en caja para cobros nuevos, sin afectar el histórico de ventas ya registradas.

## Assumptions

- Cada método de pago del catálogo puede definir cero, uno o varios campos de integración, y cada campo puede marcarse como obligatorio u opcional (ej. Nequi requiere número de celular obligatorio, con imagen QR opcional).
- El Super Admin no puede eliminar permanentemente un método de pago del catálogo una vez creado; solo puede activarlo o desactivarlo, para no invalidar las referencias históricas de ventas ni las configuraciones ya hechas por los tenants.
- Cuando el Super Admin desactiva un método de pago a nivel de plataforma, la configuración de integración que cada tenant había capturado se conserva (no se borra), de forma que si el Super Admin lo reactiva más adelante, los tenants que ya lo tenían configurado no deban volver a capturar sus datos.
- Cuando un Tenant Admin desactiva un método de pago para su propio tenant, su configuración de integración se conserva de la misma forma, permitiendo reactivarlo sin volver a capturar los datos.
- El alcance de esta funcionalidad no incluye integración real (API/webhook) con pasarelas o billeteras; el cobro sigue siendo una confirmación manual por parte del cajero, igual que hoy.
- La migración de los métodos de pago actuales (efectivo, Nequi, Bancolombia) es un proceso de una sola vez, ejecutado como parte de la puesta en marcha de esta funcionalidad, y no un flujo recurrente ni disparado por el usuario.
- Los roles Super Admin, Tenant Admin y Cajero corresponden a los roles y permisos ya existentes en el sistema; esta funcionalidad no introduce nuevos roles.
- Un tenant solo puede tener una configuración activa por método de pago del catálogo a la vez (decisión confirmada con el negocio); para cambiar los datos de integración de un método ya activo, el tenant debe desactivar la configuración actual antes de activar una nueva.
