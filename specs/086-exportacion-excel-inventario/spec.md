# Feature Specification: Exportación de Inventario a Excel

**Feature Branch**: `086-exportacion-excel-inventario`

**Created**: 2026-09-22

**Status**: Draft

**Input**: User description: "## 1. Objetivo y Contexto de Negocio
Implementar la funcionalidad de exportación a Excel en el módulo de Inventario. El objetivo
es permitir a los administradores descargar un respaldo o reporte de todo el stock de
insumos en formato `.xlsx` para realizar cruces de información, auditorías o contabilidad
externa.

## 2. Usuarios
- **Administrador / Gestor de Inventario:** Único rol con permisos para visualizar la tabla
  de inventario y, por ende, para exportar los datos.

## 3. Escenarios de Usuario (Historias)
- **Historia 1 (Descarga de reporte):** Como administrador, quiero ver un botón de
  "Exportar Inventario" en la barra de acciones superior del inventario, para que, al hacer
  clic, se genere y descargue automáticamente un archivo `.xlsx` con la información actual.
- **Historia 2 (Consistencia de columnas):** Como administrador, espero que el archivo de
  Excel contenga las mismas columnas que veo en la pantalla (Nombre, Tipo, Unidad, Stock,
  Mínimo, Costo, Estado) para que los datos sean fácilmente reconocibles.

## 4. Requisitos Funcionales
1. UI - Botón de Exportar: añadir un botón o ícono de "Exportar" en el Action Bar superior,
   alineado a la derecha junto a los filtros y el badge contador de insumos.
2. Backend - Generación de Archivo: crear un endpoint que consulte la base de datos de
   insumos, construya un archivo `.xlsx` y lo retorne como un flujo descargable.
3. Contenido del Excel: fila 1 con encabezados (Nombre, Tipo, Unidad, Stock, Mínimo, Costo,
   Estado); fila 2 en adelante con la data de cada insumo; las cantidades numéricas deben
   exportarse con formato numérico, no como texto.

## 5. Reglas de Negocio
- La exportación debe traer el total del inventario consolidado.
- Los costos deben exportarse manteniendo sus decimales originales, sin truncar.
- Ejemplo: si el inventario tiene 120 insumos registrados, el archivo debe contener
  exactamente 121 filas (1 encabezado + 120 datos); un costo de $15.00 debe quedar como el
  valor numérico 15.00, no como el string "$15.00".

## 6. Criterios de Aceptación
1. Existe un botón visible para exportar a Excel en la vista de inventario.
2. Al hacer clic, el navegador descarga un archivo .xlsx.
3. Las columnas del archivo coinciden con las de la tabla visual y los datos son fieles al
   listado de insumos.
4. Las columnas numéricas permiten autosuma en Excel (no son strings).

## 7. Casos Límite
- Inventario vacío: debe descargarse un Excel válido solo con la fila de encabezados.
- Nombres largos o con caracteres especiales (tildes, comillas, eñes) deben renderizarse
  correctamente (UTF-8)."

## Clarifications

### Session 2026-09-22

- Q: ¿La exportación a Excel debe traer siempre el total absoluto del inventario, o debe
  respetar los filtros/búsqueda que el usuario tenga activos en pantalla en ese momento? →
  A: Siempre el total absoluto del inventario, sin importar los filtros, la búsqueda o el
  orden que el usuario tenga aplicados en la tabla en ese momento.
- Q: Cuando el archivo exportado trae "el total absoluto del inventario", ¿ese total debe
  incluir los insumos marcados como inactivos (dados de baja), o solo los insumos activos? →
  A: Debe incluir todos los insumos registrados, activos e inactivos, sin agregar una
  columna nueva que los distinga; el archivo mantiene exactamente las columnas definidas en
  FR-004.
- Q: FR-001 menciona alinear el botón "Exportar Inventario" junto a un "contador de
  insumos" en la barra de acciones superior de Inventario, pero ese contador no existe
  ahí — fue excluido deliberadamente por la Spec 085 (implementada sobre el mismo archivo
  inmediatamente antes de esta spec). ¿Se agrega ese contador como parte de esta spec, o
  se corrige la referencia? → A: Se corrige la referencia; el botón se ubica junto al
  buscador y los filtros de Tipo/Estado que sí existen hoy en esa barra. No se agrega un
  contador de insumos nuevo — está fuera del alcance de esta spec (exportación a Excel).

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Descarga del reporte completo de inventario (Priority: P1)

Como administrador o gestor de inventario, quiero presionar un botón "Exportar Inventario"
desde la pantalla de Inventario y recibir automáticamente un archivo `.xlsx` descargado con
el respaldo completo del stock de insumos, para poder usarlo en cruces de información,
auditorías o contabilidad externa sin depender de otra herramienta.

**Why this priority**: Es el valor central de la funcionalidad — sin la descarga del
archivo no existe la funcionalidad. Es la única historia que, por sí sola, ya entrega un
producto mínimo viable y verificable.

**Independent Test**: Puede probarse por completo entrando a la pantalla de Inventario,
presionando "Exportar Inventario" y verificando que el navegador descarga un archivo
`.xlsx` que abre sin errores.

**Acceptance Scenarios**:

1. **Given** el administrador está en la pantalla de Inventario con insumos registrados,
   **When** presiona el botón "Exportar Inventario", **Then** el navegador descarga
   automáticamente un archivo `.xlsx` sin pasos adicionales.
2. **Given** el inventario tiene 120 insumos registrados en total, entre activos e
   inactivos, **When** el administrador exporta, sin importar qué filtro, búsqueda o
   estado activo/inactivo tenga activo en pantalla, **Then** el archivo descargado
   contiene exactamente 121 filas: 1 de encabezado más 120 de datos, correspondientes al
   total consolidado del inventario.
3. **Given** el inventario no tiene ningún insumo registrado, **When** el administrador
   presiona "Exportar Inventario", **Then** el archivo descargado es un `.xlsx` válido que
   contiene únicamente la fila de encabezados.

---

### User Story 2 - Consistencia de columnas y datos exportados (Priority: P2)

Como administrador, espero que el archivo de Excel contenga exactamente las mismas columnas
que veo hoy en la tabla de Inventario (Nombre, Tipo, Unidad, Stock, Mínimo, Costo, Estado)
y que los valores numéricos queden como números reales de Excel, para poder reconocer los
datos de inmediato y operar matemáticamente sobre ellos (sumas, promedios) sin tener que
convertir texto a número primero.

**Why this priority**: Depende de que la Historia 1 exista (el archivo ya se descargue),
pero es lo que hace que el archivo sea útil para auditoría y contabilidad en vez de solo un
volcado de datos poco confiable.

**Independent Test**: Puede probarse abriendo el archivo exportado, comparando el orden y
nombre de columnas contra la tabla en pantalla, y seleccionando la columna "Costo" o
"Stock" en Excel para confirmar que la barra de estado muestra una suma automática.

**Acceptance Scenarios**:

1. **Given** un archivo de inventario exportado, **When** se abre en una hoja de cálculo,
   **Then** la fila 1 contiene los encabezados "Nombre", "Tipo", "Unidad", "Stock",
   "Mínimo", "Costo", "Estado", en ese orden.
2. **Given** un insumo llamado "Arequipe" con costo registrado de $15.00, **When** se
   revisa su fila en el archivo exportado, **Then** la celda de la columna "Costo"
   contiene el valor numérico `15.00` (no el texto "$15.00").
3. **Given** un insumo llamado "Champiñón" con tilde y eñe en su nombre, **When** se abre
   el archivo exportado, **Then** el nombre se muestra correctamente, sin caracteres
   corruptos.
4. **Given** el archivo exportado abierto en una hoja de cálculo, **When** se selecciona
   la columna "Costo" o "Stock" completa, **Then** la herramienta de hoja de cálculo puede
   calcular automáticamente su suma, confirmando que los valores son numéricos y no texto.

---

### Edge Cases

- Inventario vacío: el archivo exportado debe ser un `.xlsx` válido que contenga
  únicamente la fila de encabezados (0 filas de datos).
- Nombres de insumos con tildes, eñes, comillas u otros caracteres especiales deben
  renderizarse correctamente en el archivo, sin símbolos corruptos.
- Un usuario sin el rol de Administrador/Gestor de Inventario no debe poder generar el
  archivo de exportación, ya que tampoco tiene acceso a la pantalla de Inventario ni a su
  tabla.
- Insumos inactivos (dados de baja): se incluyen en el archivo exportado igual que los
  insumos activos, sin una columna adicional que los distinga; el archivo mantiene
  exactamente las columnas definidas en FR-004.
- Si ocurre un error al generar el archivo (por ejemplo, un fallo de conexión a la base de
  datos a mitad del proceso), el sistema debe informar el error al administrador y no debe
  entregar un archivo descargado corrupto o incompleto.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema MUST mostrar un botón "Exportar Inventario" visible en la barra
  de acciones superior de la pantalla de Inventario, alineado a la derecha junto al
  buscador y los selectores de filtro Tipo/Estado, visible únicamente para usuarios con el
  rol que hoy tiene acceso a la tabla de Inventario (Administrador/Gestor de Inventario).
- **FR-002**: Al presionar el botón "Exportar Inventario", el sistema MUST generar y
  descargar automáticamente un archivo en formato `.xlsx`, sin pasos ni confirmaciones
  adicionales.
- **FR-003**: El archivo exportado MUST incluir el total absoluto de insumos registrados
  en el inventario, tanto activos como inactivos, sin importar los filtros, la búsqueda o
  el orden que el usuario tenga activos en la pantalla en el momento de exportar.
- **FR-004**: La primera fila del archivo exportado MUST contener los encabezados de
  columna "Nombre", "Tipo", "Unidad", "Stock", "Mínimo", "Costo", "Estado", en ese orden.
- **FR-005**: A partir de la segunda fila, el archivo MUST contener exactamente una fila
  por cada insumo registrado en el inventario, con los mismos valores que hoy se muestran
  para esas columnas en la tabla de Inventario en pantalla.
- **FR-006**: Las columnas numéricas (Stock, Mínimo, Costo) MUST exportarse como valores
  numéricos nativos de la hoja de cálculo (no como texto), conservando sus decimales
  originales sin truncar, de forma que permitan sumas u otras operaciones matemáticas
  directamente sobre esa columna.
- **FR-007**: El sistema MUST codificar el archivo exportado de forma que los nombres de
  insumos con tildes, eñes, comillas u otros caracteres especiales propios del español se
  muestren correctamente al abrirlo.
- **FR-008**: Si el inventario no tiene ningún insumo registrado, el sistema MUST generar
  y descargar igualmente un archivo `.xlsx` válido que contenga únicamente la fila de
  encabezados.
- **FR-009**: El sistema MUST rechazar la generación del archivo de exportación para
  cualquier usuario que no tenga el rol autorizado para ver la tabla de Inventario.

### Key Entities *(include if feature involves data)*

- **Insumo de Inventario**: entidad ya existente en el sistema; para esta funcionalidad
  interesan sus atributos Nombre, Tipo, Unidad, Stock, Mínimo, Costo y Estado, que son los
  que se vuelcan a cada fila del archivo exportado. Esta funcionalidad no agrega ni
  modifica atributos del insumo, solo los lee para construir el reporte.
- **Reporte de Exportación (archivo .xlsx)**: archivo generado bajo demanda en el momento
  en que el administrador presiona "Exportar Inventario"; no se persiste ni se guarda un
  historial de exportaciones, se entrega directamente como descarga al navegador.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Un administrador puede completar la descarga del reporte de inventario en un
  solo clic desde la pantalla de Inventario, sin pasos de configuración adicionales.
- **SC-002**: El archivo descargado contiene siempre exactamente una fila de encabezado
  más una fila por cada insumo registrado en el inventario, activo o inactivo (por ejemplo,
  120 insumos producen 121 filas), sin variar según los filtros, la búsqueda o el estado
  activo/inactivo aplicados en pantalla.
- **SC-003**: El 100% de los valores en las columnas Stock, Mínimo y Costo del archivo
  exportado permite operaciones de suma nativas de la hoja de cálculo, sin requerir
  conversión manual de texto a número.
- **SC-004**: El 100% de los nombres de insumos con tildes, eñes o caracteres especiales
  se visualiza correctamente, sin símbolos corruptos, al abrir el archivo exportado.

## Assumptions

- El rol autorizado para ver la tabla de Inventario (Administrador/Gestor de Inventario)
  es el mismo autorizado para exportarla; esta funcionalidad no introduce un permiso nuevo
  y diferenciado.
- El nombre del archivo descargado incluye la fecha de generación para diferenciarlo de
  descargas previas (por ejemplo, "inventario_2026-09-22.xlsx"); el formato exacto del
  nombre queda a criterio de la fase de planeación.
- Los valores que hoy calcula y muestra la columna "Estado" en la tabla de Inventario en
  pantalla (por ejemplo "Normal", "Stock bajo", "Agotado") son los mismos que se exportan;
  esta funcionalidad no introduce nuevas reglas para calcular el estado de un insumo.
- La exportación es síncrona: el administrador espera la descarga dentro de la misma
  interacción, sin necesidad de notificación posterior ni generación en segundo plano.
- Esta funcionalidad no requiere guardar un historial de exportaciones ni registrar
  auditoría de quién exportó y cuándo; cubre únicamente la generación y descarga del
  archivo.
- El volumen actual de insumos del negocio no representa un riesgo de rendimiento
  relevante para una generación síncrona del archivo en el momento del clic.
