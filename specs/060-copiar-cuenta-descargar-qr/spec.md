# Feature Specification: Copiar número de cuenta y descargar QR en pagos por transferencia

**Feature Branch**: `060-copiar-cuenta-descargar-qr`

**Created**: 2026-08-30

**Status**: Draft

**Naturaleza de esta spec**: mejora de UX sobre una pantalla ya existente del flujo de pedido por
QR del comensal (`/menu/t/:token/checkout/...`, "Paso 3 de 3: Datos de transferencia",
`TransferDetailsStepComponent`). Esta pantalla ya se muestra únicamente para métodos de pago
distintos a efectivo (los que requieren transferencia) y ya renderiza de forma genérica, por
método, cualquier campo de texto (p. ej. "Número de celular") y cualquier campo de imagen (el QR)
que el administrador haya configurado para ese método — no hay nada hardcodeado a "Nequi", el
mismo comportamiento debe aplicar a cualquier método de transferencia configurado (Nequi,
Daviplata, Bancolombia, etc.). Hoy ninguno de esos valores se puede copiar ni el QR se puede
descargar: el comensal solo puede leer el número a mano o tomar una captura de pantalla del código.
La app ya cuenta con un sistema de notificaciones no bloqueantes (toasts efímeros) usado en varias
pantallas, que esta mejora reutiliza para las dos confirmaciones pedidas.

**Input**: User description (verbatim): "en las opciones de pago que son diferentes a efectivo,
quiero agregar la opcion de que el numero de cuenta se pueda copiar al portapapeles, usa iconos
elegantes para describir la opcion de copiar y notifica cuando se haya copiado correctamente y
tambien agregar la opcion de que es usuario pueda descargar la imagen del qr para que no tenga que
estar tomando captura de pantalla y notifica cuando la imagen se haya descargado no uses alertas
modales para notificar y cada alerta debe durar 5 segundos"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Copiar el número de cuenta/celular al portapapeles (Priority: P1)

Un comensal llega al paso "Datos de transferencia" tras elegir un método de pago distinto a
efectivo (p. ej. Nequi) y ve el número de celular/cuenta al que debe transferir. Hoy tiene que
leerlo y volver a escribirlo manualmente en su app bancaria, con riesgo de equivocarse un dígito.
Con esta mejora, junto a ese número aparece un ícono claro de "copiar" que, al tocarlo, copia el
valor exacto al portapapeles del dispositivo, y el comensal ve una notificación breve confirmando
que se copió correctamente.

**Why this priority**: es el ajuste explícitamente pedido con mayor riesgo de error real para el
comensal (transferir a un número mal transcrito) — impacta directamente si el pago llega o no.

**Independent Test**: puede probarse por completo abriendo el paso "Datos de transferencia" de un
método de pago con un campo de texto (p. ej. número de celular), tocando el ícono de copiar junto
al valor, pegando en otro campo y verificando que el texto pegado coincide exactamente con el valor
mostrado, y que aparece una notificación de éxito.

**Acceptance Scenarios**:

1. **Given** el comensal está en "Datos de transferencia" de un método con un campo de texto
   configurado (p. ej. "Número de celular"), **When** observa ese valor, **Then** ve junto a él un
   ícono reconocible de "copiar".
2. **Given** el comensal toca el ícono de copiar junto al número, **When** la copia se completa,
   **Then** el valor copiado en el portapapeles coincide exactamente con el texto mostrado en
   pantalla, sin espacios ni caracteres adicionales.
3. **Given** el comensal copió el número exitosamente, **When** observa la pantalla, **Then**
   aparece una notificación no bloqueante confirmando que se copió, que desaparece por sí sola sin
   necesidad de cerrarla.
4. **Given** un método de pago define más de un campo de texto, **When** el comensal ve el paso,
   **Then** cada campo de texto tiene su propio ícono de copiar independiente.
5. **Given** el navegador o dispositivo del comensal no permite acceder al portapapeles, **When**
   toca el ícono de copiar, **Then** el sistema muestra una notificación no bloqueante indicando que
   no se pudo copiar, en vez de fallar en silencio.

---

### User Story 2 - Descargar la imagen del código QR (Priority: P2)

Un comensal en el mismo paso ve el código QR del método de pago elegido y hoy solo puede tomar una
captura de pantalla para guardarlo o abrirlo en su app bancaria. Con esta mejora, junto al QR
aparece una opción para descargar la imagen directamente al dispositivo, y al completarse la
descarga el comensal ve una notificación breve confirmándolo.

**Why this priority**: mejora la experiencia y evita errores de recorte/calidad de una captura de
pantalla manual, pero el flujo ya es usable hoy sin esto (a diferencia de copiar el número, que
previene un error de transcripción) — de ahí la prioridad menor frente a la Historia 1.

**Independent Test**: puede probarse por completo abriendo el paso "Datos de transferencia" de un
método con un campo de imagen (QR) configurado, activando la opción de descarga, y verificando que
el archivo de imagen queda guardado en el dispositivo y que aparece una notificación de éxito.

**Acceptance Scenarios**:

1. **Given** el comensal está en "Datos de transferencia" de un método con un campo de imagen (QR)
   configurado, **When** observa el QR, **Then** ve junto a él una opción clara para descargarlo.
2. **Given** el comensal activa la descarga del QR, **When** la descarga se completa, **Then** la
   imagen queda guardada en el dispositivo como un archivo utilizable sin pasos adicionales.
3. **Given** la descarga del QR se completó, **When** el comensal observa la pantalla, **Then**
   aparece una notificación no bloqueante confirmando la descarga, que desaparece por sí sola.
4. **Given** la descarga del QR falla (p. ej. problema de red al obtener la imagen), **When** el
   comensal observa la pantalla, **Then** aparece una notificación no bloqueante indicando que la
   descarga falló, en vez de no dar ninguna señal.

---

### Edge Cases

- ¿Qué pasa si un método de pago no define ningún campo de texto (solo QR)? No debe mostrarse
  ningún ícono de copiar, porque no hay ningún valor de texto que copiar.
- ¿Qué pasa si un método de pago no define ningún campo de imagen (solo texto, sin QR)? No debe
  mostrarse ninguna opción de descarga, porque no hay ninguna imagen que descargar.
- ¿Qué pasa si el comensal toca "copiar" varias veces seguidas sobre el mismo valor? Cada copia
  exitosa puede mostrar su propia notificación, pero nunca debe apilar copias idénticas visibles al
  mismo tiempo (mismo criterio que ya usa el sistema de notificaciones existente para no duplicar
  avisos idénticos).
- ¿Qué pasa si el comensal navega a otro paso del checkout (atrás o cierra el flujo) mientras una
  notificación de copiado o descarga sigue visible? La notificación simplemente termina su
  temporizador de 5 segundos sin bloquear ninguna navegación.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE permitir copiar al portapapeles, con una sola acción, el valor de
  cualquier campo de texto (p. ej. número de celular/cuenta) mostrado en el paso "Datos de
  transferencia" de un método de pago distinto a efectivo.
- **FR-002**: El sistema DEBE mostrar, junto a cada valor de texto copiable, un ícono claro y
  reconocible que identifique la acción de copiar, consistente con el resto de íconos ya usados en
  la aplicación.
- **FR-003**: El valor copiado al portapapeles DEBE coincidir exactamente con el valor mostrado en
  pantalla, sin texto adicional ni pasos manuales de selección por parte del comensal.
- **FR-004**: El sistema DEBE permitir descargar la imagen del código QR mostrado en el paso "Datos
  de transferencia" como un archivo de imagen en el dispositivo del comensal, sin requerir captura
  de pantalla.
- **FR-005**: Al copiar un valor de texto exitosamente, el sistema DEBE mostrar una notificación no
  bloqueante (no una alerta modal) confirmando que se copió correctamente.
- **FR-006**: Al descargar la imagen del QR exitosamente, el sistema DEBE mostrar una notificación
  no bloqueante (no una alerta modal) confirmando que la imagen se descargó.
- **FR-007**: Ambas notificaciones (copiado y descarga) DEBEN desaparecer automáticamente a los 5
  segundos, sin requerir que el comensal las cierre manualmente.
- **FR-008**: Si la acción de copiar falla (p. ej. el navegador o dispositivo deniega el acceso al
  portapapeles), el sistema DEBE mostrar una notificación no bloqueante indicando el error, en vez
  de no dar ninguna señal.
- **FR-009**: Si la acción de descargar el QR falla, el sistema DEBE mostrar una notificación no
  bloqueante indicando el error, en vez de no dar ninguna señal.
- **FR-010**: Estas opciones DEBEN aplicar a cualquier método de pago distinto a efectivo que
  defina un campo de texto y/o un campo de imagen, sin quedar limitadas a un método específico
  (p. ej. Nequi).
- **FR-011**: Cuando un método de pago no defina ningún campo de texto, el sistema NO DEBE mostrar
  ningún ícono de copiar. Cuando no defina ningún campo de imagen, el sistema NO DEBE mostrar
  ninguna opción de descarga.
- **FR-012**: Esta mejora NO DEBE modificar el flujo de envío del pedido ni el paso de subir el
  comprobante de pago — únicamente agrega las dos acciones (copiar y descargar) sobre datos que ya
  se muestran hoy.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: El comensal copia el número de cuenta/celular al portapapeles con una sola
  interacción (un toque/clic), sin necesidad de seleccionar texto manualmente.
- **SC-002**: El comensal descarga la imagen del QR a su dispositivo con una sola interacción, sin
  necesidad de tomar una captura de pantalla.
- **SC-003**: El 100% de las copias exitosas muestran una confirmación visible que desaparece por
  sí sola dentro de 5 segundos.
- **SC-004**: El 100% de las descargas exitosas del QR muestran una confirmación visible que
  desaparece por sí sola dentro de 5 segundos.
- **SC-005**: Ninguna de las dos confirmaciones (copiado o descarga) interrumpe ni bloquea la
  interacción del comensal con el resto de la pantalla (cero alertas modales).

## Assumptions

- Esta mejora aplica sobre la pantalla "Paso 3 de 3: Datos de transferencia" del flujo de pedido
  por QR del comensal (`TransferDetailsStepComponent`), la única pantalla hoy que muestra los datos
  de transferencia (número/cuenta y QR) de un método de pago — no se pide agregar esto a ninguna
  otra pantalla (p. ej. las de revisión de comprobante del lado del cajero, que solo muestran el
  comprobante ya subido, no los datos del método de pago).
- Cada método de pago configura sus propios campos de texto e imagen de forma independiente (según
  ya existe hoy); esta mejora debe funcionar de forma genérica para cualquier combinación de campos
  que un método defina, no solo para el ejemplo de Nequi mostrado en la captura de pantalla.
- La notificación no bloqueante reutiliza el mecanismo de notificaciones efímeras que la aplicación
  ya usa en otras pantallas, ajustando su duración a 5 segundos específicamente para estas dos
  confirmaciones (copiado y descarga), sin cambiar la duración de notificaciones existentes en otras
  partes de la app.
- El archivo de imagen descargado conserva el mismo código QR que ya se muestra en pantalla (sin
  recodificarlo ni alterar su contenido), guardado con un nombre de archivo descriptivo.
- "Iconos elegantes para describir la opción de copiar" se interpreta como un ícono reconocible de
  copiar (no un botón de solo texto), visualmente consistente con el resto de íconos ya usados en
  la aplicación — no se especifica un ícono ni estilo gráfico particular más allá de eso.
