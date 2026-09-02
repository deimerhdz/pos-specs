# Contrato: búsqueda y selección de productos en una regla (FR-006 a FR-011)

**Normativo.** Reemplaza `visibleVariantsForRule` de
`promotions-page.component.ts:867-875` por dos métodos con responsabilidades separadas. Sin
cambios de API: todo opera sobre `catalogVariants()`, ya cargado en el cliente.

---

## 1. Los dos listados

| Método | Reemplaza a | Entrada | Regla |
|---|---|---|---|
| `selectedVariantsForRule(ruleIndex)` | (ya existe, sin cambio de firma) | `rule.variantIds` | Devuelve las variantes del catálogo cuyo id está en el conjunto de la regla. Es el listado de FR-006 — nunca depende del filtro. |
| `searchResultsForRule(ruleIndex)` | `visibleVariantsForRule` | `ruleFilters[ruleIndex]` (`{category, text}`) | Ver tabla de FR-007/FR-008 abajo. Es independiente de si una variante ya está o no en el conjunto. |

El *template* pinta `selectedVariantsForRule` como el listado principal ("CONJUNTO (N)"), y
`searchResultsForRule` como un listado aparte que solo tiene filas cuando no está vacío — visible
mientras el administrador está buscando.

## 2. `searchResultsForRule` — tabla de casos (FR-007, FR-008)

| `category` | `text` (trim) | Resultado |
|---|---|---|
| `''` (Todas las categorías) | `''` | `[]` — nunca lista el catálogo completo por defecto (FR-008). |
| `''` | `"gaseosa"` | Todas las variantes cuyo `productName + variantName` contiene el texto (case-insensitive), de cualquier categoría. |
| `'<id-categoría>'` | `''` | Todas las variantes de esa categoría — sin exigir texto (FR-007). |
| `'<id-categoría>'` | `"gaseosa"` | Intersección: variantes de esa categoría **y** que coinciden con el texto. |

Esta tabla reemplaza el comportamiento actual, donde `category: ''` y `text: ''` devuelven **todo
el catálogo** (el defecto que reporta esta spec).

## 3. Marcar y desmarcar (FR-009, FR-010)

- `toggleVariantForRule(ruleIndex, variantId)` no cambia de firma ni de guarda
  (`canEditRuleSet()`, ver `contracts/edicion-en-pausada.md`) — sigue siendo la única función que
  muta `rule.variantIds`, se llame desde una fila de `selectedVariantsForRule` o de
  `searchResultsForRule`.
- Marcar una variante en `searchResultsForRule` la agrega a `variantIds` → en el siguiente ciclo
  de detección de cambios aparece también en `selectedVariantsForRule` (FR-009).
- Desmarcar una variante — desde cualquiera de los dos listados — la quita de `variantIds` → deja
  de aparecer en `selectedVariantsForRule` de inmediato; puede seguir apareciendo en
  `searchResultsForRule` si sigue coincidiendo con el filtro activo, ahora con la casilla vacía
  (FR-010).

## 4. "Agregar visibles" (FR-011)

`selectAllFilteredForRule(ruleIndex)` (sin cambio de nombre) pasa a unionar `rule.variantIds` con
`searchResultsForRule(ruleIndex)` en vez de `visibleVariantsForRule(ruleIndex)` — mismo cuerpo,
misma semántica de unión (nunca quita nada fuera del filtro), ahora también capaz de agregar una
categoría completa de una sola vez cuando `searchResultsForRule` la lista entera (caso 3 de la
tabla del §2).

## 5. Fuera de alcance

- No hay paginación ni *debounce* de red: todo es un `Array.prototype.filter` en memoria sobre un
  catálogo ya cargado (research.md D2). Si el catálogo de un tenant creciera a un tamaño que
  hiciera perceptible el filtrado en cliente, eso es una spec de rendimiento aparte, no esta.
- El filtro de categoría reutiliza `categoryFilterOptions()`, sin cambios.
