# Phase 0 Research: Mejora de Visibilidad del Buscador y Filtros en Inventario

El spec no dejó ningún `NEEDS CLARIFICATION` pendiente (ver [spec.md](./spec.md), sección
Assumptions y Clarifications). La investigación de esta fase es puramente técnica: cómo
lograr, sobre el componente real, la mejora visual que la sesión de clarificación del
2026-09-22 fijó como objetivo — fusionar la zona de búsqueda/filtros como encabezado de la
tarjeta de la tabla, replicando como referencia visual exacta el mockup
`design/inventory-search-filters-redesign.html`.

## ⚠️ Estado actual del código (revisión de esta versión de research.md)

`inventory-page.component.ts` (`pos-heladeria`) tiene **hoy** implementada una iteración
**anterior** de este mismo spec 085, resultado de `tasks.md` T001-T013 (rama
`feat/085-improve-inventory-filters-visibility`):

- Un `<div>` de filtros **separado**, hermano directo de la tabla (líneas 111-131):
  `bg-indigo-50/40 rounded-xl border border-indigo-100 p-4 flex flex-wrap gap-3`, apilado
  con `space-y-6` sobre el `<div>` de la tabla.
- El `<div>` de la tabla (líneas 133-199): `bg-white rounded-xl border border-gray-100
  overflow-hidden`, sin ningún cambio.
- Buscador con ícono de lupa (`app-mi-icon name="search"`) ya agregado, `pl-9 pr-3`.
- Los tres controles (input + 2 selects) con `border-gray-300` uniforme.

Esa iteración se basaba en las decisiones *previas* de este `research.md` (tarjeta separada,
acento `indigo` genérico de Tailwind) — decisiones que la sesión de clarificación del mismo
día (2026-09-22, ver spec.md) **reemplazó explícitamente** antes de que existiera el mockup.
Este documento describe el **nuevo objetivo**; reconciliar el código ya implementado con este
nuevo objetivo es trabajo de implementación pendiente, fuera del alcance de `/speckit-plan` —
debe replanificarse en `tasks.md` vía `/speckit-tasks` y re-ejecutarse.

## 1. ¿La zona de búsqueda/filtros debe seguir siendo una tarjeta separada de la tabla, o fusionarse con ella?

- **Decisión**: Fusionar la zona de búsqueda y filtros como encabezado superior de la
  **misma tarjeta** que contiene la tabla de resultados, separada de las filas por un borde
  (`border-b border-gray-200`), replicando la estructura `data-purpose="table-card-header"`
  del mockup (`design/inventory-search-filters-redesign.html`, líneas 259-304): el
  `<div>` de filtros deja de ser hermano de la tabla y pasa a ser su primer hijo, dentro del
  mismo `<div class="bg-white rounded-xl ... overflow-hidden">`.
- **Rationale**: la clarificación Q1 de spec.md es explícita: "Fusionarla como encabezado
  superior de la misma tarjeta de la tabla: un único contenedor blanco redondeado que agrupa
  buscador+filtros arriba ... y la tabla debajo, separados entre sí por un borde. Esto
  reemplaza la decisión previa de `research.md` de una tarjeta separada." FR-001 codifica lo
  mismo.
- **Alternativas consideradas**: mantener la tarjeta separada ya implementada en el código
  actual — descartada porque la clarificación Q1 la reemplaza explícitamente como decisión ya
  tomada con el usuario a partir del mockup de referencia.

## 2. ¿Qué acento de color usar: el `indigo` genérico de Tailwind ya usado en el resto del producto, o el color exacto del mockup?

- **Decisión**: usar el acento morado exacto del mockup, `#4a3aff`, vía clases Tailwind de
  **valor arbitrario** (p. ej. `focus:ring-[#4a3aff]/30 focus:border-[#4a3aff]`), en lugar
  del `indigo` genérico de Tailwind (`indigo-500`/`indigo-600`) que usa el resto del
  producto (botón "Nuevo insumo", pestaña activa, anillos de foco por defecto). Esto
  reemplaza la decisión previa de este `research.md` de reutilizar el acento de marca
  genérico.
- **Rationale**: la clarificación Q3 de spec.md fija `design/inventory-search-filters-redesign.html`
  como "referencia visual exacta ... colores, bordes, espaciados y estructura ... incluso
  donde difieran de tokens usados hoy en otras pantallas del producto (por ejemplo, el
  acento morado `#4a3aff` del mockup en lugar del índigo genérico usado en otras pantallas)."
  Las Assumptions de spec.md confirman lo mismo.
- **Alcance del color exacto** (para no exceder lo que spec.md autoriza): se aplica
  únicamente al subárbol `data-purpose="table-card-header"` del mockup (buscador + selects)
  y al contenedor de la tarjeta que lo envuelve (`shadow-sm border-slate-200/80` en vez de
  `border-gray-100`, ver punto 4). **No** se extiende al resto de la página del mockup —
  sidebar morado de fondo completo, header superior, fondo de página `#f8fafc`/`#fafbfe` —
  ni a las filas, columnas o datos de la tabla, que quedan fuera de alcance de spec 085
  (Contexto: "cubre **exclusivamente** el ajuste visual de esa zona"; FR-008; Assumptions:
  "Las filas, columnas y datos de la tabla de resultados tampoco cambian de apariencia").
- **Implementación**: clase Tailwind de valor arbitrario (`[#4a3aff]`) directamente en el
  template de este componente, **sin** extender `tailwind.config` con un nuevo token de
  marca compartido — así el cambio de color no se filtra a otras pantallas que sí usan hoy
  el acento `indigo` genérico (botón "Nuevo insumo", pestaña activa, etc.), que no forman
  parte del alcance de este spec y no deben cambiar.
- **Alternativas consideradas**:
  - Extender `tailwind.config` con un token `brand-085` o similar — descartada por alcance
    (Principio V/VI de la constitución): una config de Tailwind es compartida por toda la
    app; introducir un token ahí para un solo componente excede el spec y arriesga afectar
    pantallas fuera de alcance.
  - Reutilizar el `indigo` genérico ya usado en el resto del producto (decisión de la
    iteración anterior) — descartada porque la clarificación Q3 la reemplazó explícitamente.

## 3. ¿Cómo reforzar el campo de búsqueda específicamente (FR-002) sin lógica nueva?

- **Decisión**: mantener `app-mi-icon name="search"` dentro del input (ya agregado en la
  iteración anterior, ya catalogado en `icon-catalog.ts`), ajustando el padding del input a
  `pl-10 pr-3` para calzar exactamente con el mockup (línea 270 del mockup: `pl-10 pr-12` —
  el `pr-12` del mockup reserva espacio para el badge "⌘K", que esta spec excluye
  explícitamente por FR-009, por lo que se usa `pr-3` en su lugar, suficiente sin ese
  badge). Se agregan además `bg-white text-slate-800 placeholder-slate-400 shadow-sm
  transition-all` y `rounded-lg` (el mockup usa `rounded-lg`, no `rounded-xl`, para los
  controles individuales) para calzar con el resto de clases del input del mockup.
- **Rationale**: sigue siendo la señal convencional de "campo de búsqueda"; el ajuste de
  padding y clases adicionales es la única forma de calzar exactamente con el mockup (Q3)
  sin arrastrar el badge "⌘K" que la clarificación Q2 excluye.
- **Alternativas consideradas**: usar `pr-12` igual que el mockup dejando el espacio vacío
  sin badge — descartada por generar un padding derecho injustificado y visualmente extraño
  una vez removido el badge que lo motivaba; `<input type="search">` nativo — descartada
  (igual que en la iteración anterior) porque cambia comportamiento del control
  (FR-005/FR-007).

## 4. ¿Cómo agrupar y separar visualmente el buscador de los dos selects entre sí (FR-003)?

- **Decisión**: replicar la agrupación izquierda/derecha del mockup en vez del
  `flex flex-wrap gap-3` plano de la iteración anterior (donde los tres controles eran
  hermanos directos sin agrupación):
  - Contenedor del encabezado fusionado: `flex items-center justify-between gap-4 p-4
    border-b border-gray-200 bg-gray-50/50 flex-wrap` (se agrega `flex-wrap` sobre el
    `justify-between` del mockup, ausente en el mockup porque asume suficiente ancho, para
    seguir cumpliendo FR-006 en ventanas angostas).
  - Buscador: `<div class="relative flex-1 max-w-md">` a la izquierda.
  - Selects: agrupados en `<div class="flex items-center gap-3 flex-wrap">` a la derecha.
  - Cada `<select>` se envuelve en `<div class="relative">` con `appearance-none` +
    un ícono de flecha (chevron) SVG decorativo posicionado en forma absoluta, replicando
    la estructura del mockup (líneas 276-299) — no solo su color. `[ngModel]`,
    `(ngModelChange)` y las `<option>` de ambos selects no cambian.
  - Contenedor de la tarjeta que envuelve todo: `bg-white rounded-xl shadow-sm border
    border-slate-200/80 overflow-hidden`, reemplazando el `border-gray-100` sin `shadow-sm`
    de hoy — es parte de la "tarjeta contenedora" que las Assumptions de spec.md
    explícitamente permiten ajustar al fusionar el encabezado, distinta de "el resto de la
    tabla" (filas/columnas/datos) que permanece intacto.
- **Rationale**: FR-003 exige distinguir cada control como interactivo y separado entre sí;
  la clarificación Q3 exige además replicar la estructura del mockup, no solo su color —de
  ahí incorporar también el wrapper de flecha personalizada de los selects y el
  `shadow-sm`/`border-slate-200/80` del contenedor, que son parte de esa estructura.
- **Alternativas consideradas**: mantener los tres controles como hermanos directos en un
  único `flex flex-wrap gap-3` (decisión de la iteración anterior) — descartada porque ya no
  calza con la agrupación izquierda/derecha del mockup que la clarificación fija como
  referencia exacta; usar la flecha nativa del `<select>` sin wrapper personalizado —
  descartada por la misma razón (la "estructura" exacta del mockup incluye ese wrapper).

## 5. Verificación de contraste AA (FR-004, SC-005)

- **Decisión**: el texto permanece en tonos oscuros sobre fondos claros
  (`text-slate-800`/`text-slate-700` sobre `white`/`bg-gray-50/50`), igual criterio que
  antes; el acento `#4a3aff` se usa únicamente para bordes y anillos de foco (no como color
  de texto), donde el requisito de contraste AA es el de elementos gráficos/UI (3:1), no el
  de texto (4.5:1). Contraste aproximado de `#4a3aff` sobre blanco: ≈ 6.3:1 (cálculo de
  luminancia relativa WCAG), muy por encima del mínimo de 3:1 exigido para bordes/anillos de
  foco. Se valida de forma concreta en la fase de implementación con el checker de
  contraste de las herramientas de desarrollo del navegador (ver `quickstart.md`).
- **Rationale**: reutilizar la misma estrategia de texto oscuro/fondo claro que el resto del
  producto ya usa (y que nunca ha sido reportada como problema de contraste), aplicándola
  ahora también al color de acento nuevo (`#4a3aff`) que reemplaza al `indigo` genérico.
- **Alternativas consideradas**: usar `#4a3aff` como color de texto (p. ej. para labels) —
  no aplica: ni el mockup ni esta spec introducen texto nuevo en la zona (FR-009 prohíbe
  agregar elementos nuevos como el contador "N Insumos"), así que no hay texto en ese color
  que verificar más allá de bordes/anillos.

## 6. Estrategia de verificación (Principio X)

- **Decisión**: (a) la suite `ng test` existente debe seguir en verde sin modificaciones —
  `inventory-page.component.spec.ts` (spec 082, íconos) no ejercita el bloque de filtros ni
  la tarjeta de la tabla, así que un cambio de clases/estructura no la afecta; (b) no se
  añaden tests automatizados nuevos porque no hay comportamiento nuevo que probar (cambio de
  presentación); (c) la verificación real de SC-001/SC-002/SC-003 es cualitativa y manual,
  documentada en `quickstart.md`, incluyendo el checklist de contraste AA (SC-005), de
  no-regresión funcional (SC-004) y de que los elementos excluidos por FR-009 (badge "⌘K",
  contador de insumos) efectivamente no se agregaron.
- **Nota de esta revisión**: T001-T013 de `tasks.md` ya ejecutaron esta estrategia contra la
  iteración *anterior* del diseño (tarjeta separada, acento `indigo` genérico). Esa
  verificación queda invalidada para el nuevo objetivo (encabezado fusionado, acento
  `#4a3aff`) descrito en este `research.md`; debe repetirse íntegra una vez se regenere
  `tasks.md` (vía `/speckit-tasks`) y se re-implemente el componente.
- **Rationale**: coherente con el Principio X (verificación obligatoria, proporcional al
  tipo de cambio) y con la propia spec, que anticipa que su validación principal es
  cualitativa/de usuario, no una métrica automatizada.
