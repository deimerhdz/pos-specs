# Feature Specification: Vista de Pasos para Revisión y Pago del Menú QR

**Feature Branch**: `034-checkout-qr-vista-pasos`

**Created**: 2026-08-24

**Status**: Draft

**Naturaleza de esta spec**: funcionalidad **nueva** (fase de evolución funcional, Principio I de la
[Constitución](../../.specify/memory/constitution.md)). Se construye sobre
[spec 007](../007-menu-carrito-qr/spec.md) (menú y carrito del comensal),
[spec 024](../024-pagos-ordenes-mesa/spec.md) y
[spec 025](../025-revision-pago-antes-envio/spec.md) (revisión y pago antes de enviar el pedido) y
[spec 032](../032-catalogo-metodos-pago/spec.md) (catálogo de métodos de pago por tenant). Ninguna
regla de negocio de esas specs cambia: esta spec cambia **cómo se presenta** la interacción de
revisión y pago (de un modal superpuesto a una vista propia con pasos) y **su resiliencia** ante una
recarga de página, además de corregir cómo se muestran los datos de pago del tenant y modernizar los
iconos de esta pantalla.

**Input**: User description: "mejorar el flujo de checkout del menú QR, actualmente se muestra
mediante un modal, pero si el usuario llega a la sección de adjuntar el comprobante de pago y
recarga la página tiene que empezar de nuevo y se pierden los pasos; migrar esa interacción del
modal a una vista diferente, mediante un formulario de pasos, para prevenir que la información se
pierda; también mostrar la información de pago con el número de cuenta e imagen del código QR
configurado por el tenant; y mejorar los iconos para que se vean más profesionales."

## Clarifications

### Session 2026-08-24

- Q: ¿El progreso recuperable de pago (paso alcanzado, método elegido) debe funcionar solo en el
  mismo dispositivo/navegador donde el comensal empezó, o también debe recuperarse al reabrir la
  sesión desde otro dispositivo mediante el enlace compartido? → A: Solo en el mismo
  dispositivo/navegador — reabrir el enlace de sesión en otro dispositivo mantiene el carrito y
  cualquier pedido ya existente (comportamiento ya vigente, spec 007), pero reinicia la selección de
  método de pago y el paso alcanzado en esta vista.
- Q: ¿El criterio de éxito sobre los iconos debe medirse de forma objetiva (sin emojis, todos del
  set de iconos ya existente en la aplicación), o mediante un estudio cualitativo de percepción con
  usuarios reales? → A: De forma objetiva — 0% de iconos como emoji o imagen rasterizada, 100%
  provenientes del set de iconos ya existente en la aplicación; consistente con cómo se verifican el
  resto de los criterios de esta spec y de las demás specs del proyecto.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - El comensal retoma el pago exactamente donde iba si recarga la página (Priority: P1)

Un comensal está en medio de la revisión de su pedido y el pago — ya eligió un método de
transferencia, o ya cargó su comprobante — y por cualquier motivo recarga la página (se le cierra el
navegador, pierde señal un instante, refresca por error). Al volver a abrir el flujo, el sistema lo
regresa exactamente al paso en el que estaba, con sus selecciones ya hechas conservadas, en lugar de
obligarlo a empezar de nuevo desde la carta.

**Why this priority**: es el problema concreto que motiva esta spec — hoy perder el progreso de pago
a mitad de camino genera fricción, confusión (¿ya pagué o no?) y abandono. Sin esta garantía, migrar
a una vista propia no resuelve el dolor real del comensal.

**Independent Test**: avanzar el flujo hasta elegir un método de transferencia, recargar la página, y
verificar que el comensal vuelve al mismo paso con el método ya elegido, sin tener que seleccionarlo
de nuevo.

**Acceptance Scenarios**:

1. **Given** un comensal que ya eligió un método de transferencia en la pantalla de revisión, **When**
   recarga la página antes de cargar el comprobante, **Then** vuelve a ver la pantalla de ese método
   con sus datos de pago, sin tener que volver a elegirlo.
2. **Given** un comensal que ya cargó su comprobante con éxito antes de que el pedido se creara,
   **When** recarga la página justo después, **Then** el sistema reconoce que el comprobante ya fue
   recibido y le permite continuar sin pedírselo de nuevo (mismo principio ya vigente en spec 025 para
   un fallo de red, extendido aquí al caso de recarga de página).
3. **Given** un comensal que solo había seleccionado un archivo en su dispositivo sin que la carga se
   confirmara con éxito, **When** recarga la página, **Then** el sistema lo regresa al mismo paso
   pidiéndole seleccionar el archivo de nuevo (no puede recuperar una selección de archivo que nunca
   llegó a subirse), pero conserva el método de pago ya elegido.
4. **Given** un comensal cuyo pedido ya se creó (pago en efectivo confirmado, o comprobante ya
   cargado), **When** recarga la página, **Then** ve la confirmación de su pedido ya creado, no la
   pantalla de revisión — esta garantía de recuperación aplica mientras el pedido todavía no existe.

---

### User Story 2 - El comensal navega la revisión y el pago en una vista propia, con pasos claros (Priority: P1)

En lugar de un modal superpuesto sobre la carta, la revisión del pedido, la selección de método de
pago, los datos de transferencia y la carga del comprobante se presentan como una vista propia,
organizada en pasos con indicación visible de en cuál se encuentra el comensal y la posibilidad de
volver a un paso anterior sin perder lo ya hecho más adelante.

**Why this priority**: es el cambio de interfaz que hace posible la Historia 1 (una vista propia,
navegable, es lo que permite recuperar el progreso tras recargar) y además resuelve por sí sola la
confusión de un modal que se siente "temporal" y desconectado de la navegación normal.

**Independent Test**: abrir la revisión desde el carrito y verificar que aparece como una vista
propia (no como una superposición sobre la carta), con un indicador de paso visible, y que se puede
volver a un paso anterior sin perder el progreso de los pasos siguientes que sigan siendo válidos.

**Acceptance Scenarios**:

1. **Given** un comensal con productos en el carrito, **When** presiona "Enviar pedido", **Then** se
   abre la vista de revisión y pago como pantalla propia, mostrando en qué paso se encuentra de la
   secuencia completa.
2. **Given** un comensal en el paso de datos de un método de transferencia, **When** decide volver al
   paso de selección de método, **Then** puede hacerlo y elegir uno distinto (efectivo u otra
   transferencia), sin ningún rastro del método anterior — mismo comportamiento ya definido en spec
   025 (Historia 4), ahora expresado como navegación entre pasos de esta vista.
3. **Given** un comensal en cualquier paso previo a que el pedido exista, **When** decide salir de la
   vista sin completar el pago, **Then** no se crea ningún pedido y su carrito conserva exactamente
   los mismos productos (mismo comportamiento ya definido en spec 025).

---

### User Story 3 - El comensal ve el número de cuenta y el código QR de pago con claridad (Priority: P2)

Cuando el comensal elige un método de transferencia, ve los datos de pago que el tenant configuró
para ese método — como el número de cuenta — como texto legible, y el código QR de pago, cuando el
método lo tiene configurado, como una imagen que puede escanear o ampliar, no como un texto o enlace
plano.

**Why this priority**: sin esto, el comensal no puede completar su transferencia con confianza — un
código QR mostrado como texto no sirve para escanear, y obliga a copiar manualmente un número de
cuenta mezclado entre otros datos sin etiquetar con claridad.

**Independent Test**: elegir un método de transferencia cuyo tenant tenga configurados número de
cuenta e imagen de código QR, y verificar que el número se lee como texto y el código QR se muestra
como una imagen visible.

**Acceptance Scenarios**:

1. **Given** un método de transferencia con número de cuenta e imagen de código QR configurados por
   el tenant, **When** el comensal lo elige, **Then** ve el número de cuenta como texto legible y
   claramente identificado, y la imagen del código QR renderizada como imagen, no como una URL en
   texto plano.
2. **Given** un método de transferencia sin imagen de código QR configurada, **When** el comensal lo
   elige, **Then** ve el resto de los datos de pago disponibles, sin ningún espacio roto ni ícono de
   "imagen no encontrada".
3. **Given** el método de pago que el comensal había elegido antes de recargar la página deja de
   estar activo para el tenant mientras tanto, **When** vuelve a la vista, **Then** el sistema se lo
   indica y le pide elegir uno de los métodos actualmente activos, conservando el resto de su
   progreso (resumen de su pedido).

---

### User Story 4 - Los iconos de la pantalla de pago se ven profesionales y consistentes (Priority: P3)

Los iconos que identifican los métodos de pago, el estado del comprobante, y la navegación entre
pasos de esta vista usan gráficos consistentes con la identidad visual del resto de la aplicación, en
lugar de emojis o símbolos informales.

**Why this priority**: mejora la percepción de seriedad y confianza del comensal al momento de pagar,
pero no bloquea ninguna de las otras historias — es una mejora visual sobre un flujo que ya funciona.

**Independent Test**: recorrer los pasos de la vista de revisión y pago y verificar que ningún icono
es un emoji; todos son gráficos vectoriales consistentes entre sí y con el resto del producto.

**Acceptance Scenarios**:

1. **Given** la vista de revisión y pago en cualquiera de sus pasos, **When** el comensal la recorre,
   **Then** todos los iconos (métodos de pago, adjuntar/quitar comprobante, volver, confirmar) son
   gráficos vectoriales de un mismo estilo, sin emojis.

---

### Edge Cases

- **El comensal recarga la página después de que su pedido ya se creó** (efectivo confirmado, o
  comprobante ya cargado): no aplica la recuperación de esta spec — ve la confirmación de su pedido
  ya existente, continuando con el flujo ya definido en spec 025.
- **El comensal solo seleccionó un archivo localmente sin que la carga se confirmara** antes de
  recargar: debe volver a seleccionarlo — limitación inherente de los navegadores (no puede
  restaurarse la selección de un input de archivo tras una recarga), no una carencia de esta spec.
- **La sesión del comensal expira mientras su progreso de pago está pendiente de recarga**: aplica el
  mismo límite de sesión ya vigente (spec 007) — al expirar, debe reingresar por su QR o enlace como
  ya ocurre hoy; no se introduce un mecanismo de expiración distinto para el progreso de pago.
- **El tenant desactiva el método de transferencia que el comensal tenía elegido mientras este no
  había completado el pago**: al volver a la vista (por recarga o navegación), el sistema se lo
  indica y le pide elegir otro método activo, sin perder el resumen de su pedido.
- **Un método de transferencia no tiene imagen de código QR configurada**: se muestran los demás
  datos de pago disponibles, sin espacio roto ni ícono de imagen no encontrada.
- **El comensal reabre el enlace de sesión compartido en otro dispositivo** mientras tiene un pago en
  curso: el carrito y cualquier pedido ya existente se ven igual (comportamiento ya vigente, spec
  007, Historia 11), pero la selección de método de pago y el paso alcanzado en esta vista NO se
  recuperan en el dispositivo nuevo — deben elegirse de nuevo, porque ese progreso es local al
  dispositivo/navegador donde se inició. Si ya había cargado un comprobante con éxito antes de
  cambiar de dispositivo, esa carga sí se reconoce, porque vive en el servidor desde antes de esta
  spec (FR-006).
- **El comensal abre el mismo enlace de sesión en dos pestañas del mismo dispositivo/navegador**
  mientras tiene un pago en curso: comparten el mismo progreso local; la última acción real es la
  que prevalece, sin mecanismo de bloqueo entre pestañas.

## Requirements *(mandatory)*

### Functional Requirements

**Vista de pasos (reemplazo del modal)**

- **FR-001**: El sistema DEBE presentar la revisión del pedido, la selección de método de pago, los
  datos de un método de transferencia y la carga del comprobante como una vista propia organizada en
  pasos, en lugar de como un modal superpuesto sobre la carta.
- **FR-002**: La vista DEBE mostrar en todo momento un indicador visible de en qué paso de la
  secuencia se encuentra el comensal.
- **FR-003**: El sistema DEBE permitir al comensal volver a un paso anterior de esta vista y cambiar
  su elección (por ejemplo, de método de transferencia) sin restricción, mientras el pedido todavía
  no exista (mismo comportamiento ya definido en spec 025, Historia 4).
- **FR-004**: El sistema DEBE permitir al comensal salir de esta vista en cualquier paso previo a que
  el pedido exista, sin que eso cree ningún pedido ni afecte los productos ya agregados a su carrito
  (mismo comportamiento ya definido en spec 025).

**Resiliencia ante recarga de página**

- **FR-005**: Si el comensal recarga la página, en el mismo dispositivo y navegador donde inició el
  pago, en cualquier paso de esta vista mientras su pedido todavía no existe, el sistema DEBE
  devolverlo exactamente al mismo paso en que estaba, con el método de pago ya elegido preservado
  (Clarification Session 2026-08-24: esta garantía es local al dispositivo/navegador, no
  multi-dispositivo).
- **FR-006**: Si el comensal ya había cargado con éxito un comprobante de transferencia antes de
  recargar la página, el sistema NO DEBE pedirle cargarlo de nuevo — debe reconocer que ya fue
  recibido y permitirle continuar (crear el pedido) sin repetir la carga. Esto extiende a la recarga
  de página la misma garantía que spec 025 (FR-012) ya define para un fallo de red tras la carga.
- **FR-007**: Si el comensal solo había seleccionado un archivo de comprobante en su dispositivo sin
  que la carga se confirmara con éxito antes de recargar, el sistema DEBE permitirle seleccionar el
  archivo de nuevo, conservando el resto de su progreso (método de pago ya elegido).
- **FR-008**: Si el pedido del comensal ya se creó (pago en efectivo confirmado, o comprobante ya
  cargado) antes de recargar la página, el sistema NO DEBE mostrarle de nuevo la vista de revisión —
  debe mostrarle la confirmación de su pedido ya existente.
- **FR-009**: El progreso recuperable de esta vista (paso alcanzado, método elegido, comprobante ya
  cargado) NO DEBE sobrevivir más allá de la vigencia de la sesión del comensal ya definida en spec
  007 (ventana deslizante de inactividad y tope absoluto) — no se introduce un mecanismo de
  expiración distinto.
- **FR-010**: Si el método de pago que el comensal había elegido deja de estar activo para el tenant
  mientras su progreso estaba pendiente de recarga, el sistema DEBE indicárselo y pedirle elegir uno
  de los métodos actualmente activos, conservando el resto de su progreso.

**Datos de pago del tenant**

- **FR-011**: Cuando el comensal elige un método de transferencia, el sistema DEBE mostrarle los datos
  de pago configurados por el tenant para ese método específico (por ejemplo número de cuenta, tipo
  de cuenta, número de celular) como texto legible y claramente identificado.
- **FR-012**: Cuando el método de transferencia elegido tenga configurada una imagen de código QR de
  pago, el sistema DEBE mostrarla como una imagen visible, no como una URL o texto plano.
- **FR-013**: Si el método de transferencia elegido no tiene imagen de código QR configurada, el
  sistema DEBE mostrar el resto de los datos de pago disponibles sin dejar un espacio roto ni un
  ícono de imagen no encontrada.

**Iconos**

- **FR-014**: Los iconos usados en esta vista (métodos de pago, adjuntar/quitar comprobante,
  navegación entre pasos, confirmación) DEBEN ser gráficos vectoriales consistentes entre sí y con la
  identidad visual del resto de la aplicación, en lugar de emojis o símbolos informales.

**Sin cambio de reglas de negocio**

- **FR-015**: El sistema DEBE seguir aplicando, sin cambios, todas las reglas ya vigentes de las
  specs 024/025 sobre este flujo: el pedido solo se crea al confirmar efectivo o al cargar el
  comprobante con éxito, el bloqueo de una segunda orden activa, la garantía de una sola creación de
  pedido por confirmación (incluso duplicada), y la aprobación/rechazo de comprobante por el cajero.
  Esta spec cambia únicamente la interfaz (modal → vista de pasos), su resiliencia ante recarga, la
  presentación de los datos de pago y los iconos — no las reglas de negocio del pago en sí.

### Key Entities *(include if feature involves data)*

- **Progreso de Revisión y Pago** (nuevo, conceptual): representa el estado recuperable de la vista
  de revisión y pago de un comensal mientras su pedido todavía no existe — qué paso alcanzó y qué
  método de pago eligió. Es local al dispositivo/navegador donde el comensal inició el pago
  (Clarification Session 2026-08-24) — no se recupera al reabrir la sesión desde otro dispositivo. Si
  ya cargó un comprobante con éxito, ese hecho sí se reconoce sin importar el dispositivo, porque vive
  en el servidor como parte de **Comprobante** (spec 025), no de este progreso local. El Progreso de
  Revisión y Pago vive mientras la sesión del comensal siga vigente (spec 007); deja de tener sentido
  en cuanto el pedido se crea, momento en el que pasan a aplicar **Orden**, **Intento de Pago** y
  **Comprobante**, ya definidos en spec 024/025.
- **Configuración de Método de Pago por Tenant**: ya definida en spec 032; esta spec no le agrega
  campos — solo corrige cómo se presenta al comensal (número de cuenta como texto, imagen de código
  QR como imagen) en la vista de pago, sin afectar su uso ya definido en la pantalla de cobro de caja
  (spec 032, FR-012a, que permanece sin cambios).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El 100% de los comensales que recargan la página durante la revisión o el pago (antes
  de que el pedido exista) retoman exactamente en el mismo paso, sin tener que rehacer selecciones ya
  hechas.
- **SC-002**: El 100% de los comensales que ya subieron su comprobante con éxito antes de recargar la
  página no tienen que volver a seleccionarlo ni subirlo de nuevo.
- **SC-003**: El 100% de los métodos de transferencia con imagen de código QR configurada la muestran
  como imagen visible (no como texto o URL) en la vista de pago del comensal.
- **SC-004**: El comensal puede identificar en qué paso de la revisión y el pago se encuentra sin
  necesitar explicación adicional, verificado con usuarios reales.
- **SC-005**: El 0% de los iconos de la vista de revisión y pago son emojis o imágenes rasterizadas;
  el 100% provienen del set de iconos vectoriales ya existente en la aplicación (Clarification
  Session 2026-08-24).
- **SC-006**: El 0% de las veces que un comensal sale de esta vista sin completar el pago queda un
  pedido registrado para el staff (mismo criterio ya vigente en spec 025, verificado de nuevo tras
  este cambio de interfaz).

## Out of Scope

- **El modal de reintento de pago tras un rechazo de comprobante** (spec 024, Historia 5), que actúa
  sobre un pedido **ya creado**: sigue siendo un modal; esta spec no lo migra a la nueva vista de
  pasos.
- Nuevos métodos de pago o cambios a las reglas de negocio de pago (efectivo/transferencia,
  aprobación/rechazo del cajero, cálculo de cambio) — sin cambios, ya definidos en specs 024/025/032.
- Rediseño de iconos en el resto de la aplicación (menú, carrito, panel de cocina o de caja) — el
  alcance de iconografía de esta spec se limita a la vista de revisión y pago del comensal.
- Edición del catálogo de métodos de pago o de sus datos de integración por el Tenant Admin — cubierto
  por spec 032, sin cambios aquí.
- Integración directa vía API/webhook con pasarelas o bancos — el proceso sigue siendo manual vía
  comprobante y verificación humana del cajero, igual que en specs 024/025.

## Assumptions

- **Se construye enteramente sobre specs 007, 024, 025 y 032**: ninguna regla de negocio de pago,
  sesión o catálogo de métodos cambia — solo la interfaz (modal → vista de pasos), la resiliencia
  ante recarga de página, la presentación de los datos de pago del tenant, y los iconos de esta
  pantalla.
- **La vigencia del progreso recuperable está acotada por la sesión del comensal ya definida en spec
  007** (ventana deslizante de 4h, tope absoluto de 24h) — no se introduce un mecanismo de expiración
  nuevo ni distinto.
- **Seleccionar un archivo de comprobante localmente sin confirmarlo no sobrevive a una recarga**: es
  una limitación inherente de los navegadores, no una decisión de producto; una vez la carga se
  confirma con éxito, sí se preserva.
- **El alcance de "migrar el modal a una vista distinta" cubre el flujo completo de revisión de
  pedido, selección de método, datos/comprobante de transferencia y confirmación de efectivo** (spec
  025); no incluye el modal de reintento tras rechazo de comprobante sobre un pedido ya creado (spec
  024), que queda fuera de alcance explícito.
- **La mejora de iconos se limita a esta vista de revisión y pago**: busca consistencia con la
  identidad visual ya existente en el resto de la aplicación, sin definir aquí un sistema de iconos
  nuevo para todo el producto.
- **La restricción de spec 032 (FR-012a) sobre la pantalla de cobro en caja no cambia**: esa pantalla
  sigue mostrando los métodos de pago solo por nombre, sin datos de integración, exclusivamente para
  el cajero; esta spec solo mejora cómo el **comensal** ve esos datos en su propia vista de pago, ya
  requerido desde spec 025 (FR-005).
