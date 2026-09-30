# Feature Specification: Reporte de Caja Legible al Imprimir (Cierre de Turno e Historial)

**Feature Branch** (rama de la spec en `pos-specs`): `090-fix-cash-report-print-blank`. Las ramas de código de `pos-heladeria` siguen el Principio XIV y se definen en `tasks.md`

**Created**: 2026-09-30

**Status**: Draft

**Input**: User description: "Hay un problema que aún no se ha solucionado: al imprimir el resumen de
caja, ya sea al cerrar el turno o desde el historial de turnos, los documentos PDF se muestran en
blanco, pero no es porque no haya contenido, es porque el texto es de color blanco. Corregir esto y
hacer el texto visible, tanto desde el historial como desde la otra opción."

Usuario: **Cajero / Administrador** — necesita entregar o archivar el reporte de cierre de caja en
papel o PDF, tanto justo después de cerrar el turno como al reimprimirlo más adelante desde el
historial de turnos.

## Clarifications

### Session 2026-09-30

- Q: ¿Con qué código se vio la hoja en blanco? → A: **según el usuario, con el arreglo de la spec 089
  ya aplicado** (la historia 5 de la spec 089, rama `fix/089-cash-report-print` de `pos-heladeria`).
  El historial de git indica que esa rama **no está fusionada a `develop`**; por eso el diagnóstico
  (FR-009, research D1) prueba los dos estados, `develop` a secas y `develop` + 089, antes de asumir que
  ese arreglo no basta. La causa real debe **diagnosticarse de nuevo**; no se asume que basta con
  integrarlo.
- Q: ¿Ocurre con el tema claro, el oscuro o ambos? → A: **en ambos**: no depende del tema.
- Q: ¿En qué navegador debe quedar garantizado? → A: **Brave** (basado en Chromium), con la vista
  previa de impresión y "Guardar como PDF", que es lo que muestran las capturas del usuario.
- Q: ¿Alcance? → A: **solo el reporte de caja**, en sus dos rutas (al cerrar el turno y desde el
  historial de turnos). Recibo de venta, recibo de mesa y hoja de QR **no cambian**.

### Hallazgos del reconocimiento (contexto, no decisiones)

- Las dos rutas —"Imprimir / Exportar reporte" tras cerrar el turno y el reporte abierto desde el
  historial— muestran **la misma pantalla de reporte** y usan la **misma acción de imprimir**; un
  defecto ahí se manifiesta idéntico en ambas.
- Las capturas del usuario muestran una hoja con **solo el encabezado y pie que agrega el navegador**
  (fecha y hora, nombre del archivo `heladeria-30-09-2026`, dirección de la página y "1/1"): el
  contenido del reporte no se ve.
- La spec 089 (Historia 5) **ya declaró corregido** este síntoma y lo verificó con un Chrome
  automatizado sin fondos; la vista previa real de impresión de un navegador de escritorio y Firefox
  **nunca se comprobaron** (tarea T070 abierta). Esa es la brecha de verificación que esta spec cierra.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Imprimir el reporte justo después de cerrar el turno (Priority: P1)

Como cajero, al cerrar mi turno quiero pulsar "Imprimir / Exportar reporte" y obtener un documento
(vista previa, papel o PDF) donde **todo el contenido del reporte se ve**, en texto oscuro sobre fondo
claro, para entregarlo al administrador o archivarlo.

**Why this priority**: es el flujo principal y hoy produce una hoja aparentemente vacía: el cajero no
puede entregar el cierre de caja, que es una obligación diaria de control.

**Independent Test**: cerrar un turno con movimientos, pulsar "Imprimir / Exportar reporte" en Brave y
comprobar que la vista previa y el PDF guardado muestran el reporte completo legible.

**Acceptance Scenarios**:

1. **Given** el reporte de un turno recién cerrado en pantalla, **When** el cajero pulsa "Imprimir /
   Exportar reporte", **Then** la vista previa muestra todos los bloques del reporte (datos del turno,
   arqueo, totales por método de pago, movimientos) con texto oscuro y legible sobre fondo claro.
2. **Given** esa misma vista previa, **When** el cajero elige "Guardar como PDF", **Then** el archivo
   guardado se llama `<negocio>-<DD-MM-YYYY>` (spec 087) y al abrirlo **todo el contenido es visible**
   sin seleccionarlo ni cambiarle el contraste.
3. **Given** un turno cerrado **sin movimientos**, **When** se imprime, **Then** el reporte sale con
   sus encabezados y totales en cero, no como hoja vacía.
4. **Given** la impresión del reporte, **Then** no aparecen el menú lateral, el encabezado de la
   aplicación ni los botones de la pantalla.

---

### User Story 2 - Reimprimir el reporte desde el historial de turnos (Priority: P1)

Como cajero o administrador, quiero abrir un turno anterior desde el historial de turnos y poder
imprimir o guardar su reporte con exactamente la misma legibilidad que al cerrar el turno.

**Why this priority**: es la segunda ruta que el usuario reportó; se usa para reponer un reporte
perdido o auditar turnos pasados, y hoy falla igual.

**Independent Test**: desde el historial, abrir un turno cerrado de un día anterior, imprimir en Brave
y verificar que la vista previa y el PDF muestran el reporte completo legible.

**Acceptance Scenarios**:

1. **Given** el historial de turnos, **When** se abre un turno cerrado y se pulsa imprimir, **Then** la
   vista previa muestra todo el contenido legible, idéntico en forma al de la Historia 1.
2. **Given** ese turno de un día distinto al actual, **When** se guarda como PDF, **Then** el nombre
   usa la **fecha de cierre del turno**, no la de hoy (spec 087).
3. **Given** el reporte abierto desde el historial, **When** el cajero pulsa "Volver al historial"
   tras imprimir o cancelar la impresión, **Then** la pantalla vuelve a verse normal (sin estilos de
   impresión pegados) y el historial funciona como siempre.

---

### User Story 3 - Legibilidad garantizada y verificada de verdad (Priority: P2)

Como responsable del producto, quiero que la legibilidad de la impresión **no dependa del tema, del
tamaño del reporte ni de la configuración de fondos del navegador**, y que se haya comprobado en una
vista previa real de impresión, para que no vuelva a declararse resuelto sin estarlo.

**Why this priority**: la spec 089 dio por corregido este defecto con una verificación parcial y el
síntoma persiste; sin una verificación real el riesgo de reincidencia es alto.

**Independent Test**: repetir las Historias 1 y 2 con tema claro y tema oscuro, con la opción de
imprimir gráficos de fondo activada y desactivada, y con un reporte de más de una hoja.

**Acceptance Scenarios**:

1. **Given** el tema claro o el oscuro, **When** se imprime el reporte, **Then** el resultado es
   idéntico: ningún texto blanco, transparente o gris claro sobre blanco.
2. **Given** "Gráficos de fondo" activado o desactivado en el diálogo de impresión, **When** se
   imprime, **Then** el contenido se ve igual de legible en ambos casos.
3. **Given** un reporte extenso que ocupa más de una hoja, **When** se imprime, **Then** todas las
   hojas tienen contenido, ninguna sale en blanco y las filas de las tablas no se parten entre hojas.
4. **Given** cualquier otra pantalla imprimible (recibo de venta, recibo de mesa, hoja de QR),
   **Then** su impresión **no cambia**.

---

### Edge Cases

- **Cancelar el diálogo de impresión**: al cancelar, la pantalla debe volver a su aspecto normal
  (menú, encabezado y colores) sin recargar la página.
- **Imprimir dos veces seguidas** (vista previa, cancelar, volver a imprimir): ambas salen legibles.
- **Reporte abierto desde el historial tras haber impreso uno recién cerrado**: no arrastra estado ni
  estilos de la impresión anterior.
- **Cierre del turno y reimpresión con la pestaña ya abierta desde antes del despliegue de la
  corrección**: la corrección llega al recargar; no se exige comportamiento retroactivo en pestañas
  viejas.
- **Turno sin movimientos o con muchos movimientos**: ambos imprimen con contenido visible.
- **Navegador con "Gráficos de fondo" desactivado** (valor por defecto de Brave/Chrome): no puede
  ocultar el contenido.
- **Extensiones o escudos del navegador** (Brave Shields, modo oscuro forzado por extensiones): el
  documento impreso no debe depender de que estén apagados; si una configuración externa inevitable
  altera la salida, se documenta como limitación conocida.
- **Nombre del archivo**: sigue siendo `<negocio>-<DD-MM-YYYY>` (spec 087); esta spec no lo cambia.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: Al imprimir el reporte de caja tras cerrar un turno, el sistema MUST mostrar **todo su
  contenido** (datos del turno, arqueo, totales y movimientos) de forma legible en la vista previa de
  impresión, en el PDF guardado y en papel.
- **FR-002**: Al imprimir el reporte de un turno abierto desde el historial de turnos, el sistema MUST
  producir **el mismo resultado legible** que en FR-001.
- **FR-003**: En el documento impreso, todo texto MUST contrastar claramente con su fondo (texto
  oscuro sobre fondo claro); MUST NOT quedar ningún texto blanco, transparente o de color claro sobre
  fondo claro.
- **FR-004**: El resultado de FR-001 a FR-003 MUST ser **el mismo con el tema claro y con el oscuro**
  de la aplicación y **el mismo con la opción "Gráficos de fondo" del navegador activada o
  desactivada**.
- **FR-005**: La legibilidad MUST mantenerse en reportes de una o de varias hojas: sin hojas en
  blanco, y con las filas de las tablas sin partirse entre hojas.
- **FR-006**: La impresión MUST seguir ocultando el menú lateral, el encabezado de la aplicación y los
  botones de la pantalla, y MUST conservar el nombre de archivo sugerido
  `<negocio>-<DD-MM-YYYY>` con la fecha de cierre del turno (spec 087).
- **FR-007**: Al terminar o cancelar la impresión, la pantalla MUST volver a su aspecto normal en
  ambas rutas, sin que la impresión anterior deje estilos o estado residuales.
- **FR-008**: La impresión de otras pantallas (recibo de venta, recibo de mesa, hoja de QR) MUST NOT
  cambiar.
- **FR-009**: Antes de corregir, la causa del texto invisible MUST **diagnosticarse de nuevo en Brave**
  con la vista previa real de impresión, con el arreglo de la spec 089 aplicado, y registrarse en
  `implementation-notes.md` (qué elemento o regla deja el contenido invisible y por qué el arreglo
  anterior no bastó). No se toca la solución hasta tener ese registro.
- **FR-010**: La corrección MUST verificarse **en Brave con la vista previa real y con el PDF
  guardado**, en ambas rutas, con tema claro y oscuro y con "Gráficos de fondo" activado y
  desactivado; además MUST quedar una comprobación automática repetible que imprima a PDF emulando el
  medio de impresión **sin depender de fondos** y confirme que el PDF contiene el texto del reporte.
- **FR-011**: La relación con la spec 089 MUST quedar resuelta y documentada: la Historia 5 de la spec
  089 queda **reemplazada** por esta spec en lo que respecta a la impresión del reporte, y se registra
  la anomalía correspondiente en el registro de anomalías (Principio XII).

### Key Entities *(include if feature involves data)*

- **Reporte de cierre de turno**: el documento imprimible que resume un turno de caja cerrado (datos
  del turno, arqueo, totales por método de pago, movimientos). Se obtiene al cerrar el turno o desde
  el historial; es **la misma pantalla** en ambos casos. No se crean ni modifican datos.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: En el 100% de las impresiones del reporte de caja (cierre de turno e historial) hechas en
  Brave, la vista previa muestra todo el contenido legible sin que el cajero tenga que cambiar ningún
  ajuste.
- **SC-002**: Un PDF guardado del reporte, abierto en un visor externo, muestra el texto completo del
  reporte; al extraer su texto se recuperan los totales y movimientos del turno.
- **SC-003**: El resultado es equivalente en las 4 combinaciones de {tema claro, tema oscuro} ×
  {gráficos de fondo activados, desactivados}.
- **SC-004**: Un reporte de 60 o más movimientos se imprime en varias hojas y **ninguna** sale sin
  contenido.
- **SC-005**: Cero cambios visibles en la impresión de recibo de venta, recibo de mesa y hoja de QR.
- **SC-006**: Un cajero sin ayuda técnica imprime o guarda el reporte en menos de 30 segundos desde que
  lo ve en pantalla, en ambas rutas.

## Assumptions

- El problema se reproduce en **Brave** (Chromium) de escritorio con la vista previa de impresión y
  "Guardar como PDF"; se asume que el comportamiento de Chrome es equivalente, aunque la
  verificación obligatoria es en Brave.
- El arreglo de la spec 089 (rama `fix/089-cash-report-print`, **aún no fusionada a `develop`** en el
  momento de esta spec) se considera **aplicado** para el diagnóstico, según lo confirmó el usuario.
  Si al reproducir se comprueba que la versión probada no lo incluía, el plan lo registra y la
  corrección se reduce a integrarlo y verificarlo; el resto de requisitos sigue vigente.
- Firefox, Safari, móviles e impresoras físicas o térmicas **no** son objetivo de esta spec; quedan
  fuera de alcance salvo que el arreglo les sirva sin costo adicional.
- No hay cambios de datos, de backend ni de contratos: se trata de presentación de impresión en el
  cliente web.
- El nombre del archivo `<negocio>-<DD-MM-YYYY>` y el ocultamiento de menú, encabezado y botones
  (spec 087) son comportamiento vigente que esta spec conserva (Principio II).
- El idioma de toda la documentación es español de Colombia (Principio XIII).
