# Feature Specification: Estandarización de Íconos en el Panel de Administración

**Feature Branch**: `082-estandarizacion-iconos-admin`

**Created**: 2026-09-16

**Status**: Draft

**Input**: User description: "necesito estandarizar los iconos usados en angular, de tal forma que sea un componente reutilizable para esta version enfocate en cambia solamente los iconos que hoy se muestra en todo el panel de administracion, no puede quedar ningun icono generado por ia en formato svg ni usar algun icono relacionado con heladeria de los que hoy se muestran, la libreria que vas a instalar que hoy no esta disponible es material icons"

## Clarifications

### Session 2026-09-16

- Q: Should icon-only elements (action buttons, status indicators with no visible label) expose an accessible text description for screen readers? → A: Sí, todo ícono usado sin texto visible debe tener una descripción accesible equivalente.
- Q: Does the Material Icons library need to keep working when the admin panel has no internet connection, or is it acceptable for icons to depend on an external connection (e.g., Google Fonts CDN)? → A: Debe funcionar sin conexión a internet, empaquetada con la aplicación.

### Session 2026-09-16 (durante `/speckit-plan`)

- Q: Al investigar el código real (`pos-heladeria`) para planear la implementación, se descubrió que el componente SVG artesanal (`IconComponent`/`app-icon`, `src/app/shared/icon/icon.component.ts`) no es exclusivo del panel de administración: también lo usan hoy `public-menu.component.ts` y los cuatro pasos de `checkout/*-step.component.ts`, que son parte del flujo público de menú QR declarado fuera de alcance (FR-010). Eliminar el archivo del componente, tal como pedía la redacción original de FR-005, rompería ese flujo público que esta misma spec prohíbe modificar. ¿Cómo se ajusta FR-005 ante este hallazgo? → A: El componente `IconComponent`/`app-icon` NO se elimina del código; su archivo permanece intacto y sigue sirviendo, sin cambios, al flujo público de menú QR. Lo que se elimina es únicamente su uso dentro de las pantallas del panel de administración — "eliminado" en el resto de esta spec (FR-005, SC-002) significa "sin referencias activas dentro del panel de administración", no "borrado del repositorio".
- Q: También se descubrió que algunos componentes con emojis usados como íconos (`cart.component.ts`, `product-select.component.ts`, `payment-attempt-review-panel.component.ts`, `pos-catalog-drawer.component.ts`) son compartidos: los usa tanto `manual-order-page.component.ts` (panel de administración, dentro de alcance) como el menú público del cliente (fuera de alcance). No es técnicamente posible migrar el ícono para el personal sin que el mismo cambio visual aparezca también en la pantalla pública que reutiliza el mismo componente. ¿Se migran estos íconos compartidos o se dejan sin tocar para no afectar el flujo público? → A: Sí se migran. FR-010 prohíbe modificar el *comportamiento* del flujo público de menú QR (su lógica, sus rutas, sus reglas de negocio), no impide que un componente visual compartido reciba el mismo ícono estandarizado en ambos contextos como efecto secundario de una migración que sí es obligatoria en el panel de administración (FR-004). Bifurcar el componente compartido en dos variantes de ícono (una para personal, otra para clientes) sería la alternativa más compleja e innecesaria (Principio IX de la constitución) para un cambio puramente visual sin impacto funcional.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Íconos consistentes en todo el panel de administración (Priority: P1)

Como usuaria o usuario del panel de administración (personal del negocio: administradores, cajeros, meseros, encargados de inventario), quiero ver íconos consistentes y de aspecto profesional en cada módulo (panel principal, caja, inventario, mesas/terminal, usuarios, promociones), en lugar de la mezcla actual de emojis y símbolos SVG hechos a mano, para que la interfaz se vea uniforme y confiable sin importar en qué pantalla me encuentre.

**Why this priority**: Es el problema central que motiva la funcionalidad: hoy conviven dos sistemas de íconos distintos (un componente SVG artesanal y emojis sueltos) y ninguno de los dos es consistente en todo el panel. Sin resolver esto, el resto de la funcionalidad no tiene sentido.

**Independent Test**: Se puede verificar navegando cada módulo del panel de administración y confirmando que todos sus íconos provienen del mismo componente de íconos y de la misma librería visual, sin emojis ni SVG artesanales sueltos.

**Acceptance Scenarios**:

1. **Given** que estoy autenticada/o en el panel de administración, **When** navego entre el panel principal, caja, inventario, mesas/terminal, usuarios y promociones, **Then** todos los íconos de navegación, botones de acción, indicadores de estado y tarjetas de estadísticas se ven visualmente consistentes entre sí (mismo estilo de trazo, mismo componente).
2. **Given** una pantalla del panel de administración que hoy muestra un ícono mediante el componente SVG artesanal (`app-icon`) o mediante un emoji, **When** se completa esta funcionalidad, **Then** esa misma pantalla muestra el ícono equivalente a través del nuevo componente reutilizable respaldado por Material Icons.

---

### User Story 2 - Ningún ícono con temática de heladería (Priority: P1)

Como usuaria o usuario del panel de administración de un negocio que puede no ser una heladería, quiero que ningún ícono del sistema haga referencia visual a "heladería" (por ejemplo, el cono de helado usado hoy como logo por defecto o en la sección de productos), para que la marca del sistema sea neutra y no confunda a negocios de otro tipo.

**Why this priority**: Es una restricción explícita y no negociable del pedido: ningún ícono de heladería puede permanecer. Tiene el mismo nivel de urgencia que la consistencia visual porque afecta directamente la percepción de marca frente a clientes del negocio (tenants) que no venden helados.

**Independent Test**: Se puede verificar revisando el logo por defecto del panel lateral y la tarjeta/acceso rápido de "Productos" del panel principal (los dos puntos identificados hoy con el emoji de cono de helado 🍦) y confirmando que fueron reemplazados por un ícono neutro equivalente en significado.

**Acceptance Scenarios**:

1. **Given** un tenant que no tiene un logo propio configurado, **When** se muestra la marca por defecto en el panel lateral, **Then** se usa un ícono neutro de negocio/tienda en lugar del cono de helado.
2. **Given** el panel principal de administración, **When** se muestra el acceso o tarjeta correspondiente a "Productos", **Then** el ícono mostrado es neutro y no hace referencia a helados.
3. **Given** cualquier pantalla del panel de administración, **When** se revisa exhaustivamente en busca de íconos con temática de heladería, **Then** no se encuentra ninguno.

---

### User Story 3 - Componente de ícono reutilizable respaldado por Material Icons (Priority: P2)

Como persona que mantiene la aplicación, quiero un único componente reutilizable que renderice cualquier ícono a partir de la librería Material Icons (hoy no instalada en el proyecto), en lugar de depender de un componente SVG hecho a mano con rutas de ícono copiadas manualmente, para poder agregar o cambiar íconos de forma simple y consistente en el futuro.

**Why this priority**: Es el medio técnico que permite sostener en el tiempo lo logrado en las historias 1 y 2; sin un componente reutilizable respaldado por una librería estándar, cualquier consistencia lograda hoy se volvería a fragmentar con el próximo ícono que alguien agregue a mano.

**Independent Test**: Se puede verificar confirmando que existe un solo componente de ícono en el código, que ese componente puede mostrar cualquier ícono de Material Icons a partir de un nombre, y que el componente SVG artesanal anterior (con sus rutas de ícono escritas a mano) ya no se usa en ninguna pantalla del panel de administración.

**Acceptance Scenarios**:

1. **Given** que un desarrollador necesita mostrar un ícono nuevo en el panel de administración, **When** usa el componente reutilizable de íconos indicando el nombre del ícono de Material Icons, **Then** el ícono se renderiza sin necesidad de escribir o pegar un SVG a mano.
2. **Given** el estado final de esta funcionalidad, **When** se revisa el código del panel de administración, **Then** el componente SVG artesanal anterior (y sus rutas de ícono codificadas a mano) ya no tiene referencias activas en las pantallas del panel de administración.

---

### Edge Cases

- ¿Qué pasa con un ícono de estado que hoy comunica su significado por color además de por símbolo (por ejemplo, un punto rojo o verde, o una marca de check/equis/advertencia)? El nuevo ícono de Material Icons equivalente debe seguir comunicando el mismo estado (combinando ícono + color), no solo el símbolo.
- ¿Qué pasa con un emoji que aparece dentro de un texto o mensaje (por ejemplo, dentro de la frase de una notificación), en lugar de usarse como ícono independiente de navegación, botón o tarjeta? Ese uso se considera contenido de texto, no un ícono, y queda fuera del alcance de esta funcionalidad — salvo que sea un emoji con temática de heladería, caso en el cual igual debe eliminarse por la restricción explícita de la Historia 2.
- ¿Qué pasa con el menú digital público que el cliente ve al escanear el código QR de la mesa (navegación de menú, carrito, pago), dado que ese flujo no requiere autenticación y no forma parte del panel de administración? Ese flujo queda fuera del alcance de esta funcionalidad (ver Assumptions) y no debe verse alterado por este cambio.
- ¿Qué pasa si un ícono actual (emoji o SVG artesanal) no tiene un equivalente exacto y obvio dentro del catálogo de Material Icons? Debe elegirse el ícono de Material Icons semánticamente más cercano, de forma que la acción, estado o entidad representada siga siendo reconocible; ninguna pantalla del panel de administración puede quedar sin ícono o con un ícono vacío/roto.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE incorporar la librería Material Icons como fuente de íconos del panel de administración, dado que hoy no está disponible en el proyecto. La librería DEBE quedar empaquetada con la aplicación (no depender de una conexión externa en tiempo de ejecución), de forma que los íconos sigan mostrándose correctamente cuando el panel opera sin conexión a internet, igual que ya ocurre hoy con el modo híbrido del terminal de mesas.
- **FR-002**: El sistema DEBE proveer un único componente de Angular reutilizable, capaz de renderizar cualquier ícono de Material Icons a partir de un nombre/identificador, para uso en todo el panel de administración. Este componente DEBE aceptar además una descripción accesible (texto equivalente para lectores de pantalla) asociada al significado del ícono.
- **FR-003**: El sistema DEBE reemplazar, en cada pantalla del panel de administración, todo ícono actualmente mostrado mediante el componente SVG artesanal existente por el ícono equivalente del nuevo componente reutilizable.
- **FR-004**: El sistema DEBE reemplazar, en cada pantalla del panel de administración, todo carácter emoji usado hoy como ícono funcional (ítem de navegación, botón de acción, indicador de estado, ícono de tarjeta/estadística o de entidad) por el ícono equivalente del nuevo componente reutilizable.
- **FR-005**: Al finalizar la funcionalidad, ninguna pantalla del panel de administración DEBE mostrar íconos mediante el componente SVG artesanal anterior. Dicho componente permanece en el código, sin modificarse, únicamente para seguir sirviendo al flujo público de menú QR (FR-010) que también lo usa hoy; no se elimina del repositorio como parte de esta funcionalidad (ver Clarifications, sesión durante `/speckit-plan`).
- **FR-006**: Al finalizar la funcionalidad, ningún ícono del panel de administración DEBE hacer referencia visual al negocio de heladería (incluyendo, entre otros, el ícono de cono de helado usado hoy como marca por defecto del panel lateral y en el acceso/tarjeta de "Productos"); estos DEBEN reemplazarse por íconos neutros equivalentes en significado.
- **FR-007**: Cada ícono reemplazado DEBE conservar el mismo significado funcional (la acción, el estado o la entidad que representa) que tenía el ícono o emoji al que reemplaza.
- **FR-008**: Los indicadores de estado que hoy combinan color y símbolo mediante emoji DEBEN seguir comunicando el mismo estado después del reemplazo por Material Icons.
- **FR-009**: El nuevo componente reutilizable de íconos DEBE quedar disponible para ser usado por cualquier parte de la aplicación, aunque esta funcionalidad solo migra los íconos que hoy se muestran dentro del panel de administración.
- **FR-010**: El flujo público del menú digital por código QR (menú, carrito y pago que ve el cliente sin autenticarse) queda fuera del alcance de esta funcionalidad; su comportamiento, lógica y rutas NO deben modificarse como parte de este cambio. Cuando ese flujo reutilice un componente visual compartido con una pantalla del panel de administración, el reemplazo del ícono de ese componente compartido (requerido por FR-004 dentro del panel de administración) puede reflejarse también en el flujo público como efecto visual secundario, sin que eso constituya una modificación de su funcionalidad (ver Clarifications, sesión durante `/speckit-plan`).
- **FR-011**: Todo ícono del panel de administración que se muestre sin texto visible junto a él (por ejemplo, un botón de acción de solo ícono o un indicador de estado sin etiqueta) DEBE tener una descripción accesible equivalente a su significado, disponible para personas que usan lector de pantalla.

### Key Entities

- **Ícono**: Representa un glifo visual identificado por un nombre semántico (por ejemplo, "mesa", "inventario", "editar", "estado activo") que el componente reutilizable resuelve contra un ícono concreto de la librería Material Icons. No se persiste como dato de negocio; es un concepto de presentación usado de forma consistente en todo el panel de administración.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de las pantallas del panel de administración (panel principal, caja, inventario, mesas/terminal, usuarios, promociones) muestran sus íconos a través del mismo componente reutilizable de íconos.
- **SC-002**: Cero (0) íconos SVG artesanales quedan visibles en cualquier pantalla del panel de administración.
- **SC-003**: Cero (0) íconos con temática de heladería quedan visibles en cualquier pantalla del panel de administración.
- **SC-004**: El 100% de los íconos migrados conservan, tras una revisión pantalla por pantalla, el mismo significado funcional que tenían antes del cambio.
- **SC-005**: El personal del negocio puede identificar cada acción, estado o sección del panel de administración por su ícono con la misma facilidad que antes del cambio, sin requerir capacitación adicional.
- **SC-006**: El 100% de los íconos que se muestran sin texto visible junto a ellos (botones de solo ícono, indicadores de estado sin etiqueta) exponen una descripción accesible verificable con herramientas de auditoría de accesibilidad.
- **SC-007**: El panel de administración sigue mostrando correctamente el 100% de sus íconos cuando se usa sin conexión a internet, con el mismo comportamiento de disponibilidad que ya ofrece hoy el modo híbrido del terminal de mesas.

## Assumptions

- "Panel de administración" se interpreta como toda la aplicación autenticada de uso interno del negocio (panel principal, caja, inventario, mesas/terminal para el personal, usuarios, configuración y promociones). Se excluye explícitamente el menú digital público que el cliente ve sin autenticarse al escanear el código QR de la mesa (navegación de menú, carrito y pago), por tratarse de una superficie de producto distinta ya cubierta por otra línea de especificaciones (menú QR).
- Un emoji o ícono se considera "de heladería" cuando representa visualmente helados, conos, paletas u otros elementos propios de ese negocio específico (por ejemplo, el emoji 🍦); el ícono de escudo 🛡️ usado para el rol de super-administrador no es de heladería, pero igual se estandariza al nuevo componente por la Historia 1.
- "Material Icons" se refiere a la librería/fuente de íconos de Google indicada explícitamente por quien solicitó la funcionalidad. La variante visual "Outlined" fue elegida durante la planeación (`research.md`, Decisión D1) por ser la más parecida al estilo de trazo fino que tienen los íconos actuales.
- Un emoji que aparece como parte del texto de un mensaje o notificación (no como ícono independiente de navegación, botón, tarjeta o indicador de estado) no se considera "ícono" para efectos de esta funcionalidad y no requiere reemplazo, salvo que tenga temática de heladería.
- Reemplazar el ícono asociado a una marca o etiqueta existente (por ejemplo, el logo por defecto de un tenant) no cambia la lógica de negocio que decide cuándo mostrarlo, únicamente el glifo visual usado.
- No se requieren cambios de backend, de permisos ni de roles para esta funcionalidad; es un cambio de presentación exclusivo del frontend del panel de administración.
