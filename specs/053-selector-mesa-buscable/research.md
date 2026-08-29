# Research: Selector de mesa buscable en la creación de orden manual

Todas las incógnitas de esta spec son de diseño técnico dentro del frontend `pos-heladeria`
(Angular); no hay dependencias externas nuevas que investigar ni cambios de backend. Cada decisión
cita el código real leído para tomarla.

## D1 — Reutilizar el select buscable ya existente en el proyecto

**Decision**: Reutilizar `SearchableSelectComponent`
(`pos-heladeria/src/app/shared/searchable-select/searchable-select.component.ts`, selector
`app-searchable-select`), ya usado hoy en `product-form.component.ts`,
`purchase-form.component.ts`, `option-form.component.ts` e `inventory-page.component.ts`. Es un
`ControlValueAccessor` (`[ngModel]`/`(ngModelChange)`), con botón que abre un panel flotante con
buscador (`normalizeText`, insensible a tildes/mayúsculas — ya resuelve FR-002), lista filtrada,
navegación por teclado (flechas + Enter) y cierre por click afuera o `Escape`.

**Rationale**: no existe ningún otro select/combobox ni librería de terceros en el proyecto
(`package.json`: sin `ng-select`/`primeng`/Angular Material) — construir uno nuevo duplicaría
exactamente lo que este componente ya resuelve, incluida la búsqueda insensible a tildes que
`SearchableSelectComponent.spec.ts` ya protege con tests. Es la spec.md, Assumptions, aplicada.

**Alternatives considered**: construir un componente nuevo específico para mesas — rechazado,
violaría Principio V (no reinventar algo que ya existe y ya está probado) sin ninguna necesidad
real; el único gap real (mostrar nombre + estado, y mesas no seleccionables) se resuelve extendiendo
el componente compartido (D2), no reemplazándolo.

## D2 — Cómo mostrar "nombre + estado" y mesas no seleccionables (FR-003, FR-004)

**Decision**:
1. `SearchableSelectOption` (hoy `{ id: string; label: string }`) gana un tercer campo opcional:
   `disabled?: boolean`. Los 4 consumidores existentes no lo pasan, por lo que siguen funcionando
   idénticos (`undefined` es falsy).
2. `selectOption(opt)` retorna sin efecto si `opt.disabled` es `true` — cubre tanto el click como
   `Enter` (ambos llaman a `selectOption`), sin necesidad de guardas duplicadas.
3. El `<li>` de cada opción agrega una clase condicional (`text-gray-300 cursor-not-allowed` si
   `o.disabled`, si no la combinación resaltado/hover que ya existía) para que se vea claramente no
   seleccionable, sin ocultarla.
4. El `label` de cada mesa combina nombre y estado en un único string: `` `${nombre} — ${estado}` ``
   (p. ej. `"Mesa 3 — Libre"`), calculado en `manual-order-page.component.ts` (no en el componente
   compartido, que no sabe nada de mesas).

**Rationale**: es la extensión mínima que cierra exactamente el gap que spec.md pide (mesas
ocupadas visibles pero no seleccionables, con nombre + estado) sin tocar la lógica de filtrado ni
de teclado ya existente y ya probada. Combinar nombre y estado en un solo `label` evita agregar un
segundo campo visual (`sublabel`) al componente compartido — un cambio de superficie mayor sobre 4
consumidores existentes — para un beneficio puramente cosmético (una segunda línea/badge) que
spec.md no exige explícitamente (solo pide que ambos datos "se muestren", no en qué layout).

**Alternatives considered**:
- *Filtrar las mesas ocupadas fuera de `options` en vez de marcarlas `disabled`*: rechazado —
  contradice FR-004 (deben seguir siendo visibles, mismo criterio que la rejilla que reemplaza, que
  las mostraba deshabilitadas, no las ocultaba).
- *Agregar `sublabel` en vez de combinar todo en `label`*: rechazado por alcance — exige tocar el
  template del componente compartido en dos lugares (botón cerrado + lista) y no aporta nada que
  spec.md pida explícitamente; se puede reconsiderar en una spec futura si el negocio pide un
  layout de dos líneas.

## D3 — Etiqueta "nombre de la mesa" cuando la mesa no tiene nombre personalizado

**Decision (corregida tras implementación)**: `` `Mesa ${t.number}${t.name ? ` - ${t.name}` : ''} -
${t.statusLabel}` `` — el número de mesa **siempre** encabeza la etiqueta; el nombre personalizado
(`Table.name`, cuando está poblado) se agrega después, y el estado al final. Ejemplos: "Mesa 1 -
Libre" (sin nombre propio) o "Mesa 1 - Terraza - Libre" (con nombre propio "Terraza").

**Decisión original (implementada primero, corregida el mismo día)**: `t.name ?? \`Mesa
${t.number}\`` — usaba el nombre personalizado **en vez de** "Mesa {número}" cuando existía,
perdiendo el número. El dueño/desarrollador reportó el gap tras ver el resultado ("faltó incluir el
número de la mesa en el selector, por ejemplo Mesa 1 - terraza - libre") — corregido antes de
commitear ningún cambio de esta spec.

**Rationale**: el número de mesa es el identificador primario que usa el resto de la aplicación
(tarjetas de la Terminal de Mesas, "Nueva orden" → "Mesa N") — un nombre personalizado
("Terraza", "Ventana") es un alias adicional, no un reemplazo del número; ocultarlo cuando hay
nombre propio rompía la forma en que el negocio ya identifica sus mesas en todo el resto del
sistema.

**Alternatives considered**: usar siempre `M{{ t.number }}` (la forma corta que tenía la rejilla) —
rechazado, era una abreviación aceptable solo por el espacio angosto de un botón de rejilla; en un
listado de texto libre "Mesa 3" es más claro y consistente con el resto de la pantalla (post-052).

## D4 — Estado del listado sin coincidencias (Edge Cases)

**Decision**: Ninguno — `SearchableSelectComponent` ya renderiza `"Sin resultados"` cuando
`filteredOptions()` devuelve `[]` (línea 61-63 del componente). No requiere ningún cambio.

**Rationale**: es exactamente el edge case que spec.md pide cubrir, y ya está resuelto por el
componente que se reutiliza (D1).

## Resumen de impacto en tests existentes

- `searchable-select.component.spec.ts` (8 casos): ninguno pasa `disabled` hoy, así que los 8 casos
  existentes no cambian de comportamiento con la extensión de D2 (campo opcional, sin afectar
  `filteredOptions()`/`selectOption()` para opciones sin `disabled`). Se agrega cobertura nueva para
  el campo `disabled`.
- `manual-order-page.component.spec.ts` (post-052, 17 casos): los casos que buscan botones "M2"/"M3"
  por `textContent` (`querySelectorAll('button')...textContent?.includes('M2')`) dejan de encontrar
  esos botones — el picker deja de ser una rejilla de `<button>`, pasa a ser
  `app-searchable-select`. Estos casos se reescriben para abrir el select
  (`store.tablesView()`/clic en el botón `app-searchable-select`, escribir en el buscador, clic en
  la opción) en vez de buscar un botón `M{n}` directo — mismo comportamiento final
  (`store.selectedTableId()` cambia), interacción distinta. El resto de los casos (imagen de
  producto, título "Mesas", ubicación en el panel derecho, ancho del panel) no dependen de cómo se
  implementa la selección de mesa en sí, así que no cambian.
- El caso de spec 052 "el listado de mesas usa una rejilla, sin scroll horizontal" queda retirado
  (no adaptado): verificaba exactamente el `grid grid-cols-4` que esta spec reemplaza por completo.
  El nuevo caso de esta spec ("el listado de mesas ya no es una rejilla de botones, sino un select
  buscable") lo sucede como la prueba vigente de ese mismo bloque de UI.
- 0 tests `"CONGELA comportamiento actual:"` en `pos-heladeria/src/` (mismo hallazgo que specs
  045-052).
