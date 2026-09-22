# Feature Specification: Mejora de Visibilidad del Buscador y Filtros en Inventario

**Feature Branch**: `085-mejora-visibilidad-filtros-inventario`

**Created**: 2026-09-22

**Status**: Draft

**Input**: User description: "necesito realizar una mejora a nivel visual, este cambio se
implementara en la seccion de inventario, es una mejora a nivel visual solamente ya que el
usuario me dice que para el es poco visible la seccion del buscador, quisiera mejorar la
accesibilidad a esa zona, tanto al buscador como a los filtros que tiene, por temas visuales
no alcanzaba a distinguir donde estaba esa seccion"

## Clarifications

### Session 2026-09-22

- Q: ¿La zona de búsqueda y filtros debe fusionarse dentro de la misma tarjeta de la tabla de resultados (como encabezado superior, pegada arriba de la tabla y separada por un borde, tal como muestra el diseño de referencia en `code.html`), o debe seguir siendo una tarjeta propia y separada de la tabla como plantea hoy la spec? → A: Fusionarla como encabezado superior de la misma tarjeta de la tabla: un único contenedor blanco redondeado que agrupa buscador+filtros arriba (con fondo ligeramente distinto) y la tabla debajo, separados entre sí por un borde. Esto reemplaza la decisión previa de `research.md` de una tarjeta separada.
- Q: El diseño de referencia agrega un badge "⌘K" dentro del campo de búsqueda y una etiqueta con el conteo de insumos (p. ej. "20 Insumos") junto a los filtros, ninguno de los cuales existe hoy en la pantalla; siendo esta spec un cambio exclusivamente visual, ¿qué se hace con estos dos elementos nuevos? → A: Excluirlos de esta spec. No se agrega el badge "⌘K" (mostrar un atajo que no funciona confundiría al usuario) ni el contador de insumos; la zona de búsqueda/filtros adopta el nuevo estilo visual pero mantiene exactamente los mismos elementos y comportamiento de hoy.
- Q: `code.html` usa colores específicos (morado `#4a3aff`, fondos slate/gris) distintos del acento índigo que `research.md` ya había decidido reutilizar; ¿el diseño de `code.html` debe tomarse como referencia visual exacta a implementar, o solo como guía direccional a adaptar con los tokens de marca ya existentes? → A: Referencia visual exacta. `code.html` (guardado como `design/inventory-search-filters-redesign.html` en esta carpeta de spec) es el objetivo visual a implementar tal cual —colores, bordes, espaciados y estructura—, reemplazando la decisión previa de `research.md` de reutilizar el acento índigo genérico.

## Contexto

Un usuario del módulo de Inventario (pestaña "Insumos") reportó que la zona de búsqueda y
filtros —el campo de texto para buscar por nombre, el selector de tipo de insumo y el selector
de estado (activo/inactivo)— es poco visible dentro de la pantalla: no lograba distinguir a
simple vista dónde estaba esa sección respecto al resto del contenido de la página. Esta spec
cubre **exclusivamente** el ajuste visual de esa zona; no cambia ninguna lógica de búsqueda,
filtrado, datos ni comportamiento. Las pestañas "Compras" y "Movimientos" de Inventario no
tienen esta misma barra de búsqueda/filtros y quedan fuera de alcance.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Encontrar la zona de búsqueda y filtros de un vistazo (Priority: P1)

Un usuario que administra el inventario abre la pestaña "Insumos" y necesita ubicar rápidamente
dónde buscar un insumo por nombre, sin tener que leer detenidamente toda la pantalla para
encontrar el campo.

**Why this priority**: Es la queja original y de mayor impacto: si el usuario no puede ubicar el
buscador, no puede usar la función en absoluto, sin importar que esta funcione correctamente por
detrás.

**Independent Test**: Puede probarse mostrando la pantalla de Inventario (pestaña Insumos) a un
usuario y pidiéndole que señale dónde buscaría un insumo por nombre; se considera exitoso si lo
ubica de inmediato, sin necesidad de ningún otro cambio en la spec.

**Acceptance Scenarios**:

1. **Given** el usuario abre la pestaña "Insumos" de Inventario, **When** la página termina de
   cargar, **Then** la zona de búsqueda y filtros se distingue visualmente del fondo de la página
   y de los demás elementos sueltos (encabezado, tarjeta de alerta de "Bajo mínimo") al mostrarse
   como el encabezado superior de la tarjeta de la tabla de resultados, de forma que pueda
   ubicarse en un primer vistazo.
2. **Given** el usuario está mirando la pantalla completa de Inventario, **When** compara la zona
   de búsqueda y filtros con los elementos vecinos, **Then** dicha zona comunica claramente que
   es un área interactiva de entrada de datos y no un bloque de contenido estático más.

---

### User Story 2 - Distinguir el buscador de los filtros dentro de la misma zona (Priority: P2)

Una vez que el usuario ubica la zona de búsqueda y filtros, necesita distinguir con claridad cuál
control es el buscador por nombre y cuáles son los filtros (tipo de insumo, estado
activo/inactivo), para poder usar cada uno sin confundirlos entre sí ni con elementos decorativos.

**Why this priority**: Es un refinamiento sobre la Historia 1: resuelve la queja de que, además
de la zona en general, ni el buscador ni los filtros individuales se distinguían por temas
visuales. Depende de que la zona ya sea visible (Historia 1), por eso es P2.

**Independent Test**: Puede probarse pidiéndole a un usuario que, dentro de la zona ya ubicada,
señale específicamente el campo de búsqueda por nombre y cada uno de los filtros; se considera
exitoso si identifica correctamente cada control sin dudar.

**Acceptance Scenarios**:

1. **Given** el usuario ya identificó la zona de búsqueda y filtros, **When** observa el campo de
   búsqueda por nombre, **Then** ese campo se percibe claramente como un lugar para escribir texto
   de búsqueda.
2. **Given** el usuario ya identificó la zona de búsqueda y filtros, **When** observa los
   selectores de tipo de insumo y de estado, **Then** cada selector se percibe claramente como un
   control de filtro separado del buscador y separado entre sí.

---

### Edge Cases

- ¿Qué pasa cuando la pantalla se ve en un ancho de ventana angosto y el buscador y los filtros
  se acomodan en varias líneas? La zona debe seguir siendo igual de reconocible en ese acomodo.
- ¿Qué pasa cuando el filtro "Bajo mínimo" (la tarjeta de alerta) está activo al mismo tiempo?
  La zona de búsqueda y filtros debe seguir distinguiéndose igual de bien, sin competir
  visualmente con esa tarjeta ni perderse junto a ella.
- ¿Qué pasa con un usuario con baja visión o dificultad para distinguir colores? La mejora no
  puede depender únicamente del color; debe apoyarse también en otras señales visuales (por
  ejemplo, contraste, bordes, espaciado o iconografía).
- ¿Qué pasa con la tabla de resultados y el resto del contenido de la pestaña "Insumos"? Deben
  conservar su apariencia y jerarquía visual actuales; el cambio no debe hacer que otros
  elementos de la pantalla pierdan protagonismo por exceso de énfasis en la zona de búsqueda.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: El sistema DEBE distinguir visualmente la zona de búsqueda y filtros de la pestaña
  "Insumos" de Inventario respecto al fondo de la página y a los demás elementos sueltos de la
  pantalla (encabezado, tarjeta de alerta "Bajo mínimo"), integrándola como el encabezado
  superior de la misma tarjeta que contiene la tabla de resultados —separada de las filas de la
  tabla por un borde—, de modo que un usuario la ubique sin tener que buscarla deliberadamente.
- **FR-002**: El sistema DEBE presentar el campo de búsqueda por nombre de forma que comunique
  visualmente que es un lugar para escribir una búsqueda, y no un elemento más que se confunde
  con el fondo de la página.
- **FR-003**: El sistema DEBE distinguir visualmente cada filtro (tipo de insumo, estado
  activo/inactivo) como un control interactivo de filtrado, separado del contenido estático que
  lo rodea y separado entre sí.
- **FR-004**: El sistema DEBE mantener un contraste suficiente entre la zona de búsqueda y
  filtros y el fondo de la página, cumpliendo pautas estándar de accesibilidad de contraste, para
  que sea legible también para usuarios con baja visión.
- **FR-005**: El sistema DEBE conservar el 100% del comportamiento actual de búsqueda y filtrado
  (búsqueda por nombre, filtro por tipo, filtro por estado activo/inactivo, filtro de "Bajo
  mínimo"): este es un cambio exclusivamente visual, sin cambios de lógica ni de datos.
- **FR-006**: La distinción visual de la zona de búsqueda y filtros DEBE mantenerse legible y
  reconocible en los distintos tamaños de pantalla que ya soporta hoy la página de Inventario
  (incluyendo el acomodo en varias líneas en ventanas angostas).
- **FR-007**: El sistema NO DEBE alterar el orden, los resultados ni el comportamiento de la
  búsqueda o los filtros como parte de este cambio.
- **FR-008**: El sistema DEBE conservar la apariencia y jerarquía visual del resto de la pestaña
  "Insumos" (tarjeta de alerta, tabla de resultados) sin degradarla al resaltar la zona de
  búsqueda y filtros.
- **FR-009**: El sistema NO DEBE agregar elementos nuevos que no existen hoy en la zona de
  búsqueda y filtros (por ejemplo, un indicador de atajo de teclado tipo "⌘K" o una etiqueta con
  el conteo de insumos filtrados): este cambio se limita a re-estilizar los controles ya
  existentes (buscador por nombre, filtro de tipo, filtro de estado), sin incorporar controles,
  indicadores ni datos adicionales.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: En una prueba de visibilidad con usuarios que ven la pantalla de Inventario por
  primera vez, al menos el 90% ubica el campo de búsqueda por nombre en menos de 3 segundos.
- **SC-002**: En la misma prueba, al menos el 90% de los usuarios identifica los filtros de tipo
  y estado como un grupo de controles distinto en menos de 5 segundos.
- **SC-003**: El usuario que reportó el problema original confirma, al revisar el cambio, que
  ahora distingue con claridad y de un vistazo dónde está la zona de búsqueda y filtros.
- **SC-004**: Cero regresiones funcionales reportadas en la búsqueda o los filtros de Inventario
  tras publicar el cambio (los resultados de búsqueda y filtrado son idénticos a los de antes del
  cambio).
- **SC-005**: La zona de búsqueda y filtros cumple con las pautas estándar de contraste de texto
  y fondo (nivel AA de WCAG), verificado mediante una revisión de accesibilidad.

## Assumptions

- El alcance de esta mejora es exclusivamente la zona de búsqueda y filtros de la pestaña
  "Insumos" de Inventario (campo "Buscar por nombre...", selector de tipo, selector de estado),
  que es la que el usuario describió como poco visible. Las pestañas "Compras" y "Movimientos" no
  tienen esta misma barra y quedan fuera de alcance.
- La tarjeta de alerta "Bajo mínimo" no cambia de apariencia. Las filas, columnas y datos de la
  tabla de resultados tampoco cambian de apariencia; solo su tarjeta contenedora incorpora la
  zona de búsqueda y filtros como encabezado superior fusionado (ver Clarifications), sin alterar
  el resto de la tabla.
- El cambio se implementará siguiendo como referencia visual exacta el mockup
  `design/inventory-search-filters-redesign.html` (provisto por el usuario, ver Clarifications):
  colores, bordes, espaciados y la estructura de encabezado de tabla fusionado que allí se
  muestran son el objetivo a implementar, incluso donde difieran de tokens usados hoy en otras
  pantallas del producto (por ejemplo, el acento morado `#4a3aff` del mockup en lugar del índigo
  genérico usado en otras pantallas). Esto reemplaza la suposición anterior de reutilizar
  únicamente patrones visuales ya existentes sin introducir lenguaje visual nuevo.
- No se requiere ningún cambio en la lógica de negocio, en la API ni en el modelo de datos: es un
  ajuste de presentación únicamente.
- La validación de esta mejora es principalmente cualitativa (confirmación del usuario que
  reportó el problema) más que una métrica cuantitativa formal, dado que se trata de un cambio de
  percepción visual y no de una funcionalidad nueva.
